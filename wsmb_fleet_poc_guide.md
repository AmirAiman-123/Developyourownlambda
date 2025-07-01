# WSMB 2025 Cloud Computing Challenge: Fleet POC

## Comprehensive AWS Deployment Guide

### Step 1: IAM Role Creation

- Navigate to IAM → Roles → Create role.
- Choose "AWS Service" → Lambda.
- Name the role `FleetPOC-Lambda-Role`.
- Attach policies:
  - **AWSLambdaBasicExecutionRole**
  - **AWSLambdaVPCAccessExecutionRole**
  - **SecretsManagerReadWrite**
  - **AmazonRDSDataFullAccess**
  - **CloudWatchLogsFullAccess**
- Create the role.

### Step 2: VPC Creation

- **VPC Name:** FleetPOC-VPC
- **CIDR:** `10.0.0.0/16`
- **Availability Zones:** `ap-southeast-1a`, `ap-southeast-1b`

**Subnets:**

- Public Subnet 1: `10.0.1.0/24`
- Public Subnet 2: `10.0.2.0/24`
- Private Subnet 1: `10.0.3.0/24`
- Private Subnet 2: `10.0.4.0/24`

**Internet Gateway:** Attach to VPC.

**Route Tables:**

- Public: `0.0.0.0/0` → IGW
- Private: `0.0.0.0/0` → NAT Gateway

**NAT Gateway:** Place in Public Subnet 1.

### Step 3: RDS MySQL DB Deployment

- **Engine:** MySQL 8.0
- **Instance:** `db.t3.micro`, 20 GB SSD
- **Identifier:** `fleetdb`
- **Username:** `admin`, Password: `YourPassword123!`
- **Subnet Group:** Use Private Subnets
- **Public Access:** NO
- **Security Group:** Allow inbound MySQL (3306) from Lambda SG

### Step 4: Configure RDS

```bash
sudo yum install mysql -y
mysql -h <your-RDS-endpoint> -u admin -p
```

Then execute:

```sql
CREATE DATABASE fleetdb;
USE fleetdb;
CREATE TABLE fleet_logs (
  fleet_id VARCHAR(20),
  timestamp DATETIME,
  speed INT,
  location VARCHAR(100)
);
```

### Step 5: Secrets Manager

- Name: `fleetdb/credentials`
- Store DB credentials, save the ARN.

### Step 6: Lambda Functions

- Create 2 functions: `FleetPOSTLambda`, `FleetGETLambda`
- **Runtime:** Node.js 18.x
- **Role:** FleetPOC-Lambda-Role
- **VPC:** FleetPOC-VPC
- **Subnets:** Private 1 & 2
- **Security Group:** Outbound TCP 3306 → RDS SG
- **Environment Variables:**
  - `DB_SECRET_ARN`
  - `DB_HOST`

Upload code for:

- `lambda_post.js`
- `lambda_get.js`

### Step 7: API Gateway Setup

- Create a REST API named `FleetPOC-API`
- Add Resources:
  - `/addlog` (POST) → `FleetPOSTLambda`
  - `/getlogs` (GET) → `FleetGETLambda`
- Deploy stage: `prod`
- **Copy your Invoke URL** under **API Gateway → Stages → prod → Invoke URL**.
  - Example: `https://abc123xyz.execute-api.ap-southeast-1.amazonaws.com/prod`
  - From this URL:
    - **API ID** = `abc123xyz`
    - **Region** = `ap-southeast-1`

### Step 8: EC2 Launch & Frontend Setup

- **AMI:** Ubuntu 22.04
- **Instance Type:** `t2.micro`
- **Subnet:** Public Subnet 1
- **Security Group:** Allow HTTP (80), SSH (22)

Install Apache + PHP:

```bash
sudo apt update
sudo apt install apache2 php curl -y
```

Upload files:

```bash
scp -i your-key.pem index.html getlogs.php ubuntu@<your-ec2-ip>:/home/ubuntu
sudo mv index.html getlogs.php /var/www/html/
```

Update `getlogs.php`:

```php
<?php
echo file_get_contents("https://<your-api-id>.execute-api.<your-region>.amazonaws.com/prod/getlogs");
?>
```

### Step 9: CloudWatch Logging

- View Lambda logs under CloudWatch → Log groups

### Step 10: Alarms & Cost Control (Optional)

- Create alarms for Lambda error count and RDS CPU
- Use AWS Cost Explorer to monitor usage

### Step 11: Testing

**POST Test with curl or Postman**

```bash
curl -X POST https://<your-api-id>.execute-api.<your-region>.amazonaws.com/prod/addlog \
-H 'Content-Type: application/json' \
-d '{"fleet_id":"fleet001","timestamp":"2025-07-01 12:00:00","speed":60,"location":"City Center"}'
```

- Replace `<your-api-id>` and `<your-region>` using your Invoke URL from API Gateway.
- **You do NOT need to open HTTPS (443) in EC2 inbound rules.**
- This command sends traffic from your local device to AWS API Gateway (not EC2).
- **EC2 security group has no effect on this test.**

**GET Test:**

```bash
curl https://<your-api-id>.execute-api.<your-region>.amazonaws.com/prod/getlogs
```

**Frontend Test:**

- In browser: `http://<your-ec2-ip>/index.html`

### README.md

```
# WSMB Fleet POC

## Architecture
Browser → EC2 → API Gateway → Lambda → RDS

## Deployment
Step-by-step AWS setup for real-time fleet logging

## Usage
Send logs (POST) → View logs (GET)

## Monitoring
CloudWatch Logs & Alarms
```

