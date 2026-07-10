# Static Site on S3 + CloudFront with GitHub Actions

A static website hosted on a private S3 bucket, served through CloudFront over HTTPS, and deployed automatically with GitHub Actions every time I push to main. No AWS access keys are stored anywhere - GitHub authenticates to AWS using short-lived OIDC tokens.

Live site: https://d2x7th1mrhcitx.cloudfront.net

## What each piece does

The S3 bucket holds the site files. It blocks all public access, so nothing is served straight from S3.

CloudFront sits in front of the bucket. It serves the site over HTTPS, caches it at edge locations around the world, and is the only thing allowed to read from the bucket (through Origin Access Control).

GitHub Actions handles deployment. On every push to main, it authenticates to AWS with a temporary OIDC token, syncs the site folder to S3, and tells CloudFront to clear its cache so the new version goes live.

## Why OIDC instead of access keys

The usual shortcut is to create AWS access keys and paste them into GitHub secrets. Those keys don't expire, so if they ever leak, someone has access until they're found and rotated.

This setup uses OIDC instead. GitHub gets a fresh token for each run that expires in about an hour, nothing sensitive is stored in the repo, and the IAM role is locked to this one repository and can only touch this one bucket and distribution.

## Repo structure

    s3-cloudfront-static-site/
    ├── .github/
    │   └── workflows/
    │       └── deploy.yml     # the automation
    ├── site/
    │   └── index.html         # the actual website
    └── README.md

## The workflow file, line by line

This lives at `.github/workflows/deploy.yml`. Here is what each part does:

    name: Deploy site to S3 + CloudFront

A label for the workflow, shown in the Actions tab.

    on:
      push:
        branches: [main]

The trigger. This workflow runs whenever code is pushed to the main branch.

    permissions:
      id-token: write
      contents: read

`id-token: write` is what lets the workflow request an OIDC token from GitHub - without it the whole keyless login fails. `contents: read` lets it read the repo code.

    jobs:
      deploy:
        runs-on: ubuntu-latest

Defines one job called deploy, running on a GitHub-provided Linux machine.

    - name: Checkout repo
      uses: actions/checkout@v4

Downloads the repo's files onto that machine so the later steps can use them.

    - name: Configure AWS credentials via OIDC
      uses: aws-actions/configure-aws-credentials@v4
      with:
        role-to-assume: arn:aws:iam::596055752057:role/github-actions-deploy-site
        aws-region: us-east-1

The login step. It presents the OIDC token to AWS, assumes the IAM role, and receives temporary credentials. This is where the keyless auth happens.

    - name: Sync site folder to S3
      run: aws s3 sync ./site s3://devops-journey-site --delete

Uploads everything in the site folder to the bucket. `--delete` removes files from the bucket that no longer exist in the repo, keeping them identical.

    - name: Invalidate CloudFront cache
      run: |
        aws cloudfront create-invalidation \
          --distribution-id E485VDAP1XU8G \
          --paths "/*"

Tells CloudFront to drop its cached copies and fetch fresh ones from S3, so changes show up right away instead of waiting for the cache to expire.

## How to set it up yourself

If I were rebuilding this from scratch, the order is:

1. Create a private S3 bucket with all public access blocked, and upload the site files.

2. Create a CloudFront distribution with the bucket as the origin, using Origin Access Control so the bucket can stay private. Set the Default root object to `index.html` so the bare URL loads the site.

3. In IAM, add an OpenID Connect identity provider pointing at `https://token.actions.githubusercontent.com` with audience `sts.amazonaws.com`. This is how AWS learns to trust GitHub's tokens.

4. Create an IAM role that GitHub can assume, with a trust policy scoped to this repo, and an inline permissions policy that only allows syncing to this bucket and invalidating this distribution.

5. Add the workflow file above to `.github/workflows/deploy.yml`, filling in the account ID, role name, bucket name, and distribution ID.

6. Push to main and watch the Actions tab.

## To deploy a change

    git add .
    git commit -m "Update site"
    git push

The site updates on its own within a minute. No AWS console needed.

## Notes to self

CloudFront does not treat `/` as `index.html` automatically the way a normal web server does. The Default root object setting has to be filled in, or the bare URL returns AccessDenied.

Never leave placeholder values in commands or policies. An unreplaced distribution ID in the IAM policy caused the deploy to fail on the very last step - the file synced fine, only the cache invalidation was denied. Reading the IAM error message is what pointed to the fix.

## Stack

S3, CloudFront (with OAC), IAM (OIDC provider, scoped role, least-privilege policy), GitHub Actions
