# ☁️ Amazon S3 File Sharing with Event Notifications

## 📖 Project Description

This project demonstrates how to configure an **Amazon S3 bucket** for secure file sharing and implement **real-time email notifications** using **Amazon SNS**. The scenario simulates an external media user (`mediacouser`) who uploads and manages image files for a café. Administrators receive email alerts whenever files are added or removed.

This project is ideal for beginners learning **AWS S3**, **IAM**, and **event-driven architecture**, and is based on a lab from the **AWS re/Start Program**.

<p align="center">
  <img src="https://github.com/Adib2024/amazon-s3-event-notifications/blob/main/Architecture.png?raw=true" alt="Architecture" />
</p>


The following steps describe the usage flow in the diagram:
1.	When new product pictures are available or when existing pictures must be updated, a representative from the media company signs in to the AWS Management Console as (`mediacouser`) to upload, change, or delete the bucket contents.
2.	As an alternative, (`mediacouser`) can use the AWS Command Line Interface (AWS CLI) to change the contents of the S3 bucket.
3.	When Amazon S3 detects a change in the contents of the bucket, it publishes an email notification to the s3NotificationTopic Amazon Simple Notification Service (Amazon SNS) topic.
4.	The administrator who is subscribed to the s3NotificationTopic SNS topic receives an email message that contains the details of the changes to the contents of the bucket. 

Note: In real-world implementations, external users might not receive direct access to CLI Host as depicted in the diagram.

---

## ✨ Features

- 🔐 Secure IAM access for external users to an S3 bucket
- 📤 Upload and delete images in an S3 folder (`images/`)
- 📩 Email alerts via SNS when bucket content is modified
- ❌ Block unauthorized actions (e.g., changing ACLs or uploading outside the allowed prefix)
- ⚙️ CLI-based interaction with AWS services

---

## 🧰 Tech Stack

- **Amazon S3** – Object storage
- **Amazon SNS** – Simple Notification Service for alerts
- **IAM** – Access management and policies
- **AWS CLI** – Command-line interface for AWS
- **Linux (Ubuntu)** – CLI host environment

---

## 🛠 Getting Started

### Prerequisites

- AWS account with necessary permissions
- AWS CLI installed and configured
- Linux environment (CLI Host)

---

### 1. **Create an S3 bucket**

Choose a unique bucket name (must be globally unique):

```bash
aws s3 mb s3://<your-bucket-name> --region us-west-2
```

📌 _Example_:  
```bash
aws s3 mb s3://cafe-demo-012345 --region us-west-2
```

---

### 2. **Upload sample images to the `images/` folder**

```bash
aws s3 sync ~/initial-images/ s3://<your-bucket-name>/images
```

---

### 3. **Verify that the files are in the S3 bucket**

```bash
aws s3 ls s3://<your-bucket-name>/images/ --human-readable --summarize
```



✅ S3 bucket with uploaded image files — _"S3 Bucket View"_

<img align="center" alt="S3 Bucket files" src="https://github.com/Adib2024/amazon-s3-event-notifications/blob/main/S3%20Bucket%20Initial.png" />

---

### 4. **Set up IAM user `(`mediacouser`)` with limited access**

Attach a policy like this:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": [
                "s3:ListAllMyBuckets",
                "s3:GetBucketLocation"
            ],
            "Resource": [
                "arn:aws:s3:::*"
            ],
            "Effect": "Allow",
            "Sid": "AllowGroupToSeeBucketListInTheConsole"
        },
        {
            "Action": [
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::cafe-*",
                "arn:aws:s3:::cafe-*/*"
            ],
            "Effect": "Allow",
            "Sid": "AllowRootLevelListingOfTheBucket"
        },
        {
            "Action": [
                "s3:PutObject",
                "s3:GetObject",
                "s3:GetObjectVersion",
                "s3:DeleteObject",
                "s3:DeleteObjectVersion"
            ],
            "Resource": "arn:aws:s3:::cafe-*/images/*",
            "Effect": "Allow",
            "Sid": "AllowUserSpecificActionsOnlyInTheSpecificPrefix"
        }
    ]
}
```

### Policy Statements Description

- **`AllowGroupToSeeBucketListInTheConsole`**  
  Grants permissions that allow the user to use the Amazon S3 console to view the list of S3 buckets in the account.

- **`AllowRootLevelListingOfTheBucket`**  
  Grants permissions that allow the user to use the Amazon S3 console to view the list of first-level objects in the `cafe` bucket and other objects in the bucket.

- **`AllowUserSpecificActionsOnlyInTheSpecificPrefix`**  
  Grants permissions for actions the user can perform on the objects in the `cafe-*/images/*` folder.  
  The main operations include:
  - `GetObject` (read)
  - `PutObject` (write)
  - `DeleteObject` (delete)  
  Additionally, it includes operations for version-related actions.

---

Sign into (`mediacouser`) account and view the content of S3 Bucket. The user able to upload new file into the S3 bucket:

<img align="center" alt="mediacouser S3 Bucket" src="https://github.com/Adib2024/amazon-s3-event-notifications/blob/main/mediacouserS3.png" />


User do not have permission to rename the object:

<img align="center" alt="Rename denied" src="https://github.com/Adib2024/amazon-s3-event-notifications/blob/main/rename%20denied.png" />

User have permission to delete the object:

<img align="center" alt="delete permission" src="https://github.com/Adib2024/amazon-s3-event-notifications/blob/main/delete%20permission.png" />


---

### 5. **Create an SNS topic and subscribe your email**

Back to root user

```bash
aws sns create-topic --name s3NotificationTopic
```

Then subscribe:

```bash
aws sns subscribe   --topic-arn arn:aws:sns:us-west-2:xxxxxxxxxxxx:s3NotificationTopic   --protocol email   --notification-endpoint your-email@example.com
```

---

### 6. **Add SNS permissions to allow S3 to publish**

Use this SNS topic policy:

```json
{
  "Version": "2008-10-17",
  "Id": "S3PublishPolicy",
  "Statement": [
    {
      "Sid": "AllowPublishFromS3",
      "Effect": "Allow",
      "Principal": {
        "Service": "s3.amazonaws.com"
      },
      "Action": "SNS:Publish",
      "Resource": "<ARN of s3NotificationTopic>",
      "Condition": {
        "ArnLike": {
          "aws:SourceArn": "arn:aws:s3:*:*:<your-bucket-name>"
        }
      }
    }
  ]
}


```

### Subscribe to the SNS Topic

1. In the **Topic ARN** box, choose the `s3NotificationTopic` SNS topic that appears as an option.
2. From the **Protocol** dropdown list, choose **Email**.
3. In the **Endpoint** box, enter an email address that you can access.
4. Choose **Create subscription**.
5. Check your email inbox — a **Subscription Confirmation** email will be sent.
6. Confirm the subscription by clicking the link in the email.


---

### 7. **Create and apply `s3EventNotification.json`**

```json
{
    "TopicConfigurations": [
      {
        "TopicArn": "<ARN of s3NotificationTopic>",
        "Events": ["s3:ObjectCreated:*","s3:ObjectRemoved:*"],
        "Filter": {
          "Key": {
            "FilterRules": [
              {
                "Name": "prefix",
                "Value": "images/"
              }
            ]
          }
        }
      }
    ]
  }


```

To associate the event configuration file with the S3 share bucket, run the following command
```bash
aws s3api put-bucket-notification-configuration   --bucket <your-bucket-name>   --notification-configuration file://s3EventNotification.json
```

Email should be like:

<img align="center" alt="SNS S3 email" src="https://github.com/Adib2024/amazon-s3-event-notifications/blob/main/SNS%20S3%20email.png" />

---

### 8. **Test the configuration**

Test the S3 bucket event notifications by simulating (`mediacouser`) actions: uploading and deleting objects (triggering email alerts), and attempting an unauthorized action (to confirm it's blocked), using the AWS s3api CLI.

Run 'aws configure' and sign into (`mediacouser`) to test the configuration:


CREATE and DELETE Object on S3 Bucket

<img align="center" alt="mediacouser-createdelete" src="https://github.com/Adib2024/amazon-s3-event-notifications/blob/main/mediacouser-createdelete.png" />

#### ✅ SNS email alert after uploading/deleting a file — _"Notification Email Sample"_

<img align="center" alt="Object Created" src="https://github.com/Adib2024/amazon-s3-event-notifications/blob/main/Object%20Created.png" />

<img align="center" alt="Object Removed" src="https://github.com/Adib2024/amazon-s3-event-notifications/blob/main/Object%20Removed.png" />



#### ✅ Upload an object
```bash
aws s3 cp ~/new-images/sample.jpg s3://<your-bucket-name>/images/
```

#### ❌ Try unauthorized action
```bash
aws s3api put-object-acl   --bucket <your-bucket-name>   --key images/sample.jpg   --acl public-read
```

#### 🗑️ Delete an object
```bash
aws s3 rm s3://<your-bucket-name>/images/sample.jpg
```

✅ Access denied message — _"Unauthorized Operation Test"_

<img align="center" alt="Access denied" src="https://github.com/Adib2024/amazon-s3-event-notifications/blob/main/Access%20denied.png" />

---

## 💡 What I Learned / Takeaways

- Hands-on with **IAM policies** for fine-grained access control
- Using **AWS CLI** to automate S3 and SNS operations
- Applying **event-driven design** using S3 + SNS
- Securing cloud resources while allowing collaboration

---

## 🚀 Future Improvements

- Add **CloudWatch logging** for monitoring and auditing
- Add a **Lambda function** to process images on upload
- Build a **simple UI** for easier file uploads
- Use **S3 versioning** and lifecycle policies

---

## 🙌 Credits

This project was completed as part of the **AWS re/Start Program**.  
Special thanks to **AWS Training & Certification** for the lab materials.

---

## 📄 License

This project is licensed under the [MIT License](LICENSE).

---
