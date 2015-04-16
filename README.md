# An Implemetation of Flume,Elastic Search and Kibana for log analysis using AWS

## Setting up Webserver to generate logs

One EC2 machine will act as a webserver. Private DNS ip-172-31-18-234.us-west-1.compute.internal
```
ssh -i hortonworks.pem ec2-user@ec2-52-8-50-96.us-west-1.compute.amazonaws.com
```
Install  Nginx web server
```
sudo yum -y install nginx
```
start the Nginx web server:
```
[ec2-user@ip-172-31-18-234 ~]$ sudo /etc/init.d/nginx start
Starting nginx:                                            [  OK  ]
```

Go to public IP and see the welcome page. (Make sure that we have HTTP inbound access in security group for the instance)


