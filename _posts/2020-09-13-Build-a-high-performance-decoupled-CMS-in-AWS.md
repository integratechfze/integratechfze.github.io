---
title: Build a high-performance 'decoupled CMS' in AWS 
author: amal_jith
date: 13 September 2020 11:33:00 +0400
categories: [AWS, CMS, DevOps]
tags: [CloudFront, CodeBuild]
published: true
typora-root-url: ../../integratechfze.github.io
---


### Web Content Management Systems
From the inception of the World Wide Web (www), websites and pages were a popularly adopted concept in the digital world whether it was for used for commercial, government, research or community purposes. 



Making websites more interactive and useful for daily transactions led to implementation of dynamic websites which were developed in either in a high level programming language such as PHP, Java or JavaScript.  As the need of having an online presence surged, the pain of developing feature rich web applications increased as it involved different software frameworks and coding. That is where Web Content Management Systems came to the rescue for easily creating and managing content.


![CMS-Creativity](/public/img/posts/decoupled-cms-aws-01.jpg)
<br/> 
Web Content Management Systems (WCMS) provides website authoring, collaboration, and administration tools that help users with little knowledge of web programming languages or markup languages create and manage website content. It also controls a dynamic collection of web material, including HTML documents, images, and other forms of media. As of today, WordPress is the leading content management system (CMS) with 63% market share and 36.2% of all websites built with it, followed by Joomla, Drupal and so many other open sources as well as paid options. 

<br/>
The increase in number of internet users and mobile devices and rapid digitalization caused explosive growth in web traffic. The www project happened in 1991 with one website and today we have around 1.8 billion websites and growing rapidly.

* Speed and performance of a website with that of a similar competitor matters, and users will move if they are dissatisfied.
* Website contents include high-definition media (images, animation, videos, audio etc.) that should be delivered without any delay.
* End user devices (clients) are becoming more than just a presentation platform and their processing capabilities should be leveraged rather than doing processing on the server-side.
* With the wide adoption of public cloud, there are more ways to host web applications, be it static webhosting, serverless, or container based microservices. They can be very secure and costs less as well.

Suddenly a traditional CMS seems unappealing and too dated, which leads us to....

<br/>
### Decoupled CMS with Frontend WebApp and a Headless CMS
In a traditional CMS, the CMS application will be responsible for both storing and managing the content as well as for dynamically generating and presenting the content to the audience. This can cause performance issues as the monolith application would need to keep serving content, even in times of a surge in web traffic, while continuing to be accessible to content authors for new content to be created and published. 

<br/>
![traditional-CMS](/public/img/posts/decoupled-cms-aws-02.jpeg)



WordPress, Joomla, Drupal etc. fall into the category of traditional CMS.

In order to maintain better speed and performance, the best option would be to decouple the ‘head’ (Frontend delivery/presentation) from the ‘body’ (Backend processing/content management) with an API in the middle. i.e. Run the CMS as headless and implement the ‘head’ as a Progressive Web Application with a JS framework of your choice.



![decoupled-CMS](/public/img/posts/decoupled-cms-aws-03.jpeg)  
<br/>  
This separation of concerns helps in fine-tuning the content delivery to be more optimized to be a mobile friendly, and the ReactJS website will be running perfectly in the client side taking advantage of the client browser computing power. We can also utilize Static Webhosting to serve the frontend website through a content delivery network (CDN) to make it improve overall performance. There are many reasons we use GraphQL with GatsbyJS for pulling the CMS content to generate the static pages, compared to using a REST API:

*	GraphQL responses are typed & organized – it can support complex object & data types like json scalars, enumerations, unions, interfaces and supports nullability. Helps to fetch a wide variety of data from the CMS and also the requesting app can use types to avoid writing manual parsing code.
*	GraphQL queries always return predictable results. There is no under/over utilization of the API request. Apps using GraphQL will fast and stable because they control the data they get, not the server. This leads to less time in building/generating the static site.
*	GraphQL APIs can get all the required data in a single request, while typical REST APIs require loading from multiple URLs which will be time consuming. GraphQL queries access not just the properties of one resource but also smoothly follow references between them.
*	Integrating GraphQL with GatsbyJS and CMS(eg: WordPress, Drupal) is very easy with community developed stable packages & plugins.

<br/>
### Architecting the solution in AWS
When architecting the solution in this case, it is better if we follow a modular approach which leads to better integration of these complex components. ie. We can build the network(with subnets) first, then hosting infrastructure and finally the deployment automation.

As the AWS-Well Architected Framework says, we should follow the AWS recommended best practices across the following pillars when we architect a solution, especially for Production workloads:
*	Security
*	Reliability
*	Operational Excellence
*	Performance Efficiency
*	Cost Optimization

<br/>

### Virtual Private Network in AWS: 


Create an AWS VPC of a network CIDR of 10.0.0.0/17 with 2 public & 4 private subnets spread across 2 availability zones (AZ). Egress internet access for servers can be enabled using AWS managed NAT gateways. The application & database servers should be launched in private subnets because these shouldn’t be reachable from internet thus ensuring security.
There will be an Elastic Load Balancer (Application Load Balancer to be specific) deployed in the public subnets and all the traffic to the CMS application is expected to go through the load balancer. The ALB receives inbound traffic at HTTP/HTTPS ports 80/443 from the internet. A free SSL certificate can be created and attached to the ALB with AWS Certificate Manager. Any insecure http requests will be routed to https 443 port for ensuring security.

<br/>

### Drupal 8 as the Headless CMS:



We can create a base AMI with latest stable PHP, Drupal 8 and necessary Apache 2.4 configurations. We might need to install Amazon CloudWatch agent, AWS Sessions Manager agent, AWS CodeDeploy agent for logging/monitoring, SSH logins and automated deployment respectively. This CMS app instances can be launched as an AutoScaling group (ASG) with minimum: 1, desired: 1 and maximum: 2 instances as the Scaling Policy. The ASG will be associated with the 2 different private subnets so that we can ensure high availability in case of data center failure. The EC2 instance’s Security group will have inbound traffic allowed from only the Load Balancer Security Group.

The CMS database will be created as an RDS Aurora MySQL cluster with a Writer/Primary DB instance running in one private subnet and another Reader/Secondary DB instance running a different private subnet in another AZ. The database should not be publicly accessible to enhance security. The Security Group of the RDs will have inbound traffic allowed only from the Application EC2 instance’s Security group.

<br/>

### Automated Deployment Pipeline (Continuous Integration and Continuous Deployment): 



We can create a separate code repository (Github/CodeCommit) for keeping the Drupal8 source code. At least 2 branches should be maintained to ensure no buggy code reaches Production deployment pipeline. As a common strategy, we can have the Master branch and a dev branch for development code.

AWS CodePipeline helps to orchestrate the end-to-end deployment. We can integrate the source code repository (Github/CodeCommit) end enable automatic trigger of the pipeline (like webhook) with the help of CloudWatch events starting the pipeline when there is a code push in the designated branch. The resulting artifact will be versioned and stored in an S3 bucket as an encrypted object, which is used by CodeDeploy in the deployment phase once the Manual Action stage (Approval) is passed.

We can customize the deployment with AWS CodeDeploy by creating an application & deployment groups so that we can leverage the out-of-the-box deployment strategies like Rolling/Canary/BlueGreen etc. Any custom steps involved in the application deployment could be configured by creating/checking-in an appspec.yml (application deployment specification) and related shell scripts (BASH) in the code repository branch – which will be used by CodeDeploy during the deployment phase.

We can optionally mount an Elastic File System (EFS) shared filesystem mount point at /var/www/drupal8 /sites/default/files so all the Static Assets (newly uploaded images, icons, cached CSS, JS files) – which are not part of the Source code could be preserved and attached to the new instances launching in the AutoScaling group. This step can be achieved with a simple shell script and appspec.yml. 

When new Amazon EC2 instances are launched as part of an Amazon EC2 Auto Scaling group, CodeDeploy can deploy the code revisions to the new instances automatically. We can also coordinate deployments in CodeDeploy with Amazon EC2 Auto Scaling instances registered with Elastic Load Balancing load balancers (ALB).

<br/>

### Deployment automation of Frontend website with S3 Static Webhosting:
<br/>
![build-and-deploy-frontend](/public/img/posts/decoupled-cms-aws-04.jpeg)  
<br/>
Similar to the Drupal8 CMS deployment pipeline, we will have another pipeline for the frontend as well. The only difference is that, we need to configure Amazon CodeBuild buildspec.yml (Build specification/configuration) so that build process can be automated without requiring a CI/Build server running always. We can have the build logs populated in a CloudWatch logstream/S3 bucket, which makes debugging/troubleshooting easy.


Here is a an example buildspec.yml :

<br/>
```yml
version: 0.2
run-as: root
phases:
  install:
    runtime-versions: 
      nodejs: 10
      python: 3.7 
    commands: 
      - "echo \"### Entered the install phase...#####\""
      - "node -v"
      - "npm -v"
      - "python --version"
      - "aws --version"
      - "echo Default Linux shell is `ps -p $$`"
      - "echo \"Installing all dependencies as per package.json\""
    finally: 
      - "npm install"
  pre_build: 
    commands: 
      - "echo \"### Entered the pre_build phase...#####\""
      - "echo \"List of installed dependencies:\""
      - "npm list -g"
      - "echo \"### Initiating the build...#####\""
      - "echo Build started at `date`"
      - "npm run build && export static_web_build_stat=\"PASS\""
    finally: 
      - "echo ${static_web_build_stat}"
      - "echo Build ended at `date`" 
  build: 
    commands: 
      - "echo \"### Entered the build phase...#####\""
      - "echo \"### Removing .map files from the build...#####\""
      - "find ./public -name '*.map' -type f -delete"
      - "echo \"### Deploying the built code to S3 bucket...#####\""
      - "aws s3 sync \"public\" \"s3://prod-frontend-s3website/\" --region eu-west-1 --delete"
      - "aws s3 sync \"public\" \"s3://prod-frontend-s3website2/\" --region me-south-1 --delete"
    finally: 
      - "echo \"Deployed site to S3\""
  post_build: 
    commands: 
      - "echo \"### Entered the pre_build phase...### \""
      - "echo \"Initiating CloudFront invalidation\""
      - "aws cloudfront create-invalidation --distribution-id YORB6XXXRPDD --paths '/*'"
      - "echo \"Invalidation completed\""
    finally: 
      - "echo Deployment ended at `date`"
artifacts: 
  base-directory: public
  discard-paths: "no"
  files: 
    - "**/*"
```

The resulting artifacts will be the ReactJS static website with html, js, cs, fonts etc. This will be synced to two S3 buckets as a deployment. These 2 S3 buckets in 2 different AWS regions (for example one in Ireland and another one in Singapore) to host the static website. In this way the website will survive even in case of a regional disaster thus high availability can be ensured.

</br>

### Improving the site performance with a global CDN:


The S3 static website can directly be used to serve the website. But the whole purpose of generating the static version of CMS frontend is to serve it as Progressive Web Application (PWA), through a high-speed global Content Delivery Network (CDN). We can utilize Amazon CloudFront which offers much more functionality than just a CDN. 

We can create a Cloudfront web distribution with custom SSL certificate for the domain – which can be created in N.Virginia AWS Certificate Manager for free. Create three origins – one or for the ALB for dynamic requests and two custom origins, each for the S3 static webhosting endpoints (Ireland and Singapore buckets). Create an origin group with the two S3 static webhosting endpoint origins to provide rerouting during a failover event. We can associate an origin group with a cache behavior to have requests routed from a primary origin to a secondary origin for failover. We must have two origins for the distribution before we can create an origin group.

<br/>
![enh ance-performance-security-frontend](/public/img/posts/decoupled-cms-aws-05.png)  
<br/>
We can associate a Lambda@Edge custom code to append custom headers, redirection, additional processing logic for each request reaching any Cloudfront cache behaviors. We will be able to create up to 25 cache behaviors per Cloudfront distribution. We should create a global Web Application Firewall (WAF) with OWASP top 10 rules to protect the Cloudfront endpoint from attacks/exploits.  
<br/>

### The complete AWS architecture
<br/>
![complete-architecture](/public/img/posts/decoupled-cms-aws-06.jpeg)  
<br/>

### Conclusion  
Running a decoupled CMS as headless is a great way for hosting CMS with improved performance, security and less running cost. The advanced eco system of devops related services in AWS, makes it very easier to build complex workload solutions with ease - making AWS a complete platform to build innovative solutions. 
