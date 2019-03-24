## Lab 1 - Containerize the Mythical Mysfits adoption agency platform

The Mythical Mysfits adoption agency infrastructure has always been running directly on EC2 VMs. Our first step will be to modernize how our code is packaged by containerizing the current Mythical Mysfits adoption platform, which we'll also refer to as the monolith application.  To do this, you will create a [Dockerfile](https://docs.docker.com/engine/reference/builder/), which is essentially a recipe for [Docker](https://aws.amazon.com/docker) to build a container image.  You'll use your [AWS Cloud9](https://aws.amazon.com/cloud9/) development environment to author the Dockerfile, build the container image, and run it to confirm it's able to process adoptions.

[Containers](https://aws.amazon.com/what-are-containers/) are a way to package software (e.g. web server, proxy, batch process worker) so that you can run your code and all of its dependencies in a resource isolated process. You might be thinking, "Wait, isn't that a virtual machine (VM)?" Containers virtualize the operating system, while VMs virtualize the hardware. Containers provide isolation, portability and repeatability, so your developers can easily spin up an environment and start building without the heavy lifting.  More importantly, containers ensure your code runs in the same way anywhere, so if it works on your laptop, it will also work in production.

### Here's what you're going to work on in lab 1:

![Lab 1 Architecture](images/01-arch.png)

1. Review the draft Dockerfile and add the missing instructions indicated by comments in the file:

    *Note: If you're already familiar with how Dockerfiles work and want to focus on breaking the monolith apart into microservices, skip down to ["HINT: Final Dockerfile"](#final-dockerfile) near the end of step 5, create a Dockerfile in the monolith directory with the hint contents, build the "monolith" image, and continue to step 6.  Otherwise continue on...*

    One of the Mythical Mysfits' developers started working on a Dockerfile in her free time, but she was pulled to a high priority project before she finished.

    In the Cloud9 file tree, navigate to `workshop-1/app/monolith-service`, and double-click on **Dockerfile.draft** to open the file for editing.

    *Note: If you would prefer to use the bash shell and a text editor like vi or emacs instead, you're welcome to do so.*

    Review the contents, and you'll see a few comments at the end of the file noting what still needs to be done.  Comments are denoted by a "#".

    Docker builds container images by stepping through the instructions listed in the Dockerfile.  Docker is built on this idea of layers starting with a base and executing each instruction that introduces change as a new layer.  It caches each layer, so as you develop and rebuild the image, Docker will reuse layers (often referred to as intermediate layers) from cache if no modifications were made.  Once it reaches the layer where edits are introduced, it will build a new intermediate layer and associate it with this particular build.  This makes tasks like image rebuild very efficient, and you can easily maintain multiple build versions.

    ![Docker Container Image](images/01-container-image.png)

    For example, in the draft file, the first line - `FROM ubuntu:latest` - specifies a base image as a starting point.  The next instruction - `RUN apt-get -y update` - creates a new layer where Docker updates package lists from the Ubuntu repositories.  This continues until you reach the last instruction which in most cases is an `ENTRYPOINT` *(hint hint)* or executable being run.

    Add the remaining instructions to Dockerfile.draft.

    <details>
    <summary>HINT: Helpful links for completing Dockefile.draft</summary>
    <pre>
    Here are links to external documentation to give you some ideas:

    #[TODO]: Copy the "service" directory into container image

    - Consider the [COPY](https://docs.docker.com/engine/reference/builder/#copy) command
    - You're copying both the python source files and requirements.txt from the "monolith-service/service" directory on your EC2 instance into the working directory of the container, which can be specified as "."

    #[TODO]: Install dependencies listed in the requirements.txt file using pip

    - Consider the [RUN](https://docs.docker.com/engine/reference/builder/#run) command
    - More on [pip and requirements files](https://pip.pypa.io/en/stable/user_guide/#requirements-files)
    - We're using pip and python binaries from virtualenv, so use "bin/pip" for your command

    #[TODO]: Specify a listening port for the container

    - Consider the [EXPOSE](https://docs.docker.com/engine/reference/builder/#expose) command
    - App listening portNum can be found in the app source - mythicalMysfitsService.py

    #[TODO]: Run "mythicalMysfitsService.py" as the final step. We want this container to run as an executable. Looking at ENTRYPOINT for this?

    - Consider the [ENTRYPOINT](https://docs.docker.com/engine/reference/builder/#entrypoint) command
    - Our ops team typically runs 'python mythicalMysfitsService.py' to launch the application on our servers.
    </pre>
    </details>

    Once you're happy with your additions OR if you get stuck, you can check your work by comparing your work with the hint below.

    <details>
    <summary>HINT: Completed Dockerfile</summary>
    <pre>
    FROM ubuntu:latest
    RUN apt-get update -y
    RUN apt-get install -y python-pip python-dev build-essential
    RUN pip install --upgrade pip
    COPY ./service /MythicalMysfitsService
    WORKDIR /MythicalMysfitsService
    RUN pip install -r ./requirements.txt
    EXPOSE 80
    ENTRYPOINT ["python"]
    CMD ["mythicalMysfitsService.py"]
    </pre>
    </details>

    If your Dockerfile looks good, rename your file from "Dockerfile.draft" to "Dockerfile" and continue to the next step.

    <pre>
    $ mv Dockerfile.draft Dockerfile
    </pre>

2. Build the image using the [Docker build](https://docs.docker.com/engine/reference/commandline/build/) command.

    This command needs to be run in the same directory where your Dockerfile is. **Note the trailing period** which tells the build command to look in the current directory for the Dockerfile.

    <pre>
    $ docker build -t monolith-service .
    </pre>

    You'll see a bunch of output as Docker builds all layers of the image.  If there is a problem along the way, the build process will fail and stop (red text and warnings along the way are fine as long as the build process does not fail).  Otherwise, you'll see a success message at the end of the build output like this:

    <pre>
    Step 9/10 : ENTRYPOINT ["python"]
     ---> Running in 7abf5edefb36
    Removing intermediate container 7abf5edefb36
     ---> 653ccee71620
    Step 10/10 : CMD ["mythicalMysfitsService.py"]
     ---> Running in 291edf3d5a6f
    Removing intermediate container 291edf3d5a6f
     ---> a8d2aabc6a7b
    Successfully built a8d2aabc6a7b
    Successfully tagged monolith-service:latest
    </pre>

    *Note: Your output will not be exactly like this, but it will be similar.*

    Awesome, your Dockerfile built successfully, but our developer didn't optimize the Dockefile for the microservices effort later.  Since you'll be breaking apart the monolith codebase into microservices, you will be editing the source code (e.g. `mythicalMysfitsService.py`) often and rebuilding this image a few times.  Looking at your existing Dockerfile, what is one thing you can do to improve build times?

    <details>
    <summary>HINT</summary>
    Remember that Docker tries to be efficient by caching layers that have not changed.  Once change is introduced, Docker will rebuild that layer and all layers after it.

    Edit mythicalMysfitsService.py by adding an arbitrary comment somewhere in the file.  If you're not familiar with Python, [comments](https://docs.python.org/2/tutorial/introduction.html) start with the hash character, '#' and are essentially ignored when the code is interpreted.

    For example, here a comment (`# Author: Mr Bean`) was added before importing the time module:
    <pre>
    # Author: Mr Bean

    import time
    from flask import Flask
    from flask import request
    import json
    import requests
    ....
    </pre>

    Rebuild the image using the 'docker build' command from above and notice Docker references layers from cache, and starts rebuilding layers starting from Step 5, when mythicalMysfitsService.py is copied over since that is where change is first introduced:

    <pre>
    Step 5/10 : COPY ./service /MythicalMysfitsService
     ---> 9ec17281c6f9
    Step 6/10 : WORKDIR /MythicalMysfitsService
     ---> Running in 585701ed4a39
    Removing intermediate container 585701ed4a39
     ---> f24fe4e69d88
    Step 7/10 : RUN pip install -r ./requirements.txt
     ---> Running in 1c878073d631
    Collecting Flask==0.12.2 (from -r ./requirements.txt (line 1))
    </pre>

    Try reordering the instructions in your Dockerfile to copy the monolith code over after the requirements are installed.  The thinking here is that the Python source will see more changes than the dependencies noted in requirements.txt, so why rebuild requirements every time when we can just have it be another cached layer.
    </details>

    Edit your Dockerfile with what you think will improve build times and compare it with the Final Dockerfile hint below.


    #### Final Dockerfile
    <details>
    <summary>HINT: Final Dockerfile</summary>
    <pre>
    FROM ubuntu:latest
    RUN apt-get update -y
    RUN apt-get install -y python-pip python-dev build-essential
    RUN pip install --upgrade pip
    COPY service/requirements.txt .
    RUN pip install -r ./requirements.txt
    COPY ./service /MythicalMysfitsService
    WORKDIR /MythicalMysfitsService
    EXPOSE 80
    ENTRYPOINT ["python"]
    CMD ["mythicalMysfitsService.py"]
    </pre>
    </details>

    To see the benefit of your optimizations, you'll need to first rebuild the monolith image using your new Dockerfile (use the same build command at the beginning of step 5).  Then, introduce a change in `mythicalMysfitsService.py` (e.g. add another arbitrary comment) and rebuild the monolith image again.  Docker cached the requirements during the first rebuild after the re-ordering and references cache during this second rebuild.  You should see something similar to below:

    <pre>
    Step 6/11 : RUN pip install -r ./requirements.txt
     ---> Using cache
     ---> 612509a7a675
    Step 7/11 : COPY ./service /MythicalMysfitsService
     ---> c44c0cf7e04f
    Step 8/11 : WORKDIR /MythicalMysfitsService
     ---> Running in 8f634cb16820
    Removing intermediate container 8f634cb16820
     ---> 31541db77ed1
    Step 9/11 : EXPOSE 80
     ---> Running in 02a15348cd83
    Removing intermediate container 02a15348cd83
     ---> 6fd52da27f84
    </pre>

    You now have a Docker image built.  The -t flag names the resulting container image.  List your docker images and you'll see the "monolith-service" image in the list. Here's a sample output, note the monolith image in the list:

    <pre>
    $ docker images
    REPOSITORY                                                              TAG                 IMAGE ID            CREATED              SIZE
    monolith-service                                                        latest              29f339b7d63f        About a minute ago   506MB
    ubuntu                                                                  latest              ea4c82dcd15a        4 weeks ago          85.8MB
    golang                                                                  1.9                 ef89ef5c42a9        4 months ago         750MB
    </pre>

    *Note: Your output will not be exactly like this, but it will be similar.*

    Notice the image is also tagged as "latest".  This is the default behavior if you do not specify a tag of your own, but you can use this as a freeform way to identify an image, e.g. monolith-service:1.2 or monolith-service:experimental.  This is very convenient for identifying your images and correlating an image with a branch/version of code as well.

3. Run the docker container and test the adoption agency platform running as a container:

    Use the [docker run](https://docs.docker.com/engine/reference/run/) command to run your image; the -p flag is used to map the host listening port to the container listening port.

    <pre>
    $ docker run -p 8000:80 -e AWS_DEFAULT_REGION=<b><i>REGION</i></b> -e DDB_TABLE_NAME=<b><i>TABLE_NAME</i></b> monolith-service
    </pre>

    *Note: You can find your DynamoDB table name in the file `workshop-1/cfn-output.json` derived from the outputs of the CloudFormation stack.*

    Here's sample output as the application starts:

    ```
    * Running on http://0.0.0.0:80/ (Press CTRL+C to quit)
    ```

    *Note: Your output will not be exactly like this, but it will be similar.*

    To test the basic functionality of the monolith service, query the service using a utility like [cURL](https://curl.haxx.se/), which is bundled with Cloud9.

    Click on the plus sign next to your tabs and choose **New Terminal** or click **Window** -> **New Terminal** from the Cloud9 menu to open a new shell session to run the following curl command.

    <pre>
    $ curl http://localhost:8000/mysfits
    </pre>

    You should see a JSON array with data about a number of Mythical Mysfits.

    *Note: Processes running inside of the Docker container are able to authenticate with DynamoDB because they can access the EC2 metadata API endpoint running at `169.254.169.254` to retrieve credentials for the instance profile that was attached to our Cloud9 environment in the initial setup script. Processes in containers cannot access the `~/.aws/credentials` file in the host filesystem (unless it is explicitly mounted into the container).*

    Switch back to the original shell tab where you're running the monolith container to check the output from the monolith.

    The monolith container runs in the foreground with stdout/stderr printing to the screen, so when the request is received, you should see a `200`. "OK".

    Here is sample output:

    <pre>
    INFO:werkzeug:172.17.0.1 - - [16/Nov/2018 22:24:18] "GET /mysfits HTTP/1.1" 200 -
    </pre>

    In the tab you have the running container, type **Ctrl-C** to stop the running container.  Notice, the container ran in the foreground with stdout/stderr printing to the console.  In a production environment, you would run your containers in the background and configure some logging destination.  We'll worry about logging later, but you can try running the container in the background using the -d flag.

    <pre>
    $ docker run -d -p 8000:80 -e AWS_DEFAULT_REGION=<b><i>REGION</i></b> -e DDB_TABLE_NAME=<b><i>TABLE_NAME</i></b> monolith-service
    </pre>

    List running docker containers with the [docker ps](https://docs.docker.com/engine/reference/commandline/ps/) command to make sure the monolith is running.

    <pre>
    $ docker ps
    </pre>

    You should see monolith running in the list. Now repeat the same curl command as before, ensuring you see the same list of Mysfits. You can check the logs again by running [docker logs](https://docs.docker.com/engine/reference/commandline/ps/) (it takes a container name or id fragment as an argument).

    <pre>
    $ docker logs <b><i>CONTAINER_ID</i></b>
    </pre>

    Here's sample output from the above commands:

    <pre>
    $ docker run -d -p 8000:80 -e AWS_DEFAULT_REGION=<b><i>REGION</i></b> -e DDB_TABLE_NAME=<b><i>TABLE_NAME</i></b> monolith-service
    51aba5103ab9df25c08c18e9cecf540592dcc67d3393ad192ebeda6e872f8e7a
    $ docker ps
    CONTAINER ID        IMAGE                           COMMAND                  CREATED             STATUS              PORTS                  NAMES
    51aba5103ab9        monolith-service:latest         "python mythicalMysf…"   24 seconds ago      Up 23 seconds       0.0.0.0:8000->80/tcp   awesome_varahamihira
    $ curl localhost:8000/mysfits
    {"mysfits": [...]}
    $ docker logs 51a
     * Running on http://0.0.0.0:80/ (Press CTRL+C to quit)
    172.17.0.1 - - [16/Nov/2018 22:56:03] "GET /mysfits HTTP/1.1" 200 -
    INFO:werkzeug:172.17.0.1 - - [16/Nov/2018 22:56:03] "GET /mysfits HTTP/1.1" 200 -
    </pre>

    In the sample output above, the container was assigned the name "awesome_varahamihira".  Names are arbitrarily assigned.  You can also pass the docker run command a name option if you want to specify the running name.  You can read more about it in the [Docker run reference](https://docs.docker.com/engine/reference/run/).  Kill the container using `docker kill` now that we know it's working properly.

4. Now that you have a working Docker image, tag and push the image to [Elastic Container Registry (ECR)](https://aws.amazon.com/ecr/).  ECR is a fully-managed Docker container registry that makes it easy to store, manage, and deploy Docker container images. In the next lab, we'll use ECS to pull your image from ECR.

    In the AWS Management Console, navigate to [Repositories](https://console.aws.amazon.com/ecs/home#/repositories) in the ECS dashboard.  You should see repositories for the monolith service and like service.  These were created by CloudFormation and named like <code><b><i>STACK_NAME</i></b>-mono-xxx</code> and <code><b><i>STACK_NAME</i></b>-like-xxx</code> where ***STACK_NAME*** is the name of the CloudFormation stack (the stack name may be truncated).

    ![ECR repositories](images/01-ecr-repo.png)

    Click on the repository name for the monolith, and note down the Repository URI (you will use this value again in the next lab):

    ![ECR monolith repo](images/01-ecr-repo-uri.png)

    *Note: Your repository URI will be unique.*

    Tag and push your container image to the monolith repository.

    <pre>
    $ docker tag monolith-service:latest <b><i>ECR_REPOSITORY_URI</i></b>:latest
    $ docker push <b><i>ECR_REPOSITORY_URI</i></b>:latest
    </pre>

    When you issue the push command, Docker pushes the layers up to ECR.

    Here's sample output from these commands:

    <pre>
    $ docker tag monolith-service:latest 873896820536.dkr.ecr.us-east-2.amazonaws.com/mysfit-mono-oa55rnsdnaud:latest
    $ docker push 873896820536.dkr.ecr.us-east-2.amazonaws.com/mysfit-mono-oa55rnsdnaud:latest
    The push refers to a repository [873896820536.dkr.ecr.us-east-2.amazonaws.com/mysfit-mono-oa55rnsdnaud:latest]
    0f03d692d842: Pushed
    ddca409d6822: Pushed
    d779004749f3: Pushed
    4008f6d92478: Pushed
    e0c4f058a955: Pushed
    7e33b38be0e9: Pushed
    b9c7536f9dd8: Pushed
    43a02097083b: Pushed
    59e73cf39f38: Pushed
    31df331e1f23: Pushed
    630730f8c75d: Pushed
    827cd1db9e95: Pushed
    e6e107f1da2f: Pushed
    c41b9462ea4b: Pushed
    latest: digest: sha256:a27cb7c6ad7a62fccc3d56dfe037581d314bd8bd0d73a9a8106d979ac54b76ca size: 3252
    </pre>

    *Note: Typically, you'd have to log into your ECR repo. However, you did not need to authenticate docker with ECR because the [Amazon ECR Credential Helper](https://github.com/awslabs/amazon-ecr-credential-helper) has been installed and configured for you on the Cloud9 Environment.  This was done earlier when you ran the setup script. You can read more about the credentials helper in this [article](https://aws.amazon.com/blogs/compute/authenticating-amazon-ecr-repositories-for-docker-cli-with-credential-helper/).*

    If you refresh the ECR repository page in the console, you'll see a new image uploaded and tagged as latest.

    ![ECR push complete](images/01-ecr-push-complete.png)

### Checkpoint:
At this point, you should have a working container for the monolith codebase stored in an ECR repository and ready to deploy with ECS in the next lab.

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
Congratulations, you've successfully rolled out the like microservice from the monolith.  If you have time, try repeating this lab to break out the adoption microservice.  Otherwise, please remember to follow the steps below in the **Workshop Cleanup** to make sure all assets created during the workshop are removed so you do not see unexpected charges after today.

## Workshop Cleanup

This is really important because if you leave stuff running in your account, it will continue to generate charges.  Certain things were created by CloudFormation and certain things were created manually throughout the workshop.  Follow the steps below to make sure you clean up properly.

Delete manually created resources throughout the labs:

* ECS service(s) - first update the desired task count to be 0.  Then delete the ECS service itself.
* ECR - delete any Docker images pushed to your ECR repository.
* CloudWatch logs groups
* ALBs and associated target groups

Finally, [delete the CloudFormation stack](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cfn-console-delete-stack.html) launched at the beginning of the workshop to clean up the rest.  If the stack deletion process encountered errors, look at the Events tab in the CloudFormation dashboard, and you'll see what steps failed.  It might just be a case where you need to clean up a manually created asset that is tied to a resource goverened by CloudFormation.
