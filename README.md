# Load Balancer Solution With Apache

After completing Devops Tooling Website Solution, we might wonder how a user will be accessing each of the webservers using 3 different IP addreses or 3 different DNS names. We might also wonder what is the point of having 3 different servers doing exactly the same thing.

When we access a website in the Internet we use an URL and we do not really know how many servers are out there serving our requests, this complexity is hidden from a regular user, but in case of websites that are being visited by millions of users per day (like Google or facebook) it is impossible to serve all the users from a single Web Server (it is also applicable to databases, but for now we will not focus on distributed DBs).

Each URL contains a domain name part, which is translated (resolved) to IP address of a target server that will serve requests when open a website in the Internet. Translation (resolution) of domain names is perormed by DNS servers.

## Load balancing

It refers to efficiently distributing incoming network traffic across a group of backend servers, also known as a server farm or server pool. Modern high‑traffic websites must serve hundreds of thousands, if not millions, of concurrent requests from users or clients and return the correct text, images, video, or application data, all in a fast and reliable manner. To cost‑effectively scale to meet these high volumes, modern computing best practice generally requires adding more servers.

## A load balancer

Acts as the “traffic cop” sitting in front of your servers and routing client requests across all servers capable of fulfilling those requests in a manner that maximizes speed and capacity utilization and ensures that no one server is overworked, which could degrade performance. If a single server goes down, the load balancer redirects traffic to the remaining online servers. When a new server is added to the server group, the load balancer automatically starts to send requests to it.

In this manner, a load balancer performs the following functions:

* Distributes client requests or network load efficiently across multiple servers
* Ensures high availability and reliability by sending requests only to servers that are online
* Provides the flexibility to add or subtract servers as demand dictates

![Load Balancing](https://user-images.githubusercontent.com/117458922/224293449-60d241ca-936c-4bdb-b33f-374f0ceee9bf.png)

### Difference between L4 and L7 Load Balancing

* L4 load balancing offers traffic management of transactions at the network protocol layer (TCP/UDP). L4 load balancing delivers traffic with limited network information with a load balancing algorithm (i.e. round-robin) and by calculating the best server based on fewest connections and fastest server response times.

* L7 load balancing works at the highest level of the OSI model. L7 bases its routing decisions on various characteristics of the HTTP/HTTPS header, the content of the message, the URL type, and information in cookies.

## Project Overview

In this project we will enhance our Tooling Website solution by adding a Load Balancer to disctribute traffic between Web Servers and allow users to access our website using a single URL.

![3-tier](https://user-images.githubusercontent.com/117458922/224294190-c89f52ab-f218-400b-91ff-b1901c5aa86c.png)

## Task

We will deploy and configure an Apache Load Balancer for Tooling Website solution on a separate Ubuntu EC2 instance. We will make sure that users can be served by Web servers through the Load Balancer. We can implement this solution with 2 Web Servers for simplicity, the approach will be the same for 3 and more Web Servers.

## Prerequisites

Make sure that we have the following servers installed and configured within Devops Tooling Website Solution:

* Two RHEL8 Web Servers
* One MySQL DB Server (based on Ubuntu 20.04)
* One RHEL8 NFS server

![project print](https://user-images.githubusercontent.com/117458922/224294823-ec4e5fae-b5f0-4d15-b20e-51e589ddfad4.png)

## Configure Apache As A Load Balancer

* Create an Ubuntu Server EC2 instance and name it Apache-lb.

* Open TCP port 80 on Apache-lb by creating an Inbound Rule in Security Group.

* Install Apache Load Balancer on Project-8-apache-lb server and configure it to point traffic coming to LB to both Web Servers 

## Install apache2

```
sudo apt update
sudo apt install apache2 -y
sudo apt-get install libxml2-dev
```

##  Enable following modules:

```
sudo a2enmod rewrite
sudo a2enmod proxy
sudo a2enmod proxy_balancer
sudo a2enmod proxy_http
sudo a2enmod headers
sudo a2enmod lbmethod_bytraffic
```

## Restart apache2 service

```
sudo systemctl restart apache2
```

##  Configure Load Balancing

```
sudo vi /etc/apache2/sites-available/000-default.conf
```

* Add this configuration into this section <VirtualHost *:80>  </VirtualHost>

```
<Proxy "balancer://mycluster">
               BalancerMember http://<WebServer1-Private-IP-Address>:80 loadfactor=5 timeout=1
               BalancerMember http://<WebServer2-Private-IP-Address>:80 loadfactor=5 timeout=1
               ProxySet lbmethod=bytraffic
               # ProxySet lbmethod=byrequests
        </Proxy>

        ProxyPreserveHost On
        ProxyPass / balancer://mycluster/
        ProxyPassReverse / balancer://mycluster/
```

* Restart apache server

*bytraffic balancing method will distribute incoming load between your Web Servers according to current traffic load. We can control in which proportion the traffic must be distributed by loadfactor parameter.*

* Verify that our configuration works - try to access your LB’s public IP address or Public DNS name from your browser:

```
http://<Load-Balancer-Public-IP-Address>/index.php
```

![Screen Shot 2023-03-10 at 12 56 45 PM](https://user-images.githubusercontent.com/117458922/224343812-c15ba5ee-51ac-4e75-b324-550b2bce69bb.png)

![Screen Shot 2023-03-10 at 12 53 49 PM](https://user-images.githubusercontent.com/117458922/224343871-f4bbfbe1-2e61-4972-8dd4-935ac3b60f4c.png)


*Note: If in the Devops Tooling Website Solution we mounted /var/log/httpd/ from our Web Servers to the NFS server - unmount them and make sure that each Web Server has its own log directory.*

Open two terminal consoles for both Web Servers and run the following command:

```
sudo tail -f /var/log/httpd/access_log
```

![logss](https://user-images.githubusercontent.com/117458922/224345181-af190fdc-4683-4c58-9d9e-bdd38d7e8652.png)

![logs](https://user-images.githubusercontent.com/117458922/224345203-d424d38d-f95b-4413-b4a8-e45686301e5b.png)

* Try to refresh your browser page http:///index.php several times and make sure that both servers receive HTTP GET requests from your LB – new records must appear in each server’s log file. The number of requests to each server will be approximately the same since we set loadfactor to the same value for both servers – it means that traffic will be disctributed evenly between them.

*If all is properly configured, our users will not even notice that their requests are served by more than one server.*


##  Optional Step - Configure Local DNS Names Resolution

Sometimes it is tedious to remember and switch between IP addresses, especially if we have a lot of servers under our management. What we can do, is to configure local domain name resolution. The easiest way is to use /etc/hosts file, although this approach is not very scalable, but it is very easy to configure and shows the concept well. So let us configure IP address to domain name mapping for our LB

* Open this file on our LB server

```
sudo vi /etc/hosts
```

* Add 2 records into this file with Local IP address and arbitrary name for both of our Web Servers

```
<WebServer1-Private-IP-Address> Web1
<WebServer2-Private-IP-Address> Web2
```

* Now we can update your LB config file with those names instead of IP addresses.

```
BalancerMember http://Web1:80 loadfactor=5 timeout=1
BalancerMember http://Web2:80 loadfactor=5 timeout=1
```

* We can try to curl our Web Servers from LB locally curl http://Web1 or curl http://Web2 - it will work.

* Remember, this is only internal configuration and it is also local to our LB server, these names will neither be ‘resolvable’ from other servers internally nor from the Internet.

*We just finished the Apache LoadBanlancer on our Ubuntu Server and the architecture looks like the diagram below.*

![Apache LB](https://user-images.githubusercontent.com/117458922/224347492-dc03855b-feab-42a9-bdbe-73a561170681.png)
