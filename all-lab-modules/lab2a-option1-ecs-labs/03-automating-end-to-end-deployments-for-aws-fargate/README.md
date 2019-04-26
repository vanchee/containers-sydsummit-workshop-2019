# Continuous Deployment to Amazon ECS Using AWS CodePipeline, AWS CodeBuild, and AWS CloudFormation

The ECS Continuous Deployment reference architecture demonstrates how to achieve continuous deployment of an application to Amazon EC2 Container Service (Amazon ECS) using AWS CodePipeline, AWS CodeBuild, and AWS CloudFormation. With continuous deployment, software revisions are deployed to a production environment without explicit approval from a developer, automating the entire software release process.

## Launching this AWS CloudFormation stack provisions the following

    A continuous deployment process that uses AWS CodePipeline to monitor a GitHub repository for new commits

    AWS CodeBuild to create a new Docker container image and to push it into Amazon EC2 Container Registry (Amazon ECR),

    AWS CloudFormation to deploy the new container image to production on Amazon ECS.

# Step 1: Fork and clone the GitHub repository

Fork  the Amazon ECS mythical mysfits app  GitHub repository into your GitHub account.

    Browse to the Amazon ECS mythical mysfits app.

    Choose fork from the upper right corner of the screen.

From your terminal application, run the following command (replace <your_github_username> with your GitHub username):
<pre>
git clone https://github.com/<your_github_username>/ecs-demo-php-simple-app
</pre>

This creates a directory named ecs-demo-php-simple-app in your current directory, which contains the code for the Amazon ECS sample app.

# Step 2: Create a personal access token

The personal access token is used by AWS CodePipeline to access the contents of your GitHub repository.

    Browse to https://github.com/settings/tokens.

    Choose Generate new token.

    In Token description, type a description.

    Under Select scopes, choose repo.

    Choose Generate token.

    Copy the generated token to the clipboard.

# Step 3: Create the CloudFormation stack

Choose Launch Stack to launch the template in the US-west-2 (Oregon) Region in your account.
**Oregon** (us-west-2) | [![Launch Mythical Mysfits Stack into Oregon with CloudFormation](/images/deploy-to-aws.png)](https://console.aws.amazon.com/cloudformation/home?region=us-west-2#/stacks/new?stackName=cicd-mysfits-fargate&templateURL=https://s3-us-west-2.amazonaws.com/pv-ecs-refarch-continuous-deployment/ecs-refarch-continuous-deployment.yaml) |


The CloudFormation template requires the following parameters:

    GitHub configuration

        Repo: The repo name of the sample service; for example, <github_username>/ecs-demo-php-simple-app

        Branch: The branch of the repo to deploy continuously; for example, master

        User: Your user name on GitHub

        Personal Access Token: The token for the user specified in the preceding procedure

The CloudFormation stack provides the following output:

    ServiceUrl: The sample service that is being continuously deployed.

    PipelineUrl: The continuous deployment pipeline in the AWS Management Console.

# Step 4: Testing the example

After the CloudFormation stack is created, the latest commit to the GitHub repository is run through the pipeline and deployed to Amazon ECS. Open the PipelineUrl to watch the first revision run through the AWS CodePipeline pipeline. After the deploy step turns green, open the URL from ServiceUrl, which loads a page similar to this:

![Sample_App_page](images/Sample_PHP_App.png)

To review the pipeline stages and test continuous deployment, make a change to src/index.php in the ecs-demo-php-simple-app repository and push it to GitHub.

From your terminal application, change the working directory to your local copy of the repository, and then run the following commands to push the changes to your remote repository:

    git add .

    git commit –m “changes to index.php”

    git push origin master  
    
AWS CodePipeline detects the change, builds the new application, and deploys it to your cluster automatically. After the pipeline finishes deploying the revision, refresh the page to see your changes.

This pipeline is now in production, listening for new code in the source code repository, and ready to ship future changes that your team pushes into production. The pipeline is extensible, meaning that you can add stages to include additional steps. For example, add a test stage to run unit and acceptance tests. This helps ensure new code revisions are safe to deploy to production. You can also include a notification step that alerts your team, through email or a Slack channel, that a new version is live. The notification can include details about the change that was deployed to production.

# Once complete, please remember to follow the steps below in the **Workshop Cleanup** to make sure all assets created during the workshop are removed so you do not see unexpected charges after today.

## Workshop Cleanup

This is really important because if you leave stuff running in your account, it will continue to generate charges.  Certain things were created by CloudFormation and certain things were created manually throughout the workshop.  Follow the steps below to make sure you clean up properly.

Delete any manually created resources throughout the labs:
* Delete the CI-CD Cloud Formation Master Template which will delete any nested stacks and all assosiated resources 
* ECS service(s) - first update the desired task count to be 0.  Then delete the ECS service itself.
* ECR - delete any Docker images pushed to your ECR repository.
* CloudWatch logs groups
* ALBs and associated target groups

Finally, [delete the CloudFormation stack](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cfn-console-delete-stack.html) launched at the beginning of the workshop to clean up the rest.  If the stack deletion process encountered errors, look at the Events tab in the CloudFormation dashboard, and you'll see what steps failed.  It might just be a case where you need to clean up a manually created asset that is tied to a resource goverened by CloudFormation.
