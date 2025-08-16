# Deploy Java Application on AWS 3‑Tier Architecture – Full Guide
**Lift and Shift Application Workload on AWS Cloud**

Welcome! This project walks through hosting and running a Java application on **AWS** using a **lift‑and‑shift strategy** suitable for production.

---

## Project Overview
You will provision a classic three‑tier stack (Load Balancer → App → Backend) and deploy a Java web app on EC2.

After completing this guide, you’ll understand how to run application workloads on AWS using a lift‑and‑shift approach and how to secure and automate the core building blocks.

---

## AWS Services Used
- **EC2 instances**: Tomcat (app tier), RabbitMQ, Memcached, MySQL (backend tier)
- **Elastic Load Balancer (ALB)**: Fronts the app tier
- **Auto Scaling**: Scales Tomcat instances
- **S3 / EFS**: Artifact or shared storage
- **Route 53**: Private DNS
- **ACM (AWS Certificate Manager)**: HTTPS certificates
- **IAM & EBS**: Supporting services

---

## Project Objectives
- Flexible, scalable infrastructure
- **Pay‑as‑you‑go** cost model
- Modernize via managed AWS services
- Emphasize **automation & IaC**

---

## Architectural Design
<p align="center">
  <img src="images/architecture.jpg" alt="Architecture Diagram" width="520"/>
</p>

- Users access a public URL (managed by your registrar/DNS provider, e.g., Namecheap).
- **ACM** issues TLS certificates; HTTPS terminates at the **Application Load Balancer**.
- The ALB routes requests to **Tomcat** instances in an **Auto Scaling Group** across subnets/AZs.
- Backend services (MySQL, Memcached, RabbitMQ) are reachable via **Route 53 private hosted zone** names.
- Security groups are separated for **Load Balancer**, **Application**, and **Backend** tiers.

---

## AWS Resources in Use
- **ACM** for SSL/TLS
- **EC2**: Tomcat, Memcached, RabbitMQ, MySQL
- **Route 53**: Private DNS
- **S3**: Artifact storage (WARs, configs, etc.)

---

## Execution Flow
1. Log into AWS.
2. Create **key pair(s)** for EC2 SSH access.
3. Create **security groups** for Load Balancer, Tomcat, and Backend.
4. Launch EC2 instances with **user data**.
5. Create private DNS records in Route 53.
6. Build application locally and upload artifacts to S3.
7. Deploy artifacts from S3 to Tomcat instances.
8. Create an **Application Load Balancer** with HTTPS (ACM certs).
9. Map the ALB DNS name in your public DNS (registrar).
10. Create an **Auto Scaling Group** for Tomcat.

---

# Prerequisites
- AWS CLI configured with appropriate permissions
- VPC and subnets created
- Your workstation’s public IP (for SSH)

For convenience, export a few variables you’ll reuse:

```bash
export AWS_REGION=<your-aws-region>
export VPC_ID=<your-vpc-id>
export MY_IPv4_CIDR=<your-public-ipv4>/32   # e.g., 203.0.113.45/32
```

> Replace placeholders like `<your-vpc-id>` with your actual values before running commands.

---

## 1) Create the Security Group for the **Load Balancer**
Allows HTTP/HTTPS from anywhere over IPv4 and IPv6.

```bash
# Create
SG_LB_ID=$(aws ec2 create-security-group \
  --group-name JavaApp-LoadBalancer-SG \
  --description "Allow HTTP and HTTPS from IPv4 and IPv6" \
  --vpc-id "$VPC_ID" \
  --query 'GroupId' --output text)

# (Optional) Tag it
aws ec2 create-tags --resources "$SG_LB_ID" \
  --tags Key=Name,Value=JavaApp-LoadBalancer-SG

# Ingress: 80/443 from everywhere (IPv4 + IPv6)
aws ec2 authorize-security-group-ingress \
  --group-id "$SG_LB_ID" \
  --ip-permissions '[
    {"IpProtocol":"tcp","FromPort":80,"ToPort":80,
     "IpRanges":[{"CidrIp":"0.0.0.0/0"}],
     "Ipv6Ranges":[{"CidrIpv6":"::/0"}]},
    {"IpProtocol":"tcp","FromPort":443,"ToPort":443,
     "IpRanges":[{"CidrIp":"0.0.0.0/0"}],
     "Ipv6Ranges":[{"CidrIpv6":"::/0"}]}
  ]'
```

---

## 2) Create the Security Group for the **Tomcat (App Tier)**
Allows SSH from your IP, app ports from the ALB SG (preferred), and optional HTTP/8080 for testing.

```bash
# Create
SG_APP_ID=$(aws ec2 create-security-group \
  --group-name JavaApp-Tomcat-SG \
  --description "Tomcat app tier" \
  --vpc-id "$VPC_ID" \
  --query 'GroupId' --output text)

aws ec2 create-tags --resources "$SG_APP_ID" \
  --tags Key=Name,Value=JavaApp-Tomcat-SG

# Ingress rules
# - SSH from your workstation
# - 80/8080 from ALB SG (secure) – swap SG_LB_ID below after creation
# - (Optional) 8080 from 0.0.0.0/0 for quick tests – remove in production

aws ec2 authorize-security-group-ingress \
  --group-id "$SG_APP_ID" \
  --ip-permissions "[
    {\"IpProtocol\":\"tcp\",\"FromPort\":22,\"ToPort\":22,
     \"IpRanges\":[{\"CidrIp\":\"$MY_IPv4_CIDR\"}]},
    {\"IpProtocol\":\"tcp\",\"FromPort\":80,\"ToPort\":80,
     \"UserIdGroupPairs\":[{\"GroupId\":\"$SG_LB_ID\"}]},
    {\"IpProtocol\":\"tcp\",\"FromPort\":8080,\"ToPort\":8080,
     \"UserIdGroupPairs\":[{\"GroupId\":\"$SG_LB_ID\"}]}
  ]"

# (Optional – testing only)
# aws ec2 authorize-security-group-ingress \
#   --group-id "$SG_APP_ID" --protocol tcp --port 8080 --cidr 0.0.0.0/0
```

---

## 3) Create the Security Group for the **Backend Tier**
Allows specific service ports from the app SG, SSH from your IP, and self‑reference for internal clustering/sync.

```bash
# Create
SG_BE_ID=$(aws ec2 create-security-group \
  --group-name JavaApp-Backend-SG \
  --description "Backend: MySQL, Memcached, RabbitMQ" \
  --vpc-id "$VPC_ID" \
  --query 'GroupId' --output text)

aws ec2 create-tags --resources "$SG_BE_ID" \
  --tags Key=Name,Value=JavaApp-Backend-SG

# Ingress: MySQL(3306), Memcached(11211), RabbitMQ(5672) from APP SG; SSH from your IP; self-ref for node-to-node traffic
aws ec2 authorize-security-group-ingress \
  --group-id "$SG_BE_ID" \
  --ip-permissions "[
    {\"IpProtocol\":\"tcp\",\"FromPort\":3306,\"ToPort\":3306,
     \"UserIdGroupPairs\":[{\"GroupId\":\"$SG_APP_ID\"}]},
    {\"IpProtocol\":\"tcp\",\"FromPort\":11211,\"ToPort\":11211,
     \"UserIdGroupPairs\":[{\"GroupId\":\"$SG_APP_ID\"}]},
    {\"IpProtocol\":\"tcp\",\"FromPort\":5672,\"ToPort\":5672,
     \"UserIdGroupPairs\":[{\"GroupId\":\"$SG_APP_ID\"}]},
    {\"IpProtocol\":\"tcp\",\"FromPort\":22,\"ToPort\":22,
     \"IpRanges\":[{\"CidrIp\":\"$MY_IPv4_CIDR\"}]},
    {\"IpProtocol\":\"-1\",\"UserIdGroupPairs\":[{\"GroupId\":\"$SG_BE_ID\"}]}
  ]"
```

> Note: Granting **All traffic** from the app SG to the backend SG is not recommended when you already open the specific ports. Keep access least‑privileged.

---

## 4) Create an **EC2 Key Pair**
Use this key to SSH into instances.

```bash
aws ec2 create-key-pair \
  --key-name JavaAppKey \
  --query 'KeyMaterial' \
  --output text > JavaAppKey.pem

chmod 400 JavaAppKey.pem
```

> Save the `.pem` securely. You can’t re‑download it later.

---

## Key Takeaways
- Demonstrates migrating a multi‑tier web application to AWS with **segregated security groups**.
- Uses EC2, ALB, Auto Scaling, S3, Route 53, and ACM.
- Emphasizes security (least privilege), scalability, and automation.

---

## Next Steps (Optional)
- Create target groups and an ALB listener (HTTP→HTTPS redirect, 443 with ACM cert).
- Create a Launch Template and Auto Scaling Group for Tomcat, attaching `JavaApp-Tomcat-SG`.
- Add Route 53 records (public: ALB; private: backend service names).
- Store artifacts in S3 and automate deployments (e.g., via CodeDeploy, Ansible, or user data).
