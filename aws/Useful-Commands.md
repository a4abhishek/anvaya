# AWS: Useful Commands

> **The Anvaya:** *The AWS CLI is the scriptable API — every console click has a CLI equivalent, and every CLI command has an IAM permission behind it.*

---

## 🔑 Auth & Identity

**Check who you're authenticated as**

* **Why:** Verify which IAM identity (user or role) your current credentials represent.
* **Command:**

```bash
aws sts get-caller-identity
# Returns: Account ID, User ID, ARN — confirm you're using the right profile/role
```

**Switch between AWS profiles**

* **Why:** Manage multiple accounts (personal, work, prod, staging) cleanly.
* **Command:**

```bash
export AWS_PROFILE=work-prod
aws sts get-caller-identity   # Verify the switch took effect

# Or per-command:
aws s3 ls --profile personal
```

**Assume an IAM role (get temporary credentials)**

* **Why:** Test a role's permissions, or automate cross-account access.
* **Command:**

```bash
CREDS=$(aws sts assume-role \
  --role-arn arn:aws:iam::123456789:role/my-role \
  --role-session-name debug-session \
  --query 'Credentials.[AccessKeyId,SecretAccessKey,SessionToken]' \
  --output text)

export AWS_ACCESS_KEY_ID=$(echo $CREDS | awk '{print $1}')
export AWS_SECRET_ACCESS_KEY=$(echo $CREDS | awk '{print $2}')
export AWS_SESSION_TOKEN=$(echo $CREDS | awk '{print $3}')
aws sts get-caller-identity   # Verify you're now acting as the role
```

---

## 🏛️ IAM

**List all policies attached to a role**

* **Why:** Audit what permissions a role has without clicking through the console.
* **Command:**

```bash
aws iam list-attached-role-policies --role-name my-role
aws iam list-role-policies --role-name my-role  # Inline policies
```

**Check what actions a policy allows**

* **Why:** Get the full JSON of a managed or inline policy.
* **Command:**

```bash
# Get policy ARN first, then get the version document
aws iam get-policy --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
aws iam get-policy-version \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess \
  --version-id v1 \
  --query 'PolicyVersion.Document'
```

**Simulate whether a role can perform an action**

* **Why:** Debug `AccessDenied` without actually making the API call.
* **Command:**

```bash
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::123456789:role/myapp-task-role \
  --action-names s3:GetObject \
  --resource-arns arn:aws:s3:::my-bucket/path/to/key
# Returns: allowed | implicitDeny | explicitDeny
```

---

## 🖥️ EC2

**List running instances with name and IP**

* **Why:** Quick overview of all EC2 instances without opening the console.
* **Command:**

```bash
aws ec2 describe-instances \
  --filters Name=instance-state-name,Values=running \
  --query 'Reservations[*].Instances[*].[Tags[?Key==`Name`].Value|[0],InstanceId,PrivateIpAddress,PublicIpAddress,InstanceType]' \
  --output table
```

**SSM Session Manager — SSH without a key or port 22**

* **Why:** Access instances in private subnets, with full CloudTrail audit, no open ports.
* **Command:**

```bash
# Requires: SSM agent on instance + AmazonSSMManagedInstanceCore policy on instance role
aws ssm start-session --target i-0abc123def456789

# Port-forward (e.g., connect to RDS via private instance):
aws ssm start-session \
  --target i-0abc123def456789 \
  --document-name AWS-StartPortForwardingSessionToRemoteHost \
  --parameters '{"host":["mydb.cluster-xxx.ap-south-1.rds.amazonaws.com"],"portNumber":["5432"],"localPortNumber":["5432"]}'
# Then: psql -h localhost -p 5432 -U admin mydb
```

**Find the latest Amazon Linux 2023 AMI in your region**

* **Why:** AMI IDs are region-specific and change with new releases; this always gets the latest.
* **Command:**

```bash
aws ec2 describe-images \
  --owners amazon \
  --filters "Name=name,Values=al2023-ami-2023*" "Name=architecture,Values=x86_64" \
  --query 'sort_by(Images, &CreationDate)[-1].ImageId' \
  --output text
```

**Stop and start all EC2 instances with a specific tag (dev cost-saving)**

* **Why:** Stop dev/staging instances overnight. Saves 60-65% compared to running 24/7.
* **Command:**

```bash
# Get IDs for instances tagged Environment=staging
INSTANCE_IDS=$(aws ec2 describe-instances \
  --filters "Name=tag:Environment,Values=staging" "Name=instance-state-name,Values=running" \
  --query 'Reservations[*].Instances[*].InstanceId' --output text)

aws ec2 stop-instances --instance-ids $INSTANCE_IDS
```

---

## 🪣 S3

**List all buckets with their creation dates and regions**

* **Why:** Audit all S3 buckets in the account.
* **Command:**

```bash
aws s3api list-buckets --query 'Buckets[*].[Name,CreationDate]' --output table
```

**Sync local directory to S3 (deploy static site or upload artifacts)**

* **Why:** Efficient, incremental upload — only changed files are transferred.
* **Command:**

```bash
aws s3 sync ./dist s3://my-static-site/ --delete   # --delete removes files in S3 not in local
aws s3 sync s3://my-backups/2024/ ./local-restore/  # Download from S3 to local
```

**Find and delete unversioned objects older than N days**

* **Why:** Clear out old CI artifacts or logs that lifecycle rules missed.
* **Command:**

```bash
# List objects older than 30 days (for review first):
aws s3api list-objects-v2 --bucket my-artifacts \
  --query 'Contents[?LastModified<=`2024-01-01`].[Key,LastModified,Size]' \
  --output table
```

**Check if public access is blocked on all buckets (security audit)**

* **Why:** Verify no bucket was accidentally made public.
* **Command:**

```bash
for bucket in $(aws s3api list-buckets --query 'Buckets[*].Name' --output text); do
  STATUS=$(aws s3api get-public-access-block --bucket $bucket \
    --query 'PublicAccessBlockConfiguration.BlockPublicPolicy' --output text 2>/dev/null || echo "NOT_SET")
  echo "$bucket: $STATUS"
done
```

---

## 📦 ECR

**Login, build, tag, and push an image**

* **Why:** The full ECR push workflow in one go.
* **Command:**

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
REGION=ap-south-1
REPO=$ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com

aws ecr get-login-password --region $REGION | docker login --username AWS --password-stdin $REPO
docker build -t $REPO/myapp:$(git rev-parse --short HEAD) .
docker push $REPO/myapp:$(git rev-parse --short HEAD)
```

**List image tags in a repository**

* **Why:** Check what versions are available without opening the console.
* **Command:**

```bash
aws ecr describe-images --repository-name myapp \
  --query 'sort_by(imageDetails, &imagePushedAt)[-10:].[imageTags[0],imagePushedAt,imageSizeInBytes]' \
  --output table
```

---

## 🐳 ECS

**Force a new deployment (rolling update with latest image)**

* **Why:** Trigger ECS to pull the latest image and replace running tasks without changing the Task Definition.
* **Command:**

```bash
aws ecs update-service \
  --cluster myapp-cluster \
  --service myapp \
  --force-new-deployment
```

**Watch ECS service deployment status**

* **Why:** Monitor a rolling deployment to confirm all tasks are healthy.
* **Command:**

```bash
aws ecs wait services-stable --cluster myapp-cluster --services myapp
# Blocks until deployment is complete or fails (use in CI/CD pipelines)
```

**Get logs of a stopped ECS task (debugging crashes)**

* **Why:** Stopped tasks disappear from the console quickly — get their logs before they expire.
* **Command:**

```bash
# Get the log stream name for a stopped task
TASK_ID=abc123
aws logs get-log-events \
  --log-group-name /ecs/myapp \
  --log-stream-name ecs/myapp/$TASK_ID \
  --query 'events[*].[timestamp,message]' \
  --output text
```

**Run a one-off ECS task (like a DB migration)**

* **Why:** Run a command in the same environment as your service without spinning up a new service.
* **Command:**

```bash
aws ecs run-task \
  --cluster myapp-cluster \
  --task-definition myapp:latest \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-priv-a],securityGroups=[sg-app],assignPublicIp=DISABLED}" \
  --overrides '{"containerOverrides":[{"name":"myapp","command":["./migrate"]}]}'
```

**Execute a shell inside a running ECS task (like kubectl exec)**

* **Why:** Debug a live Fargate container without SSH. Requires ECS Exec to be enabled on the service.
* **Command:**

```bash
# Enable Exec on the service first (one-time):
aws ecs update-service --cluster myapp-cluster --service myapp --enable-execute-command

aws ecs execute-command \
  --cluster myapp-cluster \
  --task <task-id> \
  --container myapp \
  --interactive \
  --command "/bin/sh"
```

---

## ☸️ EKS

**Update kubeconfig and switch to EKS cluster**

* **Why:** Configure `kubectl` to point at your EKS cluster — replaces doing this manually.
* **Command:**

```bash
aws eks update-kubeconfig --name myapp-prod --region ap-south-1
kubectl config current-context   # Verify
kubectl get nodes
```

**Get EKS add-on versions available for upgrade**

* **Why:** Check if VPC CNI, CoreDNS, kube-proxy can be updated.
* **Command:**

```bash
aws eks describe-addon-versions --kubernetes-version 1.31 \
  --query 'addons[*].[addonName,addonVersions[0].addonVersion]' \
  --output table
```

---

## 🔐 Secrets Manager

**Create and retrieve secrets**

* **Why:** Core secret lifecycle management from the CLI.
* **Command:**

```bash
# Create
aws secretsmanager create-secret \
  --name myapp/prod/db-password \
  --secret-string '{"username":"admin","password":"sup3rS3cret"}'

# Retrieve (as JSON)
aws secretsmanager get-secret-value \
  --secret-id myapp/prod/db-password \
  --query SecretString \
  --output text | jq .

# Update/rotate manually
aws secretsmanager put-secret-value \
  --secret-id myapp/prod/db-password \
  --secret-string '{"username":"admin","password":"n3wP4ssword"}'
```

---

## 📊 CloudWatch

**Tail application logs in real-time**

* **Why:** The `kubectl logs -f` equivalent for ECS/EC2 applications.
* **Command:**

```bash
aws logs tail /ecs/myapp --follow --format short

# Filter for errors in the last 2 hours:
aws logs filter-log-events \
  --log-group-name /ecs/myapp \
  --start-time $(date -d '2 hours ago' +%s)000 \
  --filter-pattern "ERROR" \
  --query 'events[*].[timestamp,message]' \
  --output text
```

**Query logs with CloudWatch Log Insights**

* **Why:** SQL-like ad-hoc log analysis — find error rates, slow requests, etc.
* **Command:**

```bash
aws logs start-query \
  --log-group-name /ecs/myapp \
  --start-time $(date -d '1 hour ago' +%s) \
  --end-time $(date +%s) \
  --query-string 'fields @timestamp, @message | filter @message like /ERROR/ | sort @timestamp desc | limit 20'

# Then get results with the query ID returned above:
aws logs get-query-results --query-id <query-id>
```

**List all log groups and their retention settings**

* **Why:** Audit for log groups without retention policies (will grow unbounded and cost money).
* **Command:**

```bash
aws logs describe-log-groups \
  --query 'logGroups[*].[logGroupName,retentionInDays,storedBytes]' \
  --output table | sort
# retentionInDays of "None" = never expires = cost risk
```

---

## 🌐 VPC & Networking

**List all VPCs, subnets, and their AZs**

* **Why:** Get a quick overview of the network topology without using the console.
* **Command:**

```bash
# VPCs
aws ec2 describe-vpcs --query 'Vpcs[*].[VpcId,CidrBlock,Tags[?Key==`Name`].Value|[0]]' --output table

# Subnets in a VPC
aws ec2 describe-subnets \
  --filters Name=vpc-id,Values=vpc-xxx \
  --query 'Subnets[*].[SubnetId,CidrBlock,AvailabilityZone,Tags[?Key==`Name`].Value|[0]]' \
  --output table
```

**Check Security Group rules for an instance**

* **Why:** Debug connectivity issues — see what inbound/outbound rules are actually applied.
* **Command:**

```bash
# Get SG IDs for an instance
aws ec2 describe-instances --instance-ids i-xxx \
  --query 'Reservations[0].Instances[0].SecurityGroups'

# Get rules for a specific SG
aws ec2 describe-security-group-rules \
  --filters Name=group-id,Values=sg-xxx \
  --query 'SecurityGroupRules[*].[IsEgress,IpProtocol,FromPort,ToPort,CidrIpv4,ReferencedGroupInfo.GroupId]' \
  --output table
```

---

## 💰 Cost Debugging

**Find top 5 most expensive services this month**

* **Why:** Quickly identify the biggest cost drivers without navigating the Cost Explorer UI.
* **Command:**

```bash
aws ce get-cost-and-usage \
  --time-period Start=$(date -d 'first day of this month' +%Y-%m-%d),End=$(date +%Y-%m-%d) \
  --granularity MONTHLY \
  --metrics UnblendedCost \
  --group-by Type=DIMENSION,Key=SERVICE \
  --query 'ResultsByTime[0].Groups | sort_by(@, &Metrics.UnblendedCost.Amount)[-5:].[Keys[0],Metrics.UnblendedCost.Amount]' \
  --output table
```

**Find unattached EBS volumes (paying for nothing)**

* **Why:** Stopped instances leave EBS volumes behind. ~$0.08/GB/month unused.
* **Command:**

```bash
aws ec2 describe-volumes \
  --filters Name=status,Values=available \
  --query 'Volumes[*].[VolumeId,Size,CreateTime,AvailabilityZone]' \
  --output table
```
