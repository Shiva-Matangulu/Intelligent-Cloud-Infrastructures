# Testing & Cleanup

I used the following tests to verify that the infrastructure was working as expected. I also included the cleanup steps I followed to remove the AWS resources after testing and avoid unnecessary charges.

---

## 1. Testing

I performed five main tests covering recovery, scaling, security, auditing, and monitoring.

### Test 1 — ECS Self-Healing

**Goal:** Check whether ECS automatically replaces a failed task.

1. Go to **ECS → Clusters → `intelligent-cloud-infra-cluster` → Tasks**.
2. Select one running task and click **Stop**.
3. Check the ECS service and target group.
4. The stopped task should move to `STOPPED`.
5. ECS should automatically start a new task to maintain the desired count.
6. Wait for the new task to become healthy in the ALB target group.

**Expected result:** A replacement task starts automatically and the application remains available through the ALB.

---

### Test 2 — Auto Scaling

**Goal:** Check whether ECS adds tasks when CPU usage increases.

I generated traffic against the ALB using ApacheBench:

```bash
ab -n 50000 -c 50 http://<ALB-DNS-NAME>/
```

Then I monitored:

**CloudWatch → ECS → `ECSServiceAverageCPUUtilization`**

**Expected result:**

* CPU usage increases during the test.
* The scaling policy responds when the CPU target is exceeded.
* The number of tasks can increase from **2 up to 4**.
* After the load decreases, the service can scale back down.

---

### Test 3 — Private Network Access

**Goal:** Verify that ECS containers are not directly exposed to the internet.

I checked the ECS task networking configuration and security groups.

I verified that:

* ECS tasks have **no public IP**.
* Public IP assignment is disabled.
* The application security group allows traffic only from the ALB security group.
* The containers cannot be accessed directly from the internet.

**Expected result:** Application traffic must go through the ALB before reaching the ECS tasks.

---

### Test 4 — CloudTrail & AWS Config

**Goal:** Verify that AWS activity is being recorded and compliance rules are working.

For CloudTrail, I checked:

**CloudTrail → Event history**

I searched for events such as:

* `StopTask`
* `UpdateService`

I also checked:

**AWS Config → Rules**

and verified the configured rules and their compliance status.

**Expected result:** AWS actions are visible in CloudTrail and the configured resources are evaluated by AWS Config.

---

### Test 5 — CloudWatch Dashboard

**Goal:** Check whether the main application metrics are visible from one place.

I opened:

**CloudWatch → Dashboards → `intelligent-cloud-infra-dashboard`**

I checked the following metrics:

* ECS CPU utilization
* ECS memory utilization
* ALB target health
* Request count
* Target response time

**Expected result:** The dashboard displays current metrics and reflects changes during testing.

---

# 2. Cleanup

After completing the tests, I removed the resources I created for this project.

Some AWS resources, especially **NAT Gateways, Elastic IPs, and Application Load Balancers**, can continue to generate charges while they are running. So I followed a dependency-based cleanup order.

## Cleanup Checklist

| Step | Resource             | Action                                                      |
| ---- | -------------------- | ----------------------------------------------------------- |
| 1    | CloudWatch Alarms    | Delete the project alarms                                   |
| 2    | CloudWatch Dashboard | Delete `intelligent-cloud-infra-dashboard`                  |
| 3    | ECS Service          | Set desired tasks to `0` and delete the service             |
| 4    | ECS Cluster          | Delete `intelligent-cloud-infra-cluster`                    |
| 5    | ECR Repository       | Delete `intelligent-cloud-infra-app` and its images         |
| 6    | S3 Buckets           | Empty and delete project log buckets                        |
| 7    | SNS Subscription     | Delete the email subscription                               |
| 8    | SNS Topic            | Delete `intelligent-cloud-infra-alerts`                     |
| 9    | AWS Config           | Stop recording and remove configured rules                  |
| 10   | CloudTrail           | Delete `intelligent-cloud-infra-trail`                      |
| 11   | Load Balancer        | Delete `intelligent-cloud-infra-alb`                        |
| 12   | Target Group         | Delete `intelligent-cloud-infra-tg`                         |
| 13   | NAT Gateway          | Delete the project NAT Gateway and wait until it is deleted |
| 14   | Elastic IP           | Release the associated Elastic IP                           |
| 15   | Security Groups      | Delete the application and ALB security groups              |
| 16   | VPC                  | Delete `intelligent-cloud-infra-vpc`                        |

> **Important:** I checked that the NAT Gateway was fully deleted before releasing its Elastic IP.

---

## 3. Final Verification

After cleanup, I checked the AWS Console to make sure the resources created for this project were removed.

I specifically checked:

* ECS services and clusters
* Load Balancers
* NAT Gateways
* Elastic IPs
* ECR repositories
* CloudWatch dashboards and alarms
* SNS topics
* CloudTrail trails
* AWS Config resources
* S3 log buckets
* VPC resources

This final check helps make sure that no unnecessary resources are left running.

---

## Related Documentation

* [Architecture](architecture.md) — How I designed the infrastructure
* [Deployment Guide](deployment-guide.md) — Steps I followed to build it
* [README](../README.md) — Project overview
* [Full Visual Guide](../shiva_final_intelligent_cloud_infrastructure%20-%20FINAL.pdf) — Complete AWS Console walkthrough
