---
title: Building a high-performance decoupled CMS in AWS 
author: amal_jith
date: 13 September 2020 11:33:00 +0400
categories: [AWS, CMS, DevOps]
tags: [CloudFront, CodeBuild]
published: true
typora-root-url: ../../integratechfze.github.io
---


### Web Content Management Systems
From the inception of the World Wide Web (www), websites and pages were a popularly adopted concept in the digital world whether it was for used for commercial, government, research or community purposes. 

<br/>

Making websites more interactive and useful for daily transactions led to implementation of dynamic websites which were developed in either in a high level programming language such as PHP, Java or JavaScript.  As the need of having an online presence surged, the pain of developing feature rich web applications increased as it involved different software frameworks and coding. That is where Web Content Management Systems came to the rescue for easily creating and managing content.


<img src="/public/img/posts/decoupled-cms-aws-01.jpg" alt="CMS-Creativity" style="zoom:80%;" />
<br/> 
Web Content Management Systems (WCMS) provides website authoring, collaboration, and administration tools that help users with little knowledge of web programming languages or markup languages create and manage website content. It also controls a dynamic collection of web material, including HTML documents, images, and other forms of media. As of today, WordPress is the leading content management system (CMS) with 63% market share and 36.2% of all websites built with it, followed by Joomla, Drupal and so many other open sources as well as paid options. 

<br/>
The increase in number of internet users and mobile devices and rapid digitalization caused explosive growth in web traffic. The www project happened in 1991 with one website and today we have around 1.8 billion websites and growing rapidly.

<br/>

* Speed and performance of a website with that of a similar competitor matters, and users will move if they are dissatisfied.
* Website contents include high-definition media (images, animation, videos, audio etc.) that should be delivered without any delay.
* End user devices (clients) are becoming more than just a presentation platform and their processing capabilities should be leveraged rather than doing processing on the server-side.
* With the wide adoption of public cloud, there are more ways to host web applications, be it static webhosting, serverless, or container based microservices. They can be made very secure and costs less as well.

<br/>

With all these advances in technology, a traditional CMS seems unappealing and too dated. 

<br/>

### Decoupled CMS with Frontend WebApp and a Headless CMS
In a traditional CMS, the CMS application is responsible for storing and managing the content as well as dynamically generating and presenting the content to the audience. This can cause performance issues as the monolith application would need to keep serving content, even in times of a surge in web traffic, while continuing to be accessible to content authors for new content to be created and published. 

<br/>
<img src="/public/img/posts/decoupled-cms-aws-02.jpeg" alt="traditional-CMS" style="zoom:80%;" />



WordPress, Joomla, Drupal etc. fall into the category of traditional CMS.

<br/>

In order to improve speed and performance, the best option would be to decouple the 'head' (the frontend) from the 'body' (backend processing / content management) and integrate them with an API, i.e., run the CMS headless and have the frontend as a progressive web application with a JS framework of your choice.



<img src="/public/img/posts/decoupled-cms-aws-03.jpeg" alt="decoupled-CMS" style="zoom:80%;" />  
<br/>  
This separation will help create multiple client screen size friendly static files. ReactJS code will run in the client side. Static pages can be served through a Amazon CloudFront which is a Content Delivery Network (CDN). The advantages of using GraphQL with GatsbyJS for pulling the CMS content to generate the static pages over using a REST API are<br/>

<br/>

*	GraphQL responses are typed & organized – it can support complex object & data types like json scalars, enumerations, unions, interfaces and supports nullability. It helps fetch a wide variety of data from the CMS and the requesting app can use types to avoid writing manual parsing code.
*	GraphQL queries always return predictable results. There is no under or over utilization of the API request. Apps using GraphQL are fast and stable because they control the data they get rather than the server. This leads to less time in generating the static site files.
*	GraphQL APIs can get all the required data in a single request, while typical REST APIs require loading from multiple URLs. GraphQL queries access not just the properties of one resource but smoothly follows references between them.
*	Integrating GraphQL with GatsbyJS and CMS(eg: WordPress, Drupal) is easy with community developed packages and plugins.

<br/>

### Architecting the solution in AWS
Following the AWS-Well Architected Framework, the following pillars are to be kept in mind when designing the architecture.
*	Security
*	Reliability
*	Operational Excellence
*	Performance Efficiency
*	Cost Optimization

<br/>

### AWS Infrastructure

We will create an AWS VPC of a network CIDR of 10.0.0.0/16 with 2 public and 4 private subnets spread across 2 Availability Zones (AZ). Egress from the private subnets will be through NAT gateways. The application and database servers will be lauched in the private subnets.

<br/>

We will also deply an Application Load Balance in the public subnets and all the traffic to the CMS application will go through the load balancer. The ALB receives inbound traffic at HTTP/HTTPS ports 80/443 from the internet. A SSL certificate should be created and attached to the ALB with AWS Certificate Manager (ACM). All requests coming over 80/http will be redirected to 443/https.

<br/>

### Drupal 8 as Headless CMS

Next we need to create a base AMI with latest (stable) PHP, Drupal 8 and Apache 2.4.  Amazon CloudWatch agent, AWS Sessions Manager agent and AWS CodeDeploy agent should be installed. We will then create an AutoScaling group (ASG) with minimum 1, desired 1 and maximum 2 instances as the Scaling Policy. The ASG will be associated with different private subnets to ensure high availability. The EC2 instance Security Group (SG) will have inbound traffic allowed from only the Load Balancer SG. <br/>

<br/>

We will now create the database, an RDS Aurora MySQL cluster, with the Writer and Reader instances in different subnets. The database will need to set to be not publically accessible. The SG of the RDS instances will have inbound traffic allowed only from the Application EC2 instances SG.

<br/>

### Automated Deployment Pipeline

<br/>

We need to create a CodeCommit repository for keeping the Drupal 8 code.  A master and dev branch should be created at the minimum. <br/>

AWS CodePipeline helps orchestrate end-to-end deployment. Automatic triggerring of the pipeline will be done with the help of CloudWatch events whenever there is a code push in the designated branch. The resulting artifact will be versioned and stored in an S3 bucket as an encrypted object. This will be used by CodeDeploy in the deployment phase once the Manual Action stage (Approval) is passed. <br/><br/>

We can customize the deployment with AWS CodeDeploy by creating an application as well as deployment groups so that we can leverage out-of-the-box deployment strategies like Rolling, Canary or Blue-Green. Any custom steps involved in the application deployment can be configured by creating or checking-in appspec.yml  and related shell scripts (bash) in the code repository branch. This will be used by CodeDeploy during the deployment phase. <br/>

<br/>

We can optionally mount an Elastic File System (EFS) shared filesystem mount point at /var/www/drupal8 /sites/default/files where static assets  which are not part of the source code (newly uploaded images, icons, cached CSS and  js files etc.) will be saved. This enables the availability of these assets when new instances are launched by the ASG. This step can be achieved with a simple shell script and appspec.yml. <br/>

<br/>

When new EC2 instances are launched as part of an Amazon EC2 Auto Scaling group, CodeDeploy will deploy code revisions to the new instances automatically.

<br/>

### Deployment automation of Frontend website with S3 Static Webhosting:
<br/>
<img src="/public/img/posts/decoupled-cms-aws-04.jpeg" alt="build-and-deploy-frontend" style="zoom:67%;" />  
<br/>

Similar to the Drupal8 CMS deployment pipeline, we will have another pipeline for the frontend. However, we need to configure the Amazon CodeBuild buildspec.yml (build specification/configuration) so that build process can be automated without requiring a continuously running build server. The build logs can be populated in a CloudWatch logstream or written to a S3 bucket,to help with debugging in case of any issues.<br/>


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

The resulting artifacts will be the ReactJS static website with html, js, cs, fonts etc. This should be synced to two S3 buckets in different regions as a deployment. This will ensure site availability even in case of an AWS region being unavailable.

</br>

### Improving the site performance with a global CDN:


The S3 static website can directly be used to serve the website. However, we can serve the site as a Progressive Web Application (PWA) through a high-speed global Content Delivery Network (CDN), preferably Amazon CloudFront.<br/>

A CloudFront web distribution with custom SSL certificate for the domain has to be created. We create three origins – one for the ALB for dynamic requests and two custom origins, one for each of the S3 static web hosting endpoints (say, Ireland and Singapore buckets). Create an origin group with the two S3 static web hosting endpoint origins to provide re-routing during a failover event. We can associate an origin group with a cache behavior to have requests routed from a primary origin to a secondary origin for failover. Something to note - we must have two origins for the distribution before we can create an origin group.

<br/>
<img src="/public/img/posts/decoupled-cms-aws-05.png" alt="enh ance-performance-security-frontend" style="zoom:67%;" />  
<br/>Furthermore, we can associate Lambda@Edge functions to append custom headers, perform redirection or have additional processing logic for each request reaching each Cloudfront cache behaviors. We can create up to 25 cache behaviors per Cloudfront distribution. A Web Application Firewall (WAF) with OWASP top 10 rules should be deployed to protect the CloudFront endpoint from attacks or to thwart common exploits.  
<br/>

### The complete AWS architecture
<br/>
![complete-architecture](/public/img/posts/decoupled-cms-aws-06.jpeg)  
<br/>

### Conclusion  
Running a application headless is a great way to improve performance, security, availability and to reduce costs. The advanced ecosystem of DevOps related services on AWS makes it easy to create, deploy and maintain complex web applications.
