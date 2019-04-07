
# Mythical Mysfits: A tale of love, loss, and cuddles

![mysfits-welcome](/images/mysfits-welcome.png)

# Will you help us find their forever homes?

**Mythical Mysfits** is a (fictional) pet adoption non-profit dedicated to helping abandoned, and often misunderstood, mythical creatures find a new forever family! Mythical Mysfits believes that all creatures deserve a second chance, even if they spent their first chance hiding under bridges and unapologetically robbing helpless travelers.

Our business has been thriving with only a single Mysfits adoption center, located inside Devils Tower National Monument. Speak, friend, and enter should you ever come to visit.

We've just had a surge of new mysfits arrive at our door with nowhere else to go! They're all pretty distraught after not only being driven from their homes... but an immensely grumpy ogre has also denied them all entry at a swamp they've used for refuge in the past.

That's why we've hired you to be our first Full Stack Engineer. We need a more scalable way to show off our inventory of mysfits and let families adopt them. We'd like you to build the first Mythical Mysfits adoption website to help introduce these lovable, magical, often mischievous creatures to the world!

### Requirements:
* AWS account - if you don't have one, it's easy and free to [create one](https://aws.amazon.com/)
* AWS IAM account with elevated privileges allowing you to interact with various AWS Services
* Familiarity with Python, vim/emacs/nano, [Docker](https://www.docker.com/), AWS and microservices - not required but a bonus

### Building your docker image

* Step1 : [Setting up the workshop environment (Cloud9)](https://github.com/vanchee/containers-sydsummit-workshop-2019/blob/master/all-lab-modules/lab0-setting-up-environment/READ.md)   
  * Step1(a) (Applies ONLY for EKS users) : [Trigger clusters creation for EKS](https://github.com/vanchee/containers-sydsummit-workshop-2019/tree/master/all-lab-modules/lab0-setup-eks-cluster)        
* Step2 : [Build your Docker image and push it to container repository (ECR)](https://github.com/vanchee/containers-sydsummit-workshop-2019/tree/master/all-lab-modules/lab1-building-docker-image)       


 * Step3a : Deploy on ECS(with Fargate)   
   * Step 3a.1 :[deploy-your-container-to-ecs-fargate-cluster](https://github.com/vanchee/containers-sydsummit-workshop-2019/blob/master/all-lab-modules/lab2a-option1-ecs-labs/01-deploy-your-ecs-fargate-cluster/READ.md)     
   * Step 3a.2 :[breaking-monolith-image-ecs](https://github.com/vanchee/containers-sydsummit-workshop-2019/blob/master/all-lab-modules/lab2a-option1-ecs-labs/02-breaking-monolith-image-ecs/README.md)     
   * Step 3a.3 :[automating-end-to-end-deployments-for-aws-fargate](https://github.com/vanchee/containers-sydsummit-workshop-2019/tree/master/all-lab-modules/lab2a-option1-ecs-labs/03-automating-end-to-end-deployments-for-aws-fargate)      
   * Step 3a.4 :[log analysis with cloudwatch logs and elasticsearch]()    
            
 or      
 
 * Step3b : Deploy on EKS 

### ECS Learning Path

You will learn,
  
            * Configure ECS Cluster with Fargate  
            * Splitiing monolith into microservices, leveraging ALB  
            * logging & Application monitoring  
            * CI/CD Pipeline for ECS 

### EKS Learning Path
You will learn,
  
            * Configure EKS Cluster, provisoning worker nodes  
            * Splitiing monolith into microservices, leveraging ALB   
            * Logging & Monitoring with 
            * CI/CD Pipeline for EKS

