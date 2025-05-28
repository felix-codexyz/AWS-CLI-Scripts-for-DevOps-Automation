# 🛠️ 15 Essential AWS CLI Scripts for Everyday DevOps Automation



## 1. Check EC2 Instance Status
### Checks if a specific EC2 instance is in a desired state (e.g., running) before proceeding with further actions.
```
#!/bin/bash

INSTANCE_ID="i-0abcdef1234567890"
EXPECTED_STATE="running"

CURRENT_STATE=$(aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --query "Reservations[0].Instances[0].State.Name" \
  --output text)

if [ "$CURRENT_STATE" == "$EXPECTED_STATE" ]; then
  echo "Instance $INSTANCE_ID is $CURRENT_STATE. Proceeding with action..."
  # Add your next command here
else
  echo "Instance $INSTANCE_ID is not $EXPECTED_STATE (Current: $CURRENT_STATE). Aborting."
  exit 1
fi

```


## 2. Stop EC2 Instances by Tag
### Finds and stops all running EC2 instances with a given tag key and value.

```
#!/bin/bash

TAG_KEY="Environment"
TAG_VALUE="Development"

INSTANCE_IDS=$(aws ec2 describe-instances \
  --filters "Name=tag:${TAG_KEY},Values=${TAG_VALUE}" "Name=instance-state-name,Values=running" \
  --query "Reservations[*].Instances[*].InstanceId" \
  --output text)

if [ -z "$INSTANCE_IDS" ]; then
  echo "No running instances found with tag $TAG_KEY=$TAG_VALUE."
  exit 0
fi

echo "Stopping instances: $INSTANCE_IDS"
aws ec2 stop-instances --instance-ids $INSTANCE_IDS
```


## 3. Upload Build to S3 with Timestamp
### Uploads a build artifact (like a zip file) to an S3 bucket, appending a timestamp to avoid overwriting.

```
#!/bin/bash

BUCKET_NAME="your-build-artifacts-bucket"
ARTIFACT_PATH="./app/build/app.zip"
ARTIFACT_NAME=$(basename "$ARTIFACT_PATH")
TIMESTAMP=$(date +"%Y%m%d-%H%M%S")
S3_KEY="builds/${ARTIFACT_NAME}-${TIMESTAMP}"

if [ ! -f "$ARTIFACT_PATH" ]; then
  echo "Artifact $ARTIFACT_PATH not found!"
  exit 1
fi

aws s3 cp "$ARTIFACT_PATH" "s3://${BUCKET_NAME}/${S3_KEY}"

if [ $? -eq 0 ]; then
  echo "Upload successful."
else
  echo "Upload failed."
  exit 1
fi
```

## 4. Check CloudFormation Stack Status
### Verifies the status of a CloudFormation stack (e.g., complete, failed, in-progress) and exits based on its health.

```
#!/bin/bash

STACK_NAME="my-application-stack"

STATUS=$(aws cloudformation describe-stacks \
  --stack-name "$STACK_NAME" \
  --query "Stacks[0].StackStatus" \
  --output text)

echo "Stack Status: $STATUS"

case "$STATUS" in
  CREATE_COMPLETE|UPDATE_COMPLETE)
    echo "Stack is in a good state."
    ;;
  *ROLLBACK*|*FAILED*)
    echo "Stack failed or rolled back."
    exit 1
    ;;
  *PROGRESS*)
    echo "Stack operation in progress."
    exit 2
    ;;
  *)
    echo "Unknown or deleted state."
    exit 1
    ;;
esac
```


## 5. Fetch Latest CloudWatch Logs (Lambda)
### Retrieves recent logs from the most recent log stream of a specific Lambda function.

```
#!/bin/bash

FUNCTION_NAME="my-lambda-function"
LOG_GROUP_NAME="/aws/lambda/${FUNCTION_NAME}"
LIMIT=20

LATEST_STREAM=$(aws logs describe-log-streams \
  --log-group-name "$LOG_GROUP_NAME" \
  --order-by LastEventTime \
  --descending \
  --limit 1 \
  --query "logStreams[0].logStreamName" \
  --output text)

if [ "$LATEST_STREAM" == "None" ] || [ -z "$LATEST_STREAM" ]; then
  echo "No log streams found."
  exit 1
fi

aws logs get-log-events \
  --log-group-name "$LOG_GROUP_NAME" \
  --log-stream-name "$LATEST_STREAM" \
  --limit "$LIMIT" \
  --query "events[*].message" \
  --output text
```


## 6. Invalidate CloudFront Cache
### Triggers a CloudFront invalidation to refresh cached content like HTML, CSS, or JS files.
```
#!/bin/bash

DISTRIBUTION_ID="E12345ABCDEFGH"
INVALIDATION_PATHS="/index.html /css/* /js/*"
CALLER_REFERENCE=$(date +"%Y%m%d%H%M%S")

aws cloudfront create-invalidation \
  --distribution-id "$DISTRIBUTION_ID" \
  --paths $INVALIDATION_PATHS \
  --caller-reference "$CALLER_REFERENCE"

if [ $? -eq 0 ]; then
  echo "Invalidation submitted."
else
  echo "Invalidation failed."
  exit 1
fi
```


## 7. Check Target Group Health
### Checks the health status of registered targets in an ALB Target Group and flags any unhealthy instances.
```
#!/bin/bash

command -v jq >/dev/null 2>&1 || { echo >&2 "jq is required but not installed."; exit 1; }

TARGET_GROUP_ARN="arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/my-targets/abcdef1234567890"

UNHEALTHY_TARGETS=$(aws elbv2 describe-target-health \
  --target-group-arn "$TARGET_GROUP_ARN" \
  --query "TargetHealthDescriptions[?TargetHealth.State!='healthy'].Target.Id" \
  --output json | jq -r '.[]')

if [ -z "$UNHEALTHY_TARGETS" ]; then
  echo "All targets are healthy."
else
  echo "Unhealthy targets found:"
  echo "$UNHEALTHY_TARGETS"
  exit 1
fi
```


## 8. Get EC2 Public IP
### Retrieves the public IP address of a specified EC2 instance.
```
#!/bin/bash

INSTANCE_ID="i-0abcdef1234567890"

PUBLIC_IP=$(aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --query "Reservations[0].Instances[0].PublicIpAddress" \
  --output text)

if [ "$PUBLIC_IP" == "None" ] || [ -z "$PUBLIC_IP" ]; then
  echo "No public IP found."
  exit 1
else
  echo "Public IP: $PUBLIC_IP"
fi
```

## 9. Find Large Files in S3 Bucket
### Lists objects in an S3 bucket larger than a defined size (e.g., 100 MB), sorted by size.

```
#!/bin/bash

BUCKET_NAME="your-large-data-bucket"
MIN_SIZE_MB=100
MIN_SIZE_BYTES=$((MIN_SIZE_MB * 1024 * 1024))

aws s3api list-objects-v2 \
  --bucket "$BUCKET_NAME" \
  --query "Contents[?Size > \`$MIN_SIZE_BYTES\`].[Key, Size]" \
  --output text | \
  awk '{size=$NF; $NF=""; key=$0; printf "%-100s %.2f MB\n", key, size/1024/1024}' | \
  sort -k2 -n -r
```



## 10. Wait for EC2 to Start
### Waits for an EC2 instance to reach the ‘running’ state before continuing with further commands.

```
#!/bin/bash

INSTANCE_ID="i-0abcdef1234567890"

echo "Waiting for instance $INSTANCE_ID to be 'running'..."

aws ec2 wait instance-running --instance-ids "$INSTANCE_ID"

if [ $? -eq 0 ]; then
  echo "Instance is running."
else
  echo "Instance failed to reach 'running' state."
  exit 1
fi
```

## 11. Describe Security Groups for EC2
### Lists security groups attached to an EC2 instance and displays detailed ingress/egress rules (uses jq).

```
#!/bin/bash

command -v jq >/dev/null 2>&1 || { echo >&2 "jq is required but not installed."; exit 1; }

INSTANCE_ID="i-0abcdef1234567890"

GROUP_IDS=$(aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --query "Reservations[0].Instances[0].SecurityGroups[*].GroupId" \
  --output text)

if [ -z "$GROUP_IDS" ]; then
  echo "No security groups found."
  exit 1
fi

aws ec2 describe-security-groups \
  --group-ids $GROUP_IDS \
  --query "SecurityGroups[*].{ID:GroupId, Ingress:IpPermissions, Egress:IpPermissionsEgress}" \
  --output json | jq .
```

## 12. Find Old EBS Snapshots
### Lists EBS snapshots owned by you that are older than a given number of days (e.g., 90).

```
#!/bin/bash

DAYS_OLD=90
OWNER_ID=$(aws sts get-caller-identity --query Account --output text)

if date --version >/dev/null 2>&1 ; then
  CUTOFF_DATE=$(date -d "-$DAYS_OLD days" +"%Y-%m-%d")
else
  CUTOFF_DATE=$(date -v-${DAYS_OLD}d +"%Y-%m-%d")
fi

aws ec2 describe-snapshots --owner-ids "$OWNER_ID" \
  --query "Snapshots[?StartTime<'$CUTOFF_DATE'].{ID:SnapshotId, Size:VolumeSize, Time:StartTime}" \
  --output json | jq .
```

## 13. Invoke Lambda Function
### Calls a Lambda function with input payload and displays the result and status code.

```
#!/bin/bash

FUNCTION_NAME="my-utility-lambda"
PAYLOAD='{"action": "processData", "dataId": "xyz789"}'
OUTPUT_FILE="/tmp/lambda_output.json"

aws lambda invoke \
  --function-name "$FUNCTION_NAME" \
  --invocation-type RequestResponse \
  --payload "$PAYLOAD" \
  --cli-binary-format raw-in-base64-out \
  "$OUTPUT_FILE" > /dev/null

if [ $? -ne 0 ]; then
  echo "Lambda invocation failed."
  cat "$OUTPUT_FILE"
  exit 1
fi

cat "$OUTPUT_FILE"
STATUS_CODE=$(jq -r '.statusCode // "N/A"' "$OUTPUT_FILE")
echo "Status Code: $STATUS_CODE"
```

## 14. Fetch Secure Parameter from Parameter Store
### Retrieves a decrypted SSM Parameter Store value (e.g., database password) securely.

```
#!/bin/bash

PARAMETER_NAME="/config/app/db_password"

VALUE=$(aws ssm get-parameter \
  --name "$PARAMETER_NAME" \
  --with-decryption \
  --query "Parameter.Value" \
  --output text)

if [ $? -ne 0 ] || [ -z "$VALUE" ]; then
  echo "Failed to retrieve parameter."
  exit 1
fi

echo "Parameter retrieved successfully."
# export DB_PASSWORD=$VALUE
```

## 15. Sync Local Folder to S3 (Deploy or Backup)
### Syncs a local directory to an S3 bucket for website hosting or backup, excluding unnecessary files.

```
#!/bin/bash

SOURCE_DIR="./website/public/"
DEST_S3_URI="s3://your-website-bucket/live/"
DELETE_FLAG="--delete"
ACL_FLAG="--acl public-read"

if [ ! -d "$SOURCE_DIR" ]; then
  echo "Source directory not found."
  exit 1
fi

aws s3 sync "$SOURCE_DIR" "$DEST_S3_URI" $DELETE_FLAG $ACL_FLAG \
  --exclude ".git/*" --exclude "node_modules/*"

if [ $? -eq 0 ]; then
  echo "Sync complete."
else
  echo "Sync failed."
  exit 1
fi
```
