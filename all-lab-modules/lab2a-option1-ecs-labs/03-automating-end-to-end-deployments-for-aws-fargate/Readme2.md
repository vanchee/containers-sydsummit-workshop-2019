##Continuous Deployment to Amazon ECS Using AWS CodePipeline, AWS CodeBuild, and AWS CloudFormation

The ECS Continuous Deployment reference architecture demonstrates how to achieve continuous deployment of an application to Amazon EC2 Container Service (Amazon ECS) using AWS CodePipeline, AWS CodeBuild, and AWS CloudFormation. With continuous deployment, software revisions are deployed to a production environment without explicit approval from a developer, automating the entire software release process.

##Launching this AWS CloudFormation stack provisions the following

    A continuous deployment process that uses AWS CodePipeline to monitor a GitHub repository for new commits

    AWS CodeBuild to create a new Docker container image and to push it into Amazon EC2 Container Registry (Amazon ECR),

    AWS CloudFormation to deploy the new container image to production on Amazon ECS.

#Prerequisites

This getting started guide uses Git to clone and push files to and from GitHub repositories. You can install the Git command line tools by following the instructions at https://git-scm.com/downloads, or by using a package manager like NuGet or Homebrew.
Step 1: Fork and clone the GitHub repository

Fork  the Amazon ECS mythical mysfits app  GitHub repository into your GitHub account.

    Browse to the Amazon ECS mythical mysfits app.

    Choose fork from the upper right corner of the screen.

From your terminal application, run the following command (replace <your_github_username> with your GitHub username):
<pre>
git clone https://github.com/<your_github_username>/ecs-demo-php-simple-app
</pre>
This creates a directory named ecs-demo-php-simple-app in your current directory, which contains the code for the Amazon ECS sample app.
Step 2: Create a personal access token

The personal access token is used by AWS CodePipeline to access the contents of your GitHub repository.

    Browse to https://github.com/settings/tokens.

    Choose Generate new token.

    In Token description, type a description.

    Under Select scopes, choose repo.

    Choose Generate token.

    Copy the generated token to the clipboard.

Step 3: Create the CloudFormation stack

Choose Launch Stack to launch the template in the US East (N.Virginia) Region in your account. The CloudFormation template requires the following parameters:

    GitHub configuration

        Repo: The repo name of the sample service; for example, <github_username>/ecs-demo-php-simple-app

        Branch: The branch of the repo to deploy continuously; for example, master

        User: Your user name on GitHub

        Personal Access Token: The token for the user specified in the preceding procedure

The CloudFormation stack provides the following output:

    ServiceUrl: The sample service that is being continuously deployed.

    PipelineUrl: The continuous deployment pipeline in the AWS Management Console.

Step 4: Testing the example