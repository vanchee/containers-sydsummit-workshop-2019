Dockerizing an app and deploying onto AWS Fargate, Amazon EKS
================================================================

Welcome to the Mythical Mysfits team!

In this lab, you'll build the Mythical Mysfits adoption platform with Docker, deploy it on Amazon ECS. Let's get started!

### Requirements:

* AWS account - if you don't have one, it's easy and free to [create one](https://aws.amazon.com/).
* AWS IAM account with elevated privileges allowing you to interact with CloudFormation, IAM, EC2, ECS, ECR, ELB/ALB, VPC, SNS, CloudWatch, Cloud9. [Learn how](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html).
* Familiarity with [Python](https://wiki.python.org/moin/BeginnersGuide/Programmers), [Docker](https://www.docker.com/), and [AWS](httpts://aws.amazon.com) - *not required but a bonus*.

### What you'll do:

These labs are designed to be completed in sequence, and the full set of instructions are documented below.  Read and follow along to complete the labs.  If you're at a live AWS event, the workshop staff will give you a high-level overview of the labs and help answer any questions.  Don't worry if you get stuck, we provide hints along the way.


* **Workshop Setup:** [Setup working environment on AWS](#lets-begin)
* **Lab 1:** [Containerize the Mythical Mysfits application](#lab-1---containerize-the-mythical-mysfits-adoption-agency-platform)
* **Lab 2:** [Deploy the container using AWS Fargate](#lab-2---deploy-your-container-using-ecrecs)
* **Lab 3:** [Scale the adoption platform application with an ALB and an ECS Service](#lab-3---scale-the-adoption-platform-monolith-with-an-alb)
* **Cleanup** [Put everything away nicely](#workshop-cleanup)

### Conventions:

Throughout this workshop, we will provide commands for you to run in the terminal.  These commands will look like this:

<pre>
$ ssh -i <b><i>PRIVATE_KEY.PEM</i></b> ec2-user@<b><i>EC2_PUBLIC_DNS_NAME</i></b>
</pre>
