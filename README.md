# ⚖️ AWS Auto Scaling + Load Balancing — High Availability Architecture
  
**Lab:** Lab 4 — Configure High Availability for Your Application  
**Console:** AWS Management Console — Region: US East (N. Virginia) `us-east-1`

---

## 📋 Objective

Transform a single EC2 instance running the Employee Directory App into a fully high-availability, auto-scaling architecture — one that automatically distributes traffic across multiple instances, monitors their health, replaces unhealthy ones without human intervention, and scales the number of servers up or down based on demand. This is accomplished by wiring together four AWS services: an Application Load Balancer, a Target Group, an EC2 Launch Template, and an Auto Scaling Group — and confirmed by observing the app running through the load balancer URL and by watching the ASG automatically launch replacement instances.

---

## 🛠️ Tools Used

| Tool | Purpose |
|------|---------|
| **AWS Management Console** | Browser-based interface for all AWS service configuration |
| **Employee Directory App** | Live web app on EC2 used to verify configuration, test connectivity, and observe which instance is serving traffic |
| **Elastic Load Balancing (ALB)** | Application Load Balancer to distribute HTTP traffic across multiple EC2 instances |
| **EC2 Target Groups** | Defines which instances receive traffic and monitors their health for the ALB |
| **EC2 Launch Template** | Reusable, versioned instance configuration (AMI, type, security group, User Data) used by the ASG |
| **EC2 Auto Scaling Groups** | Automatically launches/terminates instances to maintain desired capacity across multiple AZs |
| **Amazon SNS** | Simple Notification Service — email alerts when the ASG launches or terminates instances |
| **Amazon EC2 Instances** | Observing multiple running instances created and managed by the ASG |

---

## ✅ Skills Learned

- Reading the **Employee Directory App Configuration page** to verify EC2 instance identity, AZ placement, S3 and DynamoDB connectivity status
- Understanding the **three AWS load balancer types**: Application (ALB), Network (NLB), and Gateway (GWLB) — and when to use each
- Creating a **Target Group** with target type `Instances`, protocol HTTP:80, and naming it `lab-app-target-group`
- Configuring **Target Group health check settings**: health check port (Traffic port), healthy threshold (2), unhealthy threshold (5), and timeout (20 seconds)
- Understanding what target health states mean: **Healthy**, **Unhealthy**, **Unused**, **Initial**, **Draining**
- Creating an **Application Load Balancer** (`Web-Application-ALB`) as internet-facing with IPv4, spanning two AZs
- Configuring an **ALB Listener** on HTTP:80 with a default action to forward traffic to `lab-app-target-group` at 100% weight
- Understanding ALB **routing actions**: Forward to target groups, Redirect to URL, Return fixed response
- Creating an **EC2 Launch Template** (`lab-app-launch-template`) with Auto Scaling guidance enabled, versioned and reusable
- Naming and configuring an **Auto Scaling Group** (`app-asg`) using the launch template
- Selecting **two public subnets** across `us-east-1a` and `us-east-1b` for multi-AZ redundancy with **Balanced best effort** distribution
- Attaching the ASG to the existing load balancer via the **target group** (`lab-app-target-group | HTTP`)
- Subscribing to **SNS notifications** for ASG scaling events (launch/terminate) and confirming the email subscription
- Observing **three EC2 instances** in the console — one terminated (original), two running (ASG-launched) — proving the ASG is managing capacity automatically
- Comparing the app **configuration page through the ALB URL** vs. the direct EC2 URL to verify the load balancer is serving traffic
- Identifying a **broken instance** by its missing DynamoDB/S3 configuration (red ✗) compared to a healthy one (green ✓) — a real-world troubleshooting scenario
- Understanding that when an instance is served through the ALB URL, the **EC2 Instance ID shown in the app changes** as the load balancer routes to different healthy targets

---

## 📸 Step-by-Step Walkthrough

---

### Step 1 — Verifying the Starting EC2 Instance (Direct Access)

**URL:** `ec2-13-222-161-169.compute-1.amazonaws.com/#/configuration`

Before building the high availability architecture, accessed the Employee Directory App directly via its EC2 public DNS to verify it is working. The Configuration Settings page confirms:

| Setting | Value |
|---------|-------|
| **DynamoDB Enabled** | ✅ Connected |
| **S3 Access Enabled** | ✅ Connected |
| **S3 Bucket** | `labstack-d9ec4d90-6cea-41d0-9560-6836-imagesbucket-fhesrzoh9y9n` |
| **Region** | `us-east-1` |
| **Availability Zone** | `us-east-1a` |
| **EC2 Instance ID** | `i-0298c4a8db69c19df` |

This baseline confirms the app is healthy and connected to both S3 and DynamoDB before the load balancer and ASG are added. The instance ID (`i-0298c4a8db69c19df`) will be used to track which instance is serving requests once the ALB is in front.

<img width="1366" height="768" alt="auto scaling (1)" src="https://github.com/user-attachments/assets/3793e0f8-9ec9-4c7f-b5d4-0486e122c948" />

---

### Step 2 — Comparing Load Balancer Types

**Service:** EC2 → Load Balancers → Compare and select load balancer type

Reviewed the three AWS Elastic Load Balancer types before creating one:

| Type | Best For | Protocols |
|------|----------|-----------|
| **Application Load Balancer (ALB)** | HTTP/HTTPS web apps, flexible routing rules, Lambda targets | HTTP, HTTPS |
| **Network Load Balancer (NLB)** | Ultra-high performance, TLS offloading, static IPs, VPC endpoints | TCP, UDP, TLS |
| **Gateway Load Balancer (GWLB)** | Routing traffic through third-party security appliances (firewalls, inspection tools) | All IP traffic |

Selected Application Load Balancer — the correct choice for a web application receiving HTTP traffic, where layer-7 content-based routing and health checks are needed. ALB is the most commonly used load balancer type for web apps in AWS.

<img width="1366" height="768" alt="auto scaling (2)" src="https://github.com/user-attachments/assets/cd458c7a-8051-47dc-b8b1-4ed0a4579de1" />

---

### Step 3 — Creating the Target Group

**Service:** EC2 → Target Groups → Create target group

Configured the `lab-app-target-group` target group with target type Instances — meaning the ALB will route traffic directly to registered EC2 instances (as opposed to IP addresses, Lambda functions, or another ALB). Named it `lab-app-target-group`.

The four target type options and their use cases:
- **Instances** — standard EC2 targets; integrates with Auto Scaling Groups ✅ selected
- **IP addresses** — for VPC/on-premises resources or IPv6
- **Lambda function** — serverless targets (ALB only)
- **Application Load Balancer** — chaining NLB → ALB for PrivateLink patterns

<img width="1366" height="768" alt="auto scaling (3)" src="https://github.com/user-attachments/assets/0eb8126d-0954-459a-b711-3f305bc71a09" />

---

### Step 4 — Configuring Target Group Health Checks

**Service:** EC2 → Target Groups → Create target group → Health check settings

Configured the health check parameters that determine when the ALB considers an instance healthy or unhealthy enough to receive or stop receiving traffic:

| Setting | Value | Meaning |
|---------|-------|---------|
| **Health check port** | Traffic port | Use the same port as the target group (port 80) |
| **Healthy threshold** | **2** | 2 consecutive successful checks = healthy |
| **Unhealthy threshold** | **5** | 5 consecutive failed checks = unhealthy |
| **Timeout** | **20 seconds** | No response in 20s = failed check |

These thresholds balance responsiveness with stability — 2 successes to confirm recovery, 5 failures before pulling an instance from rotation to avoid flapping (repeatedly adding/removing an instance on transient errors).

<img width="1366" height="768" alt="auto scaling (4)" src="https://github.com/user-attachments/assets/8be890b5-add3-4a7b-b95d-4078d3abbc86" />

---

### Step 5 — Creating the Auto Scaling Group — Launch Template Selection

**Service:** EC2 → Auto Scaling Groups → Create Auto Scaling group — Step 1

Named the Auto Scaling Group app-asg and selected the pre-built launch template lab-app-launch-template (Default version 1). The 7-step wizard visible on the left shows the full ASG creation flow:

1. **Choose launch template or configuration** ← current step
2. Choose instance launch options (VPC + subnets)
3. Integrate with other services (attach load balancer)
4. Configure group size and scaling
5. Add notifications (SNS)
6. Add tags
7. Review

A Launch Template is a saved, versioned EC2 instance configuration (AMI, instance type, key pair, security group, User Data) that the ASG uses every time it needs to launch a new instance — ensuring every instance that scales out is identical to the original.

<img width="1366" height="768" alt="auto scaling (5)" src="https://github.com/user-attachments/assets/62cdc69e-caba-469f-bf37-5786f3c77b23" />

---

### Step 6 — Selecting VPC and Multi-AZ Subnets

**Service:** EC2 → Auto Scaling Groups → Create Auto Scaling group — Step 2

Selected the Lab VPC (`vpc-061ffdd0ad79aa678`, CIDR `10.0.0.0/16`) and chose two public subnets across different Availability Zones for high availability:

| Subnet | Availability Zone | CIDR |
|--------|------------------|------|
| `subnet-0e632327ce3749644` (Public Subnet 1) | `us-east-1a` | `10.0.0.0/24` |
| `subnet-0ed1f2c93e1364f88` (Public Subnet 2) | `us-east-1b` | `10.0.2.0/24` |

Set Availability Zone distribution to Balanced best effort — if one AZ has a launch failure, the ASG will attempt to launch in another healthy AZ rather than fail entirely. This is the foundation of fault-tolerant cloud architecture: if an entire AWS data center (AZ) goes down, the application keeps running from the other AZ.

<img width="1366" height="768" alt="auto scaling (6)" src="https://github.com/user-attachments/assets/d4c25a59-95ae-4f16-bd20-8d994a6a798a" />

---

### Step 7 — Target Group Successfully Created

**Service:** EC2 → Target Groups → `lab-app-target-group`

The green banner confirms: *"Successfully created the target group: lab-app-target-group. Anomaly detection is automatically applied to all registered targets."*

Target group details:
| Field | Value |
|-------|-------|
| **ARN** | `arn:aws:elasticloadbalancing:us-east-1:253397837122:targetgroup/lab-app-target-group/487c283e2f6adf22` |
| **Target type** | Instance |
| **Protocol : Port** | HTTP : 80 |
| **Protocol version** | HTTP1 |
| **VPC** | `vpc-061ffdd0ad79aa678` |
| **Load balancer** | None associated (yet) |
| **Total targets** | 1 (Unused — not yet receiving traffic) |

The target shows status Unused (1) rather than Healthy or Unhealthy — because no load balancer has been attached yet. Once the ALB is created and associated, targets will move to Initial → Healthy.

<img width="1366" height="768" alt="auto scaling (7)" src="https://github.com/user-attachments/assets/842299a0-08e0-4a90-99e4-8a138da55687" />

---

### Step 8 — Configuring the ALB Listener

**Service:** EC2 → Load Balancers → Create Application Load Balancer → Listeners and routing

Configured the Listener for the `Web-Application-ALB` — the rule that tells the load balancer what to do with incoming traffic:

| Setting | Value |
|---------|-------|
| **Protocol** | HTTP |
| **Port** | **80** |
| **Default action** | Forward to target groups |
| **Target group** | `lab-app-target-group` (HTTP, Instance, IPv4) |
| **Weight** | 1 (100%) |
| **Target stickiness** | Off |

The routing action options shown:
- **Forward to target groups** ✅ selected — sends traffic to registered healthy instances
- **Redirect to URL** — used for HTTP → HTTPS redirects
- **Return fixed response** — returns a static HTTP response (used for maintenance pages)

Weight `1` at `100%` means all traffic goes to this single target group. In more complex architectures, multiple target groups with different weights enable blue/green deployments and canary releases.

<img width="1366" height="768" alt="auto scaling (8)" src="https://github.com/user-attachments/assets/7168348c-9219-4c21-9a62-da112cc0bbf8" />

---

### Step 9 — Application Load Balancer Successfully Created

**Service:** EC2 → Load Balancers → `Web-Application-ALB`

The green banner confirms: *"Successfully created load balancer: Web-Application-ALB. It might take a few minutes for your load balancer to fully set up and route traffic."*

ALB details:
| Field | Value |
|-------|-------|
| **Load balancer type** | Application |
| **Status** | Provisioning → (becomes Active) |
| **Scheme** | Internet-facing |
| **VPC** | `vpc-061ffdd0ad79aa678` |
| **Availability Zones** | `us-east-1b` (subnet `0ed1f2c93e1364f88`) + `us-east-1a` (subnet `0e632327ce3749644`) |
| **IP address type** | IPv4 |
| **Hosted zone** | `Z35SXDOTRQ7X7K` |
| **Date created** | April 5, 2026, 17:07 UTC-08:00 |

The ALB spans both AZs, meaning it can route traffic to healthy instances in either AZ — if all instances in one AZ fail health checks, all traffic automatically shifts to the surviving AZ.

<img width="1366" height="768" alt="auto scaling (9)" src="https://github.com/user-attachments/assets/91c36b02-0e1f-4372-b3cb-a22bba1cbc84" />

---

### Step 10 — App Accessible Through the ALB URL

**URL:** `web-application-alb-1406601950.us-east-1.elb.amazonaws.com/#/configuration`

With the ALB provisioned and the target group registered, accessed the Employee Directory App through the load balancer DNS name instead of the direct EC2 URL. The configuration page confirms the app is healthy and reachable through the ALB:

| Setting | Value |
|---------|-------|
| **DynamoDB Enabled** | ✅ |
| **S3 Access Enabled** | ✅ |
| **Region** | `us-east-1` |
| **Availability Zone** | `us-east-1a` |
| **EC2 Instance ID** | `i-0298c4a8db69c19df` |
| **CPU Usage** | 3% |
| **Stress Testing** | Available (for load testing) |

The URL in the browser bar has changed from a direct EC2 hostname to the ALB DNS name(`*.elb.amazonaws.com`) — confirming all traffic now flows through the load balancer. The Stress Application Server tool visible in Admin Tools can be used to simulate CPU load and trigger Auto Scaling policies.

<img width="1366" height="768" alt="auto scaling (10)" src="https://github.com/user-attachments/assets/6fd46df3-d4e5-4c7b-8fce-d9bce5562e38" />

---

### Step 11 — Creating the EC2 Launch Template

**Service:** EC2 → Launch Templates → Create launch template

Created the lab-app-launch-template — the reusable instance definition that the Auto Scaling Group uses every time it needs to launch a new EC2 instance. Key configuration:

| Field | Value |
|-------|-------|
| **Launch template name** | `lab-app-launch-template` |
| **Template version description** | "A web server for the employee directory app" |
| **Auto Scaling guidance** | ✅ Enabled — optimizes the template for use with EC2 Auto Scaling |

Enabling Auto Scaling guidance ensures the console highlights settings that are required or recommended for ASG compatibility (such as not using instance-specific settings that would prevent identical copies from launching). The summary panel on the right (AMI, instance type, firewall, storage) fills in as each setting is configured.

Launch templates support multiple versions — meaning you can update an app's configuration (e.g., new AMI, new User Data script) and deploy the new version to the ASG without recreating everything from scratch.

<img width="1366" height="768" alt="auto scaling (11)" src="https://github.com/user-attachments/assets/c981bfc6-afb4-43b7-9fcd-be97e0cc9f10" />

---

### Step 12 — Launch Template Successfully Created

**Service:** EC2 → Launch Templates → `lab-app-launch-template`

The green success banner confirms: *"Successfully created lab-app-launch-template (lt-0ec007f82557d4b96)."*

The Next Steps panel explains the three things you can do with a launch template:

1. **Launch an instance** — spin up a one-time On-Demand EC2 instance from this template
2. **Create an Auto Scaling group** — use this template as the basis for automatic scaling ← the path taken in this lab
3. **Create Spot Fleet** — use the template to request discounted Spot Instances for cost-sensitive workloads

The console also describes Auto Scaling directly: *"Amazon EC2 Auto Scaling helps you maintain application availability and allows you to scale your Amazon EC2 capacity up or down automatically according to conditions you define."* This is the definition every cloud professional needs to know.

<img width="1366" height="768" alt="auto scaling (12)" src="https://github.com/user-attachments/assets/bc95387d-4959-4511-98de-abe2da8507d3" />

---

### Step 13 — App Served Through ALB — New Instance ID

**URL:** `ec2-13-222-161-169.compute-1.amazonaws.com/#/configuration`

Revisited the Employee Directory App configuration page at a different point in time. The EC2 Instance ID has changed to `i-03c21da37b4d2058c` — a different instance than the one seen in Step 1 (`i-0298c4a8db69c19df`). This proves the Auto Scaling Group has launched at least one new instance, and the load balancer is routing traffic to it. Both DynamoDB and S3 show ✅ healthy — confirming the new ASG-launched instance has the correct IAM Role permissions and is fully functional.

Critically, the app URL and behavior are identical even though a completely different EC2 instance is serving the request — this is the entire point of a load balancer: the user experience is seamless regardless of which backend instance handles the request.

<img width="1366" height="768" alt="auto scaling (13)" src="https://github.com/user-attachments/assets/1d71c925-ccfb-40c4-bf74-1b670cf34d82" />

---

### Step 14 — Attaching the ASG to the Load Balancer

**Service:** EC2 → Auto Scaling Groups → Create Auto Scaling group — Step 3: Integrate with other services

In the ASG wizard's Load balancing step, attached the Auto Scaling Group to the existing `Web-Application-ALB` by selecting:

- **Attach to an existing load balancer** ✅
- **Choose from your load balancer target groups** ✅
- **Selected target group:** `lab-app-target-group | HTTP` (Application Load Balancer: Web-Application-ALB)

This is the critical wiring step — it tells the ASG that every new instance it launches should automatically be registered as a target in the load balancer's target group. Without this, new instances would launch but never receive any traffic from the ALB. With it, scaling out is seamless: instance launches → registers in target group → passes health checks → starts receiving traffic, all automatically.

<img width="1366" height="768" alt="auto scaling (14)" src="https://github.com/user-attachments/assets/99d38c3a-51aa-4847-b2ec-dd3e51c1446a" />

---

### Step 15 — SNS Notification Subscription Confirmed

**Service:** Amazon SNS → Subscription confirmation page

After configuring the Auto Scaling Group to send notifications via Amazon SNS, confirmed the email subscription by clicking the confirmation link in the notification email. The SNS confirmation page shows:

> *"Subscription confirmed! You have successfully subscribed."*  
> Subscription ID: `arn:aws:sns:us-east-1:253397837122:lab-app-sns-topic:c221b334-01e3-400e-b726-d0a6746171c3`

SNS notifications are configured to fire on ASG lifecycle events — specifically when instances are launched (scale-out) or terminated (scale-in). This means when the ASG replaces an unhealthy instance or scales up due to CPU load, an email alert is sent immediately. In production, these alerts go to on-call engineers and are often routed into Slack, PagerDuty, or ticketing systems.

<img width="1366" height="768" alt="auto scaling (15)" src="https://github.com/user-attachments/assets/2695f218-98de-4191-8809-a81e0f141abc" />

---

### Step 16 — Auto Scaling Group Has Launched Multiple Instances

**Service:** EC2 → Instances

Navigated to the EC2 Instances list to observe the result of the Auto Scaling Group being active. Three instances are visible:

| Name | Instance ID | State | Type | AZ |
|------|-------------|-------|------|----|
| *(unnamed)* | `i-0f9d771f3120551a9` | ✅ Running | t3.micro | us-east-1b |
| Web Application | `i-0298c4a8db69c19df` | ⊘ Terminated | t3.micro | us-east-1a |
| *(unnamed)* | `i-03c21da37b4d2058c` | ✅ Running | t3.micro | us-east-1a |

This view tells the full story:
- The original "Web Application" instance (`i-0298c4a8db69c19df`) has been Terminated — it was replaced by the ASG-managed fleet
- Two new instances are Running — one in each AZ (`us-east-1a` and `us-east-1b`), confirming the ASG's multi-AZ distribution is working
- All new instances pass 3/3 status checks — fully healthy and serving traffic

The ASG has taken over instance management: it launched, placed, and health-checked new instances automatically, without any manual EC2 launch actions.

<img width="1366" height="768" alt="auto scaling (16)" src="https://github.com/user-attachments/assets/583fe612-650f-48ca-bb55-9cb976ab34fe" />

---

### Step 17 — Observing a Broken Instance (DynamoDB + S3 Disconnected)

**URL:** `ec2-13-222-161-169.compute-1.amazonaws.com/#/configuration`

In a critical real-world troubleshooting scenario, accessed an instance that is showing failure symptoms. The Configuration page shows:

| Setting | Value |
|---------|-------|
| **DynamoDB Enabled** | ❌ Disconnected |
| **S3 Access Enabled** | ❌ Disconnected |
| **Region** | — (blank) |
| **Availability Zone** | `us-east-1a` |
| **EC2 Instance ID** | `i-0298c4a8db69c19df` |
| **CPU Usage** | 0% |

Both DynamoDB and S3 show red ✗ — and the Region is blank. This is the terminated instance (`i-0298c4a8db69c19df`) being accessed directly via its old EC2 URL, which is no longer connected to the correct IAM role permissions or network path. Comparing this to Step 13 (a healthy instance showing green ✅) demonstrates exactly how to distinguish a misconfigured or broken instance from a healthy one — a core cloud support troubleshooting skill. A cloud engineer seeing this would know the IAM role is missing, the instance is in a bad state, or the instance was not launched from the correct template.

<img width="1366" height="768" alt="auto scaling (17)" src="https://github.com/user-attachments/assets/f3b92697-03fc-46df-ac22-e2e7a5039909" />

---
