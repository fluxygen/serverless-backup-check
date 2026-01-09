# Serverless backup summary

This repo contains a Serverless application that exposes a secure HTTP API to analyze backups stored in S3.
It uses the Serverless Framework + AWS Lambdato perform on-demand checks.

## Primary Use Case

This API is designed to be queried by **Grafana** (or other monitoring tools) to visualize backup health. It accepts a JSON payload pointing to a specific S3 bucket and folder, checks if the backup exists, and validates if it has grown within expected tolerances.

## Backup Structure
This tool assumes backups are organized by date in an S3 bucket:

- S3 bucket
    - path/to/backup/folder
        - 2018.04.26
            - file1
            - file2
            - ...
            - fileN
        - 2018.04.25
        - ...

The script then performs the check to see if the backups from today (if available) are same as ones from last time,
and if they are within tolerance (X% difference per day).

Backup size change limitations are based on the calculations in
https://drive.google.com/open?id=1tiQXgoRs9gfTeVeEpIu1l0TDDkTfh4dDDm1Eud0zhxQ

# Setup & Requirements

- **Python 3.13+**
- **Node.js & NPM**
- **Serverless Framework V4** (Local dependency)

To install dependencies and setup the environment:

```bash
# Create and activate virtual environment
python3.13 -m venv .venv313
source .venv313/bin/activate

# Install Python requirements
pip install -r requirements.txt

# Install Node requirements (Serverless V4 + Plugins)
npm install

```

# Deploying

Make sure you have access to the AWS account: `dovetail-backups`.

**Important:** You must use `npx` to ensure the local Serverless V4 version is used. Global installations (v3) will fail with Python 3.13.

```bash
# Authenticate once (required for V4)
npx serverless login

# Deploy to Production
aws-vault exec dovetail-backups -- npx serverless deploy --stage prod

```

# Adding more buckets to check

In order to do backup checks we need access to the specific backup bucket.
In case the bucket is in a different account you need to grant this access both in the account that owns the bucket as well as the `dovetail-backups` account (for the lambda function) itself.
The access for the execution role of the lambda function on the `dovetail-backups` account side is managed by serverless.yaml. So there you will need to append the last two lines to the policy section of serverless.yaml:

```yaml
      Resource:
        - "... other buckets ..."
        - "arn:aws:s3:::<BUCKET_NAME>"
        - "arn:aws:s3:::<BUCKET_NAME>/*"

```

This policy needs to be added to the account that hosts the s3 bucket.

**Note:** Update the role ARN to match the `prod` stage if deploying to production.

```json
{
    "Sid": "AllowBackupCheck",
    "Effect": "Allow",
    "Principal": {
        "AWS": "arn:aws:iam::190384451510:role/serverless-backup-analysis-prod-eu-west-1-lambdaRole"
    },
    "Action": "s3:ListBucket",
    "Resource": [
        "arn:aws:s3:::<BUCKET_NAME>",
        "arn:aws:s3:::<BUCKET_NAME>/*"
    ]
}

```

# Trying the check locally

**Requirements:**

* `pip install fire` (or install via `requirements.txt`)
* `aws-vault exec dovetail-backups` (But the script will notify you if you forgot)

The file `run_local.py` is a [Fire](https://github.com/google/python-fire) script for running this check locally
Fire helps with nice cli apps. For example if you run `./run_local.py -h` you get this output:

```bash
NAME
    run_local.py

SYNOPSIS
    run_local.py BACKUP_FOLDER <flags>

POSITIONAL ARGUMENTS
    BACKUP_FOLDER

FLAGS
    --bucket_name=BUCKET_NAME
    --file_date_format=FILE_DATE_FORMAT

NOTES
    You can also use flags syntax for POSITIONAL ARGUMENTS
```