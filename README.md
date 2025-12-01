This guide documents the steps to set up monitoring for EC2 instances using CloudWatch alarms that send notifications to Gmail and Lambda via SNS.

## Architecture Overview

```
EC2 Instance (NginxWebPage)
         ↓
CloudWatch Metrics
         ↓
CloudWatch Alarm (EC2StatusCheckAlarm)
         ↓
SNS Topic (WebServerUsageDemo)
         ↓
    ┌────┴────┐
    ↓         ↓
  Email    Lambda Function
           (NGINX_Web_Server_Function)
              ↓
          S3 Bucket
          (nginx-web-server-lambda-receiver)
```

---

## Step 1: Create SNS Topic

1. Navigate to **SNS Console**
2. Click **Topics** → **Create topic**
3. Configure topic:
   - **Type**: Standard
   - **Name**: `WebServerUsageDemo`
4. Click **Create topic**
5. Note the ARN: `arn:aws:sns:us-east-2:207214252187:WebServerUsageDemo`

---

## Step 2: Create Email Subscription

1. From the topic page, click **Create subscription**
2. Configure subscription:
   - **Protocol**: EMAIL
   - **Endpoint**: `aryan.vij@gmail.com`
3. Click **Create subscription**
4. Check email and **confirm subscription**
5. Status should change to **Confirmed**

---

## Step 3: Create CloudWatch Alarm

1. Navigate to **CloudWatch Console**
2. Click **Alarms** → **Create alarm**
3. Click **Select metric**
4. Choose **EC2** → **Per-Instance Metrics**
5. Select instance: `i-089853de77178796a` (NginxWebPage)
6. Select metric: **StatusCheckFailed_Instance**
7. Configure alarm:
   - **Statistic**: Average
   - **Period**: 1 minute
   - **Threshold**: Greater than 0.99
   - **Datapoints**: 1 out of 1
8. Click **Next**

---

## Step 4: Configure Alarm Actions

1. Under **Notification**:
   - **Alarm state trigger**: In alarm
   - **SNS topic**: `WebServerUsageDemo`
   - **Endpoint**: `aryan.vij@gmail.com`
2. Click **Next**

---

## Step 5: Name and Create Alarm

1. **Alarm name**: `EC2StatusCheckAlarm`
2. **Description**: Monitors EC2 instance status checks
3. Click **Create alarm**

---

## Step 6: Test Alarm via CLI
```bash
aws configure list
aws cloudwatch set-alarm-state --alarm-name "EC2StatusCheckAlarm" --state-value ALARM --state-reason "Manual test to verify notification actions for instance i-089853de77178796a"
```

**Result**: Email notification received successfully

---

## Step 7: Create Lambda Function for S3 Storage

1. Navigate to **Lambda Console**
2. Click **Create function**
3. Configure function:
   - **Author from scratch**: Selected
   - **Function name**: `NGINX_Web_Server_Function`
   - **Runtime**: Python 3.14
   - **Architecture**: x86_64
4. Click **Create function**

---

## Step 8: Add SNS Trigger to Lambda

1. Click **Add trigger**
2. Select **SNS**
3. **SNS topic**: `arn:aws:sns:us-east-2:207214252187:WebServerUsageDemo`
4. Click **Add**
5. Verify SNS subscription is **Confirmed**

---

## Step 9: Create S3 Bucket for Alarm Storage

1. Navigate to **S3 Console**
2. Click **Create bucket**
3. Configure bucket:
   - **Bucket name**: `nginx-web-server-lambda-receiver`
   - **Region**: us-east-2
   - **Block all public access**: Enabled
   - **Versioning**: Disabled
   - **Encryption**: SSE-S3
4. Click **Create bucket**

---

## Step 10: Add S3 Destination to Lambda

1. In Lambda function, click **Add destination**
2. Configure destination:
   - **Source**: Asynchronous invocation
   - **Condition**: On failure
   - **Destination type**: S3 bucket
   - **Destination**: `arn:aws:s3:::nginx-web-server-lambda-receiver`
3. Click **Save**

---

## Step 11: Configure Lambda IAM Permissions

1. Go to **IAM Console** → **Roles**
2. Find role: `NGINX_Web_Server_Function-role-32v9p6ot`
3. Click **Add permissions** → **Create inline policy**
4. Use JSON editor and paste:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::nginx-web-server-lambda-receiver",
        "arn:aws:s3:::nginx-web-server-lambda-receiver/*"
      ]
    }
  ]
}
```

5. Click **Review policy**
6. **Policy name**: `S3WritePolicy`
7. Click **Create policy**

---

## Step 12: Write Lambda Function Code

1. In Lambda console, go to **Code** tab
2. Replace code with:
```python
import json
import boto3
from datetime import datetime

s3 = boto3.client('s3')
BUCKET_NAME = 'nginx-web-server-lambda-receiver'

def lambda_handler(event, context):
    try:
        # Parse SNS message
        sns_message = json.loads(event['Records'][0]['Sns']['Message'])
        
        # Create filename with timestamp
        timestamp = datetime.now().strftime('%Y-%m-%d_%H-%M-%S')
        instance_id = sns_message.get('Trigger', {}).get('Dimensions', [{}])[0].get('value', 'unknown')
        filename = f'i-{instance_id}/alarm-{timestamp}.json'
        
        # Save to S3
        s3.put_object(
            Bucket=BUCKET_NAME,
            Key=filename,
            Body=json.dumps(sns_message, indent=2),
            ContentType='application/json'
        )
        
        return {
            'statusCode': 200,
            'body': json.dumps(f'Saved alarm to {filename}')
        }
    except Exception as e:
        print(f"Error: {str(e)}")
        raise
```

3. Click **Deploy**

---

## Step 13: Create Test Event

1. Click **Test** tab
2. Configure test event:
   - **Event name**: `S3LambdaTest`
   - **Template**: Hello World
   - **Event sharing**: Private
3. Click **Save**
4. Click **Test**

---

## Step 14: Verify Test Results

**Lambda Execution Result**:
```json
{
  "statusCode": 200,
  "body": "Saved alarm to i-089853de77178796a/alarm-2025-12-01T22-02-27.json"
}
```

**Summary**:
- Execution time: 11 seconds
- Duration: 347.31 ms
- Memory used: 92 MB

---

## Step 15: Verify S3 Storage

1. Navigate to S3 bucket: `nginx-web-server-lambda-receiver`
2. Check folder: `i-089853de77178796a/`
3. Verify file: `alarm-2025-12-01T22-02-27.json`
4. **Size**: 572 B
5. **Last modified**: December 1, 2025, 17:02:28 (UTC-05:00)

---

## Step 16: View CloudWatch Logs

1. Navigate to **CloudWatch** → **Logs** → **Log groups**
2. Open log group: `/aws/lambda/NGINX_Web_Server_Function`
3. View log stream: `2025/12/01/[$LATEST]054c40fe29a046fc92b7d2799cfe`
4. Verify execution logs show successful S3 upload

---

## Step 17: Check Additional SNS Messages

1. Return to S3 bucket
2. View additional files:
   - `sns-message-2025-12-01_22-15-52.txt` (6.0 B)
   - `sns-message-2025-12-01_22-16-58.txt` (22.0 B)
3. Verify timestamps match alarm trigger times

---

## Architecture Summary

```
EC2 Instance (NginxWebPage)
         ↓
CloudWatch Alarm (EC2StatusCheckAlarm)
         ↓
SNS Topic (WebServerUsageDemo)
         ↓
    ┌────┴────┐
    ↓         ↓
  Email    Lambda Function
(aryan.vij  (NGINX_Web_Server_Function)
@gmail.com)     ↓
            S3 Bucket
            (nginx-web-server-lambda-receiver)
```

---

## Testing the Complete Flow

1. **Trigger alarm manually**:
```bash
aws cloudwatch set-alarm-state \
  --alarm-name "EC2StatusCheckAlarm" \
  --state-value ALARM \
  --state-reason "Manual test"
```

2. **Expected outcomes**:
   - Email notification received
   - Lambda function executes
   - Alarm data saved to S3
   - CloudWatch logs updated

---

## Troubleshooting

- **No email received**: Check SNS subscription confirmation status
- **Lambda fails**: Verify IAM role permissions for S3
- **S3 upload fails**: Check bucket policy and Lambda execution role
- **Alarm not triggering**: Verify metric data and threshold values

---

## Best Practices

- Use descriptive names for all resources
- Test alarms regularly with manual triggers
- Monitor Lambda execution logs in CloudWatch
- Set up S3 lifecycle policies for old alarm data
- Document threshold values and their rationale
- Enable S3 versioning for critical alarm data

---

## Resources

- CloudWatch Alarm ARN: `arn:aws:cloudwatch:us-east-2:207214252187:alarm:EC2StatusCheckAlarm`
- SNS Topic ARN: `arn:aws:sns:us-east-2:207214252187:WebServerUsageDemo`
- Lambda Function ARN: `arn:aws:lambda:us-east-2:207214252187:function:NGINX_Web_Server_Function`
- S3 Bucket: `nginx-web-server-lambda-receiver`
- Instance ID: `i-089853de77178796a`
