# Day 2: AWS IAM (Identity and Access Management) 🚀

## 📌 What is AWS IAM?
AWS IAM (Identity and Access Management) is a secure and flexible service that allows you to:
- Manage **users**, **groups**, and **roles** in AWS.
- Control **who** can access **what** resources.
- Assign **fine-grained permissions** using **policies**.

With IAM, you can manage:
✅ **User accounts**  
✅ **Group permissions**  
✅ **Temporary credentials** for applications  
✅ **Multi-Factor Authentication (MFA)**  
✅ **Federated access** using external identity providers.

---

## 📌 Why Use IAM? (Use Cases)
👉 **User Management:** Create users for developers, admins, testers, etc.  
👉 **Least Privilege:** Grant only the permissions users/services need.  
👉 **Secure Access:** Use roles instead of hardcoded credentials.  
👉 **Cross-Account Access:** Share resources with other AWS accounts securely.  
👉 **Application Access:** Assign roles to EC2, Lambda, or ECS tasks to interact with other AWS services (like S3, DynamoDB).

---

## 📌 Real-World Example
🎯 **Scenario:**  
Imagine you have a web app running on **EC2** that needs to upload files to an **S3 bucket**. Instead of embedding AWS keys in your code (which is insecure), you can:
1. Create an **IAM Role** with S3 write permissions.
2. Attach this role to the EC2 instance.
3. Your app uses the **temporary credentials** from the role to upload files to S3 securely.

Another example:  
- A **DevOps Engineer** in your team needs to deploy apps using **AWS CodeDeploy** but should NOT have admin rights.  
- You create an IAM user **devops-engineer** with a policy that allows CodeDeploy and read-only S3 access.

---

## 📌 How to Create IAM Resources

### 1️⃣ Create an IAM User
1. **Sign in to AWS Console** and go to **IAM**.
2. Navigate to **Users** > **Add users**.
3. Enter a **User name** (e.g., `devops-user`).
4. Choose **Access type**:
   - ✅ **Console access** (AWS Console login)
   - ✅ **Programmatic access** (Access key for CLI/SDK)
5. Click **Next: Permissions**.
   - Option 1: Attach existing policies (e.g., `AmazonS3ReadOnlyAccess`).
   - Option 2: Add user to an existing **group** with appropriate permissions.
6. Click **Next: Tags** (optional).
7. Click **Next: Review**.
8. Click **Create user**.
9. Save the **Access Key ID** and **Secret Access Key** securely for CLI/SDK.

---

### 2️⃣ Create an IAM Group (Optional)
1. Go to **IAM** > **Groups** > **Create group**.
2. Enter a **Group name** (e.g., `devops-team`).
3. Attach **policies** (e.g., `AmazonEC2FullAccess`, `AmazonS3ReadOnlyAccess`).
4. Add users to this group.

---

### 3️⃣ Create an IAM Role
Roles are useful for applications, services, or cross-account access.

#### Example: EC2 instance needing S3 access
1. Go to **IAM** > **Roles** > **Create role**.
2. Choose **Trusted entity**:
   - AWS service: **EC2**.
3. Click **Next: Permissions**.
   - Attach a policy (e.g., `AmazonS3FullAccess`).
4. Click **Next: Tags** (optional).
5. Click **Next: Review**.
6. Name the role (e.g., `ec2-s3-role`).
7. Click **Create role**.
8. Attach this role to your EC2 instance.

---

## 📌 Key Concepts
- **User:** Individual identity for a person or application.
- **Group:** A collection of users with common permissions.
- **Role:** Temporary credentials for services or cross-account access.
- **Policy:** JSON document that defines allowed/denied actions.

---

## 📌 Example Policy Snippet
Here’s an example of a policy that allows listing and uploading files to S3:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket",
        "s3:PutObject"
      ],
      "Resource": [
        "arn:aws:s3:::my-bucket",
        "arn:aws:s3:::my-bucket/*"
      ]
    }
  ]
}
