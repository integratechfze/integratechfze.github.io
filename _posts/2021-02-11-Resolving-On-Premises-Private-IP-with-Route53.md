---
title: Resolving On-Premises Hosted Domain Names with AWS Route53 Resolver
author: Rajesh Nair
date: 11 February 2021 11:07:00 +0400
categories: [AWS, DNS, Resolver, On-Premises]
tags: [DNS, Resolver]
published: true
typora-root-url: ../../integratechfze.github.io
---

### **Customer Requirement**

Due to operational requirements, the customer needed to migrate their UAT servers from AWS to On-premises. The development servers were already running on the on-premises environment. Access to the dev servers was through port forwarding. Port 80/tcp and 443/tcp was forwarded to the dev server by the Fortigate firewall. <br><br>

They wanted to deploy multiple UAT servers in house; however, the challenge was that they just had one fixed IP (static IP), assigned to the public interface of the firewall/router by the ISP. Since each UAT instance was a different virtual machine, it was not possible to forward ports 80/tcp and 443/tcp to multiple servers. Each customer needed to have an URL to access their server; and due to technical requirements, the port numbers needed to be 80/tcp or 443/tcp. If they had multiple static IP’s, they could have NAT’ed each public IP to a local internal IP. <br><br>

Since the customer domain was already on Route 53, we suggested that they leverage this advantage and use AWS Site to Site VPN to provide access to the private instances in their on-premises network.<br><br>

### **The Solution**

The solution that we provided our customer would forward the traffic sent by their customers to the on-premises network. In the on-premises network, each UAT application would run on a custom port, for example, 192.168.1.20:8081. Traffic between AWS and customer location is sent over an encrypted IPSec AWS Site to Site VPN, established between the CGW, which is a Fortigate firewall and the AWS VPG.  <br><br>

Policies and routes required for the traffic between the AWS VPC’s and the on-premises network were established.  

### Walkthrough

1. Create Customer Gateway (CGW) using the customer on-premises public IP.

2. Create Virtual Private Gateway (VPG) and attach it to the VPC.

3. Create AWS Site-Site VPN and download the customer firewall configuration.

4. Configure Site-Site VPN on the on-premises firewall using the downloaded configuration.

5. Create appropriate policies and routes on the FortiGate on-premises firewall as well as on the AWS VPC to route traffic between AWS VPC and on-premises network.  

<span style="display:block;text-align:center">![Create Title Here, Rajesh](/public/img/posts/rnair-dns-01.png)</span>    
Once the VPN tunnel has been established, we need to create a private hosted zone, say, example.com and associate it to your VPC.  <br><br>

An A record need to be created for the private hosted zone entry. For example, *test.example.com A 192.168.1.20*. This private hosted zone entry will be used to access the UAT URL from within the AWS VPC using the port configured in the on-premises application. (*https://test.example.com:8081*).   <br><br>

Route 53 Resolver helps in the resolution of the host test.example.com to the local IP of the on-premises application server.   <br><br>

Route53 Resolver can be configured using Inbound Endpoint and Outbound Endpoint. Inbound Endpoint is used when you need to forward DNS queries of your on-premises network to Route53 resolver. On the other hand, Outbound endpoint will forward DNS queries using rules to resolve your on-premises network to Route53 resolver.   <br><br>

In our case, we need to configure Outbound Endpoint to resolve the DNS queries. Internally, an Elastic Network Interface (ENI) is created in each availability zone in the VPC provided during creation of outbound endpoint.   <br><br>

 After the Outbound Endpoint has been created, a rule for routing queries for on-premises hostnames has to set up by providing the target IP address and port. This rule created should include association of Outbound endpoint, private hosted zone A record (test.example.com) and Rule type as *Forward*.  <br><br>

 Once you have successfully created and saved the configuration, you should be able to access your application hosted in your on-premises server using private hosted zone http://test.example.com:8081 from the servers inside your AWS VPC.  <br><br>

However, we still need to be able to access the on-premises server from the Internet. For this, we need to forward requests to the public hosted zone on AWS Route 53 on to the Route 53 Private hosted zone test.example.com.   <br><br>

Since we have to forward requests coming in on port 443/tcp to the internal on-premises server listening on port 8081/tcp, we need to setup a reverse proxy server.  <br><br>

A reverse proxy is a server that sits in front of web servers and forwards client (e.g., web browser) requests to those web servers. Reverse proxies are typically implemented to help increase security, performance, and reliability. Nginx is one such proxy, and we will use it in this case.  <br><br>

Nginx proxy is configured in an EC2 instance with Elastic IP (Amazon Linux AMI) in the same VPC where the Outbount Endpoint was created.   <br><br>

We need to install nginx first.  

```
# sudo amazon-linux-extras install nginx1
```

We will now configure nginx. 

```
# sudo vim /etc/nginx/conf.d/awsroute53.conf
```

The below lines need to be added to the configuration file.
```
server {   

		listen 80;   
		server_name awsroute53.com;  
		location / {     
			proxy_pass  http://test.example.com:8081/;   
		}  
}  
```
 

We need to enable and start nginx.
```
# sudo systemctl enable ngnix

# sudo systemctl start ngnix
```
 

Finally, you need to create an A record in the public hosted zone with the public IP of the nginx EC2 instance.  

### **Conclusion**

We can thus create servers on-premises and access them from the Internet using URLs. This overcomes the requirement of having multiple static IP address and one-to-one NAT for hosting multiple servers on-premises. This is especially important where ISP’s generally provide only a limited number of IP addresses, and obviates the need to have Port Address translation (PAT).   
