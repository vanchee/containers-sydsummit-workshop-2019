Next steps scaling using ALB, service and splitting monolith to microservices.
===============================================================================
* **Lab 3:** [Scale the adoption platform monolith with an ALB and an ECS Service](#lab-3---scale-the-adoption-platform-monolith-with-an-alb)
* **Lab 4:** [Incrementally build and deploy more microservices with AWS Fargate](#lab-4-incrementally-build-and-deploy-each-microservice-using-fargate)
* **Cleanup** [Put everything away nicely](#workshop-cleanup)

## Lab 3 - Scale the adoption platform monolith with an ALB

The Run Task method you used in the last lab is good for testing, but we need to run the adoption platform as a long running process.

In this lab, you will use an Elastic Load Balancing [Appliction Load Balancer (ALB)](https://aws.amazon.com/elasticloadbalancing/) to distribute incoming requests to your running containers. In addition to simple load balancing, this provieds capabilities like path-based routing to different services.

What ties this all together is an **ECS Service**, which maintains a desired task count (i.e. n number of containers as long running processes) and integrates with the ALB (i.e. handles registration/deregistration of containers to the ALB). An initial ECS service and ALB were created for you by CloudFormation at the beginning of the workshop. In this lab, you'll update those resources to host the containerized monolith service. Later, you'll make a new service from scratch once we break apart the monolith.

![Lab 3 Architecture](images/03-arch.png)


### Instructions:

1. Test the placeholder service:

    The CloudFormation stack you launched at the beginning of the workshop included an ALB in front of a placeholder ECS service running a simple container with the NGINX web server. Find the hostname for this ALB in the "LoadBalancerDNS" output variable in the `cfn-output.json` file, and verify that you can load the NGINX default page:

    ![NGINX default page](images/03-nginx.png)


3. Update the service to use your task definition:

    Find the ECS cluster named <code>Cluster-<i><b>STACK_NAME</b></i></code>, then select the service named <code><b><i>STACK_NAME</i></b>-MythicalMonolithService-XXX</code> and click "Update" in the upper right:

    ![update service](images/03-update-service.png)

    Update the Task Definition to the revision you created in the previous lab, then click through the rest of the screens and update the service.

4. Test the functionality of the website:

    You can monitor the progress of the deployment on the "Tasks" tab of the service page:

    ![monitoring the update](images/03-deployment.png)

    The update is fully deployed once there is just one instance of the Task running the latest revision:

    ![fully deployed](images/03-fully-deployed.png)

    Visit the S3 static site for the Mythical Mysfits (which was empty earlier) and you should now see the page filled with Mysfits once your update is fully deployed. Remember you can access the website at <code>http://<b><i>BUCKET_NAME</i></b>.s3-website.<b><i>REGION</i></b>.amazonaws.com/</code> where the bucket name can be found in the `workshop-1/cfn-output.json` file:

    ![the functional website](images/03-website.png)

    Click the heart icon to like a Mysfit, then click the Mysfit to see a detailed profile, and ensure that the like count has incremented:

    ![like functionality](images/03-like-count.png)

    This ensures that the monolith can read from and write to DynamoDB, and that it can process likes. Check the CloudWatch logs from ECS and ensure that you can see the "Like processed." message in the logs:

    ![like logs](images/03-like-processed.png)

<details>
<summary>INFO: What is a service and how does it differ from a task??</summary>

An [ECS service](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ecs_services.html) is a concept where ECS allows you to run and maintain a specified number (the "desired count") of instances of a task definition simultaneously in an ECS cluster.


tl;dr a **Service** is comprised of multiple **tasks** and will keep them up and running. See the link above for more detail.

</details>

### Checkpoint:
Sweet! Now you have a load-balanced ECS service managing your containerized Mythical Mysfits application. It's still a single monolith container, but we'll work on breaking it down next.

[*^ back to the top*](#monolith-to-microservices-with-docker-and-aws-fargate)

## Lab 4: Incrementally build and deploy each microservice using Fargate

It's time to break apart the monolithic adoption into microservices. To help with this, let's see how the monolith works in more detail.

> The monolith serves up several different API resources on different routes to fetch info about Mysfits, "like" them, or adopt them.
>
> The logic for these resources generally consists of some "processing" (like ensuring that the user is allowed to take a particular action, that a Mysfit is eligible for adoption, etc) and some interaction with the persistence layer, which in this case is DynamoDB.

> It is often a bad idea to have many different services talking directly to a single database (adding indexes and doing data migrations is hard enough with just one application), so rather than split off all of the logic of a given resource into a separate service, we'll start by moving only the "processing" business logic into a separate service and continue to use the monolith as a facade in front of the database. This is sometimes described as the [Strangler Application pattern](https://www.martinfowler.com/bliki/StranglerApplication.html), as we're "strangling" the monolith out of the picture and only continuing to use it for the parts that are toughest to move out until it can be fully replaced.

> The ALB has another feature called [path-based routing](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-listeners.html#path-conditions), which routes traffic based on URL path to particular target groups.  This means you will only need a single instance of the ALB to host your microservices.  The monolith service will receive all traffic to the default path, '/'.  Adoption and like services will be '/adopt' and '/like', respectively.

Here's what you will be implementing:

![Lab 4](images/04-arch.png)

*Note: The green tasks denote the monolith and the orange tasks denote the "like" microservice


As with the monolith, you'll be using [Fargate](https://aws.amazon.com/fargate/) to deploy these microservices, but this time we'll walk through all the deployment steps for a fresh service.

### Instructions:

1. First, we need to add some glue code in the monolith to support moving the "like" function into a separate service. You'll use your Cloud9 environment to do this.  If you've closed the tab, go to the [Cloud9 Dashboard](https://console.aws.amazon.com/cloud9/home) and find your environment. Click "**Open IDE**". Find the `app/monolith-service/service/mythicalMysfitsService.py` source file, and uncomment the following section:

    ```
    # @app.route("/mysfits/<mysfit_id>/fulfill-like", methods=['POST'])
    # def fulfillLikeMysfit(mysfit_id):
    #     serviceResponse = mysfitsTableClient.likeMysfit(mysfit_id)
    #     flaskResponse = Response(serviceResponse)
    #     flaskResponse.headers["Content-Type"] = "application/json"
    #     return flaskResponse
    ```

    This provides an endpoint that can still manage persistence to DynamoDB, but omits the "business logic" (okay, in this case it's just a print statement, but in real life it could involve permissions checks or other nontrivial processing) handled by the `process_like_request` function.

2. With this new functionality added to the monolith, rebuild the monolith docker image with a new tag, such as `nolike`, and push it to ECR just as before (It is a best practice to avoid the `latest` tag, which can be ambiguous. Instead choose a unique, descriptive name, or even better user a Git SHA and/or build ID):

    <pre>
    $ cd app/monolith-service
    $ docker build -t monolith-service:nolike .
    $ docker tag monolith-service:nolike <b><i>ECR_REPOSITORY_URI</i></b>:nolike
    $ docker push <b><i>ECR_REPOSITORY_URI</i></b>:nolike
    </pre>

3. Now, just as in Lab 2, create a new revision of the monolith Task Definition (this time pointing to the "nolike" version of the container image), AND update the monolith service to use this revision as you did in Lab 3.

4. Now, build the like service and push it to ECR.

    To find the like-service ECR repo URI, navigate to [Repositories](https://console.aws.amazon.com/ecs/home#/repositories) in the ECS dashboard, and find the repo named like <code><b><i>STACK_NAME</i></b>-like-XXX</code>.  Click on the like-service repository and copy the repository URI.

    ![Getting Like Service Repo](images/04-ecr-like.png)

    *Note: Your URI will be unique.*

    <pre>
    $ cd app/like-service
    $ docker build -t like-service .
    $ docker tag like-service:latest <b><i>ECR_REPOSITORY_URI</i></b>:latest
    $ docker push <b><i>ECR_REPOSITORY_URI</i></b>:latest
    </pre>

5. Create a new **Task Definition** for the like service using the image pushed to ECR.

    Navigate to [Task Definitions](https://console.aws.amazon.com/ecs/home#/taskDefinitions) in the ECS dashboard. Click on **Create New Task Definition**.

    Select **Fargate** launch type, and click **Next step**.

    Enter a name for your Task Definition, e.g. mysfits-like.

    In the "[Task execution IAM role](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_execution_IAM_role.html)" section, Fargate needs an IAM role to be able to pull container images and log to CloudWatch.  Select the role named like <code><b><i>STACK_NAME</i></b>-EcsServiceRole-XXXXX</code> that was already created for the monolith service.

    The "[Task size](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_definition_parameters.html#task_size)" section lets you specify the total cpu and memory used for the task. This is different from the container-specific cpu and memory values, which you will also configure when adding the container definition.

    Select **0.5GB** for **Task memory (GB)** and select **0.25vCPU** for **Task CPU (vCPU)**.

    Your progress should look similar to this:

    ![Fargate Task Definition](images/04-taskdef.png)

    Click **Add container** to associate the like service container with the task.

    Enter values for the following fields:

    * **Container name** - this is a logical identifier, not the name of the container image (e.g. `mysfits-like`).
    * **Image** - this is a reference to the container image stored in ECR.  The format should be the same value you used to push the like service container to ECR - <pre><b><i>ECR_REPOSITORY_URI</i></b>:latest</pre>
    * **Port mapping** - set the container port to be `80`.

    Here's an example:

    ![Fargate like service container definition](images/04-containerdef.png)

    *Note: Notice you didn't have to specify the host port because Fargate uses the awsvpc network mode. Depending on the launch type (EC2 or Fargate), some task definition parameters are required and some are optional. You can learn more from our [task definition documentation](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_definitions.html).*

    The like service code is designed to call an endpoint on the monolith to persist data to DynamoDB. It references an environment variable called `MONOLITH_URL` to know where to send fulfillment.

    Scroll down to the "Advanced container configuration" section, and in the "Environment" section, create an environment variable using `MONOLITH_URL` for the key. For the value, enter the **ALB DNS name** that currently fronts the monolith.

    Here's an example (make sure you enter just the hostname like `alb-mysfits-1892029901.eu-west-1.elb.amazonaws.com` without any "http" or slashes):

    ![monolith env var](images/04-env-var.png)

    Fargate conveniently enables logging to CloudWatch for you.  Keep the default log settings and take note of the **awslogs-group** and the **awslogs-stream-prefix**, so you can find the logs for this task later.

    Here's an example:

    ![Fargate logging](images/04-logging.png)

    Click **Add** to associate the container definition, and click **Create** to create the task definition.

6. Create an ECS service to run the Like Service task definition you just created and associate it with the existing ALB.

    Navigate to the new revision of the Like task definition you just created.  Under the **Actions** drop down, choose **Create Service**.

    Configure the following fields:

    * **Launch type** - select **Fargate**
    * **Cluster** - select your workshop ECS cluster
    * **Service name** - enter a name for the service (e.g. `mysfits-like-service`)
    * **Number of tasks** - enter `1`.

    Here's an example:

    ![ECS Service](images/04-ecs-service-step1.png)

    Leave other settings as defaults and click **Next Step**

    Since the task definition uses awsvpc network mode, you can choose which VPC and subnet(s) to host your tasks.

    For **Cluster VPC**, select your workshop VPC.  And for **Subnets**, select the private subnets; you can identify these based on the tags.

    Leave the default security group which allows inbound port 80.  If you had your own security groups defined in the VPC, you could assign them here.

    Here's an example:

    ![ECS Service VPC](images/04-ecs-service-vpc.png)

    Scroll down to "Load balancing" and select **Application Load Balancer** for *Load balancer type*.

    You'll see a **Load balancer name** drop-down menu appear.  Select the same Mythical Mysfits ALB used for the monolith ECS service.

    In the "Container to load balance" section, select the **Container name : port** combo from the drop-down menu that corresponds to the like service task definition.

    Your progress should look similar to this:

    ![ECS Load Balancing](images/04-ecs-service-alb.png)

    Click **Add to load balancer** to reveal more settings.

    For the **Production listener Port**, select **80:HTTP** from the drop-down.

    For the **Target Group Name**, you'll need to create a new group for the Like containers, so leave it as "create new" and replace the auto-generated value with `mysfits-like`.  This is a friendly name to identify the target group, so any value that relates to the Like microservice will do.

    Change the path pattern to `/mysfits/*/like`.  The ALB uses this path to route traffic to the like service target group.  This is how multiple services are being served from the same ALB listener.  Note the existing default path routes to the monolith target group.

    For **Evaluation order** enter `1`.  Edit the **Health check path** to be `/`.

    And finally, uncheck **Enable service discovery integration**.  While public namespaces are supported, a public zone needs to be configured in Route53 first.  Consider this convenient feature for your own services, and you can read more about [service discovery](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/service-discovery.html) in our documentation.

    Your configuration should look similar to this:

    ![Like Service](images/04-ecs-service-alb-detail.png)

    Leave the other fields as defaults and click **Next Step**.

    Skip the Auto Scaling configuration by clicking **Next Step**.

    Click **Create Service** on the Review page.

    Once the Service is created, click **View Service** and you'll see your task definition has been deployed as a service.  It starts out in the **PROVISIONING** state, progresses to the **PENDING** state, and if your configuration is successful, the service will finally enter the **RUNNING** state.  You can see these state changes by periodically click on the refresh button.

7. Once the new like service is deployed, test liking a Mysfit again by visiting the website. Check the CloudWatch logs again and make sure that the like service now shows a "Like processed." message. If you see this, you have succesfully factored out like functionality into the new microservice!

8. If you have time, you can now remove the old like endpoint from the monolith now that it is no longer seeing production use.

    Go back to your Cloud9 environment where you built the monolith and like service container images.

    In the monolith folder, open mythicalMysfitsService.py in the Cloud9 editor and find the code that reads:

    ```
    # increment the number of likes for the provided mysfit.
    @app.route("/mysfits/<mysfit_id>/like", methods=['POST'])
    def likeMysfit(mysfit_id):
        serviceResponse = mysfitsTableClient.likeMysfit(mysfit_id)
        process_like_request()
        flaskResponse = Response(serviceResponse)
        flaskResponse.headers["Content-Type"] = "application/json"
        return flaskResponse
    ```
    Once you find that line, you can delete it or comment it out.

    *Tip: if you're not familiar with Python, you can comment out a line by adding a hash character, "#", at the beginning of the line.*

9. Build, tag and push the monolith image to the monolith ECR repository.

    Use the tag `nolike2` now instead of `nolike`.

    <pre>
    $ docker build -t monolith-service:nolike2 .
    $ docker tag monolith-service:nolike <b><i>ECR_REPOSITORY_URI</i></b>:nolike2
    $ docker push <b><i>ECR_REPOSITORY_URI</i></b>:nolike2
    </pre>

    If you look at the monolith repository in ECR, you'll see the pushed image tagged as `nolike2`:

    ![ECR nolike image](images/04-ecr-nolike2.png)

10. Now make one last Task Definition for the monolith to refer to this new container image URI (this process should be familiar now, and you can probably see that it makes sense to leave this drudgery to a CI/CD service in production), update the monolith service to use the new Task Definition, and make sure the app still functions as before.

### Checkpoint:
Congratulations, you've successfully rolled out the like microservice from the monolith.  If you have time, try repeating this lab to break out the adoption microservice.  


