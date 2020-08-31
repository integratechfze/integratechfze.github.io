---
title: ECS Custom Constraints for Instance Lifecycle 
author: hazaq_naeem
date: 2020-08-26 11:33:00 +0400
categories: [AWS, ECS, DevOps]
tags: [Lambda, ECS]
---
---
Amazon ECS task placement constraints are a very useful way to handle the placement of containers in our ECS cluster. Task placement constraints are rules that are considered during task placement. We may want to apply these rule to make sure that the task that we are launching are placed on the correct AMI, OS, CPU architecture or the instance type. We may have situations where we are running a heterogenous ECS cluster for example a mixture of different EC2 instance types and we want to ensure that our tasks runs on the correct instance type. In these situations we have to use ECS task placement constraints to ensure that we get the desired task placements. Unfortunately Amazon ECS only provides a handful of task placement constrain attributes out of the box. However we can leverage custom attributes to build and apply our own constraints to the ECS tasks. In this blog post I will demonstrate, how can we use our own custom attributes to handle a very common use case of managing tasks to run on the correct instance lifecycle type (On-Demand | Spot).  

Spot instances are a great way to save compute cost on AWS, you pay as low as 90% of the On-Demand cost for spare compute capacity available. But these spot instances can be interrupted if the availability zone is running out of capacity or for some other reason with only few seconds of notice. This may not be ideal for some workloads for example stateful applications.  
### Configuring the Cluster with Spot instances  
If the ECS cluster was launched using the ECS console with the default setting we may not have the option to add spot instances to our cluster or auto-scale the cluster on spot instances. In order to enable this capability we need to make some modification to our autoscaling group. First we need to convert our launch configuration to a launch template.  

Go to the launch configuration's tab and select the lunch configuration created by our ECS cluster.  
Click 'Copy to launch template'  
![Copy-launch-template](/public/img/posts/ecs-custom-constrains-01.png)
  
Once the launch template is created we need to update the autoscaling group, with the launch template.  
Select the ECS Autoscaling group and click Edit  
![Autoscaling-group](/public/img/posts/ecs-custom-constrains-02.png)  
  
Under the Launch Configuration dialog box click '<span style="color:blue">Switch to launch template</span>'  
![Autoscaling-group-2](/public/img/posts/ecs-custom-constrains-03.png)

Select the launch template that was created in the previous step  
![Autoscaling-group-3](/public/img/posts/ecs-custom-constrains-04.png)
  
Now we can select spot instances as part of our ECS cluster  
![Autoscaling-group-4](/public/img/posts/ecs-custom-constrains-05.png)

I have configured my autoscaling group to launch the first 4 EC2 instances to be On-Demand but next 4 EC2 instances will have 1 EC2 (25%) with On-Demand and 3 EC2 (75%) with Spot instances.  
### Applying the Custom Attributes
While this saves us a lot of cost it also creates a problem, for some critical applications where it is not acceptable to interrupt the service if a spot instance is terminated and this can happen quite randomly from few times a day to few times a month. We need to restrict our ECS services to only launch task on On-Demand instances but AWS ECS instances does not have a built in attribute to handle instance lifecycle (On-Demand | Spot). In this case we can use custom attributes, and create a constrain based on custom attribute.  

Go to the ECS console and select the container instance click on 'Action' -> 'View/Edit Attributes'   
![ecs-cluster](/public/img/posts/ecs-custom-constrains-06.png)
  
Under 'Custom attributes' click '<span style="color:blue">Add Attribute</span>' and now you can add a custom attributes for example I am adding the following attribute   
```
{  
	'Name' : 'LifeCycle', 
	'Value' : 'On-Demand' 
} 
```
![ecs-attribute](/public/img/posts/ecs-custom-constrains-07.png)  

You also use the below aws cli command to add the attribute.  
```terminal
$ aws ecs put-attributes --attributes name=LifeCycle,value=On-Demand,targetId=arn
```

Similarly we can tag spot instances with LifeCycle: Spot  

In the task definition of the our service we can add a constrain of type memberOf with the following Expression `attribute:LifeCycle == On-Demand`  
![ecs-task](/public/img/posts/ecs-custom-constrains-08.png)  

### Automate the update of Instance attributes 
So far so good, but the container instances in our cluster are part of an autoscaling group which means that new instances could be added or removed dynamically and it will be impractical to add these attributes manually every time.  
We can add custom attributes by update the `/etc/ecs/ecs.config` file. Below is a script that I have added into the user data of the my launch template.  
```terminal
#!/bin/bash
echo ECS_CLUSTER=ecs-demo >> /etc/ecs/ecs.config
Instance_lifecycle=$(curl http://169.254.169.254/latest/meta-data/instance-life-cycle)
if [ $Instance_lifecycle == "on-demand" ]
then
  echo 'ECS_INSTANCE_ATTRIBUTES={"LifeCycle": "On-Demand"}' >> /etc/ecs/ecs.config
else
  echo 'ECS_INSTANCE_ATTRIBUTES={"LifeCycle": "Spot"}' >> /etc/ecs/ecs.config
fi
```
Now every time a new container instance starts up its attributes will be populated automatically.  

### Automate the update of Instance attributes (<span style="color:red">The Hard Way</span>)
(*Just for fun I wanted to try something different, here I have used a lambda function to update the same attributes.*)  
Let's create a Cloudwatch event to capture every time a new container instances is registered with our cluster and use a simple lambda function to update the attributes of the container instances.  
First the lambda function.  
```ruby
import boto3
import os

ecs_cluster_name = os.environ['ECS_CLUSTER_NAME']
ec2 = boto3.client('ec2')
ecs = boto3.client('ecs')

def lambda_handler(event, context):
    
    ecs_instance_arn = event['detail']['responseElements']['containerInstance']['containerInstanceArn']
    ecs_instance_id = event['detail']['responseElements']['containerInstance']['ec2InstanceId']
    
    if ecs_cluster_name == event['detail']['requestParameters']['cluster']:
        ec2_response = ec2.describe_instances( InstanceIds=[ ecs_instance_id ])
        try:
            lifecycle = ec2_response['Reservations'][0]['Instances'][0]['InstanceLifecycle']
        except KeyError:
            lifecycle = "On-Demand"
        
        try:
            ecs.put_attributes(cluster=ecs_cluster_name, attributes=[ { 'name':'LifeCycle', 'value': lifecycle, \
                                    'targetType': 'container-instance', 'targetId': ecs_instance_arn }])
            print('Added the attribute "LifeCycle": "' + lifecycle + '" to the instance with id ' + ecs_instance_id)
        except:
            print ('Failed to add the attribute to ECS cluster' + ecs_cluster_name)
    else:
        print ('ECS cluster name does not match')
        print ('Looking for Cluster: ' + ecs_cluster_name + ' event was for Cluster: ' + \
                        event['detail']['requestParameters']['cluster'])
        
```
The IAM role for the lambda function should have update attributes permission on the ECS cluster   
And finally the Cloudwatch event  
![cloudwatch-event](/public/img/posts/ecs-custom-constrains-09.png)  
The target of the above Cloudwatch event should be the lambda function we created previously.  
Now every time a new container instance is registered with ECS cluster the Cloudwatch event triggers the lambda function which will update our attribute.  

### Conclusion  
Amazon ECS task placement constrains are a very powerful feature available to the developers and cluster administrators. Not all attributes for the task placements are available but with custom attributes you can practically apply any constrain that you can think of and there will always be more than one way to it.  
