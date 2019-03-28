---
title: "CI/CD with CodePipeline"
chapter: true
weight: 42
draft: false
---

# CI/CD with CodePipeline

[Continuous integration](https://aws.amazon.com/devops/continuous-integration/) (CI) and [continuous delivery](https://aws.amazon.com/devops/continuous-delivery/) (CD)
are essential for deft organizations. Teams are more productive when they can make discrete changes frequently, release those changes programmatically and deliver updates
without disruption.

In this module, we will build a CI/CD pipeline using [AWS CodePipeline](https://aws.amazon.com/codepipeline/). The CI/CD pipeline will deploy a sample Kubernetes service,
we will make a change to the GitHub repository and observe the automated delivery of this change to the cluster.

---
title: "Create IAM Role"
date: 2018-10-087T08:30:11-07:00
weight: 10
draft: false
---

In an AWS CodePipeline, we are going to use AWS CodeBuild to deploy a sample Kubernetes service.
This requires an [AWS Identity and Access Management](https://aws.amazon.com/iam/) (IAM) role capable of interacting
with the EKS cluster.

In this step, we are going to create an IAM role and add an inline policy that we will use in the CodeBuild stage
to interact with the EKS cluster via kubectl.

Create the role:

```
cd ~/environment

ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

TRUST="{ \"Version\": \"2012-10-17\", \"Statement\": [ { \"Effect\": \"Allow\", \"Principal\": { \"AWS\": \"arn:aws:iam::${ACCOUNT_ID}:root\" }, \"Action\": \"sts:AssumeRole\" } ] }"

echo '{ "Version": "2012-10-17", "Statement": [ { "Effect": "Allow", "Action": "eks:Describe*", "Resource": "*" } ] }' > /tmp/iam-role-policy

aws iam create-role --role-name EksWorkshopCodeBuildKubectlRole --assume-role-policy-document "$TRUST" --output text --query 'Role.Arn'

aws iam put-role-policy --role-name EksWorkshopCodeBuildKubectlRole --policy-name eks-describe --policy-document file:///tmp/iam-role-policy
```

---
title: "Modify aws-auth ConfigMap"
date: 2018-10-087T08:30:11-07:00
weight: 11
draft: false
---

Now that we have the IAM role created, we are going to add the role to the [aws-auth ConfigMap](https://docs.aws.amazon.com/eks/latest/userguide/add-user-role.html)
for the EKS cluster.

Once the ConfigMap includes this new role, kubectl in the CodeBuild stage of the pipeline will be able to interact with the EKS cluster via the IAM role.

```
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

ROLE="    - rolearn: arn:aws:iam::$ACCOUNT_ID:role/EksWorkshopCodeBuildKubectlRole\n      username: build\n      groups:\n        - system:masters"

kubectl get -n kube-system configmap/aws-auth -o yaml | awk "/mapRoles: \|/{print;print \"$ROLE\";next}1" > /tmp/aws-auth-patch.yml

kubectl patch configmap/aws-auth -n kube-system --patch "$(cat /tmp/aws-auth-patch.yml)"
```


{{% notice tip %}}
If you would like to edit the aws-auth ConfigMap manually, you can run: $ kubectl edit -n kube-system configmap/aws-auth
{{% /notice %}}

---
title: "Fork Sample Repository"
date: 2018-10-087T08:30:11-07:00
weight: 12
draft: false
---

We are now going to [fork](https://help.github.com/articles/fork-a-repo/) the sample Kubernetes service
so that we will be able modify the repository and trigger builds.

Login to GitHub and fork the sample service to your own account:

[https://github.com/rnzsgh/eks-workshop-sample-api-service-go](https://github.com/rnzsgh/eks-workshop-sample-api-service-go)

![GitHub Fork](/images/codepipeline/github_fork.png)

Once the repo is forked, you can view it in your your [GitHub repositories](https://github.com).

The forked repo will look like:

![GitHub Fork](/images/codepipeline/github_fork_example.png)

---
title: "GitHub Access Token"
date: 2018-10-087T08:30:11-07:00
weight: 13
draft: false
---

In order for CodePipeline to receive callbacks from GitHub, we need to generate a personal access token.

Once created, an access token can be stored in a secure enclave and reused, so this step is only required
during the first run or when you need to generate new keys.

Open up the [New personal access page](https://github.com/settings/tokens/new) in GitHub.

{{% notice info %}}
You may be prompted to enter your GitHub password
{{% /notice %}}

Enter a value for **Token description**, check the **repo** permission scope and scroll down and click the **Generate token** button

![Generate New](/images/codepipeline/github_token_name.png)

Copy the **personal access token** and save it in a secure place for the next step

![Generate New](/images/codepipeline/github_copy_access.png)

---
title: "CodePipeline Setup"
date: 2018-10-087T08:30:11-07:00
weight: 14
draft: false
---

Now we are going to create the AWS CodePipeline using [AWS CloudFormation](https://aws.amazon.com/cloudformation/).

CloudFormation is an [infrastructure as code](https://en.wikipedia.org/wiki/Infrastructure_as_Code) (IaC) tool which
provides a common language for you to describe and provision all the infrastructure resources in your cloud environment.
CloudFormation allows you to use a simple text file to model and provision, in an automated and secure manner, all the
resources needed for your applications across all regions and accounts.

Each EKS deployment/service should have its own CodePipeline and be located in an isolated source repository.

You can modify the CloudFormation templates provided with this workshop to meet your system requirements to easily
onboard new services to your EKS cluster. For each new service the following steps can be repeated.

Click the **Launch** button to create the CloudFormation stack in the AWS Management Console.

| Launch template |  |  |
| ------ |:------:|:--------:|
| CodePipeline & EKS |  {{% cf-launch "ci-cd-codepipeline.cfn.yml" "eksws-codepipeline" %}} | {{% cf-download "ci-cd-codepipeline.cfn.yml" %}}  |

After the console is open, enter your GitHub username, personal access token (created in previous step), check the acknowledge box and then click the "Create stack" button located at the bottom of the page.

![CloudFormation Stack](/images/codepipeline/cloudformation_stack.png)

Wait for the status to change from "CREATE_IN_PROGRESS" to **CREATE_COMPLETE** before moving on to the next step.

![CloudFormation Stack](/images/codepipeline/cloudformation_stack_creating.png)

Open [CodePipeline in the Management Console](https://console.aws.amazon.com/codesuite/codepipeline/pipelines). You will see a CodePipeline that starts with **eks-workshop-codepipeline**.
Click this link to view the details.

{{% notice tip %}}
If you receive a permissions error similar to **User x is not authorized to perform: codepipeline:ListPipelines...** upon clicking the above link, the CodePipeline console may have opened up in the wrong region.  To correct this, from the **Region** dropdown in the console, choose the region you provisioned the workshop in.  Select Oregon (us-west-2) if you provisioned the workshow per the "Start the workshop at an AWS event" instructions.
{{% /notice %}}


![CodePipeline Landing](/images/codepipeline/codepipeline_landing.png)

Once you are on the detail page for the specific CodePipeline, you can see the status along with links to the change and build details.

![CodePipeline Details](/images/codepipeline/codepipeline_details.png)

{{% notice tip %}}
If you click on the "details" link in the build/deploy stage, you can see the output from the CodeBuild process.
{{% /notice %}}

To review the status of the deployment, you can run:

```
kubectl describe deployment hello-k8s
```

For the status of the service, run the following command:

```
kubectl describe service hello-k8s
```

#### Challenge:
**How can we view our exposed service?**

**HINT:** Which kubectl command will get you the Elastic Load Balancer (ELB) endpoint for this app?

 {{%expand "Expand here to see the solution" %}}

 Once the service is built and delivered, we can run the following command to get the Elastic Load Balancer (ELB) endpoint and open it in a browser.
 If the message is not updated immediately, give Kubernetes some time to deploy the change.

 ```
 kubectl get services hello-k8s -o wide
 ```

{{% notice info %}}
This service was configured with a [LoadBalancer](https://kubernetes.io/docs/tasks/access-application-cluster/create-external-load-balancer/) so,
an [AWS Elastic Load Balancer](https://aws.amazon.com/elasticloadbalancing/) is launched by Kubernetes for the service.
The EXTERNAL-IP column contains a value that ends with "elb.amazonaws.com" - the full value is the DNS address.
{{% /notice %}}
{{% /expand %}}

---
title: "Trigger New Release"
date: 2018-10-087T08:30:11-07:00
weight: 15
draft: false
---
#### Update Our Application

So far we have walked through setting up CI/CD for EKS using AWS CodePipeline and now we are going to
make a change to the GitHub repository so that we can see a new release built and delivered.

Open [GitHub](https://github.com/) and select the forked repository with the name **eks-workshop-sample-api-service-go**.

Click on **main.go** file and then click on the **edit** button, which looks like a pencil.

![GitHub Edit](/images/codepipeline/github_edit.png)

Change the text where it says "Hello World", add a commit message and then click the "Commit changes" button.

You should leave the master branch selected.

{{% notice info %}}
The main.go application needs to be compiled, so please ensure that you don't accidentally break the build :)
{{% /notice %}}

![GitHub Modify](/images/codepipeline/github_modify.png)

After you modify and commit your change in GitHub, in approximately one minute you will see a new build triggered in the [AWS Management Console](https://console.aws.amazon.com/codesuite/codepipeline/pipelines)
![CodePipeline Running](/images/codepipeline/codepipeline_building.png)

#### Confirm the Change

If you still have the ELB URL open in your browser, refresh to confirm the update. If you need to retrieve the URL again, use `kubectl get services hello-k8s -o wide`



