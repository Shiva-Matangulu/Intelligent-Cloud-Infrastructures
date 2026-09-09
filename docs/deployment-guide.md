# Deployment Guide

I built this infrastructure step by step using AWS-native services in the **`ap-south-1` (Mumbai)** region.

I have also attached the detailed pdf , you can follow from it.

This guide contains the main resources and configurations I used to deploy the application.

## 1. Network Setup

I created a custom VPC with the following configuration:

| Configuration      | Value         |
| ------------------ | ------------- |
| VPC CIDR           | `10.0.0.0/16` |
| Availability Zones | 2             |
| Public Subnets     | 2             |
| Private Subnets    | 2             |

### Subnets

| Subnet           | CIDR           | Purpose           |
| ---------------- | -------------- | ----------------- |
| Public Subnet 1  | `10.0.1.0/24`  | ALB / NAT Gateway |
| Public Subnet 2  | `10.0.2.0/24`  | ALB / NAT Gateway |
| Private Subnet 1 | `10.0.11.0/24` | ECS Tasks         |
| Private Subnet 2 | `10.0.12.0/24` | ECS Tasks         |

I configured the Internet Gateway for the public subnets and NAT Gateway connectivity for the private subnets.

---

## 2. Security Groups

I created two security groups to control application traffic.

### ALB Security Group

* HTTP: `80` from `0.0.0.0/0`
* HTTPS: `443` from `0.0.0.0/0`

### Application Security Group

* HTTP: `80`
* Source: **ALB Security Group only**

The ECS tasks do not have public IP addresses, so application traffic must pass through the load balancer.

---

## 3. Container Registry

I created a private Amazon ECR repository:

`intelligent-cloud-infra-app`

I used this repository to store the Docker image used by the ECS tasks.

---

## 4. Application Load Balancer

I created an internet-facing Application Load Balancer across the two public subnets.

### Target Group

| Setting           | Value |
| ----------------- | ----- |
| Target Type       | IP    |
| Protocol          | HTTP  |
| Port              | 80    |
| Health Check Path | `/`   |

The ALB performs health checks and sends traffic only to healthy ECS tasks.

---

## 5. ECS & Fargate

I created an ECS cluster:

`intelligent-cloud-infra-cluster`

I then created a Fargate task definition with:

| Setting        | Value           |
| -------------- | --------------- |
| Launch Type    | Fargate         |
| CPU            | 0.5 vCPU        |
| Memory         | 1 GB            |
| Container Port | 80              |
| Network        | Private Subnets |
| Public IP      | Disabled        |

### ECS Service

I created the service:

`intelligent-cloud-infra-service`

The initial configuration was:

* Desired tasks: **2**
* Minimum tasks: **1**
* Maximum tasks: **4**
* Tasks deployed across two private subnets

ECS manages the task lifecycle and automatically replaces stopped or unhealthy tasks.

---

## 6. Auto Scaling

I configured Application Auto Scaling for the ECS service.

| Setting        | Value                   |
| -------------- | ----------------------- |
| Scaling Metric | ECS Service Average CPU |
| Target         | 50%                     |
| Minimum Tasks  | 1                       |
| Maximum Tasks  | 4                       |
| Desired Tasks  | 2                       |

This allows the service to increase or decrease the number of running tasks based on CPU utilization.

---

## 7. CloudWatch Monitoring

I created a CloudWatch dashboard:

`intelligent-cloud-infra-dashboard`

The dashboard monitors:

* ECS CPU utilization
* ECS memory utilization
* ALB target health
* Request count
* Target response time

I also configured alarms for:

* Unhealthy targets
* High CPU utilization
* High memory utilization

---

## 8. SNS Notifications

I created an SNS topic:

`intelligent-cloud-infra-alerts`

CloudWatch alarms are connected to this topic so that I can receive notifications when an important condition is triggered.

---

## 9. AWS Config

I enabled AWS Config and added the required compliance rules.

The rules I used were:

* `ec2-instance-no-public-ip`
* `restricted-ssh`
* `s3-bucket-public-read-prohibited`

These rules help me monitor the security configuration of the AWS environment.

---

## 10. CloudTrail & S3

I created a CloudTrail trail:

`intelligent-cloud-infra-trail`

CloudTrail records AWS API activity and delivers the logs to an S3 bucket for centralized storage.

---

## 11. Deployment Flow

The deployment was completed in this order:

1. Created the VPC and subnets.
2. Configured Internet Gateway, NAT Gateway, and route tables.
3. Created the ALB and application security groups.
4. Created the ECR repository.
5. Built and pushed the application container image.
6. Created the ECS Fargate cluster.
7. Created the task definition.
8. Created the ALB target group and listener.
9. Created the ECS service in the private subnets.
10. Configured automatic scaling.
11. Added CloudWatch monitoring and alarms.
12. Configured SNS notifications.
13. Enabled AWS Config compliance checks.
14. Configured CloudTrail and S3 logging.
15. Tested recovery, scaling, security, and monitoring.

## 12. Verification

After deployment, I verified that:

* The application was accessible through the ALB.
* ECS was running the desired number of tasks.
* ALB health checks were passing.
* Failed tasks were automatically replaced.
* Auto Scaling responded to increased CPU usage.
* ECS tasks had no public IP addresses.
* CloudWatch metrics were being collected.
* SNS notifications were working.
* CloudTrail events were being recorded.
* AWS Config rules were evaluating the environment.

For the testing procedures and cleanup steps, see [Testing & Cleanup](testing-and-cleanup.md).
