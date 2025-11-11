# AWS EBS GP2 → GP3 Automation

This project automatically converts any newly created **Amazon EBS volumes** of type **gp2** to **gp3** using **AWS Lambda** and **Amazon EventBridge (CloudWatch Events)**.

It helps ensure that your AWS environment always stays **compliant with organizational standards** — even if someone accidentally creates an outdated gp2 volume.

## Overview

As part of the cloud engineering team, we are responsible for maintaining AWS resources in compliance with company policies.
One such policy ensures that **all EBS volumes must be of type gp3** — since gp3 volumes offer better performance and are more cost-effective than gp2.

However, it’s easy for a team member to accidentally create a gp2 volume.
To prevent this manually, we built a simple automation using **EventBridge** and **Lambda** to catch such events and automatically convert gp2 volumes to gp3.

---

## Why This Project

Even with documentation and training, accidental non-compliant resource creation can happen.
This automation ensures that:

* Every new EBS volume is checked at creation time.
* If it’s **gp3**, nothing happens.
* If it’s **gp2**, it’s automatically **modified to gp3**.

This keeps your environment consistent, cost-optimized, and policy-compliant — without manual checks.

---

## Architecture

1. **Amazon EventBridge (formerly CloudWatch Events)** watches for “CreateVolume” events.
2. When a new EBS volume is created, EventBridge triggers a **Lambda function**.
3. The **Lambda function**, written in Python, checks if the volume type is gp2.
4. If gp2, the function modifies the volume to gp3 using the EC2 API.
5. **CloudWatch Logs** captures all Lambda execution logs for debugging and tracking.

---

## How It Works

* **Trigger:** EBS `CreateVolume` event via EventBridge.
* **Function:** Lambda written in Python 3.13.
* **Action:** Checks the event details, extracts volume ID, and modifies the volume type to gp3.
* **Logging:** Results are stored in CloudWatch Logs.
* **Permissions:** Lambda’s IAM role must have EC2 permissions to describe and modify volumes.

---

## Step-by-Step Setup

Let’s recreate this automation from scratch.

---

### 1️⃣ Create Lambda Function

1. Navigate to **AWS Lambda → Create function**.
2. Choose **Author from scratch**.
3. **Function name:** `ebs-volume-check`
4. **Runtime:** `Python 3.13`
5. Under **Permissions**, keep the default — *Create a new role with basic Lambda permissions*.
6. Click **Create function**.

<img width="2900" height="1332" alt="image" src="https://github.com/user-attachments/assets/1a5498a5-70f6-4fae-ae61-ddea46f19579" />

Once created, you’ll see a default `lambda_handler` function.

<img width="2940" height="1434" alt="image" src="https://github.com/user-attachments/assets/7bf4480b-6f98-41f8-8638-7250fe58e1af" />

---

### 2️⃣ Create EventBridge Rule

Now we’ll create a rule that triggers the Lambda whenever a new EBS volume is created.

1. Go to **Amazon EventBridge → Rules → Create rule**
2. Name the rule **`EBS-volume-check`**
3. For **Event source**, choose **AWS events or EventBridge partner events**.
4. Choose **Sample event type → AWS events**
5. Select **Use pattern form**

   * **Event source:** `AWS services`
   * **AWS service:** `EC2`
   * **Event type:** `EBS Volume Notification`
   * **Specific event:** `CreateVolume`

<img width="1652" height="1182" alt="image" src="https://github.com/user-attachments/assets/bcbf2790-1e29-4fce-b9ae-52c470f2f434" />

6. Under **Target**, choose:

   * **Target type:** AWS service
   * **Select target:** Lambda function
   * **Function:** `ebs-volume-check`

7. Keep **Use execution role** checked and let EventBridge create a new role to invoke Lambda.

8. Click **Create rule**.

Your rule is now active and ready to trigger the Lambda.

---

### 3️⃣ Verify CloudWatch Logs

Before adding any custom code, let’s make sure the trigger works.

1. Go to **EC2 → Volumes → Create Volume**.
2. Choose **Type:** `gp2`.
3. Click **Create Volume**.

<img width="2380" height="450" alt="image" src="https://github.com/user-attachments/assets/3e527a9e-b826-48af-84cb-95c63a2977cb" />

Now open **CloudWatch → Log groups** and check if your Lambda function was triggered.

You should see a new log entry confirming invocation.

<img width="2898" height="852" alt="image" src="https://github.com/user-attachments/assets/09b43372-0f47-4e19-866d-300e9a1d1a0e" />

<img width="2898" height="1074" alt="image" src="https://github.com/user-attachments/assets/70a3ef17-bc69-4a21-a6a9-ece14923c571" />

---

### 4️⃣ Add Lambda Code

Now, let’s add logic to automatically convert gp2 → gp3.

Go back to the Lambda console and replace the default code with the lambda_script.py file

<img width="2366" height="1144" alt="image" src="https://github.com/user-attachments/assets/76feb300-989c-4a4d-bd26-abb71b6d1abb" />

Click **Deploy**.

---

### 5️⃣ Update IAM Role Permissions

The Lambda function needs permission to modify EBS volumes.

1. Go to **IAM → Roles**.
2. Search for the role created for your Lambda function (e.g., `ebs-volume-check-role`).

<img width="2326" height="488" alt="image" src="https://github.com/user-attachments/assets/88e90293-8792-4961-84a3-37d8af6129e0" />
   
4. Click **Add inline policy**.
5. **Service:** EC2
6. **Actions:**

   * `DescribeVolumes`
   * `ModifyVolume`
  
<img width="2854" height="1396" alt="image" src="https://github.com/user-attachments/assets/38c5a061-d29f-45c0-8bdd-be88b2766b09" />

7. **Resources:** All resources (for simplicity; you can restrict by ARN later).
8. Name the policy **`ebs-volume-check-policy`** and click **Create policy**.

<img width="2854" height="1396" alt="image" src="https://github.com/user-attachments/assets/e8d60269-4065-40e0-a267-3df3aa432618" />

---

### 6️⃣ Test the Automation

Now let’s test the full workflow:

1. Go to **EC2 → Volumes → Create Volume**.
2. Select **Type:** gp2
3. Click **Create Volume**.

<img width="2854" height="1396" alt="image" src="https://github.com/user-attachments/assets/1ac0ddde-7523-4fbe-be7b-3ba4cb9d8372" />

Within seconds, the Lambda should detect and convert it to gp3.

<img width="2394" height="348" alt="image" src="https://github.com/user-attachments/assets/9a564b92-3e77-458a-a48a-0286f51e9e75" />

Check **CloudWatch Logs** — you should see messages confirming conversion.

<img width="2394" height="1122" alt="image" src="https://github.com/user-attachments/assets/850b9c63-eb13-4e00-a660-41daf5a679ba" />

---

## Troubleshooting

**Error:** `NameError: name 'ec2' is not defined`

<img width="2228" height="252" alt="image" src="https://github.com/user-attachments/assets/8371da8a-0d77-4d6a-806d-f9da62fba9b5" />

➡️ Fix: Ensure your client is defined as `ec2_client` and used consistently in the code.

**Error:** IndentationError

<img width="2228" height="252" alt="image" src="https://github.com/user-attachments/assets/20f188b9-8679-450d-b7ed-a34263debfaa" />

➡️ Fix: Double-check spacing inside `lambda_handler`. Lambda’s inline editor is sensitive to indentation.

**Check CloudWatch Logs**
➡️ Always use CloudWatch → Log Groups → Select your Lambda → View latest stream to see detailed logs.

<img width="2898" height="852" alt="image" src="https://github.com/user-attachments/assets/de63f186-b571-435d-84c9-189d0b7ed588" />

## Closing Note

This project demonstrates how a small automation can make a big difference in maintaining compliance and reliability.
It ensures your AWS environment automatically corrects non-compliant resource configurations — saving both time and cost.
