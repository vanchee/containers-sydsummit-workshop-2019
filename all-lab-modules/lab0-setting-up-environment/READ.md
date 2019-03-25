## Let's Begin!

### Workshop Setup:

1. Open the CloudFormation launch template link below in a new tab. The link will load the CloudFormation Dashboard and start the stack creation process in the chosen region:
   
    Click on one of the **Deploy to AWS** icons below to region to stand up the core workshop infrastructure.

| Region | Launch Template |
| ------------ | ------------- | 
| **Oregon** (us-west-2) | [![Launch Mythical Mysfits Stack into Oregon with CloudFormation](/images/deploy-to-aws.png)](https://console.aws.amazon.com/cloudformation/home?region=us-west-2#/stacks/new?stackName=mythical-mysfits-fargate&templateURL=https://s3.amazonaws.com/mythical-mysfits-website/fargate/core.yml) | 
|**Singapore** (ap-southeast-1) | [![Launch Mythical Mysfits Stack into Sydney with CloudFormation](/images/deploy-to-aws.png)](https://console.aws.amazon.com/cloudformation/home?region=ap-southeast-2#/stacks/new?stackName=mythical-mysfits-fargate&templateURL=https://s3.amazonaws.com/mythical-mysfits-website/fargate/core.yml) |


2. The template will automatically bring you to the CloudFormation Dashboard and start the stack creation process in the specified region. Give the stack a name that is unique within your account, and proceed through the wizard to launch the stack. Leave all options at their default values, but make sure to check the box to allow CloudFormation to create IAM roles on your behalf:

    ![IAM resources acknowledgement](images/00-cf-create.png)

    See the *Events* tab for progress on the stack launch. You can also see details of any problems here if the launch fails. Proceed to the next step once the stack status advances to "CREATE_COMPLETE".

3. Access the AWS Cloud9 Environment created by CloudFormation:

    On the AWS Console home page, type **Cloud9** into the service search bar and select it. Find the environment named like "Project-***STACK_NAME***":

    ![Cloud9 project selection](images/00-cloud9-select.png)

    When you open the IDE, you'll be presented with a welcome screen that looks like this:
    ![cloud9-welcome](images/00-cloud9-welcome.png)

    On the left pane (Blue), any files downloaded to your environment will appear here in the file tree. In the middle (Red) pane, any documents you open will show up here. Test this out by double clicking on README.md in the left pane and edit the file by adding some arbitrary text. Then save it by clicking File and Save. Keyboard shortcuts will work as well. On the bottom, you will see a bash shell (Yellow). For the remainder of the lab, use this shell to enter all commands. You can also customize your Cloud9 environment by changing themes, moving panes around, etc. (if you like the dark theme, you can select it by clicking the gear icon in the upper right, then "Themes", and choosing the dark theme).

4. Clone the Mythical Mysfits Workshop Repository:

    In the bottom panel of your new Cloud9 IDE, you will see a terminal command line terminal open and ready to use.  Run the following git command in the terminal to clone the necessary code to complete this tutorial:

    ```
    $ git clone https://github.com/aws-samples/amazon-ecs-mythicalmysfits-workshop.git
    ```

    After cloning the repository, you'll see that your project explorer now includes the files cloned.

    In the terminal, change directory to the subdirectory for this workshop in the repo:

    ```
    $ cd amazon-ecs-mythicalmysfits-workshop/workshop-1
    ```

5. Run some additional automated setup steps with the `setup` script:

    ```
    $ script/setup
    ```

    This script will delete some unneeded Docker images to free up disk space, populate a DynamoDB table with some seed data, upload site assets to S3, and install some Docker-related authentication mechanisms that will be discussed later. Make sure you see the "Success!" message when the script completes.


### Checkpoint:
At this point, the Mythical Mysfits website should be available at the static site endpoint for the S3 bucket created by CloudFormation. You can visit the site at <code>http://<b><i>BUCKET_NAME</i></b>.s3-website.<b><i>REGION</i></b>.amazonaws.com/</code>. You can find the ***BUCKET_NAME*** in the CloudFormation outputs saved in the file `workshop-1/cfn-outputs.json`. Check that you can view the site, but there won't be much content visible yet until we launch the Mythical Mysfits monolith service:

![initial website](images/00-website.png)

###########################

### EKS Cluster Setup (Follow only if you are an EKS lab user):

In order for you to succeed in this workshop, we need you to run through a few steps to finalize the configuration of your Cloud9 environment.

=== Update and install some tools.
The first step is to update the `AWS CLI`,`pip` and a range of pre-installed packages.
[source,shell]
----
sudo yum update -y && pip install --upgrade --user awscli pip
exec $SHELL
----

=== Configure the AWS Environment
After you have the installed the latest `awscli` and `pip` we need to configure
our environment a little
[source,shell]
----
aws configure set region us-west-2
----

=== AWS Workshop Portal

NOTE: If you are running this workshop using the provided AWS Workshop Portal
environment, following the steps below, if not skip to *Clone the source
repository for this workshop*.

First thing we need to do is update the IAM Instance Profile associated with the
Cloud9 environment.

[source,shell]
----
aws ec2 associate-iam-instance-profile \
  --instance-id $(aws ec2 describe-instances \
    --filters Name=tag:Name,Values="*cloud9*" \
    --query Reservations[0].Instances[0].InstanceId \
    --output text) --iam-instance-profile Name="cl9-workshop-role"
----

Following that we'll need to turn off the Managed Temporary credentials.

The Cloud9 IDE needs to use the assigned IAM Instance profile. Open the *AWS
Cloud9* menu, go to *Preferences*, go to *AWS Settings*, and disable *AWS
managed temporary credentials* as depicted in the diagram here:

image::cloud9-credentials.png[Cloud9 Managed Credentials]



=== Spin-up an EKS cluster

=== Installing 3rd Party CLIs
During the workshop we will be using a couple of third party `CLI` tools like `eksctl` to configure our EKS cluster, and `kubectl` to interact with our Kubernetes cluster.

Step 1::
Let's make a `bin` folder where these binaries will be stored and change our directory to the new location.
[source,shell]
----
mkdir ~/bin
cd ~/bin
----

Step 2::
Download the required `CLI` tools from their locations. Make them Runnable, and add them to our `$PATH`
[source,shell]
----
curl -skL "https://github.com/weaveworks/eksctl/releases/download/latest_release/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp && mv /tmp/eksctl ~/bin/
curl -skLo ~/bin/kubectl "https://amazon-eks.s3-us-west-2.amazonaws.com/1.10.3/2018-06-05/bin/linux/amd64/kubectl"
curl -skLo ~/bin/heptio-authenticator-aws "https://github.com/kubernetes-sigs/aws-iam-authenticator/releases/download/v0.3.0/heptio-authenticator-aws_0.3.0_linux_amd64"
chmod -R +x ~/bin
echo "export PATH=~/bin:\${PATH}" >> ~/.bashrc
exec $SHELL
----

=== Launch an EKS Cluster
It takes a few minutes to launch an EKS cluster, so we will have you launch one now, while completing some initial modules. We will launch our EKS Cluster using the `eksctl` tool

Step 1::
`eksctl` requires a SSH Key to configure your EKS nodes with.
[source,shell]
----
ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa
----

Step 2::
Now that we have the key, let's launch the EKS Cluster.
[source,shell]
----
eksctl create cluster --full-ecr-access --name=mythicalmysfits
----

Step 3::
You can expect to see an output like the one below...
[.output]
....
[ℹ]  using region us-west-2
[ℹ]  setting availability zones to [us-west-2a us-west-2c us-west-2d]
[ℹ]  subnets for us-west-2a - public:192.168.0.0/19 private:192.168.96.0/19
[ℹ]  subnets for us-west-2c - public:192.168.32.0/19 private:192.168.128.0/19
[ℹ]  subnets for us-west-2d - public:192.168.64.0/19 private:192.168.160.0/19
[ℹ]  nodegroup "ng-8897c340" will use "ami-0c28139856aaf9c3b" [AmazonLinux2/1.11]
[ℹ]  creating EKS cluster "mythicalmysfitseks" in "us-west-2" region
[ℹ]  will create 2 separate CloudFormation stacks for cluster itself and the initial nodegroup
[ℹ]  if you encounter any issues, check CloudFormation console or try 'eksctl utils describe-stacks --region=us-west-2 --name=mythicalmysfitseks'
[ℹ]  building cluster stack "eksctl-mythicalmysfitseks-cluster"
[ℹ]  creating nodegroup stack "eksctl-mythicalmysfitseks-nodegroup-ng-8897c340"
[ℹ]  --nodes-min=2 was set automatically for nodegroup ng-8897c340
[ℹ]  --nodes-max=2 was set automatically for nodegroup ng-8897c340
[✔]  all EKS cluster resource for "mythicalmysfitseks" had been created
[✔]  saved kubeconfig as "/home/ec2-user/.kube/config"

We will leave this process running, and get back to it later in the workshop. So let's open a new `terminal` by pressing the combination keys `alt+t`


[*^ back to top*]
