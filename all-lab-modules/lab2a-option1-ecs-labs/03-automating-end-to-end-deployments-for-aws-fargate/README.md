# Mythical Mysfits: DevSecOps with Docker and AWS Fargate

## Lab - Automating End to End Deployments for AWS Fargate for production environments

In this lab, you will implement the end to end deployment and testing process for your like service. This lab is where it all comes together. By the end, you will be able to check in new code and have your application automatically updated.

Here's what you'll be doing:

* Create new buildspec for production environments
* Create Pipeline for deployments
* Deploy new version of Project Cuddle

1\. Configure Git credentials

Since most of the labs are going to be using git, let's set up our permissions now. There are a number of ways to authenticate with git repositories, and specifically CodeCommit in this case, but for the sake of simplicity, we'll use the CodeCommit credential helper here. Enter the following commands to configure git to access CodeCommit.

<pre>
$ git config --global credential.helper "cache --timeout=7200"
$ git config --global user.email "<b><i>REPLACEWITHYOUREMAIL</i></b>"
$ git config --global user.name "<b><i>REPLACEWITHYOURNAME</i></b>"
$ git config --global credential.helper '!aws codecommit credential-helper $@'
$ git config --global credential.UseHttpPath true
</pre>

## Lab - Starting the DevSecOps Journey

In this lab, we are going to start building in DevSecOps. Security is everyone's responsibility and in today, you will ensure that you aren't checking in any AWS secrets like AWS Access and Secret Keys.

Here's what you'll be doing:

* [Set up repos](#set-up-repos)
* [Build security right into git commits](#build-security-right-into-git-commits)
* [Remediation](#rsemediation)

### Set up repos

1\. Clone all repos

Up until now, Mythical Mysfits hasn't really been doing anything with source repos, so let's start checking things into repos like we're supposed to. First, you'll create 2 repositories. We'll do it from the CLI this time.

<pre>
$ aws codecommit create-repository --repository-name mythical-mysfits-devsecops-like-service --repository-description "mythical mysfits devsecops like service"

$ aws codecommit create-repository --repository-name mythical-mysfits-devsecops-monolith-service --repository-description "mythical mysfits devsecops monolith service"

</pre>

Next, use the batch-get-repositories command to get the clone URLs for both repositories, substituting the names you got from the previous CLI command:

<pre>
$ aws codecommit batch-get-repositories --repository-names mythical-mysfits-devsecops-monolith-service mythical-mysfits-devsecops-like-service
{
    "repositories": [
        {
            "repositoryName": "mythical-mysfits-devsecops-monolith-service",
            "cloneUrlSsh": "ssh://git-codecommit.eu-west-1.amazonaws.com/v1/repos/mythical-mysfits-devsecops-monolith-service",
            "lastModifiedDate": 1542588318.447,
            "repositoryDescription": "Repository for the Mythical Mysfits monolith service",
            "cloneUrlHttp": "https://git-codecommit.eu-west-1.amazonaws.com/v1/repos/mythical-mysfits-devsecops-monolith-service",
            "creationDate": 1542588318.447,
            "repositoryId": "c8aa761e-3ed1-4033-b830-4d9465b51087",
            "Arn": "arn:aws:codecommit:eu-west-1:123456789012:mythical-mysfits-devsecops-monolith-service",
            "accountId": "123456789012"
        },
        {
            "repositoryName": "mythical-mysfits-devsecops-like-service",
            "cloneUrlSsh": "ssh://git-codecommit.eu-west-1.amazonaws.com/v1/repos/mythical-mysfits-devsecops-like-service",
            "lastModifiedDate": 1542500073.535,
            "repositoryDescription": "Repository for the Mythical Mysfits like service",
            "cloneUrlHttp": "https://git-codecommit.eu-west-1.amazonaws.com/v1/repos/mythical-mysfits-devsecops-like-service",
            "creationDate": 1542500073.535,
            "repositoryId": "54763f98-c295-4189-a91a-7830ea085aae",
            "Arn": "arn:aws:codecommit:eu-west-1:123456789012:mythical-mysfits-devsecops-like-service",
            "accountId": "123456789012"
        }
    ],
    "repositoriesNotFound": []
}
</pre>

2\. Clone repos and copy in app code

Earlier in the workshop, we set up the CodeCommit credential helper, so we'll use the HTTPS clone URLs instead of SSH.

<pre>
$ cd ~/environment/
$ git clone <b><i>REPLACEME_LIKE_REPOSITORY_CLONEURL</b></i>
$ git clone <b><i>REPLACEME_MONOLITH_REPOSITORY_CLONEURL</b></i>
$ cp -R ~/environment/amazon-ecs-mythicalmysfits-workshop/workshop-2/app/like-service/* <b><i>REPLACEME_LIKE_REPOSITORY_NAME</b></i>
$ cp -R ~/environment/amazon-ecs-mythicalmysfits-workshop/workshop-2/app/monolith-service/* <b><i>REPLACEME_MONOLITH_REPOSITORY_NAME</b></i>
</pre>


# Mythical Mysfits: DevSecOps with Docker and AWS Fargate

## Lab 2 - Offloading Builds to AWS CodeBuild

In this lab, you will start the process of automating the entire software delivery process. The first step we're going to take is to automate the Docker container builds and push the container image into the Elastic Container Registry. This will allow you to develop and not have to worry too much about build resources. We will use AWS CodeCommit and AWS CodeBuild to automate this process. Then, we'll create a continuous delivery pipeline for our Like service in AWS Fargate.

You may be thinking, why would I want to offload my builds when I could just do it on my local machine. Well, this is going to be part of your full production pipeline. We'll use the same build system process as you will for production deployments. In the event that something is different on your local machine as it is within the full dev/prod pipeline, this will catch the issue earlier. You can read more about this by looking into **[Shift Left](https://en.wikipedia.org/wiki/Shift_left_testing)**.

Here's a reference architecture for what you'll be building:

![CodeBuild Create](images/arch-codebuild.png)

Here's what you'll be doing:

* [Create AWS CodeBuild Project](#create-aws-codebuild-project)
* [Create BuildSpec File](#create-buildspec-file)
* [Test your AWS CodeBuild Project](#test-your-aws-codebuild-project)

### Create AWS CodeBuild Project

1\. Create and configure an AWS CodeBuild project.

We will be using AWS CodeBuild to offload the builds from the local Cloud9 instance. Let's create the AWS CodeBuild project. In the AWS Management Console, navigate to the [AWS CodeBuild dashboard](https://console.aws.amazon.com/codebuild/home). Click on **Create build project**.

On the **Create build project** page, enter in the following details:

- Project Name: Enter `prod-like-service-build`
- Source Provider: Select **AWS CodeCommit**
- Repository: Choose the repo from the CloudFormation stack that looks like StackName-**like-service**

**Environment:**

![CodeBuild Create Project Part 2](images/cb-create-project-2.png)

- Environment Image: Select **Managed Image** - *There are two options. You can either use a predefined Docker container that is curated by CodeBuild, or you can upload your own if you want to customize dependencies etc. to speed up build time*
- Operating System: Select **Ubuntu** - *This is the OS that will run your build*
- Runtime: Select **Docker** - *Each image has specific versions of software installed. See [Docker Images Provided by AWS CodeBuild](http://docs.aws.amazon.com/codebuild/latest/userguide/build-env-ref-available.html)*
- Runtime version: Select **aws/codebuild/docker:17.09.0** - *This will default to the latest*
- Image version: **Leave as is**
- Privileged: **Leave as is** - *You can't actually change anything here. In order for to run Docker inside a Docker container, you need to have elevated privileges*
- Service role: **New service role** - *A service role was automatically created for you via CFN*
- Role name: Choose New service role **New service role** that has the name of the CFN stack you created previously*
- Uncheck **Allow AWS CodeBuild to modify this service role so it can be used with this build project**

![CodeBuild Create Project Part 1](images/cb-create-project-1.png)

Expand the **Additional Information** and enter the following in Environment Variables:

- Name: `AWS_ACCOUNT_ID` - *Enter this string*
- Value: ***`REPLACEME_YOUR_ACCOUNT_ID`*** - *This is YOUR account ID*

**Buildspec:**

- Build Specification: Select **Use a buildspec file** - *We are going to provide CodeBuild with a buildspec file*
- Buildspec name: Enter `buildspec_prod.yml` - *we'll be using the same repo, but different buildspecs*


**Artifacts:**

- Type: Select **No artifacts** *If there are any build outputs that need to be stored, you can choose to put them in S3.*

Click **Create build project**.

### Create BuildSpec File

1\. Create BuildSpec file

AWS CodeBuild uses a definition file called a buildspec Yaml file. The contents of the buildspec will determine what AWS actions CodeBuild should perform. The key parts of the buildspec are Environment Variables, Phases, and Artifacts. See [Build Specification Reference for AWS CodeBuild](http://docs.aws.amazon.com/codebuild/latest/userguide/build-spec-ref.html) for more details.

**At Mythical Mysfits, we want to follow best practices, so there are 2 requirements:**

1. We don't use the ***latest*** tag for Docker images. We have decided to use the Commit ID from our source control instead as the tag so we know exactly what image was deployed.

2. We want to use multiple buildspec files. One for dev, one for test, one for prod.

Another developer from the Mythical Mysfits team has started a buildspec_dev file for you, but never got to finishing it. Add the remaining instructions to the buildspec_dev.yml.draft file. The file should be in your like-service folder and already checked in. Let's create a dev branch and copy the draft to a buildspec_dev.yml file.

<pre>
$ cd ~/environment/<b><i>REPLACEME_LIKE_REPO_NAME</b></i>
$ git checkout -b dev
$ cp ~/environment/amazon-ecs-mythicalmysfits-workshop/workshop-2/Lab-2/hints/buildspec_dev.yml.draft buildspec_dev.yml
</pre>

Now that you have a copy of the draft as your buildspec, you can start editing it. The previous developer left comments indicating what commands you need to add (<b>These comments look like - #[TODO]:</b>). Add the remaining instructions to your buildspec_dev.yml.

Here are links to documentation and hints to help along the way. If you get stuck, look at the [hintspec_dev.yml](hints/hintspec_dev.yml) file in the hints folder:

<pre>
<b>#[TODO]: Command to log into ECR. Remember, it has to be executed $(maybe like this?)</b>

- http://docs.aws.amazon.com/AmazonECR/latest/userguide/Registries.html
- https://docs.aws.amazon.com/codebuild/latest/userguide/sample-docker.html#sample-docker-files

<b>#[TODO]: Build the actual image using the current commit ID as the tag...perhaps there's a CodeBuild environment variable we can use. Remember that we also added two custom environment variables into the CodeBuild project previously: AWS_ACCOUNT_ID. How can you use this?</b>

- https://docs.docker.com/get-started/part2/#build-the-app
- https://docs.aws.amazon.com/codebuild/latest/userguide/build-env-ref-env-vars.html

<b>#[TODO]: Tag the newly built Docker image so that we can push the image to ECR. See the instructions in your ECR console to find out how to do this. Make sure you use the current commit ID as the tag!</b>

<b>#[TODO]: Push the Docker image up to ECR</b>

- https://docs.aws.amazon.com/AmazonECR/latest/userguide/docker-push-ecr-image.html
- https://docs.docker.com/engine/reference/builder/#entrypoint
</pre>

<details>
  <summary>
    HINT: Click here for the completed buildspec.yml file.
  </summary>
  There are many ways to achieve what we're looking for. In this case, the buildspec looks like this:
<pre>
version: 0.2

phases:
  pre_build:
    commands:
      - echo Logging in to Amazon ECR...
      - REPOSITORY_URI=REPLACEME_REPO_URI # This was started. Just replace REPLACEME_REPO_URI with your ECR Repo URI
      - $(aws ecr get-login --no-include-email --region $AWS_DEFAULT_REGION) # <b><i>This is the login command from earlier</i></b>
  build:
    commands:
      - echo Build started on `date`
      - echo Building the Docker image...
      - docker build -t $REPOSITORY_URI:$CODEBUILD_RESOLVED_SOURCE_VERSION . # <b><i>There are a number of variables that are available directly in the CodeBuild build environment. We specified IMAGE_REPO_NAME earlier, but CODEBUILD_SOURCE_VERSION is there by default.</i></b>
      - docker tag $REPOSITORY_URI:$CODEBUILD_RESOLVED_SOURCE_VERSION $REPOSITORY_URI:$CODEBUILD_RESOLVED_SOURCE_VERSION # <b><i>This is the tag command from earlier</i></b>
  post_build:
    commands:
      - echo Build completed on `date`
      - echo Pushing the Docker image...
      - docker push $REPOSITORY_URI:$CODEBUILD_RESOLVED_SOURCE_VERSION # <b><i>This is the push command from earlier</i></b>
</pre>
<br/>

You can copy a pre-created one into your application directory. If you do, make sure you replace the REPOSITORY_URI with the one from your like-service ECR repository!
<pre>
$ cp ~/environment/amazon-ecs-mythicalmysfits-workshop/workshop-2/Lab-2/hints/hintspec_dev.yml buildspec_dev.yml
</pre>

</details>

When we created the buildspec_dev.yml file, we used CODEBUILD_RESOLVED_SOURCE_VERSION. What is CODEBUILD_RESOLVED_SOURCE_VERSION and why didn't we just use CODEBUILD_SOURCE_VERSION? You can find out in the [Environment Variables for Build Environments](http://docs.aws.amazon.com/codebuild/latest/userguide/build-env-ref-env-vars.html) documentation.

<details>
  <summary>
    HINT: Click here for a spoiler!
  </summary>
    For Amazon S3, the version ID associated with the input artifact. For AWS CodeCommit, the commit ID or branch name associated with the version of the source code to be built. For GitHub, the commit ID, branch name, or tag name associated with the version of the source code to be built. Since we will be triggering the build from CLI, the source version passed to CodeBuild will be 'dev', so that's what would show up if you use CODEBUILD_SOURCE_VERSION. Since we are using CODEBUILD_RESOLVED_SOURCE_VERSION, you will see the actual HEAD commit ID for our dev branch.<br />

</details>

### Create new buildspec for prod environments

1\. Create the buildspec_prod.yml file

In the above section, we created a buildspec for dev named buildspec_dev. That was used when CodeBuild was run directly on source code in CodeCommit, but now for production we want to build a full pipeline that will automatically deploy our environment, so we'll use CodePipeline to orchestrate that.

Ideally, we want to keep production and development branches as similar as possible, but want to control differences between dev and prod build stages. To start, we can basically copy over what we created for Lab 2, but there will be a few minor changes. We will name the new buildspec buildspec_prod.yml instead of buildspec_dev.yml.

Make sure you're in the like repository folder, which should be named something like **mythical-mysfits-devops-like-service**.

<pre>
$ cd ~/environment/<b>REPLACE_ME_LIKE_REPOSITORY_NAME</b>
$ cp buildspec_dev.yml buildspec_prod.yml
</pre>

Next, in order for CodePipeline to deploy to Fargate, we need to have an `imagedefinitions.json` file that includes the name of the container we want to replace as well as the imageUri. Then we have to surface the file to CodePipeline in an Artifacts section. The end of your buildspec_prod.yml file will look like this:

<pre>
...
    post_build:
      commands:
        - echo Build completed on `date`
        - echo Pushing the Docker image...
        - docker push $REPOSITORY_URI:$CODEBUILD_RESOLVED_SOURCE_VERSION
        - printf '[{"name":"REPLACEME_CONTAINERNAME","imageUri":"%s"}]' $REPOSITORY_URI:$CODEBUILD_RESOLVED_SOURCE_VERSION > imagedefinitions.json
artifacts:
  files: imagedefinitions.json
</pre>

Replace the container name with the name of your service, which should be `like-service`.

<details>
<summary> HINT: There's also completed file in hints/hintspec_prod.yml. Click here to see how to copy it in.</summary>
  Make sure you change REPLACEME_REPO_URI to your ECR repository URI!
  <pre>
  $ cp ~/environment/amazon-ecs-mythicalmysfits-workshop/workshop-2/Lab-3/hints/buildspec_prod.yml ~/environment/REPLACEME_REPO_NAME/buildspec_prod.yml
  </pre>
</details>

2\. Check in and push to dev

Add, commit, and push the new file to your repo. You can try to build the app again, but CodeBuild will just do the same thing because it's still looking at buildspec_prod.yml.

<pre>
  $ git add buildspec_prod.yml
  $ git commit -m "Adding a buildspec for prod"
  $ git push origin prod
</pre>

### Create Pipeline for deployments

1\. Create an AWS CodePipeline Pipeline and set it up to listen to AWS CodeCommit.

Now it's time to hook everything together. In the AWS Management Console, navigate to the [AWS CodePipeline](https://console.aws.amazon.com/codepipeline/home#/) dashboard. Click on **Create Pipeline**.

On the following pages, enter the following details:

**Choose pipeline settings:**

- Pipeline name: `prod-like-service` - *This is a production pipeline, so we'll prefix with prod*
- Service role: **Existing service role** - *A service role was automatically created for you via CFN*
- Role name: Choose **CFNStackName-CodeBuildServiceRole** - *Look for the service role that has the name of the CFN stack you created previously*
- Artifact store: Choose **Custom location** - *An artifact bucket was created for you via CFN*
- Bucket: Choose **CFNStackName-mythicalartifactbucket** - *Look for the artifact bucket that has the name of the CFN stack you created previously. Note that there are two buckets that were created for you. Look for the one that says mythicalartifactbucket*

Click **Next**

![CodePipeline Choose pipeline settings](images/cp-create-name.png)

**Add source stage:**

- Source provider: **AWS CodeCommit** - *We checked in our code to CodeCommit, so that's where we'll start. CodePipeline also supports a variety of different source providers. Try them out later!*
- Repository name: **CFNStackName-like-service** - *Name of your CodeCommit repo for the like-service*
- Branch name: **master** - *We want this to automatically trigger when we push to the master branch in our repo*
- Change detection options: **Amazon CloudWatch Events (recommended)** - *You have the option of using CodePipeline to poll CodeCommit for changes every minute, but using CloudWatch Events will trigger CodePipeline executions based on events, so it's much faster*

Click **Next**.

![CodePipeline Add source stage](images/cp-add-source.png)

**Add build stage:**

- Build provider: **AWS CodeBuild**
- Project name: Click **Create a new build project**

A new window should appear. The values here are almost identical to that of Lab-2 when you created your dev CodeBuild project, with the exception that the name is now prod-like-service-build and the buildspec will be buildspec_prod.yml. See Lab-2 instructions for detailed screenshots.

**Create build project:**

- Project name: `prod-like-service-build`
- Environment Image: Select **Managed Image** - *There are two options. You can either use a predefined Docker container that is curated by CodeBuild, or you can upload your own if you want to customize dependencies etc. to speed up build time*
- Operating System: Select **Ubuntu** - *This is the OS that will run your build*
- Runtime: Select **Docker** - *Each image has specific versions of software installed. See [Docker Images Provided by AWS CodeBuild](https://docs.aws.amazon.com/codebuild/latest/userguide/build-env-ref-available.html)*
- Runtime version: Select **aws/codebuild/docker:17.09.0** - *This will default to the latest*
- Image version: **Leave as is**
- Privileged: **Leave as is** - *You can't actually change anything here. In order for to run Docker inside a Docker container, you need to have elevated privileges*
- Service role: **Existing service role** - *A service role was automatically created for you via CFN*
- Role name: Choose **CFNStackName-CodeBuildServiceRole** - *Look for the service role that has the name of the CFN stack you created previously. It will be in the form of **CFNStackName**-CodeBuildServiceRole*

- Uncheck **Allow AWS CodeBuild to modify this service role so it can be used with this build project**

Expand the **Additional Information** and enter the following in Environment Variables:

- Name: `AWS_ACCOUNT_ID` - *Enter this string*
- Value: ***`REPLACEME_YOUR_ACCOUNT_ID`*** - *This is YOUR account ID*

**Buildspec:**

- Build Specification: Select **Use a buildspec file** - *We are going to provide CodeBuild with a buildspec file*
- Buildspec name: Enter `buildspec_prod.yml` - *Using our new buildspec*

Once confirmed, click **Continue to CodePipeline**. This should close out the popup and tell you that it **successfully created prod-like-service-build in CodeBuild.**

- Project Name: **prod-like-service-build**

Click **Next**.

![CodePipeline Created Build Project](images/cp-create-cb-complete.png)

**Add deploy stage:**

- Deployment provider: Select **Amazon ECS** - *This is the mechanism we're choosing to deploy with. CodePipeline also supports several other deployment options, but we're using ECS directly in this case.*
- Cluster Name: Select your ECS Cluster. In my case, **Cluster-mythical-mysfits-devsecops** - *This is the cluster that CodePipeline will deploy into.*
- Service Name: Enter `CFNStackName-Mythical-like-service` - *Name the CloudFormation stack that you're going to create/update*
- Image definitions file - *optional*: Enter `imagedefinitions.json` - *This is the file we created within the buildspec_prod.yml file earlier*

![CodePipeline ECS](images/cp-deploy-step.png)

Click **Next**.

Review your details and click **Create Pipeline**.

5\. Test your pipeline.

By default, when you create your pipeline, CodePipeline will automatically run through and try to deploy your application. If you see it go through all three stages with all GREEN, you're good to go. Otherwise, click into the links it gives you and troubleshoot the deployment.

![CodePipeline ECS](images/cp-deploy-success.png)

## Deploy new version of Project Cuddle

Now that you have your application deploying automatically, let's deploy a new version! We've upgraded the health check for our like service to make sure it can connect to the monolith service for the fulfillment method.

<pre>
$ cd ~/environment/REPLACEME_LIKE_REPO
$ cp ~/environment/amazon-ecs-mythicalmysfits-workshop/workshop-2/Lab-3/mysfits_like_v2.py service/mysfits_like.py
$ git add service/mysfits_like.py
$ git commit -m "Cuddles v2"
$ git push origin master
</pre>

Now sit back, relax, and watch the deployment. When it's done, congratulations! You've unlocked the automated build and deploy achievement!
