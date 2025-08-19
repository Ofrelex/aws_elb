# Configuring Auto Scaling with ALB using Launch Template
Here’s a clear, **AWS Console–only** walkthrough to set up an Auto Scaling Group (ASG) behind an **Application Load Balancer (ALB)** using a **Launch Template**, plus how to test scaling.

---

# Prerequisites (once)

* **VPC** with at least two subnets in different AZs.
* **Existing ALB** (HTTP/HTTPS listener) and a **Target Group (TG)** that matches your app port/health check (e.g., HTTP:80, path `/`). If your ALB doesn’t have a TG yet, create one now (EC2 → Target Groups → Create). You attach **the TG** to the ASG (not the ALB directly). ([AWS Documentation][1])
* **Security Groups (SGs)**

  * ALB SG: inbound 80/443 from the internet.
  * Instance SG: inbound **from the ALB SG** on the app port only (e.g., 80).
* **IAM role** (optional but recommended) with **AmazonSSMManagedInstanceCore** to allow SSM access.

---

# 1) Create the Launch Template

**Console path:** EC2 → **Launch Templates** → **Create launch template**. ([AWS Documentation][2])

1. **Name & description:** e.g., `web-lt`.
2. **Amazon Machine Image (AMI):** choose your OS (e.g., Amazon Linux 2023).
3. **Instance type:** e.g., `t3.micro` for demo; pick what fits your app.
4. **Key pair:** optional (for SSH).
5. **Network settings:**

   * **Do not** pin a specific subnet (ASG will choose subnets).
   * Select the **Instance SG** (allow from ALB SG only).
6. **IAM instance profile:** select your role (optional but helpful).
7. **Storage:** keep default or adjust as needed.
8. **Advanced details → User data (optional but handy for testing):** paste a simple web server bootstrap:

   ```bash
   #!/bin/bash
   dnf -y update
   dnf -y install nginx
   echo "Hello from $(hostname) in AZ $(curl -s http://169.254.169.254/latest/meta-data/placement/availability-zone)" > /usr/share/nginx/html/index.html
   systemctl enable --now nginx
   ```
9. **Create launch template.**

> Tip: Launch Templates are the recommended way to store AMI/instance/SG/IAM/user-data parameters for ASGs. ([AWS Documentation][2])

---

# 2) Create the Auto Scaling Group (and attach the Target Group)

**Console path:** EC2 → **Auto Scaling Groups** → **Create Auto Scaling group**.

1. **Choose launch template:** pick `web-lt` (latest version).
2. **Configure network:** choose **at least two subnets** in different AZs.
3. **Load balancing:**

   * Select **Attach to an existing load balancer**.
   * Choose **Choose from your target groups** and pick the TG used by your ALB.
   * Check **ELB health checks** (recommended). Set **Health check grace period** (e.g., 300–600 seconds) so instances have time to boot before health evaluation. ([AWS Documentation][1])
4. **Group size:** set **Desired** = 2, **Minimum** = 2, **Maximum** = 6 (adjust for your needs).
5. **Configure advanced options:**

   * Enable **Default instance warmup** and set a value (e.g., 180–300s). This helps scaling behave smoothly by ignoring noisy metrics during startup. ([AWS Documentation][3])
6. **Tags** (optional): add `Name = web-asg-instance` (propagate to instances).
7. **Create Auto Scaling group.**

> Note: You attach **Target Groups** (not the ALB itself) to the ASG. The ASG will auto-register/deregister instances with the TG as they launch/terminate. ([AWS Documentation][1])

---

# 3) Configure Scaling Policies (Target Tracking)

**Target tracking** is the simplest, most robust option. It automatically creates CloudWatch alarms and keeps a chosen metric near a target value. **Two good choices:**

### A) Track average CPU utilization (easy starting point)

**Path:** EC2 → Auto Scaling Groups → select your ASG → **Automatic scaling** tab → **Create dynamic scaling policy** → **Target tracking**.

* **Policy name:** `tt-cpu-50`.
* **Metric type:** **Average CPU utilization**.
* **Target value:** e.g., **50%**.
* **Instance warmup:** use the ASG default warmup enabled earlier.
* **Create policy.** ([AWS Documentation][4])

### B) Track **ALB Request Count per Target** (great for web traffic)

**Path:** same as above, policy type **Target tracking**, but choose **Application Load Balancer request count per target**.

* Select your **ALB + Target Group** (console handles the linkage).
* **Target value:** start with something like **1,000 requests/target** (tune for your instance size/app).
* **Create policy.** ([AWS Documentation][4])

> Tip: Keep **Default instance warmup** enabled so new instances don’t trigger premature scale-in/out while starting. ([AWS Documentation][3])

---

# 4) Verify the ALB ↔ ASG attachment

* **EC2 → Target Groups → your TG → Targets tab:** you should see the instances from your ASG registering and becoming **healthy** after a short while.
* **EC2 → Load Balancers → your ALB:** Confirm listener rules forward to **your TG**.
* Using **ELB health checks** lets the ASG replace instances that the load balancer reports as unhealthy. ([AWS Documentation][1])

---

# 5) Test Auto Scaling (Scale-out and Scale-in)

## A) Sanity check

1. **Find ALB DNS name:** EC2 → Load Balancers → select ALB → copy **DNS name**.
2. Open `http://<ALB_DNS_NAME>/` in your browser; you should see the NGINX page that includes hostname/AZ (from user data).
3. Refresh multiple times—you should occasionally hit different instances (helpful if you added unique content per instance).

## B) Watch scaling behavior under load

1. **Generate traffic** (from any machine on the internet): open multiple browser tabs or use a simple load tool (e.g., `ab` or `hey`) to hit the ALB URL for a few minutes.
2. **Observe in Console:**

   * **EC2 → Auto Scaling Groups → your ASG → Activity:** see scaling events (Launching/Terminating).
   * **CloudWatch → Metrics:**

     * For CPU policies: **EC2 → Per-Group Metrics → ASGAverageCPUUtilization**.
     * For request policies: **ALB → TargetGroup → RequestCountPerTarget**.
       The ASG should **scale out** as the metric rises above the target and **scale in** when traffic subsides (respecting warmup/grace). ([AWS Documentation][4])

## C) Quick manual trigger (optional)

* Temporarily set **Desired capacity** to a higher number on the ASG (Details tab → **Edit**), watch instances launch and register healthy in the TG, then restore your desired value.

---

## Operational Tips

* **Health checks:** Prefer **ELB health checks** for ASGs behind ALBs so failing app health is reflected in replacement decisions. Set a reasonable **Health check grace period**. ([AWS Documentation][5])
* **Warmup:** Use **Default instance warmup** (and tune it) to avoid flapping and premature scale-in. ([AWS Documentation][3])
* **Security posture:** Allow app traffic to instances **only from the ALB SG**; ALB handles public ingress.
* **User data readiness:** If your app takes longer to start, increase **grace period** and **warmup** accordingly.

---

### Handy AWS Docs (for deeper reference)

* Create **Launch Templates** (console). ([AWS Documentation][2])
* Attach **Target Groups** / ALB to an ASG (console). ([AWS Documentation][1])
* **Target Tracking** scaling policies (overview & console). ([AWS Documentation][4])
* **Default instance warmup** (concept & enable in console). ([AWS Documentation][6])
* **ASG health checks & grace period**. ([AWS Documentation][5])

