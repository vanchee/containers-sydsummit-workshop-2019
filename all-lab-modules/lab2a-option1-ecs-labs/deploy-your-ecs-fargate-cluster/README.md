## Deploy your container using ECR/ECS

Deploying individual containers is not difficult.  However, when you need to coordinate many container deployments, a container management tool like ECS can greatly simplify the task (no pun intended).

ECS refers to a JSON formatted template called a [Task Definition](http://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_definitions.html) that describes one or more containers making up your application or service.  The task definition is the recipe that ECS uses to run your containers as a **task** on your EC2 instances or AWS Fargate.

<details>
<summary>INFO: What is a task?</summary>
A task is a running set of containers on a single host. You may hear or see 'task' and 'container' used interchangeably. Often, we refer to tasks instead of containers because a task is the unit of work that ECS launches and manages on your cluster. A task can be a single container, or multiple containers that run together.

Fun fact: a task is very similar to a Kubernetes 'pod'.
</details>

Most task definition parameters map to options and arguments passed to the [docker run](https://docs.docker.com/engine/reference/run/) command which means you can describe configurations like which container image(s) you want to use, host:container port mappings, cpu and memory allocations, logging, and more.

In this lab, you will create a task definition to serve as a foundation for deploying the containerized adoption platform stored in ECR with ECS. You will be using the [Fargate](https://aws.amazon.com/fargate/) launch type, which let's you run containers without having to manage servers or other infrastructure. Fargate containers launch with a networking mode called [awsvpc](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task-networking.html), which gives ECS tasks the same networking properties of EC2 instances.  Tasks will essentially receive their own [elastic network interface](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-eni.html).  This offers benefits like task-specific security groups.  Let's get started!

![Lab 2 Architecture](images/02-arch.png)

*Note: You will use the AWS Management Console for this lab, but remember that you can programmatically accomplish the same thing using the AWS CLI, SDKs, or CloudFormation.*

### Instructions:

1. Create an ECS task definition that describes what is needed to run the monolith.

    The CloudFormation template you ran at the beginning of the workshop created some placeholder ECS resources running a simple "Hello World" NGINX container. (You can see this running now at the public endpoint for the ALB also created by CloudFormation available in `cfn-output.json`.) We'll begin to adapt this placeholder infrastructure to run the monolith by creating a new "Task Definition" referencing the container built in the previous lab.

    In the AWS Management Console, navigate to [Task Definitions](https://console.aws.amazon.com/ecs/home#/taskDefinitions) in the ECS dashboard. Find the Task Definition named <code>Monolith-Definition-<b><i>STACK_NAME</i></b></code>, select it, and click "Create new revision". Select the "monolith-service" container under "Container Definitions", and update "Image" to point to the Image URI of the monolith container that you just pushed to ECR (something like `018782361163.dkr.ecr.us-east-1.amazonaws.com/mysfit-mono-oa55rnsdnaud:latest`).

    ![Edit container example](images/02-task-def-edit-container.png)

2. Check the CloudWatch logging settings in the container definition.

    In the previous lab, you attached to the running container to get *stdout*, but no one should be doing that in production and it's good operational practice to implement a centralized logging solution.  ECS offers integration with [CloudWatch logs](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/WhatIsCloudWatchLogs.html) through an awslogs driver that can be enabled in the container definition.

    Verify that under "Storage and Logging", the "log driver" is set to "awslogs".

    The Log configuration should look something like this:

    ![CloudWatch Logs integration](images/02-awslogs.png)

    Click "Update" to save the container settings and then "Create" to create the Task Definition revision.

3. Run the task definition using the [Run Task](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ecs_run_task.html) method.

    You should be at the task definition view where you can do things like create a new revision or invoke certain actions.  In the **Actions** dropdown, select **Run Task** to launch your container.

    ![Run Task](images/02-run-task.png)

    Configure the following fields:

    * **Launch Type** - select **Fargate**
    * **Cluster** - select your workshop cluster from the dropdown menu
    * **Task Definition** - select the task definition you created from the dropdown menu

    In the "VPC and security groups" section, enter the following:

    * **Cluster VPC** - Your workshop VPC, named like <code>Mysfits-VPC-<b><i>STACK_NAME</i></b></code>
    * **Subnets** - Select a public subnet, such as <code>Mysfits-PublicOne-<b><i>STACK_NAME</i></b></code>
    * **Security goups** - The default is fine, but you can confirm that it allows inbound traffic on port 80
    * **Auto-assign public IP** - "ENABLED"

    Leave all remaining fields as their defaults and click **Run Task**.

    You'll see the task start in the **PENDING** state (the placeholder NGINX task is still running as well).

    ![Task state](images/02-task-pending.png)

    In a few seconds, click on the refresh button until the task changes to a **RUNNING** state.

    ![Task state](images/02-task-running.png)

4. Test the running task by using cURL from your Cloud9 environment to send a simple GET request.

    First we need to determine the IP of your task. When using the "Fargate" launch type, each task gets its own ENI and Public/Private IP address. Click on the ID of the task you just launched to go to the detail page for the task. Note down the Public IP address to use with your curl command.


    ![Container Instance IP](images/02-public-IP.png)

    Run the same curl command as before (or view the endpoint in your browser) and ensure that you get a list of Mysfits in the response.

    <details>
    <summary>HINT: curl refresher</summary>
    <pre>
    $ curl http://<b><i>TASK_PUBLIC_IP_ADDRESS</i></b>/mysfits
    </pre>
    </details>

    Navigate to the [CloudWatch Logs dashboard](https://console.aws.amazon.com/cloudwatch/home#logs:), and click on the monolith log group (e.g.: `mysfits-MythicalMonolithLogGroup-LVZJ0H2I2N4`).  Logging statements are written to log streams within the log group.  Click on the most recent log stream to view the logs.  The output should look very familiar from your testing in Lab 1.

    ![CloudWatch Log Entries](images/02-cloudwatch-logs.png)

    If the curl command was successful, stop the task by going to your cluster, select the **Tasks** tab, select the running monolith task, and click **Stop**.

### Checkpoint:
Nice work!  You've created a task definition and are able to deploy the monolith container using ECS.  You've also enabled logging to CloudWatch Logs, so you can verify your container works as expected.

[*^ back to the top*](#monolith-to-microservices-with-docker-and-aws-fargate)
