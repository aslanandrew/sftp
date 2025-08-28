# sftp

SFTP repository contains the setup for SFTP services

# Step-by-step instructions
https://docs.google.com/document/d/1hnKlY8Ujh3EbkUkqCcvqg6Zo4OOKimvM_xqXMHjcAPU/edit?pli=1&tab=t.0

#End user connectivity
https://docs.google.com/document/d/1QC5-Ru3T_lg22-WQWJHJ_BOqNqKMoTwsQ8sWz3S1Jws/edit?tab=t.odre8c77v02e

# key code

```
import json
import boto3
from datetime import datetime
import os
import logging
import urllib.request
import urllib.error

log = logging.getLogger()
log.setLevel(logging.INFO)

s3 = boto3.client('s3')
_secrets = None



def _get_slack_webhook_url():
    secret_id = os.getenv("SLACK_WEBHOOK_SECRET_ID")
    if secret_id:
        global _secrets
        if _secrets is None:
            _secrets = boto3.client("secretsmanager")
        resp = _secrets.get_secret_value(SecretId=secret_id)
        secret_str = resp.get("SecretString")
        if not secret_str:
            raise RuntimeError("Secret has no SecretString")
        # allow either raw string or JSON {"url": "..."}
        try:
            data = json.loads(secret_str)
            url = data.get("url") or data.get("webhook_url") or data.get("SLACK_WEBHOOK_URL")
        except json.JSONDecodeError:
            url = secret_str
        if not url:
            raise RuntimeError("Slack webhook URL not found in secret payload")
        return url

    url = os.getenv("SLACK_WEBHOOK_URL")
    if not url:
        raise RuntimeError("Missing SLACK_WEBHOOK_URL or SLACK_WEBHOOK_SECRET_ID")
    return url


def _post_to_slack(webhook_url, payload):
    data = json.dumps(payload).encode("utf-8")
    req = urllib.request.Request(
        webhook_url,
        data=data,
        headers={"Content-Type": "application/json"},
        method="POST",
    )
    with urllib.request.urlopen(req, timeout=5) as resp:
        body = resp.read().decode("utf-8", errors="ignore")
        log.info("Slack response code=%s body=%s", resp.status, body)
        if resp.status >= 300:
            raise RuntimeError(f"Slack returned HTTP {resp.status}: {body}")


def _human_size(n):
    try:
        n = int(n)
    except Exception:
        return str(n)
    for unit in ["B","KB","MB","GB","TB","PB"]:
        if n < 1024:
            return f"{n:.0f} {unit}"
        n /= 1024
    return f"{n:.0f} EB"


def _build_slack_payload(event, bucket, old_key, new_key, size_bytes, status="SUCCESS"):
    username = event.get("username") or event.get("userName") or event.get("user-name")
    size_str = _human_size(size_bytes) if size_bytes is not None else "unknown"

    blocks = [
        {
            "type": "header",
            "text": {"type": "plain_text", "text": "Aslan data file received", "emoji": True}
        },
        {
            "type": "section",
            "text": {"type": "mrkdwn", "text": f"*Status:* `{status}`\n*Bucket:* `{bucket}`"}
        },
        {
            "type": "section",
            "fields": [
                {"type": "mrkdwn", "text": f"*Original:*\n`{old_key}`"},
                {"type": "mrkdwn", "text": f"*Renamed:*\n`{new_key}`"},
                {"type": "mrkdwn", "text": f"*Size:*\n{size_str}"},
                {"type": "mrkdwn", "text": f"*User:*\n`{username or 'unknown'}`"}
            ]
        },
        {"type": "divider"}
    ]

    return {
        "text": f"Aslan data file received: {new_key} ({size_str})",  # fallback text
        "blocks": blocks,
    }



def lambda_handler(event, context):
    """
    Triggered by AWS Transfer Family WORKFLOW (preferred) or S3 event (for testing).
    Renames the uploaded file by appending a UTC timestamp, then posts to Slack.
    """
    log.info("Received event: %s", json.dumps(event))

   
    if 'fileLocation' in event:  # Transfer Family workflow payload
        bucket = event['fileLocation']['bucket']
        key = event['fileLocation']['key']
        # Try to capture file size if provided by workflow
        size_bytes = event.get("fileSize") or event.get("filesize") or None
    elif 'Records' in event and 's3' in event['Records'][0]:  # S3 event testing
        bucket = event['Records'][0]['s3']['bucket']['name']
        key = event['Records'][0]['s3']['object']['key']
        size_bytes = event['Records'][0]['s3']['object'].get('size')
    else:
        log.error("Could not determine file location from event.")
        return {'statusCode': 400,
                'body': json.dumps('Could not determine file location from event.')}

    log.info("Processing file: s3://%s/%s", bucket, key)

    
    directory = os.path.dirname(key)
    filename_with_ext = os.path.basename(key)
    filename, extension = os.path.splitext(filename_with_ext)
    timestamp = datetime.utcnow().strftime('%Y%m%dT%H%M%SZ')
    new_filename = f"{filename}_{timestamp}{extension}"
    new_key = f"{directory}/{new_filename}" if directory else new_filename

    
    try:
        copy_source = {'Bucket': bucket, 'Key': key}
        s3.copy_object(Bucket=bucket, CopySource=copy_source, Key=new_key)
        log.info("Copied s3://%s/%s to s3://%s/%s", bucket, key, bucket, new_key)

        s3.delete_object(Bucket=bucket, Key=key)
        log.info("Deleted original s3://%s/%s", bucket, key)

        # get size if we still don't have it
        if size_bytes is None:
            try:
                head = s3.head_object(Bucket=bucket, Key=new_key)
                size_bytes = head.get("ContentLength")
            except Exception as e:
                log.warning("Could not get size for s3://%s/%s: %s", bucket, new_key, e)

        status_msg = f'File successfully renamed to {new_filename}'

    except Exception as e:
        log.error("Error during rename: %s", e)
        # Attempt Slack error message (optional)
        _maybe_post_slack(event, bucket, key, new_key, size_bytes, status="ERROR")
        # Re-raise to signal failure to Transfer workflow (prevents deletion/cleanup).
        raise

    # --- Post to Slack (best-effort by default) ---
    _maybe_post_slack(event, bucket, key, new_key, size_bytes, status="SUCCESS")

    return {'statusCode': 200, 'body': json.dumps(status_msg)}


def _maybe_post_slack(event, bucket, old_key, new_key, size_bytes, status):
    """
    Posts to Slack. Controlled by env vars:
      - DISABLE_SLACK=true  -> don’t post
      - FAIL_ON_SLACK_ERROR=true -> raise if Slack fails (default: False)
    """
    if os.getenv("DISABLE_SLACK", "").lower() == "true":
        log.info("Slack disabled by DISABLE_SLACK env var.")
        return

    try:
        webhook = _get_slack_webhook_url()
        payload = _build_slack_payload(event, bucket, old_key, new_key, size_bytes, status=status)
        _post_to_slack(webhook, payload)
        log.info("Posted Slack notification.")
    except Exception as e:
        log.error("Slack post failed: %s", e)
        if os.getenv("FAIL_ON_SLACK_ERROR", "").lower() == "true":
            # Bubble up to fail the whole invocation if desired
            raise

```

# roles

## env-aslan-sftp-transfer-logging-access e.g. prod-aslan-sftp-transfer-logging access

Purpose: allows cloudwatch logging for the Transfer server
Trust relationship: Transfer
Permissions: AWS Transfer Logging Access

## prod-sftp-server-role 

Purpose: allows the execution of a lamdda function and logging of the lambda function to cloudwatch on behalf of AWS Transfer
Trust relationship: Transfer
Permissions: Custom Policy 'log and launch'

{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowTransferLogging",
            "Effect": "Allow",
            "Action": [
                "logs:CreateLogStream",
                "logs:DescribeLogStreams",
                "logs:CreateLogGroup",
                "logs:PutLogEvents"
            ],
            "Resource": "*"
        },
        {
            "Sid": "AllowLambdaInvoke",
            "Effect": "Allow",
            "Action": "lambda:InvokeFunction",
            "Resource": "arn:aws:lambda:eu-west-2:119663740913:function:sftp-transfer-rename"
        },
        {
            "Sid": "AllowTagging",
            "Effect": "Allow",
            "Action": [
                "s3:PutObjectTagging",
                "s3:PutObjectVersionTagging"
            ],
            "Resource": "arn:aws:s3:::prod-aslan-sftp/*"
        }
    ]
}

## prod-sftp-transfer-role

Purpose: allows the execution of file movements by a user on the sftp server
Trust Relationship: Transfer
Permissions: Custom Policy 'env-sftp-read-write'

{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowListingOfUserFolder",
            "Action": [
                "s3:ListBucket",
                "s3:GetBucketLocation"
            ],
            "Effect": "Allow",
            "Resource": [
                "arn:aws:s3:::prod-aslan-sftp"
            ]
        },
        {
            "Sid": "HomeDirObjectAccess",
            "Effect": "Allow",
            "Action": [
                "s3:PutObject",
                "s3:GetObject",
                "s3:GetObjectTagging",
                "s3:DeleteObject",
                "s3:DeleteObjectVersion",
                "s3:GetObjectVersion",
                "s3:GetObjectVersionTagging",
                "s3:GetObjectACL",
                "s3:PutObjectACL"
            ],
            "Resource": "arn:aws:s3:::prod-aslan-sftp/*"
        }
    ]
}

## prod-sftp-transfer-rename-role

Purpose: allows the lambda execution and s3 access to handle the file rename
Trust Relationship: Lambda
Permissions: AmazonS3FullAccess, AWSLambdaBasicExecution
