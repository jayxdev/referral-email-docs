# Deployment Guide

## Prerequisites

### Required Accounts & Services
1. **AWS Account**: Configured with appropriate IAM permissions
2. **Microsoft Graph API**: Registered application with proper permissions
3. **MediRecords API**: Valid API credentials and endpoint
4. **Python Environment**: 3.9+ with required dependencies

### Required IAM Permissions
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "secretsmanager:GetSecretValue",
                "s3:GetObject",
                "s3:PutObject",
                "s3:ListBucket",
                "textract:AnalyzeDocument",
                "bedrock:InvokeModel",
                "logs:CreateLogGroup",
                "logs:CreateLogStream",
                "logs:PutLogEvents"
            ],
            "Resource": "*"
        }
    ]
}
```

---

## Setup Steps

### 1. **Install Dependencies**
```bash
# Clone the repository
git clone <repository-url>
cd referralemailwork

# Create virtual environment
python -m venv .venv
source .venv/bin/activate  # On Windows: .venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

### 2. **Configure AWS Credentials**
```bash
# Install AWS CLI
pip install awscli

# Configure AWS credentials
aws configure
# Enter your AWS Access Key ID
# Enter your AWS Secret Access Key
# Enter your default region (ap-southeast-2)
# Enter your output format (json)
```

### 3. **Set Up AWS Secrets Manager**
```bash
# Create secret with required credentials
aws secretsmanager create-secret \
    --name "Medirecords_credentials" \
    --description "Credentials for Referral Automation" \
    --secret-string '{
        "GRAPH_TENANT_ID": "your-tenant-id",
        "GRAPH_CLIENT_ID": "your-client-id",
        "GRAPH_CLIENT_SECRET": "your-client-secret",
        "GRAPH_USER_EMAIL": "your-email@domain.com",
        "MEDIRECORDS_API_KEY": "your-api-key",
        "MEDIRECORDS_ENDPOINT": "your-api-endpoint"
    }'
```

### 4. **Configure S3 Bucket**
```bash
# Create S3 bucket for processed referrals
aws s3 mb s3://enas3bucket --region ap-southeast-2

# Enable versioning (optional but recommended)
aws s3api put-bucket-versioning \
    --bucket enas3bucket \
    --versioning-configuration Status=Enabled

# Configure bucket encryption
aws s3api put-bucket-encryption \
    --bucket enas3bucket \
    --server-side-encryption-configuration '{
        "Rules": [
            {
                "ApplyServerSideEncryptionByDefault": {
                    "SSEAlgorithm": "AES256"
                }
            }
        ]
    }'
```

### 5. **Update Configuration**
Edit `referral_automation/config.py`:
```python
# Replace with your actual values
REGION_NAME = 'ap-southeast-2'
SECRETS_MANAGER_ID = 'Medirecords_credentials'
S3_BUCKET = 'enas3bucket'
PROCESSED_FILE_KEY = "processed_emails.json"
GET_EMAIL_COUNT_PER_RUN = 10
LOG_LEVEL = 'info'
IS_LOCAL = 'True'  # Set to 'False' for Lambda deployment
```

---

## AWS Lambda Deployment

### 1. **Package Dependencies**
```bash
# Create deployment package
mkdir deployment-package
cp -r referral_automation/* deployment-package/
pip install -r requirements.txt -t deployment-package/

# Create ZIP file
cd deployment-package
zip -r ../lambda-deployment.zip .
cd ..
```

### Installing Python Packages for Lambda Deployment

Before creating your deployment zip, install all required Python packages into the `packages` directory:

```bash
pip install -r requirements.txt -t packages/
```

This ensures all dependencies are included in your Lambda deployment package.

After installing, run:

```bash
python zip.py
```

to create the deployment zip file.

### 2. **Create Lambda Function**
```bash
# Create Lambda function
aws lambda create-function \
    --function-name referral-automation \
    --runtime python3.9 \
    --role arn:aws:iam::<account-id>:role/lambda-execution-role \
    --handler lambda_function.lambda_handler \
    --zip-file fileb://lambda-deployment.zip \
    --timeout 900 \
    --memory-size 512 \
    --environment Variables='{"LOG_LEVEL":"info","IS_LOCAL":"false"}'
```

### 3. **Configure Lambda Triggers**
```bash
# Set up CloudWatch Events for scheduled execution
aws events put-rule \
    --name "referral-automation-schedule" \
    --schedule-expression "rate(1 hour)" \
    --description "Trigger referral automation every hour"

# Add permission for CloudWatch Events
aws lambda add-permission \
    --function-name referral-automation \
    --statement-id "CloudWatchEventsInvoke" \
    --action "lambda:InvokeFunction" \
    --principal "events.amazonaws.com" \
    --source-arn "arn:aws:events:ap-southeast-2:<account-id>:rule/referral-automation-schedule"

# Add CloudWatch Events target
aws events put-targets \
    --rule "referral-automation-schedule" \
    --targets "Id"="1","Arn"="arn:aws:lambda:ap-southeast-2:<account-id>:function:referral-automation"
```

---

## Local Testing

### 1. **Environment Setup**
```bash
# Set local testing flag
export IS_LOCAL=true

# Set AWS credentials (if not using aws configure)
export AWS_ACCESS_KEY_ID=your-access-key
export AWS_SECRET_ACCESS_KEY=your-secret-key
export AWS_DEFAULT_REGION=ap-southeast-2
```

### 2. **Run Local Tests**
```bash
# Test the complete pipeline
python run.py

# Test individual components
python -c "
import sys
sys.path.insert(0, 'referral_automation')
from referral_processor.email_extractor.graph_client import get_access_token
# Add your test code here
"
```

### 3. **Debug Mode**
```python
# Set debug logging in config.py
LOG_LEVEL = 'debug'

# Run with verbose output
python run.py 2>&1 | tee debug.log
```

---

## Runtime Configuration

### AWS Lambda Deployment
- **Runtime**: Python 3.9+
- **Memory**: Recommended 512MB+ for Textract processing
- **Timeout**: 15 minutes (900 seconds) for complete pipeline
- **Environment Variables**: Configured via config.py
- **Cold Start**: Considered in timeout planning

### Performance Considerations
- **Concurrent Processing**: Sequential email processing to avoid rate limits
- **Memory Management**: Temporary file cleanup after processing
- **S3 Optimization**: Batch operations for processed email tracking
- **Textract Limits**: Handles large documents with confidence filtering

### Monitoring & Alerts
```bash
# Create CloudWatch alarms for Lambda errors
aws cloudwatch put-metric-alarm \
    --alarm-name "referral-automation-errors" \
    --alarm-description "Alarm for Lambda function errors" \
    --metric-name "Errors" \
    --namespace "AWS/Lambda" \
    --statistic "Sum" \
    --period 300 \
    --threshold 1 \
    --comparison-operator "GreaterThanThreshold" \
    --evaluation-periods 1 \
    --dimensions "Name"="FunctionName","Value"="referral-automation"

# Create alarm for duration
aws cloudwatch put-metric-alarm \
    --alarm-name "referral-automation-duration" \
    --alarm-description "Alarm for Lambda function duration" \
    --metric-name "Duration" \
    --namespace "AWS/Lambda" \
    --statistic "Average" \
    --period 300 \
    --threshold 600000 \
    --comparison-operator "GreaterThanThreshold" \
    --evaluation-periods 2 \
    --dimensions "Name"="FunctionName","Value"="referral-automation"
```

---

## Execution

### Local Testing
```bash
# Run the complete pipeline locally
python run.py

# Test with specific configuration
IS_LOCAL=true LOG_LEVEL=debug python run.py
```

### Lambda Execution
```bash
# Invoke Lambda function manually
aws lambda invoke \
    --function-name referral-automation \
    --payload '{}' \
    response.json

# Check execution results
cat response.json
```

### Scheduled Execution
The Lambda function is configured to run automatically every hour via CloudWatch Events. You can modify the schedule by updating the CloudWatch Events rule.

---

## Monitoring

### CloudWatch Logs
```bash
# View recent logs
aws logs describe-log-groups --log-group-name-prefix "/aws/lambda/referral-automation"

# Get log events
aws logs get-log-events \
    --log-group-name "/aws/lambda/referral-automation" \
    --log-stream-name "latest" \
    --start-time $(date -d '1 hour ago' +%s)000
```

### S3 Bucket Monitoring
```bash
# List processed referrals
aws s3 ls s3://enas3bucket/referrals/

# Check processed emails tracking
aws s3 cp s3://enas3bucket/processed_emails.json - | jq '.'
```

### Error Log
```bash
# Check local error log
cat error_log.txt

# Monitor error log in real-time
tail -f error_log.txt
```

---

## Troubleshooting

### Common Deployment Issues

#### 1. **IAM Permission Errors**
```bash
# Verify Lambda execution role permissions
aws iam get-role --role-name lambda-execution-role

# Check attached policies
aws iam list-attached-role-policies --role-name lambda-execution-role
```

#### 2. **Secrets Manager Access Issues**
```bash
# Test secrets access
aws secretsmanager get-secret-value --secret-id Medirecords_credentials

# Verify secret exists
aws secretsmanager describe-secret --secret-id Medirecords_credentials
```

#### 3. **S3 Bucket Access Issues**
```bash
# Test S3 access
aws s3 ls s3://enas3bucket/

# Check bucket policy
aws s3api get-bucket-policy --bucket enas3bucket
```

#### 4. **Lambda Package Issues**
```bash
# Verify package size (should be < 50MB for direct upload)
ls -lh lambda-deployment.zip

# Check package contents
unzip -l lambda-deployment.zip | head -20
```

### Performance Optimization

#### 1. **Memory Optimization**
- Monitor Lambda memory usage in CloudWatch
- Increase memory allocation if needed
- Optimize temporary file handling

#### 2. **Timeout Optimization**
- Monitor execution duration
- Adjust timeout based on actual processing time
- Consider breaking large batches into smaller chunks

#### 3. **Cost Optimization**
- Monitor Lambda execution frequency
- Optimize S3 storage lifecycle policies
- Review CloudWatch log retention settings

---

## Support & Maintenance

### Regular Maintenance
- **Dependency Updates**: Regular security updates for Python packages
- **AWS Service Updates**: Monitor for service deprecations
- **API Version Updates**: Keep Microsoft Graph API client updated
- **Security Audits**: Regular review of IAM permissions and access logs

### Backup & Recovery
- **S3 Data**: Regular backup of processed referrals
- **Configuration**: Version control for configuration files
- **Secrets**: Regular rotation of API credentials
- **Disaster Recovery**: Cross-region backup procedures

### Monitoring & Alerts
- **Lambda Errors**: CloudWatch alarms for function failures
- **API Rate Limits**: Monitor Microsoft Graph API usage
- **Processing Delays**: Alert on extended processing times
- **Data Quality**: Monitor extraction success rates 