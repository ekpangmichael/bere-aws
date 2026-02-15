# Building a Production-Style AWS VPC Infrastructure with Ansible and Application Load Balancer

## A Step-by-Step Guide to Setting Up a Secure, Load-Balanced Web Application on AWS

---

## Table of Contents

1. [Introduction](#introduction)
2. [Architecture Overview](#architecture-overview)
3. [Prerequisites](#prerequisites)
4. [Step 1 — Create the VPC](#step-1--create-the-vpc)
5. [Step 2 — Create Public and Private Subnets](#step-2--create-public-and-private-subnets)
6. [Step 3 — Create and Attach an Internet Gateway](#step-3--create-and-attach-an-internet-gateway)
7. [Step 4 — Create a NAT Gateway](#step-4--create-a-nat-gateway)
8. [Step 5 — Configure Route Tables](#step-5--configure-route-tables)
9. [Step 6 — Launch the Bastion Host](#step-6--launch-the-bastion-host)
10. [Step 7 — Launch the Private Web Servers](#step-7--launch-the-private-web-servers)
11. [Step 8 — Configure SSH Access via Bastion](#step-8--configure-ssh-access-via-bastion)
12. [Step 9 — Install Ansible and Write the Playbook](#step-9--install-ansible-and-write-the-playbook)
13. [Step 10 — Deploy Nginx with Ansible](#step-10--deploy-nginx-with-ansible)
14. [Step 11 — Create the Application Load Balancer](#step-11--create-the-application-load-balancer)
15. [Step 12 — Verify Everything Works](#step-12--verify-everything-works)
16. [Architecture Diagram](#architecture-diagram)
17. [Cleanup](#cleanup)
18. [Conclusion](#conclusion)

---

## Introduction

In this guide, we will build a production-style AWS infrastructure entirely from the command line using the AWS CLI and Ansible. By the end, you will have:

- A custom VPC with public and private subnets
- A bastion host for secure SSH access to private instances
- Two Nginx web servers in a private subnet (no public IPs)
- An Application Load Balancer (ALB) distributing traffic across both web servers
- A personalized HTML page served by Nginx, displaying unique instance details

This setup mirrors real-world architectures where web servers are never directly exposed to the internet, and all public traffic flows through a load balancer.

---

## Architecture Overview

```
Internet
    │
    ▼
┌──────────────────────────────────────────────────────────┐
│  VPC (10.0.0.0/16)                                       │
│                                                          │
│  ┌─────────────────────┐  ┌─────────────────────┐       │
│  │ Public Subnet 1      │  │ Public Subnet 2      │       │
│  │ 10.0.1.0/24 (1a)    │  │ 10.0.3.0/24 (1b)    │       │
│  │                      │  │                      │       │
│  │  ┌──────────────┐   │  │                      │       │
│  │  │ Bastion Host │   │  │                      │       │
│  │  └──────────────┘   │  │                      │       │
│  │  ┌──────────────┐   │  │                      │       │
│  │  │ NAT Gateway  │   │  │                      │       │
│  │  └──────────────┘   │  │                      │       │
│  └─────────────────────┘  └─────────────────────┘       │
│           │                                              │
│  ┌────────┤──── Application Load Balancer ──────┐       │
│  │        │     (spans both public subnets)      │       │
│  └────────┤─────────────────────────────────────┘       │
│           │                                              │
│  ┌─────────────────────────────────────────────┐        │
│  │ Private Subnet                               │        │
│  │ 10.0.2.0/24 (1b)                            │        │
│  │                                               │        │
│  │  ┌──────────────┐   ┌──────────────┐        │        │
│  │  │ Web Server 1 │   │ Web Server 2 │        │        │
│  │  │ 10.0.2.23    │   │ 10.0.2.168   │        │        │
│  │  └──────────────┘   └──────────────┘        │        │
│  └─────────────────────────────────────────────┘        │
│                                                          │
│  ┌──────────────┐                                       │
│  │ Internet GW  │                                       │
│  └──────────────┘                                       │
└──────────────────────────────────────────────────────────┘
```

---

## Prerequisites

Before you begin, ensure you have:

- **AWS CLI** installed and configured with a named profile (we use `trendstack`)
- **Ansible** installed (`brew install ansible` on macOS)
- **A terminal** (macOS Terminal, iTerm2, or similar)
- An **AWS account** with permissions to create VPCs, EC2 instances, and Load Balancers

Verify your setup:

```bash
aws --version
ansible --version
```

> **Note:** Throughout this guide, we use `--profile trendstack --region us-east-1`. Replace `trendstack` with your own AWS CLI profile name.

---

## Step 1 — Create the VPC

Create a VPC with a `/16` CIDR block, giving us 65,536 IP addresses to work with:

```bash
aws ec2 create-vpc \
  --cidr-block 10.0.0.0/16 \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=my-vpc}]' \
  --query 'Vpc.VpcId' \
  --output text \
  --profile trendstack --region us-east-1
```

Save the VPC ID returned (e.g., `vpc-0dde7fcfe898431d1`). You will need it for subsequent commands.

---

## Step 2 — Create Public and Private Subnets

We need three subnets:
- **Public Subnet 1** (us-east-1a) — for the bastion host and NAT gateway
- **Public Subnet 2** (us-east-1b) — required by the ALB (ALBs need at least 2 AZs)
- **Private Subnet** (us-east-1b) — for the web servers

```bash
# Public Subnet 1
aws ec2 create-subnet \
  --vpc-id <VPC_ID> \
  --cidr-block 10.0.1.0/24 \
  --availability-zone us-east-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=my-public-subnet}]' \
  --query 'Subnet.SubnetId' --output text \
  --profile trendstack --region us-east-1

# Public Subnet 2
aws ec2 create-subnet \
  --vpc-id <VPC_ID> \
  --cidr-block 10.0.3.0/24 \
  --availability-zone us-east-1b \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=my-public-subnet-2}]' \
  --query 'Subnet.SubnetId' --output text \
  --profile trendstack --region us-east-1

# Private Subnet
aws ec2 create-subnet \
  --vpc-id <VPC_ID> \
  --cidr-block 10.0.2.0/24 \
  --availability-zone us-east-1b \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=my-private-subnet}]' \
  --query 'Subnet.SubnetId' --output text \
  --profile trendstack --region us-east-1
```

Enable auto-assign public IPs on the first public subnet:

```bash
aws ec2 modify-subnet-attribute \
  --subnet-id <PUBLIC_SUBNET_1_ID> \
  --map-public-ip-on-launch \
  --profile trendstack --region us-east-1
```

---

## Step 3 — Create and Attach an Internet Gateway

The Internet Gateway allows resources in public subnets to communicate with the internet:

```bash
# Create the Internet Gateway
aws ec2 create-internet-gateway \
  --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=my-igw}]' \
  --query 'InternetGateway.InternetGatewayId' --output text \
  --profile trendstack --region us-east-1

# Attach it to the VPC
aws ec2 attach-internet-gateway \
  --internet-gateway-id <IGW_ID> \
  --vpc-id <VPC_ID> \
  --profile trendstack --region us-east-1
```

---

## Step 4 — Create a NAT Gateway

The NAT Gateway allows private subnet instances to reach the internet (for updates, package installs, etc.) without being directly accessible from the internet.

First, allocate an Elastic IP:

```bash
aws ec2 allocate-address \
  --domain vpc \
  --tag-specifications 'ResourceType=elastic-ip,Tags=[{Key=Name,Value=my-nat-eip}]' \
  --query 'AllocationId' --output text \
  --profile trendstack --region us-east-1
```

Then create the NAT Gateway in the **public** subnet:

```bash
aws ec2 create-nat-gateway \
  --subnet-id <PUBLIC_SUBNET_1_ID> \
  --allocation-id <EIP_ALLOCATION_ID> \
  --tag-specifications 'ResourceType=natgateway,Tags=[{Key=Name,Value=my-nat-gw}]' \
  --query 'NatGateway.NatGatewayId' --output text \
  --profile trendstack --region us-east-1
```

Wait for it to become available:

```bash
aws ec2 wait nat-gateway-available \
  --nat-gateway-ids <NAT_GW_ID> \
  --profile trendstack --region us-east-1
```

---

## Step 5 — Configure Route Tables

### Public Route Table

Get the VPC's main route table and add a route to the Internet Gateway:

```bash
# Get the main route table ID
aws ec2 describe-route-tables \
  --filters "Name=vpc-id,Values=<VPC_ID>" "Name=association.main,Values=true" \
  --query 'RouteTables[0].RouteTableId' --output text \
  --profile trendstack --region us-east-1

# Add default route to Internet Gateway
aws ec2 create-route \
  --route-table-id <PUBLIC_RT_ID> \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id <IGW_ID> \
  --profile trendstack --region us-east-1

# Associate both public subnets
aws ec2 associate-route-table \
  --route-table-id <PUBLIC_RT_ID> \
  --subnet-id <PUBLIC_SUBNET_1_ID> \
  --profile trendstack --region us-east-1

aws ec2 associate-route-table \
  --route-table-id <PUBLIC_RT_ID> \
  --subnet-id <PUBLIC_SUBNET_2_ID> \
  --profile trendstack --region us-east-1
```

### Private Route Table

Create a separate route table that routes internet traffic through the NAT Gateway:

```bash
# Create private route table
aws ec2 create-route-table \
  --vpc-id <VPC_ID> \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=my-private-rt}]' \
  --query 'RouteTable.RouteTableId' --output text \
  --profile trendstack --region us-east-1

# Add default route to NAT Gateway
aws ec2 create-route \
  --route-table-id <PRIVATE_RT_ID> \
  --destination-cidr-block 0.0.0.0/0 \
  --nat-gateway-id <NAT_GW_ID> \
  --profile trendstack --region us-east-1

# Associate with private subnet
aws ec2 associate-route-table \
  --route-table-id <PRIVATE_RT_ID> \
  --subnet-id <PRIVATE_SUBNET_ID> \
  --profile trendstack --region us-east-1
```

---

## Step 6 — Launch the Bastion Host

The bastion host is the only EC2 instance with a public IP. It serves as the SSH jump point to reach private instances.

### Create the Key Pair

```bash
aws ec2 create-key-pair \
  --key-name bastion-key \
  --query 'KeyMaterial' --output text \
  --profile trendstack --region us-east-1 > bastion-key.pem

chmod 400 bastion-key.pem
```

### Create the Bastion Security Group

```bash
aws ec2 create-security-group \
  --group-name bastion-sg \
  --description "Security group for bastion host - SSH access" \
  --vpc-id <VPC_ID> \
  --tag-specifications 'ResourceType=security-group,Tags=[{Key=Name,Value=bastion-sg}]' \
  --query 'GroupId' --output text \
  --profile trendstack --region us-east-1

aws ec2 authorize-security-group-ingress \
  --group-id <BASTION_SG_ID> \
  --protocol tcp --port 22 --cidr 0.0.0.0/0 \
  --profile trendstack --region us-east-1
```

> **Security Tip:** In production, replace `0.0.0.0/0` with your specific IP address (e.g., `203.0.113.50/32`).

### Launch the Instance

```bash
# Get the latest Amazon Linux 2023 AMI
AMI_ID=$(aws ec2 describe-images \
  --owners amazon \
  --filters "Name=name,Values=al2023-ami-2023.*-x86_64" "Name=state,Values=available" \
  --query 'Images | sort_by(@, &CreationDate) | [-1].ImageId' \
  --output text \
  --profile trendstack --region us-east-1)

# Launch the bastion host
aws ec2 run-instances \
  --image-id $AMI_ID \
  --instance-type t2.micro \
  --key-name bastion-key \
  --security-group-ids <BASTION_SG_ID> \
  --subnet-id <PUBLIC_SUBNET_1_ID> \
  --associate-public-ip-address \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=bastion-host}]' \
  --query 'Instances[0].{InstanceId:InstanceId,PublicIp:PublicIpAddress}' \
  --output table \
  --profile trendstack --region us-east-1
```

---

## Step 7 — Launch the Private Web Servers

### Create the Web Server Security Group

This security group only allows HTTP traffic from the ALB (we will create the ALB security group first in Step 11, but for now we allow SSH from the bastion for Ansible access):

```bash
aws ec2 create-security-group \
  --group-name webserver-sg \
  --description "Security group for web servers" \
  --vpc-id <VPC_ID> \
  --tag-specifications 'ResourceType=security-group,Tags=[{Key=Name,Value=webserver-sg}]' \
  --query 'GroupId' --output text \
  --profile trendstack --region us-east-1

# Allow SSH from bastion security group
aws ec2 authorize-security-group-ingress \
  --group-id <WEBSERVER_SG_ID> \
  --protocol tcp --port 22 \
  --source-group <BASTION_SG_ID> \
  --profile trendstack --region us-east-1
```

### Launch Two Web Servers

```bash
aws ec2 run-instances \
  --image-id $AMI_ID \
  --instance-type t2.micro \
  --count 2 \
  --key-name bastion-key \
  --security-group-ids <WEBSERVER_SG_ID> \
  --subnet-id <PRIVATE_SUBNET_ID> \
  --no-associate-public-ip-address \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=web-server}]' \
  --query 'Instances[*].{InstanceId:InstanceId,PrivateIp:PrivateIpAddress}' \
  --output table \
  --profile trendstack --region us-east-1
```

Tag them individually:

```bash
aws ec2 create-tags --resources <INSTANCE_1_ID> --tags Key=Name,Value=web-server-1 \
  --profile trendstack --region us-east-1

aws ec2 create-tags --resources <INSTANCE_2_ID> --tags Key=Name,Value=web-server-2 \
  --profile trendstack --region us-east-1
```

---

## Step 8 — Configure SSH Access via Bastion

Create an SSH config file (`~/.ssh/config`) to simplify connections through the bastion:

```
Host bastion
  HostName <BASTION_PUBLIC_IP>
  User ec2-user
  IdentityFile /path/to/bastion-key.pem
  ForwardAgent yes

Host web-server-1
  HostName <WEB_SERVER_1_PRIVATE_IP>
  User ec2-user
  IdentityFile /path/to/bastion-key.pem
  ProxyJump bastion

Host web-server-2
  HostName <WEB_SERVER_2_PRIVATE_IP>
  User ec2-user
  IdentityFile /path/to/bastion-key.pem
  ProxyJump bastion
```

Set proper permissions:

```bash
chmod 600 ~/.ssh/config
```

Add the key to your SSH agent:

```bash
eval "$(ssh-agent -s)"
ssh-add /path/to/bastion-key.pem
```

Test the connection:

```bash
ssh web-server-1
```

---

## Step 9 — Install Ansible and Write the Playbook

### Install Ansible

```bash
brew install ansible
```

### Create the Ansible Inventory

Create `inventory.ini`:

```ini
[webservers]
web-server-1 ansible_host=<WEB_SERVER_1_PRIVATE_IP>
web-server-2 ansible_host=<WEB_SERVER_2_PRIVATE_IP>

[webservers:vars]
ansible_user=ec2-user
ansible_ssh_private_key_file=/path/to/bastion-key.pem
ansible_ssh_common_args=-o ProxyJump=ec2-user@<BASTION_PUBLIC_IP> -o StrictHostKeyChecking=no
```

The key line here is `ansible_ssh_common_args` — it tells Ansible to route all SSH connections through the bastion host using ProxyJump.

### Write the Ansible Playbook

Create `deploy-nginx.yml`:

```yaml
---
- name: Install Nginx and deploy personal web page
  hosts: webservers
  become: yes
  tasks:
    - name: Install Nginx
      ansible.builtin.dnf:
        name: nginx
        state: present

    - name: Start and enable Nginx
      ansible.builtin.systemd:
        name: nginx
        state: started
        enabled: yes

    - name: Deploy custom HTML page
      ansible.builtin.copy:
        dest: /usr/share/nginx/html/index.html
        content: |
          <!DOCTYPE html>
          <html lang="en">
          <head>
            <meta charset="UTF-8">
            <meta name="viewport" content="width=device-width, initial-scale=1.0">
            <title>Peter Ekpang</title>
            <style>
              * { margin: 0; padding: 0; box-sizing: border-box; }
              body {
                font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
                background: linear-gradient(135deg, #0f2027, #203a43, #2c5364);
                color: #fff;
                min-height: 100vh;
                display: flex;
                justify-content: center;
                align-items: center;
              }
              .container {
                background: rgba(255, 255, 255, 0.05);
                backdrop-filter: blur(10px);
                border: 1px solid rgba(255, 255, 255, 0.1);
                border-radius: 20px;
                padding: 50px;
                max-width: 600px;
                width: 90%;
                text-align: center;
              }
              h1 { font-size: 2.5rem; margin-bottom: 10px; }
              .hostname {
                background: rgba(255, 255, 255, 0.1);
                border-radius: 8px;
                padding: 12px 20px;
                margin: 20px 0;
                font-family: monospace;
                font-size: 1rem;
              }
              .hostname span { color: #4fc3f7; }
              .about {
                background: rgba(255, 255, 255, 0.08);
                border-radius: 12px;
                padding: 25px;
                margin-top: 20px;
                text-align: left;
                line-height: 1.8;
              }
              .about h2 {
                margin-bottom: 12px;
                font-size: 1.3rem;
                color: #4fc3f7;
              }
              .about p { margin-bottom: 14px; }
              .about strong { color: #4fc3f7; }
            </style>
          </head>
          <body>
            <div class="container">
              <h1>Ekpang Peter Bere</h1>
              <div class="hostname">
                <p>Hostname: <span>{{ ansible_hostname }}</span></p>
                <p>Private IP: <span>{{ ansible_default_ipv4.address }}</span></p>
              </div>
              <div class="about">
                <h2>About Me</h2>
                <p>My name is Ekpang Peter Bere, and I'm a proud native of Boki Local Government Area, <strong>Cross River State</strong>, <strong>Nigeria</strong>. Coming from a community rich in culture, tradition, and strong values has shaped my discipline, character, and determination to grow—personally and professionally.</p>
                <p>I'm currently enrolled in a diploma program at <strong>AltSchool Africa</strong>, studying Cloud Engineering. Through hands-on learning, I'm building skills in cloud infrastructure, Linux, networking, DevOps practices, and automation—tools and systems that power modern digital services at scale.</p>
                <p>I'm passionate about designing reliable, scalable, and secure infrastructure that solves real-world problems. I'm committed to continuous learning and to contributing meaningfully to the global technology ecosystem. Beyond personal growth, I aim to create impact, inspire others from my community, and support the expansion of tech opportunities in Nigeria and beyond.</p>
              </div>
            </div>
          </body>
          </html>
        mode: '0644'
      notify: Restart Nginx

  handlers:
    - name: Restart Nginx
      ansible.builtin.systemd:
        name: nginx
        state: restarted
```

**Key details about this playbook:**

- `{{ ansible_hostname }}` and `{{ ansible_default_ipv4.address }}` are Ansible facts gathered automatically from each host. This ensures each server displays its own unique hostname and IP address.
- `become: yes` runs tasks with sudo privileges (required for installing packages and managing services).
- The `notify` / `handlers` pattern ensures Nginx is restarted only when the HTML file changes.

---

## Step 10 — Deploy Nginx with Ansible

Run the playbook:

```bash
ansible-playbook -i inventory.ini deploy-nginx.yml
```

Expected output:

```
PLAY RECAP *****************************************************************
web-server-1  : ok=5  changed=4  unreachable=0  failed=0  skipped=0
web-server-2  : ok=5  changed=4  unreachable=0  failed=0  skipped=0
```

Verify from the bastion:

```bash
ssh web-server-1 'curl -s localhost' | grep "Private IP"
ssh web-server-2 'curl -s localhost' | grep "Private IP"
```

You should see different IPs for each server.

---

## Step 11 — Create the Application Load Balancer

### Create the ALB Security Group

```bash
aws ec2 create-security-group \
  --group-name alb-sg \
  --description "Security group for ALB - HTTP from internet" \
  --vpc-id <VPC_ID> \
  --tag-specifications 'ResourceType=security-group,Tags=[{Key=Name,Value=alb-sg}]' \
  --query 'GroupId' --output text \
  --profile trendstack --region us-east-1

# Allow HTTP from anywhere
aws ec2 authorize-security-group-ingress \
  --group-id <ALB_SG_ID> \
  --protocol tcp --port 80 --cidr 0.0.0.0/0 \
  --profile trendstack --region us-east-1
```

### Update Web Server Security Group

Allow HTTP only from the ALB security group (not from the entire VPC or internet):

```bash
aws ec2 authorize-security-group-ingress \
  --group-id <WEBSERVER_SG_ID> \
  --protocol tcp --port 80 \
  --source-group <ALB_SG_ID> \
  --profile trendstack --region us-east-1
```

### Create the Target Group

```bash
aws elbv2 create-target-group \
  --name web-servers-tg \
  --protocol HTTP --port 80 \
  --vpc-id <VPC_ID> \
  --health-check-path / \
  --health-check-interval-seconds 30 \
  --healthy-threshold-count 2 \
  --target-type instance \
  --query 'TargetGroups[0].TargetGroupArn' --output text \
  --profile trendstack --region us-east-1
```

### Register the Web Servers

```bash
aws elbv2 register-targets \
  --target-group-arn <TARGET_GROUP_ARN> \
  --targets Id=<INSTANCE_1_ID> Id=<INSTANCE_2_ID> \
  --profile trendstack --region us-east-1
```

### Create the ALB

The ALB must span at least two Availability Zones, which is why we created two public subnets:

```bash
aws elbv2 create-load-balancer \
  --name web-alb \
  --subnets <PUBLIC_SUBNET_1_ID> <PUBLIC_SUBNET_2_ID> \
  --security-groups <ALB_SG_ID> \
  --scheme internet-facing \
  --type application \
  --query 'LoadBalancers[0].{ARN:LoadBalancerArn,DNS:DNSName}' \
  --output table \
  --profile trendstack --region us-east-1
```

### Create the Listener

```bash
aws elbv2 create-listener \
  --load-balancer-arn <ALB_ARN> \
  --protocol HTTP --port 80 \
  --default-actions Type=forward,TargetGroupArn=<TARGET_GROUP_ARN> \
  --profile trendstack --region us-east-1
```

### Wait for ALB and Check Target Health

```bash
aws elbv2 wait load-balancer-available \
  --load-balancer-arns <ALB_ARN> \
  --profile trendstack --region us-east-1

aws elbv2 describe-target-health \
  --target-group-arn <TARGET_GROUP_ARN> \
  --query 'TargetHealthDescriptions[*].{Target:Target.Id,State:TargetHealth.State}' \
  --output table \
  --profile trendstack --region us-east-1
```

Both targets should show `healthy`.

---

## Step 12 — Verify Everything Works

### Test Traffic Distribution

```bash
for i in 1 2 3 4 5 6; do
  curl -s <ALB_DNS_NAME> | grep "Private IP" | sed 's/.*<span>//;s/<\/span>.*//'
done
```

Expected output (IPs alternate between the two servers):

```
10.0.2.168
10.0.2.23
10.0.2.168
10.0.2.168
10.0.2.23
10.0.2.23
```

### Test in Your Browser

Open your browser and navigate to:

```
http://<ALB_DNS_NAME>
```

Refresh the page multiple times. You should see the **hostname** and **Private IP** change between the two instances, confirming the ALB is distributing traffic.

### Confirm Web Servers Are Not Directly Accessible

The web servers have no public IP addresses, and their security group only allows HTTP from the ALB. There is no way to access them directly from the internet — all traffic must flow through the ALB.

---

## Architecture Diagram

| Component | Resource | Details |
|---|---|---|
| VPC | `vpc-0dde7fcfe898431d1` | CIDR: 10.0.0.0/16 |
| Public Subnet 1 | `subnet-037b2a83d6628c87d` | 10.0.1.0/24, us-east-1a |
| Public Subnet 2 | `subnet-0d10655691e1843e8` | 10.0.3.0/24, us-east-1b |
| Private Subnet | `subnet-0e091fe248f2e8ff9` | 10.0.2.0/24, us-east-1b |
| Internet Gateway | `igw-0abb6d04939d575a7` | Attached to VPC |
| NAT Gateway | `nat-0e651fbfee64e4fbb` | In Public Subnet 1 |
| Bastion Host | `i-05f0c492069efd4d6` | Public IP: 54.158.55.16 |
| Web Server 1 | `i-060e13dd7609845bd` | Private IP: 10.0.2.23 |
| Web Server 2 | `i-0f604f62dc2498d22` | Private IP: 10.0.2.168 |
| ALB | `web-alb` | DNS: web-alb-1604143918.us-east-1.elb.amazonaws.com |

---

## Cleanup

To avoid ongoing charges, tear down the resources in reverse order:

```bash
# Delete ALB and Target Group
aws elbv2 delete-load-balancer --load-balancer-arn <ALB_ARN> --profile trendstack --region us-east-1
aws elbv2 wait load-balancers-deleted --load-balancer-arns <ALB_ARN> --profile trendstack --region us-east-1
aws elbv2 delete-target-group --target-group-arn <TARGET_GROUP_ARN> --profile trendstack --region us-east-1

# Terminate EC2 Instances
aws ec2 terminate-instances --instance-ids <BASTION_ID> <WEB1_ID> <WEB2_ID> --profile trendstack --region us-east-1
aws ec2 wait instance-terminated --instance-ids <BASTION_ID> <WEB1_ID> <WEB2_ID> --profile trendstack --region us-east-1

# Delete NAT Gateway and release Elastic IP
aws ec2 delete-nat-gateway --nat-gateway-id <NAT_GW_ID> --profile trendstack --region us-east-1
# Wait a few minutes for NAT Gateway to delete
aws ec2 release-address --allocation-id <EIP_ALLOCATION_ID> --profile trendstack --region us-east-1

# Delete Security Groups
aws ec2 delete-security-group --group-id <ALB_SG_ID> --profile trendstack --region us-east-1
aws ec2 delete-security-group --group-id <WEBSERVER_SG_ID> --profile trendstack --region us-east-1
aws ec2 delete-security-group --group-id <BASTION_SG_ID> --profile trendstack --region us-east-1

# Delete Subnets
aws ec2 delete-subnet --subnet-id <PUBLIC_SUBNET_1_ID> --profile trendstack --region us-east-1
aws ec2 delete-subnet --subnet-id <PUBLIC_SUBNET_2_ID> --profile trendstack --region us-east-1
aws ec2 delete-subnet --subnet-id <PRIVATE_SUBNET_ID> --profile trendstack --region us-east-1

# Detach and delete Internet Gateway
aws ec2 detach-internet-gateway --internet-gateway-id <IGW_ID> --vpc-id <VPC_ID> --profile trendstack --region us-east-1
aws ec2 delete-internet-gateway --internet-gateway-id <IGW_ID> --profile trendstack --region us-east-1

# Delete Route Tables (non-main only)
aws ec2 delete-route-table --route-table-id <PRIVATE_RT_ID> --profile trendstack --region us-east-1

# Delete VPC
aws ec2 delete-vpc --vpc-id <VPC_ID> --profile trendstack --region us-east-1

# Delete Key Pair
aws ec2 delete-key-pair --key-name bastion-key --profile trendstack --region us-east-1
```

---

## Conclusion

In this guide, we built a complete, production-style AWS infrastructure from scratch using only the AWS CLI and Ansible:

- **Network isolation** — Web servers sit in a private subnet with no public IPs
- **Secure access** — SSH access is only possible through the bastion host via ProxyJump
- **Automated configuration** — Ansible handles Nginx installation and deployment across multiple servers simultaneously
- **High availability** — The ALB distributes traffic across both instances, and each server displays unique identifying information
- **Security in depth** — Security groups are scoped to allow only necessary traffic (ALB → web servers, bastion → web servers)

This architecture is the foundation for many production workloads on AWS. From here, you could extend it with HTTPS (ACM + ALB HTTPS listener), auto-scaling groups, RDS databases in additional private subnets, or CI/CD pipelines for automated deployments.

---

*Author: Ekpang Peter Bere*
*Cloud Engineering Student, AltSchool Africa*
