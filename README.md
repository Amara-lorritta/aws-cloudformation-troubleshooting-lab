Troubleshoot AWS CloudFormation Deployments

(Full professional portfolio layout, ready to post on GitHub with diagram, commands, and explanations.)

🧠 Overview

This lab provides practical experience troubleshooting AWS CloudFormation stack creation and deletion failures using both the AWS Management Console and AWS CLI.

You’ll investigate CloudFormation deployment issues, analyze logs, detect configuration drift, and resolve stack deletion problems while keeping critical data resources intact.

🎯 Objectives & Learning Outcomes

After completing this lab, you will be able to:

Use JMESPath queries to filter JSON output in the AWS CLI.

Deploy, monitor, and troubleshoot AWS CloudFormation stacks.

Interpret AWS CloudFormation stack events and rollback causes.

Inspect EC2 userdata logs to identify script errors.

Detect and interpret resource drift using the CLI.

Resolve DELETE_FAILED stack issues involving S3 buckets containing objects.

🏗️ AWS Architecture Diagram

Below is the AWS-style visual architecture (before → after troubleshooting).

Before: Failing CloudFormation deployment with EC2 WaitCondition timeout

After: Corrected deployment, drift detection, and stack deletion success

🧩 Commands & Steps (Copy-All-in-One Block)
# ---------- TASK 1: PRACTICE JMESPath ----------
# Visit https://jmespath.org and try these JSON queries
# Example JSON:
# {"StackResources": [{"LogicalResourceId": "VPC", "ResourceType": "AWS::EC2::VPC"},
# {"LogicalResourceId": "CliHostInstance", "ResourceType": "AWS::EC2::Instance"}]}
# Query to get EC2 logical resource:
# StackResources[?ResourceType=='AWS::EC2::Instance'].LogicalResourceId


# ---------- TASK 2: TROUBLESHOOT CLOUDFORMATION ----------
# Connect to CLI Host
ssh -i labsuser.pem ec2-user@<public-ip>

# Configure AWS CLI
aws configure

# View CloudFormation template
less template1.yaml

# Attempt to create a stack (will fail)
aws cloudformation create-stack \
--stack-name myStack \
--template-body file://template1.yaml \
--capabilities CAPABILITY_NAMED_IAM \
--parameters ParameterKey=KeyName,ParameterValue=vockey

# Observe creation status
watch -n 5 -d aws cloudformation describe-stack-resources \
--stack-name myStack \
--query 'StackResources[*].[ResourceType,ResourceStatus]' \
--output table

# View CREATE_FAILED events
aws cloudformation describe-stack-events \
--stack-name myStack \
--query "StackEvents[?ResourceStatus=='CREATE_FAILED']"

# Delete failed stack
aws cloudformation delete-stack --stack-name myStack


# ---------- TASK 2.4: RECREATE WITHOUT ROLLBACK ----------
aws cloudformation create-stack \
--stack-name myStack \
--template-body file://template1.yaml \
--capabilities CAPABILITY_NAMED_IAM \
--on-failure DO_NOTHING \
--parameters ParameterKey=KeyName,ParameterValue=vockey

# Get Web Server IP
aws ec2 describe-instances \
--filters "Name=tag:Name,Values='Web Server'" \
--query 'Reservations[].Instances[].[State.Name,PublicIpAddress]'

# SSH to the Web Server
ssh -i labsuser.pem ec2-user@<web-server-ip>

# View userdata logs
sudo tail -50 /var/log/cloud-init-output.log
sudo cat /var/lib/cloud/instance/scripts/part-001

# Fix template: change "http" to "httpd" in UserData
vim template1.yaml
cat template1.yaml | grep httpd

# Delete failed stack and recreate
aws cloudformation delete-stack --stack-name myStack
aws cloudformation create-stack \
--stack-name myStack \
--template-body file://template1.yaml \
--capabilities CAPABILITY_NAMED_IAM \
--on-failure DO_NOTHING \
--parameters ParameterKey=KeyName,ParameterValue=vockey

# Wait until CREATE_COMPLETE
watch -n 5 -d aws cloudformation describe-stacks \
--stack-name myStack --output table

# Test the web server
curl http://<public-ip>


# ---------- TASK 3: DRIFT DETECTION ----------
# Modify security group manually (SSH → My IP)
# Add a file to S3 bucket
bucketName=$(aws cloudformation describe-stacks \
--stack-name myStack \
--query "Stacks[*].Outputs[?OutputKey=='BucketName'].[OutputValue]" \
--output text)
touch myfile
aws s3 cp myfile s3://$bucketName/
aws s3 ls $bucketName/

# Detect drift
aws cloudformation detect-stack-drift --stack-name myStack
# Get drift detection status
aws cloudformation describe-stack-drift-detection-status \
--stack-drift-detection-id <drift-id>
# Summarize drifted resources
aws cloudformation describe-stack-resource-drifts \
--stack-name myStack \
--stack-resource-drift-status-filters MODIFIED


# ---------- TASK 4: DELETE STACK (CHALLENGE) ----------
# Attempt to delete (will fail due to S3 object)
aws cloudformation delete-stack --stack-name myStack
# Check reason for DELETE_FAILED
aws cloudformation describe-stacks --stack-name myStack --output table

# Challenge Solution:
# Keep S3 bucket, delete rest of stack
bucketLogicalId=$(aws cloudformation list-stack-resources \
--stack-name myStack \
--query "StackResourceSummaries[?ResourceType=='AWS::S3::Bucket'].LogicalResourceId" \
--output text)
aws cloudformation delete-stack \
--stack-name myStack \
--retain-resources $bucketLogicalId

🔎 What Actually Happened

Initial Stack Failure:
CloudFormation stack creation failed due to a WaitCondition timeout caused by a yum install http command error in the EC2 instance’s UserData script.

Troubleshooting:
By disabling rollback (--on-failure DO_NOTHING), the EC2 instance persisted. SSH access and log inspection revealed that the correct package name should be httpd, not http.

Fix and Redeploy:
After correcting the script, the stack deployed successfully, creating all resources including EC2, VPC, and S3.

Drift Detection:
Manual changes to the Security Group and S3 bucket triggered drift detection, identifying discrepancies (MODIFIED state).

Delete Failure & Solution:
The stack deletion failed because the S3 bucket contained objects.
Solution: Use the --retain-resources flag to preserve the bucket while deleting other resources successfully.

💬 How to Explain It to an Interviewer

“I deployed a CloudFormation stack that initially failed due to a WaitCondition timeout. I used the AWS CLI and JMESPath queries to diagnose that the EC2 userdata script had a package error.

I fixed it, redeployed the stack, and verified successful creation. Then, I intentionally introduced configuration drift in the security group and detected it using the drift detection API.

Finally, I resolved a DELETE_FAILED issue caused by an S3 bucket with objects by retaining that resource using the --retain-resources parameter. This demonstrated my ability to troubleshoot, debug, and manage infrastructure automation end-to-end using CloudFormation and AWS CLI.”

🧰 Tools Used

AWS CloudFormation

Amazon EC2

Amazon S3

AWS CLI

JMESPath

Linux Bash utilities (watch, grep, vim)

🔑 Key Takeaways

✅ CloudFormation rollback deletes failed resources unless --on-failure DO_NOTHING is used.
✅ JMESPath queries enable precise JSON filtering in CLI outputs.
✅ Inspect /var/log/cloud-init-output.log for EC2 userdata script failures.
✅ Drift detection identifies configuration changes made outside of CloudFormation.
✅ DELETE_FAILED can be resolved via --retain-resources to preserve non-empty S3 buckets.
✅ Infrastructure as Code (IaC) enables repeatable, version-controlled AWS deployments.
