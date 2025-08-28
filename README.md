# sftp

SFTP repository contains the setup for SFTP services

# Step-by-step instructions
https://docs.google.com/document/d/1hnKlY8Ujh3EbkUkqCcvqg6Zo4OOKimvM_xqXMHjcAPU/edit?pli=1&tab=t.0

#End user connectivity
https://docs.google.com/document/d/1QC5-Ru3T_lg22-WQWJHJ_BOqNqKMoTwsQ8sWz3S1Jws/edit?tab=t.odre8c77v02e

# key code

aws lambda:

import json
import boto3
from datetime import datetime
import os

s3 = boto3.client('s3')

def lambda_handler(event, context):
    """
    This Lambda function is triggered by an AWS Transfer Family workflow.
    It renames an uploaded file by appending a unique timestamp.
    """
    print(f"Received event: {json.dumps(event)}")

    # Extract the file details from the event passed by the Transfer Family workflow
    # The event structure can vary, so we check for common patterns
    if 'fileLocation' in event:
        bucket = event['fileLocation']['bucket']
        key = event['fileLocation']['key']
    elif 'Records' in event and 's3' in event['Records'][0]: # For S3 Event Trigger testing
        bucket = event['Records'][0]['s3']['bucket']['name']
        key = event['Records'][0]['s3']['object']['key']
    else:
        print("Error: Could not determine file location from the event.")
        return {
            'statusCode': 400,
            'body': json.dumps('Could not determine file location from event.')
        }

    print(f"Processing file: s3://{bucket}/{key}")

    try:
        # Split the key into file path and extension
        directory = os.path.dirname(key)
        filename_with_ext = os.path.basename(key)
        filename, extension = os.path.splitext(filename_with_ext)

        # Generate a timestamp string (e.g., 20230815T185030Z)
        timestamp = datetime.utcnow().strftime('%Y%m%dT%H%M%SZ')
        
        # Construct the new unique filename and key
        new_filename = f"{filename}_{timestamp}{extension}"
        
        # Handle cases where the file is in the root vs. a subdirectory
        if directory:
            new_key = f"{directory}/{new_filename}"
        else:
            new_key = new_filename

        print(f"New key will be: {new_key}")

        # Copy the object to the new key (this is how you "rename" in S3)
        copy_source = {'Bucket': bucket, 'Key': key}
        s3.copy_object(Bucket=bucket, CopySource=copy_source, Key=new_key)
        print(f"Successfully copied s3://{bucket}/{key} to s3://{bucket}/{new_key}")

        # Delete the original object
        s3.delete_object(Bucket=bucket, Key=key)
        print(f"Successfully deleted original file: s3://{bucket}/{key}")

        return {
            'statusCode': 200,
            'body': json.dumps(f'File successfully renamed to {new_filename}')
        }

    except Exception as e:
        print(f"Error processing file: {str(e)}")
        # To prevent Transfer Family from deleting the file on failure,
        # you can raise the exception.
        raise e

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
