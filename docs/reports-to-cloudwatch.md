# Collect Sentry Reports to CloudWatch

- [Collect Sentry Reports to CloudWatch](#collect-sentry-reports-to-cloudwatch)
  - [Create the Lambda function](#create-the-lambda-function)
  - [Configure the S3 Notification](#configure-the-s3-notification)
  - [Test it](#test-it)

Here, I'm describing a simple way to easily get new Sentry reports to CloudWatch using Lambda.

Since the Lambda and the S3 bucket containing Sentrys output need to be in the same region one does need to replicate the same setup for all your relevant regions.

## Create the Lambda function

Go to Lambda --> Create function.

Leave it to `Author from scratch`.

Parameter | Value
--------- | -----
Name | Whatever you want, e.g. `StackSet-SentryStackSet-f-ReportCreated-07HwDBE41qO0`
Runtime | `Python 3.9`
Architecture | `x86_64`
Execution role | `Create a new role with basic Lambda permissions`

Press `[Create Function]`.

Now you modify the roles policy. Basically you need to add `s3:GetObject` on the resource `YOUR S3 Bucket ARN`.

Your resulting Lambda Policy should look like this.

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "logs:CreateLogGroup"
            ],
            "Resource": [
                "arn:aws:logs:eu-central-1:634503960501:*",
                "arn:aws:s3:::stackset-sentrystackset-f8a28-stackresourcebucket-1hnkbr59grlcj/*"
            ]
        },
        {
            "Sid": "VisualEditor1",
            "Effect": "Allow",
            "Action": [
                "logs:CreateLogStream",
                "logs:PutLogEvents"
            ],
            "Resource": "arn:aws:logs:eu-central-1:634503960501:log-group:/aws/lambda/StackSet-SentryStackSet-f-ReportCreated-07HwDBE41qO0:*"
        }
    ]
}
```

Funtion:

```python
import json
import urllib.parse
import boto3


s3 = boto3.client('s3')

def lambda_handler(event, context):
    #print("Received event: " + json.dumps(event, indent=2))

    # Get the object from the event
    bucket = event['Records'][0]['s3']['bucket']['name']
    key = urllib.parse.unquote_plus(event['Records'][0]['s3']['object']['key'], encoding='utf-8')

    try:
        response = s3.get_object(Bucket=bucket, Key=key)
        body = response['Body'].read().decode('utf-8')
        print("CONTENT TYPE: " + response['ContentType'])
        print("REPORT: " + body)
        #print("REPORT: " + json.dumps(body, indent=2))
        return response['ContentType']
    except Exception as e:
        print(e)
        print('Error getting object {} from bucket {}. Make sure they exist and your bucket is in the same region as this function.'.format(key, bucket))
        raise e
```

Copy the Lambdas ARN.

## Configure the S3 Notification

Identify the S3 Bucket for the region you want to monitor

`stackset-sentrystackset-f8a28-stackresourcebucket-1hnkbr59grlcj`

Select the bucket and choose `Properties`.

Scroll down to `Event Notifications`.

Property | Value
-------- | -----
Name | Whatever you want, e.g. `sentrystackset-f8a28-stackresourcebucket-report-notification`
Prefix | `scan-report/`
Suffix | `final_report.json`
Event types | Check `Object creation - Put`
Destination | Lambda function
Enter Lambda function ARN | The `ARN` of your Lambda

## Test it

Now trigger some scans.

```sh
FUNCTION_SPF=$(aws lambda list-functions | jq -r '.Functions[] | select(.FunctionName | contains("-SnapshotProviderFunction-")) | .FunctionName')

aws lambda invoke --function-name $FUNCTION_SPF --cli-binary-format raw-in-base64-out --payload '{}' response.json
```

When they succeed head over to the CloudWatch logs of your new Lambda.

Here's an example how it could look like:

```log
INIT_START Runtime Version: python:3.9.v16	Runtime Version ARN: arn:aws:lambda:eu-central-1::runtime:07a48df201798d627f2b950f03bb227aab4a655a1d019c3296406f95937e2525
START RequestId: 9fd7b6fc-d073-4dd7-83ce-f5fae3576d96 Version: $LATEST
CONTENT TYPE: binary/octet-stream
REPORT: 
{
    "scanID": "634503960501-4ca1994a-33e6-4b67-b69e-c7098fff4950-1676540887",
    "resourceType": "aws-ebs-volume",
    "resourceID": "vol-03b25f8105caf9f00",
    "metadata": {
        "AWSAccountID": "634503960501",
        "SnapshotID": "snap-00420b0474dcda9d5",
        "VolumeID": "vol-03b25f8105caf9f00",
        "AttachedInstances": [
            {
                "InstanceID": "i-0076dab31026905f5"
            }
        ]
    },
    "timestamp": "2023-02-16T09:50:01Z",
    "antimalwareFull": [
        {
            "file": "/home/ubuntu/eicar.com",
            "malware": "Eicar_test_file",
            "type": "Virus",
            "pathToFile": [
                "/home/ubuntu/eicar.com"
            ],
            "fileSystemGUID": "CD72A83D-AFF1-4C36-B01F-A89A2F03D063"
        },
        {
            "file": "eicar.com",
            "malware": "Eicar_test_file",
            "type": "Virus",
            "pathToFile": [
                "/home/ubuntu/eicarcom2.zip"
            ],
            "fileSystemGUID": "CD72A83D-AFF1-4C36-B01F-A89A2F03D063"
        }
    ],
    "integrityMonitoring": {
        "findings": []
    }
}

END RequestId: 9fd7b6fc-d073-4dd7-83ce-f5fae3576d96
REPORT RequestId: 9fd7b6fc-d073-4dd7-83ce-f5fae3576d96	Duration: 325.56 ms	Billed Duration: 326 ms	Memory Size: 128 MB	Max Memory Used: 70 MB	Init Duration: 350.18 ms	
START RequestId: 75cf5847-364e-4e0d-a334-1b3a5caa6350 Version: $LATEST
CONTENT TYPE: binary/octet-stream
REPORT: 
{
    "scanID": "634503960501-22d4608e-0180-4e46-851b-27519a269c41-1676540887",
    "resourceType": "aws-ebs-volume",
    "resourceID": "vol-098189ce6d2894eff",
    "metadata": {
        "AWSAccountID": "634503960501",
        "SnapshotID": "snap-0dad8deb9e3a5fc5e",
        "VolumeID": "vol-098189ce6d2894eff",
        "AttachedInstances": [
            {
                "InstanceID": "i-08a345a13318f7db0"
            }
        ]
    },
    "timestamp": "2023-02-16T09:50:13Z",
    "antimalwareFull": [
        {
            "file": "/home/ec2-user/37ea273266aa2d28430194fca27849170d609d338abc9c6c43c4e6be1bcf51f9",
            "malware": "PUA.Win32.Qjwmonkey.GZ",
            "type": "Spyware",
            "pathToFile": [
                "/home/ec2-user/37ea273266aa2d28430194fca27849170d609d338abc9c6c43c4e6be1bcf51f9"
            ],
            "fileSystemGUID": "47834BF7-764E-42F9-9507-11A3E70B99DE"
        },
        {
            "file": "/home/ec2-user/4c1dc737915d76b7ce579abddaba74ead6fdb5b519a1ea45308b8c49b950655c.bin",
            "malware": "Ransom_PETYA.E",
            "type": "Trojan",
            "pathToFile": [
                "/home/ec2-user/4c1dc737915d76b7ce579abddaba74ead6fdb5b519a1ea45308b8c49b950655c.bin"
            ],
            "fileSystemGUID": "47834BF7-764E-42F9-9507-11A3E70B99DE"
        },
        {
            "file": "/home/ec2-user/Petya.bin",
            "malware": "Ransom_PETYA.A",
            "type": "Trojan",
            "pathToFile": [
                "/home/ec2-user/Petya.bin"
            ],
            "fileSystemGUID": "47834BF7-764E-42F9-9507-11A3E70B99DE"
        },
        {
            "file": "/home/ec2-user/Petya.pdf",
            "malware": "Ransom_PETYA.A",
            "type": "Trojan",
            "pathToFile": [
                "/home/ec2-user/Petya.pdf"
            ],
            "fileSystemGUID": "47834BF7-764E-42F9-9507-11A3E70B99DE"
        },
        {
            "file": "/home/ec2-user/Petya.png",
            "malware": "Ransom_PETYA.A",
            "type": "Trojan",
            "pathToFile": [
                "/home/ec2-user/Petya.png"
            ],
            "fileSystemGUID": "47834BF7-764E-42F9-9507-11A3E70B99DE"
        },
        {
            "file": "/home/ec2-user/c0242d686b4c1707f9db2eb5afdd306507ceb5637d72662dff56c439330dbdf1",
            "malware": "TSPY_HPLOKI.SMBD",
            "type": "Trojan",
            "pathToFile": [
                "/home/ec2-user/c0242d686b4c1707f9db2eb5afdd306507ceb5637d72662dff56c439330dbdf1"
            ],
            "fileSystemGUID": "47834BF7-764E-42F9-9507-11A3E70B99DE"
        },
        {
            "file": "/home/ec2-user/eicar.txt",
            "malware": "Eicar_test_file",
            "type": "Virus",
            "pathToFile": [
                "/home/ec2-user/eicar.txt"
            ],
            "fileSystemGUID": "47834BF7-764E-42F9-9507-11A3E70B99DE"
        },
        {
            "file": "eicar.com",
            "malware": "Eicar_test_file",
            "type": "Virus",
            "pathToFile": [
                "/home/ec2-user/eicarcom2.zip"
            ],
            "fileSystemGUID": "47834BF7-764E-42F9-9507-11A3E70B99DE"
        }
    ],
    "integrityMonitoring": {
        "findings": [
            {
                "path": "/usr/sbin/",
                "change": "permissionChanged",
                "type": "directory",
                "details": {
                    "newPermissions": "690",
                    "oldPermissions": "509"
                }
            },
            {
                "path": "/usr/bin/",
                "change": "permissionChanged",
                "type": "directory",
                "details": {
                    "newPermissions": "690",
                    "oldPermissions": "509"
                }
            },
            {
                "path": "/boot/efi/EFI/amzn/",
                "change": "permissionChanged",
                "type": "directory",
                "details": {
                    "newPermissions": "690",
                    "oldPermissions": "468"
                }
            },
            {
                "path": "/boot/efi/EFI/BOOT/",
                "change": "permissionChanged",
                "type": "directory",
                "details": {
                    "newPermissions": "690",
                    "oldPermissions": "468"
                }
            },
            {
                "path": "/boot/efi/EFI/",
                "change": "permissionChanged",
                "type": "directory",
                "details": {
                    "newPermissions": "690",
                    "oldPermissions": "468"
                }
            },
            {
                "path": "/boot/grub2/locale/",
                "change": "permissionChanged",
                "type": "directory",
                "details": {
                    "newPermissions": "690",
                    "oldPermissions": "468"
                }
            },
            {
                "path": "/boot/grub2/i386-pc/",
                "change": "permissionChanged",
                "type": "directory",
                "details": {
                    "newPermissions": "690",
                    "oldPermissions": "468"
                }
            },
            {
                "path": "/boot/grub2/fonts/",
                "change": "permissionChanged",
                "type": "directory",
                "details": {
                    "newPermissions": "690",
                    "oldPermissions": "468"
                }
            }
        ]
    }
}

END RequestId: 75cf5847-364e-4e0d-a334-1b3a5caa6350
REPORT RequestId: 75cf5847-364e-4e0d-a334-1b3a5caa6350	Duration: 195.49 ms	Billed Duration: 196 ms	Memory Size: 128 MB	Max Memory Used: 71 MB	
```
