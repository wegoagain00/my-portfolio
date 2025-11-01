---
title: AWS Cost Optimisation Lambda Function (EBS Snapshot)
date: 2025-10-15
description: This project demonstrates how to automate AWS cost optimisation by using a Lambda function to identify and delete stale EBS snapshots. This helps reduce unnecessary storage costs by cleaning up stale snapshots. The project also covers setting up the necessary IAM permissions.
tag: AWS, Lambda, EC2, Serverless, IAM, Python
author: Tawfiq (wegoagain)
---

In my DevOps journey, I have always forgotten to delete resources. You could end up spinning an EC2 instance for a tutorial, create a few test volumes, and take a snapshot thinking you'll "delete this later." A month goes by, and you're hit with the "AWS surprise bill" for a pile of old resources you forgot about. I'm way more wary now though!

EBS snapshots are a major source of this silent cost-leak. They are "out of sight, out of mind" and they actually do not get terminated when you delete their original EC2 instance.

I decided to fix this. Instead of relying on my memory, I built a small, automated "robot" using AWS Lambda to find and delete these stale snapshots for me.

## The Plan and Architecture
My architecture includes:
1. Lambda Function. This is our 'robot', it will run Python code using boto3 library
2. AWS API. The lambda will talk to the AWS API to identify and delete stale EBS snapshots. It will list my EBS snapshots, active EC2 instances and compare them to find stale snapshots. It will then delete the stale snapshots.

![alt text](/images/aws-cost-optimisation-ebs/lambda-ebs.png)

# The Build, A Journey Through Errors (and a Lesson in Security)
Building this was a perfect example of the real-world DevOps workflow. It's not just about writing code; it's about troubleshooting the permissions and configuration that make it work.

## Step 1: The "Test Subject"
First, I needed a "stale" snapshot to test with. I spun up a simple EC2 instance, which automatically created a volume. Then, I manually created a snapshot of that volume from the EC2 console, just like you would for a backup. You choose the volume and click "Create Snapshot".
![alt text](/images/aws-cost-optimisation-ebs/img-1.png)


## Step 2: The Lambda "Robot"
I went to the Lambda console and created a new function from scratch (using the Python 3.13 runtime). I scrolled down to the code editor, pasted in my Boto3 script that contains the cleanup logic, and hit "Deploy."
![alt text](/images/aws-cost-optimisation-ebs/img-2.png)
(The Python code is in my repository [here](https://github.com/wegoagain00/aws-cost-optimisation))

![alt text](/images/aws-cost-optimisation-ebs/img-3.png)
I created a simple test event and hit the Test button. It failed immediately.

## Error 1: The 3-Second Timeout
The first error was a timeout. By default, a Lambda function's timeout is set to 3 seconds. This wasn't nearly enough time for my script to run, connect to the EC2 API, and process the lists.

The Fix: This was an easy one. In the function's Configuration > General configuration, I clicked "Edit" and increased the Timeout to 10 seconds. (10 seconds was a safe number but would probably need less time)

## Error 2: The "Access Denied" Wall
I clicked Test again. A new, more serious error: Access Denied.

The output log showed my function didn't have permission to ec2:DescribeSnapshots. This is the most common (and important) part of building in the cloud. By default, a Lambda function has permission to do almost nothing except write to CloudWatch Logs. It can't see your EC2 instances, it can't see your snapshots, and it definitely can't delete them.

This is a core security feature. It forces you to be deliberate about what your code is allowed to do.

## The Fix: A Lesson in "Least Privilege"
The solution was to give my Lambda function's Execution Role the exact permissions it needed.

In the Configuration > Permissions tab, I clicked on the role name. This took me straight to the IAM console, where I clicked "Add permissions" and "Create inline policy."

I could have been lazy and given it `AdministratorAccess`, but that's a terrible security practice. Instead, I followed the ***Principle of Least Privilege:*** only grant the bare minimum permissions required for the task.

Based on my script's logic, I created a policy that allowed only these four actions:
```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "EBSSnapshotCleanupPermissions",
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeSnapshots",
        "ec2:DescribeInstances",
        "ec2:DescribeVolumes",
        "ec2:DeleteSnapshot"
      ],
      "Resource": "*"
    }
  ]
}
```
![alt text](/images/aws-cost-optimisation-ebs/img-4.png)

## The Test Run: Did It Work?
With the new policy attached, I went back to Lambda and ran the test.

***Test 1:*** Instance is still alive. I ran the function. The output log showed that it found my snapshot, checked it, and correctly decided not to delete it because the instance it belonged to was still active. Perfect.
![alt text](/images/aws-cost-optimisation-ebs/img-5.png)
***Test 2:*** Instance is terminated. Now for the real test. I went to the EC2 console and terminated my instance. After a few minutes, the instance and its volume were gone, but the snapshot remained, all alone and costing me money.
![alt text](/images/aws-cost-optimisation-ebs/img-6.png)
![alt text](/images/aws-cost-optimisation-ebs/img-7.png)
I went back to my Lambda function and hit Test one more time.

***The Result:*** Success! The script ran, identified the snapshot as "stale," and deleted it.

I confirmed in the EC2 console: the snapshot was gone.

![alt text](/images/aws-cost-optimisation-ebs/img-8.png)

## The Next Step: Full Automation
This was a great success, but I don't want to log in and click "Test" every day. The final step is to make this truly automated.

The next project is to use **Amazon EventBridge** to create a rule. I can set a schedule (e.g., "Run this Lambda function every Sunday at 3 AM"), and my robot will clean up my account for me, forever.

This small project is a perfect real-world example of the DevOps mindset: identify a manual, error-prone problem (cost-leaking), build an automated solution (Lambda), and secure it properly (IAM).
