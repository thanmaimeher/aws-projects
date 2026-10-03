# Create and Launch an AMI from an Existing EC2 Instance

A hands-on lab: capture a running EC2 instance as an Amazon Machine Image (AMI), launch a clone from it, and verify the clone has the same OS, software, and data.

## Table of Contents

1. [Objective](#objective)
2. [Concepts](#concepts)
3. [Prerequisites](#prerequisites)
4. [Part 1: Create the AMI](#part-1-create-the-ami)
5. [Part 2: Launch an instance from the AMI](#part-2-launch-an-instance-from-the-ami)
6. [Part 3: Verify with a text file](#part-3-verify-with-a-text-file)
7. [Part 4: Challenge - bake a web server into an AMI](#part-4-challenge---bake-a-web-server-into-an-ami)
8. [CLI equivalent](#cli-equivalent)
9. [Questions and answers](#questions-and-answers)
10. [Cleanup](#cleanup)

## Objective

- Create an AMI from a running EBS-backed EC2 instance.
- Launch a new instance from that AMI.
- Prove the clone contains the original's files and installed software without any manual setup.

## Concepts

| Term | Meaning |
|---|---|
| **AMI** | A template containing the OS, installed software, configuration, and a reference to one or more EBS snapshots. |
| **EBS snapshot** | A point-in-time copy of an EBS volume. EC2 creates one automatically when you click **Create image**. |
| **EBS-backed AMI** | Root volume is EBS. The instance can be stopped and its data persists. |
| **Instance store-backed AMI** | Root volume is ephemeral instance store. Cannot be stopped; data is lost on termination. |

## Prerequisites

- An AWS account with permission to use EC2.
- A running EC2 instance (Amazon Linux) with an EBS root volume.
- A key pair, and a security group allowing SSH (22). For the challenge, also allow HTTP (80).

## Part 1: Create the AMI

1. Open **EC2 Console -> Instances** and select the source instance.
2. Choose **Actions -> Image and templates -> Create image**.
3. Enter:
   - **Image name:** e.g. `my-web-server-v1`
   - **Description:** optional
   - **No reboot:** leave **unchecked** (default). EC2 reboots the instance so the file system is consistent.
4. Review **Instance volumes** (size and type can be changed).
5. Click **Create image**.
6. Go to **EC2 -> Images -> AMIs** and wait until the status changes from `pending` to `available`.

> EC2 also creates an EBS snapshot automatically. It appears under **Elastic Block Store -> Snapshots**.

<!-- Screenshot: AMI status "available" -->
<!-- ![AMI available](images/ami-available.png) -->

## Part 2: Launch an instance from the AMI

1. In **Images -> AMIs**, select the AMI and click **Launch instance from AMI**.
2. Configure:
   - **Name:** e.g. `clone-instance`
   - **Instance type:** e.g. `t3.micro`
   - **Key pair:** existing or new
   - **Network settings:** VPC, subnet, and security group
   - **Storage:** same size or larger than the original (never smaller)
3. Click **Launch instance**.
4. Wait for **2/2 status checks passed**, then connect via EC2 Instance Connect or SSH.

<!-- Screenshot: clone instance running -->
<!-- ![Clone running](images/clone-running.png) -->

## Part 3: Verify with a text file

### Steps performed

1. On the original instance, created a directory under `/home/ec2-user/` and a text file inside it with sample content.
2. Created an AMI from the running instance and waited for status `available`.
3. Launched a new EC2 instance from this AMI.
4. Connected to the new instance via EC2 Instance Connect.
5. Navigated to `/home/ec2-user/<dir>/` and confirmed the file exists with identical content.

### Commands

```bash
# On the original instance
mkdir ~/my-dir
echo "Hello from original instance" > ~/my-dir/test.txt

# On the cloned instance
ls /home/ec2-user/my-dir
cat /home/ec2-user/my-dir/test.txt
```

### Note on the root login

The clone's prompt showed `[root@ip-172-31-18-191 ~]#`. EC2 Instance Connect pre-fills the **Username** field, and for a custom AMI the console cannot reliably detect the OS, so it defaulted to `root`. Nothing was changed inside the instance. The `ec2-user` home directory from the original instance was still present, and root could read it.

To log in as `ec2-user`, change the **Username** field from `root` to `ec2-user` before clicking **Connect**. Avoid using root for everyday work.

### Result

The AMI preserved the directory structure and file data, confirming the root EBS volume was captured and restored.

<!-- Screenshot: cat output on the clone -->
<!-- ![File content on clone](images/clone-file-content.png) -->

## Part 4: Challenge - bake a web server into an AMI

**Goal:** install Apache on the original instance, create an AMI, launch a clone, and confirm the clone serves the same page with no setup.

### 4.1 Allow HTTP

Security Groups -> Edit inbound rules -> Add rule -> Type: **HTTP**, Source: `0.0.0.0/0`.

### 4.2 Install and start Apache (original instance)

```bash
# Amazon Linux 2023
sudo dnf install -y httpd
# Amazon Linux 2: sudo yum install -y httpd

sudo systemctl start httpd
sudo systemctl enable httpd
```

`enable` makes Apache start on boot, so the clone serves the page immediately.

### 4.3 Create the page

```bash
sudo nano /var/www/html/index.html
```

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>AMI Challenge</title>
  <style>
    body { font-family: Arial, sans-serif; background: #232f3e; color: #fff;
           text-align: center; padding-top: 80px; }
    h1 { color: #ff9900; }
    .box { background: #37475a; display: inline-block; padding: 30px 50px;
           border-radius: 10px; }
  </style>
</head>
<body>
  <div class="box">
    <h1>Hello from the ORIGINAL instance</h1>
    <p>This page was baked into my custom AMI.</p>
    <p>Version: 1.0</p>
  </div>
</body>
</html>
```

Test:

```bash
curl localhost
```

Then open `http://<original-public-ip>` in a browser.

### 4.4 Bake the AMI

Actions -> Image and templates -> Create image -> name it `web-server-ami-v1` -> wait for `available`.

### 4.5 Launch and verify the clone

1. Launch an instance from `web-server-ami-v1` with a security group that allows port 80.
2. **Do not log in and do not install anything.**
3. Open `http://<clone-public-ip>`.

**Success:** the clone serves the same page immediately.

### Bonus tasks

1. **Prove they are separate machines.** On the clone:
   ```bash
   sudo sed -i 's/ORIGINAL/CLONE/' /var/www/html/index.html
   ```
   Only the clone's page changes.
2. **Bake version 2.** Change the original page to "Version: 2.0", create `web-server-ami-v2`, and launch a third instance.
3. **Break it on purpose.** Skip `systemctl enable httpd`, bake an AMI, launch a clone, and observe the result.

## CLI equivalent

```bash
# Create the AMI
aws ec2 create-image \
  --instance-id i-0123456789abcdef0 \
  --name "my-web-server-v1" \
  --description "AMI from existing instance"

# Check status
aws ec2 describe-images --image-ids ami-0abc1234567890def \
  --query 'Images[0].State'

# Launch from the AMI
aws ec2 run-instances \
  --image-id ami-0abc1234567890def \
  --instance-type t3.micro \
  --key-name my-key \
  --security-group-ids sg-0123456789abcdef0 \
  --subnet-id subnet-0123456789abcdef0 \
  --count 1
```

## Questions and answers

### 1. Why did the clone serve the page without any installation?

An AMI is a template of the original instance's root EBS volume. When the image was created, EC2 took a snapshot of the volume, capturing the OS, the installed Apache package, its configuration, the enabled `httpd` service, and the `index.html` file. A new instance launched from the AMI gets a volume created from that snapshot, so it boots with everything already in place. Because `httpd` was enabled, it started automatically on boot and began serving the page.

### 2. What would happen if `systemctl enable httpd` had not been run?

systemctl start httpd launches the service only in the current boot. It lives in memory and stops when the instance reboots. A web server needs to be available without manual action, so it must be enabled, which creates a symlink under /etc/systemd/system/multi-user.target.wants/. At boot, systemd starts as the first process, reads that directory, and starts every service that has a symlink there.

An AMI captures the disk, not running processes. If httpd was only started on the original instance, the AMI contains the Apache package and index.html but no enable symlink. A clone launched from it boots with httpd installed but not running, and the browser shows a connection error until someone runs sudo systemctl enable --now httpd.

### 3. Why do the two instances have different public IPs?

_TODO_

### 4. Where is the page stored: instance store, EBS root volume, or S3?

_TODO_

## Cleanup

1. Terminate all test instances.
2. Deregister the AMIs (**AMIs -> Actions -> Deregister AMI**).
3. Delete the associated snapshots. Deregistering does **not** delete them, and they continue to incur charges.

## Key takeaways

- **Create image** snapshots the EBS volume(s) and registers the AMI in one step.
- Installed software, configuration, and data on the root volume are all captured.
- Enable services (`systemctl enable`) before baking so they start on boot in clones.
- Clones are independent copies; changes on one never affect the other.
- EBS-backed AMIs can be stopped; instance store-backed AMIs cannot.
