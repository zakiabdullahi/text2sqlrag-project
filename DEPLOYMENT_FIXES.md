# Deployment Fixes Documentation

**Date:** 2026-01-22
**Status:** ✅ RESOLVED - All issues fixed, deployment successful
**Deployment Time:** ~3 minutes (with Docker layer caching)

---

## 🎯 Executive Summary

The Lambda deployment was failing because:
1. **Wrong S3 bucket name** in Lambda environment variables
2. **IAM policy pointing to wrong S3 bucket**
3. **GitHub secrets had incorrect bucket name**

All issues have been resolved and the deployment is now fully operational with S3 storage and caching working correctly.

---

## 🔍 Root Cause Analysis

### Issue #1: Lambda Environment Variable - Wrong S3 Bucket Name

**Symptom:**
```bash
Failed to initialize S3 storage: S3 bucket 'rag-bucket-doc-cache' does not exist. Falling back to local.
```

**Root Cause:**
- Lambda environment variable `S3_CACHE_BUCKET` was set to: `rag-bucket-doc-cache`
- Actual S3 bucket name: `rag-cache-docsqa`
- This caused S3StorageBackend initialization to fail with 404 error
- System fell back to LocalStorageBackend (cache_service.py:50)

**Impact:**
- Deployment tests expected `storage_backend: "s3"` but got `storage_backend: "local"`
- S3 caching completely disabled
- Documents stored in ephemeral `/tmp` storage (lost on container restart)

---

### Issue #2: IAM Policy - Wrong S3 Bucket ARN

**Symptom:**
```bash
Access denied to S3 bucket (403 Forbidden)
```

**Root Cause:**
- IAM role policy granted access to: `arn:aws:s3:::rag-cache-bucket`
- Actual bucket: `arn:aws:s3:::rag-cache-docsqa`
- Even after fixing env var, S3 validation would fail with 403 Forbidden

**Impact:**
- Lambda couldn't validate S3 bucket access on startup
- S3StorageBackend initialization failed
- Fallback to local storage

---

### Issue #3: GitHub Secrets - Incorrect Values

**Symptom:**
- Workflow used wrong API Gateway URL for testing
- Wrong S3 bucket name propagated to Lambda on deployment

**Root Cause:**
- `API_GATEWAY_URL` secret: pointed to non-existent endpoint
- `S3_CACHE_BUCKET` secret: set to `rag-cache-bucket` instead of `rag-cache-docsqa`

**Impact:**
- Workflow tests failed to reach API
- Every deployment propagated wrong bucket name to Lambda

---

### Issue #4: Warm Lambda Containers

**Symptom:**
- After fixing environment variables, Lambda still reported `storage_backend: "local"`

**Root Cause:**
- Lambda keeps containers warm for 5-15 minutes
- Old containers initialized with old environment variables
- Tests were hitting old warm containers instead of new ones

**Impact:**
- Environment variable changes didn't take effect immediately
- Manual container cycling required

---

## 🔧 All Fixes Applied

### Fix #1: Update Lambda Environment Variable

**Command:**
```bash
# Step 1: Fetch current environment variables
aws lambda get-function-configuration \
  --function-name rag-text-to-sql-serverless \
  --region us-east-1 \
  --output json > /tmp/lambda-config.json

cat /tmp/lambda-config.json | jq '.Environment.Variables' > /tmp/current-env.json

# Step 2: Update S3_CACHE_BUCKET to correct value
cat /tmp/current-env.json | jq '.S3_CACHE_BUCKET = "rag-cache-docsqa"' > /tmp/updated-env.json

# Step 3: Apply updated environment variables
cat /tmp/updated-env.json | jq '{Variables: .}' > /tmp/env-config.json

aws lambda update-function-configuration \
  --function-name rag-text-to-sql-serverless \
  --region us-east-1 \
  --environment file:///tmp/env-config.json

# Step 4: Wait for update to complete
aws lambda wait function-updated \
  --function-name rag-text-to-sql-serverless \
  --region us-east-1
```

**Verification:**
```bash
aws lambda get-function-configuration \
  --function-name rag-text-to-sql-serverless \
  --region us-east-1 \
  --output json | jq -r '.Environment.Variables.S3_CACHE_BUCKET'
# Expected output: rag-cache-docsqa
```

---

### Fix #2: Update IAM Policy

**Command:**
```bash
# Create new IAM policy with correct bucket ARN
cat > /tmp/s3-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "S3CacheAccess",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject",
        "s3:ListBucket",
        "s3:HeadObject",
        "s3:HeadBucket"
      ],
      "Resource": [
        "arn:aws:s3:::rag-cache-docsqa",
        "arn:aws:s3:::rag-cache-docsqa/*"
      ]
    }
  ]
}
EOF

# Apply updated policy
aws iam put-role-policy \
  --role-name rag-lambda-execution-role \
  --policy-name S3CacheAccess \
  --policy-document file:///tmp/s3-policy.json
```

**Verification:**
```bash
# Check IAM policy is updated
aws iam get-role-policy \
  --role-name rag-lambda-execution-role \
  --policy-name S3CacheAccess \
  --output json | jq '.PolicyDocument'

# Test S3 bucket access from CLI
aws s3 ls s3://rag-cache-docsqa/ --region us-east-1
```

**Notes:**
- Added `s3:HeadBucket` permission (required for bucket validation)
- Changed bucket ARN from `rag-cache-bucket` to `rag-cache-docsqa`

---

### Fix #3: Update GitHub Secrets

**Commands:**
```bash
# Update S3_CACHE_BUCKET secret
gh secret set S3_CACHE_BUCKET --body "rag-cache-docsqa"

# Update API_GATEWAY_URL secret
gh secret set API_GATEWAY_URL --body "https://vjtotquohk.execute-api.us-east-1.amazonaws.com/prod"

# Verify secrets updated
gh secret list | grep -E "API_GATEWAY_URL|S3_CACHE_BUCKET"
```

**How to find correct values:**
```bash
# Find API Gateway URL
aws apigatewayv2 get-apis --region us-east-1 \
  --query 'Items[?contains(Name, `rag`) || contains(Name, `sql`)].{Name:Name, ApiEndpoint:ApiEndpoint}' \
  --output table

# Find S3 bucket name
aws s3 ls | grep -i rag
```

---

### Fix #4: Force Lambda Container Restart

**Command:**
```bash
# Option 1: Update function description (forces container restart)
aws lambda update-function-configuration \
  --function-name rag-text-to-sql-serverless \
  --region us-east-1 \
  --description "Multi-Source RAG + Text-to-SQL API (Updated $(date +%s))"

aws lambda wait function-updated \
  --function-name rag-text-to-sql-serverless \
  --region us-east-1

# Option 2: Cycle through containers by invoking multiple times
API_URL="https://vjtotquohk.execute-api.us-east-1.amazonaws.com/prod"
for i in {1..20}; do
  curl -s "$API_URL/health" > /dev/null
  sleep 0.5
done
```

**Why this works:**
- Lambda terminates old containers when configuration changes
- Multiple invocations ensure all warm containers are cycled
- New containers use updated environment variables

---

### Fix #5: Improve Deployment Workflow

**File:** `.github/workflows/deploy.yml`

**Change:** Replaced passive wait with active container cycling

**Before:**
```yaml
# Wait 120 seconds for Lambda containers to expire
echo "⏳ Waiting 120 seconds for Lambda to fully switch to new containers..."
for i in {120..1}; do
  sleep 1
done
```

**After:**
```yaml
# Force Lambda container refresh by invoking multiple times
echo "🔄 Forcing Lambda container refresh (cycling through warm containers)..."
echo "   (Invoking 20 times to ensure all old containers are replaced)"
SUCCESS_COUNT=0
for i in {1..20}; do
  if curl -f -s "$API_URL/health" > /dev/null 2>&1; then
    SUCCESS_COUNT=$((SUCCESS_COUNT + 1))
  fi
  sleep 0.5
done
```

**Benefits:**
- Reduces wait time from 120s to ~10s
- Actively forces container cycling instead of passive waiting
- More reliable container refresh

---

## 🧪 Verification Commands

### 1. Check Lambda Configuration

```bash
# Get all Lambda configuration
aws lambda get-function-configuration \
  --function-name rag-text-to-sql-serverless \
  --region us-east-1 \
  --output json

# Check specific environment variables
aws lambda get-function-configuration \
  --function-name rag-text-to-sql-serverless \
  --region us-east-1 \
  --output json | jq -r '.Environment.Variables |
    {STORAGE_BACKEND, S3_CACHE_BUCKET, ENVIRONMENT}'

# Check Lambda IAM role
aws lambda get-function-configuration \
  --function-name rag-text-to-sql-serverless \
  --region us-east-1 \
  --output json | jq -r '.Role'
```

---

### 2. Check IAM Permissions

```bash
# Get Lambda role name
ROLE_ARN=$(aws lambda get-function-configuration \
  --function-name rag-text-to-sql-serverless \
  --region us-east-1 \
  --output json | jq -r '.Role')
ROLE_NAME=$(echo $ROLE_ARN | awk -F'/' '{print $NF}')

# List all policies attached to role
aws iam list-role-policies --role-name "$ROLE_NAME"

# Get S3 policy details
aws iam get-role-policy \
  --role-name "$ROLE_NAME" \
  --policy-name S3CacheAccess \
  --output json | jq '.PolicyDocument'
```

---

### 3. Check S3 Bucket

```bash
# List S3 buckets
aws s3 ls | grep -i rag

# Check bucket contents
aws s3 ls s3://rag-cache-docsqa/ --recursive | head -20

# Test bucket access
aws s3 ls s3://rag-cache-docsqa/txt/ 2>&1
```

---

### 4. Test Lambda API

```bash
API_URL="https://vjtotquohk.execute-api.us-east-1.amazonaws.com/prod"

# Test 1: Health check
curl -s "$API_URL/health" | jq '{status, services, configuration}'

# Test 2: Upload document (check storage backend)
echo "Test document $(date)" > /tmp/test.txt
curl -s -X POST "$API_URL/upload" \
  -F "file=@/tmp/test.txt" | jq '{status, storage_backend, cache_hit}'

# Test 3: Re-upload same document (check S3 caching)
sleep 2
curl -s -X POST "$API_URL/upload" \
  -F "file=@/tmp/test.txt" | jq '{status, storage_backend, cache_hit}'
# Expected: storage_backend="s3", cache_hit=true
```

---

### 5. Check CloudWatch Logs

```bash
# Tail recent logs
aws logs tail /aws/lambda/rag-text-to-sql-serverless \
  --region us-east-1 \
  --since 5m \
  --follow

# Search for S3-related logs
aws logs tail /aws/lambda/rag-text-to-sql-serverless \
  --region us-east-1 \
  --since 10m | grep -i "s3\|storage"

# Search for errors
aws logs tail /aws/lambda/rag-text-to-sql-serverless \
  --region us-east-1 \
  --since 10m | grep -i "error\|failed\|warning"
```

---

### 6. Check GitHub Actions

```bash
# List recent workflow runs
gh run list --workflow=deploy.yml --limit 5

# View specific run
gh run view <RUN_ID>

# View failed logs
gh run view <RUN_ID> --log-failed

# Watch current run
gh run watch
```

---

## 📊 Deployment Test Results

### Final Verification (2026-01-22 08:27 UTC)

```json
Test 1: Health Check
{
  "status": "healthy",
  "services": {
    "embedding_service": true,
    "vector_service": true,
    "rag_service": true,
    "sql_service": true,
    "query_cache": true
  },
  "configuration": {
    "openai_configured": true,
    "pinecone_configured": true,
    "database_configured": true,
    "opik_configured": true,
    "redis_cache_configured": true
  }
}

Test 2: Upload Document (S3 Storage)
{
  "status": "success",
  "storage_backend": "s3",
  "cache_hit": false,
  "chunks_created": 1
}

Test 3: Re-upload Document (S3 Cache)
{
  "status": "success",
  "storage_backend": "s3",
  "cache_hit": true
}
```

**Results:**
- ✅ All services healthy
- ✅ Lambda using S3 storage
- ✅ S3 caching working
- ✅ Deployment fully operational
- ✅ Deployment time: 3 minutes (with Docker cache)

---

## 🚀 Future Deployment Checklist

Before deploying, verify these settings:

### 1. GitHub Secrets (Required)

```bash
# Check all secrets are set
gh secret list

# Required secrets:
# - AWS_ACCESS_KEY_ID
# - AWS_SECRET_ACCESS_KEY
# - API_GATEWAY_URL (format: https://{api-id}.execute-api.{region}.amazonaws.com/prod)
# - S3_CACHE_BUCKET (must match actual bucket name)
# - OPENAI_API_KEY
# - PINECONE_API_KEY
# - PINECONE_ENVIRONMENT
# - PINECONE_INDEX_NAME
# - DATABASE_URL
# - UPSTASH_REDIS_URL
# - UPSTASH_REDIS_TOKEN
# - OPIK_API_KEY
```

### 2. AWS Resources (Required)

```bash
# Lambda function must exist
aws lambda get-function --function-name rag-text-to-sql-serverless --region us-east-1

# S3 bucket must exist and be accessible
aws s3 ls s3://rag-cache-docsqa/

# Lambda IAM role must have S3 permissions
aws iam get-role-policy \
  --role-name rag-lambda-execution-role \
  --policy-name S3CacheAccess

# API Gateway must exist and point to Lambda
aws apigatewayv2 get-apis --region us-east-1
```

### 3. Deployment Workflow Settings

```yaml
# .github/workflows/deploy.yml
env:
  AWS_REGION: us-east-1
  ECR_REPOSITORY: rag-text-to-sql-doc
  LAMBDA_FUNCTION: rag-text-to-sql-serverless
```

---

## 🔧 Debugging Commands Cheat Sheet

### Quick Health Check
```bash
API_URL="https://vjtotquohk.execute-api.us-east-1.amazonaws.com/prod"
curl -s "$API_URL/health" | jq .
```

### Check Storage Backend
```bash
echo "test" > /tmp/test.txt
curl -s -X POST "$API_URL/upload" -F "file=@/tmp/test.txt" | jq -r .storage_backend
# Expected: "s3"
```

### Force Lambda Restart
```bash
aws lambda update-function-configuration \
  --function-name rag-text-to-sql-serverless \
  --region us-east-1 \
  --description "Force restart $(date +%s)"
```

### View Recent Logs
```bash
aws logs tail /aws/lambda/rag-text-to-sql-serverless \
  --region us-east-1 --since 5m | grep -i "error\|warning\|storage"
```

### Trigger Manual Deployment
```bash
git commit --allow-empty -m "Trigger deployment"
git push origin main
gh run watch
```

### Check GitHub Actions Status
```bash
gh run list --workflow=deploy.yml --limit 3
gh run view --log-failed  # View latest failed run
```

---

## 📝 Environment Variables Reference

### Lambda Environment Variables (Current Configuration)

```bash
ENVIRONMENT=production
STORAGE_BACKEND=s3
S3_CACHE_BUCKET=rag-cache-docsqa

# OpenAI
OPENAI_API_KEY=sk-proj-...

# Pinecone
PINECONE_API_KEY=pcsk_...
PINECONE_ENVIRONMENT=us-east-1-aws
PINECONE_INDEX_NAME=rag-documents

# Database
DATABASE_URL=postgresql://...

# Redis Cache
UPSTASH_REDIS_URL=https://...
UPSTASH_REDIS_TOKEN=...

# Monitoring
OPIK_API_KEY=...

# Lambda paths
HOME=/tmp
TMPDIR=/tmp
TEMP=/tmp
TMP=/tmp
HF_HOME=/tmp/.cache/huggingface
TRANSFORMERS_CACHE=/tmp/.cache/huggingface
UNSTRUCTURED_CACHE_DIR=/tmp/.cache/unstructured
DOCLING_CACHE_DIR=/tmp/.cache/docling
```

---

## 🎯 Key Takeaways

1. **Always verify bucket names match** across:
   - Lambda environment variables
   - IAM policies
   - GitHub secrets
   - Application code

2. **Lambda warm containers can cache old config**:
   - Changes to environment variables don't apply immediately
   - Force restart by updating configuration
   - Or cycle through containers with multiple invocations

3. **IAM permissions must be exact**:
   - Bucket ARN must match exactly
   - Include all required S3 actions (HeadBucket, HeadObject, etc.)

4. **GitHub secrets are deployment-critical**:
   - Wrong secrets propagate wrong config to Lambda
   - Always verify after updating

5. **CloudWatch logs are essential for debugging**:
   - Check logs immediately when deployment fails
   - Search for "S3", "storage", "failed", "warning"

---

## 📞 Support Commands

If deployment fails in the future, run these diagnostic commands:

```bash
# 1. Check Lambda is using correct config
aws lambda get-function-configuration \
  --function-name rag-text-to-sql-serverless \
  --region us-east-1 | jq '.Environment.Variables | {STORAGE_BACKEND, S3_CACHE_BUCKET}'

# 2. Test S3 bucket access
aws s3 ls s3://rag-cache-docsqa/ 2>&1

# 3. Check IAM policy
aws iam get-role-policy \
  --role-name rag-lambda-execution-role \
  --policy-name S3CacheAccess | jq '.PolicyDocument.Statement[0].Resource'

# 4. Test API endpoint
curl -s "https://vjtotquohk.execute-api.us-east-1.amazonaws.com/prod/health" | jq .

# 5. Check recent errors in logs
aws logs tail /aws/lambda/rag-text-to-sql-serverless \
  --region us-east-1 --since 10m | grep -i "error\|failed"
```

---

## 📚 Additional Resources

- **AWS Lambda Docs**: https://docs.aws.amazon.com/lambda/
- **AWS IAM Policies**: https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies.html
- **GitHub Actions**: https://docs.github.com/en/actions
- **AWS CLI Reference**: https://docs.aws.amazon.com/cli/latest/reference/

---

**Document Version:** 1.0
**Last Updated:** 2026-01-22
**Status:** ✅ All issues resolved, deployment operational
