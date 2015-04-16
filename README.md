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

Generate Sample Data (using https://github.com/wg/wrk)

wrk -c 2 -d 20m http://52.8.43.205  (After restart, AWS assigns new public IP)

```
run:wrk-master arun$ ./wrk -c 2 -d 20m http://52.8.43.205
Running 20m test @ http://52.8.43.205
  2 threads and 2 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    29.76ms   33.27ms 649.41ms   93.44%
    Req/Sec    40.49     12.77    70.00     76.94%
  95770 requests in 20.00m, 366.06MB read
Requests/sec:     79.80
Transfer/sec:    312.34KB
```
Create a spool directory

[ec2-user@ip-172-31-18-234 ~]$ pwd
/home/ec2-user
[ec2-user@ip-172-31-18-234 ~]$ mkdir spool

move log to spool (logrotate)
```
[ec2-user@ip-172-31-18-234 ~]$ cat rotateAccess.conf 
/var/log/nginx/access.log {
    missingok
    notifempty
    rotate 0
    copytruncate
    sharedscripts
    olddir /home/ec2-user/spool
    postrotate
        chown ec2-user:ec2-user /home/ec2-user/spool/access.log.1
        ts=$(date +%s)
        mv /home/ec2-user/spool/access.log.1 /home/ec2-user/spool/access.log.$ts
    endscript
}
[ec2-user@ip-172-31-18-234 ~]$ 
```


[ec2-user@ip-172-31-18-234 ~]$ sudo /usr/sbin/logrotate -d /home/ec2-user/rotateAccess.conf (debug run)
sudo /usr/sbin/logrotate -f /home/ec2-user/rotateAccess.conf (real run)

[ec2-user@ip-172-31-18-234 ~]$ ls -l spool/
total 3948
-rw-r--r-- 1 ec2-user ec2-user 4038697 Apr 16 18:02 access.log.1429207345
[ec2-user@ip-172-31-18-234 ~]$ 


sudo ls -l /var/log/nginx/
total 20
-rw-r--r-- 1 root root 14792 Apr 16 18:05 access.log
-rw-r--r-- 1 root root   443 Apr 16 16:48 error.log
[ec2-user@ip-172-31-18-234 ~]$ 



run the job peroidically

[ec2-user@ip-172-31-18-234 ~]$ sudo cat  /etc/cron.d/rotateLogsToSpool
# Move files to spool directory every 5 minutes
*/5 * * * * root /usr/sbin/logrotate -f /home/ec2-user/accessRotate.conf
[ec2-user@ip-172-31-18-234 ~]$ 



ec2-user@ip-172-31-18-234 ~]$ ls -l spool/
total 5104
-rw-r--r-- 1 ec2-user ec2-user 4038697 Apr 16 18:02 access.log.1429207345
-rw-r--r-- 1 ec2-user ec2-user 1182758 Apr 16 18:05 access.log.1429207514
[ec2-user@ip-172-31-18-234 ~]$ 


## Now Set up Elastic Search Machine

wget https://download.elasticsearch.org/elasticsearch/elasticsearch/elasticsearch-1.4.1.noarch.rpm

```
[ec2-user@ip-172-31-14-72 ~]$ wget https://download.elasticsearch.org/elasticsearch/elasticsearch/elasticsearch-1.4.1.noarch.rpm
--2015-04-16 18:46:07--  https://download.elasticsearch.org/elasticsearch/elasticsearch/elasticsearch-1.4.1.noarch.rpm
Resolving download.elasticsearch.org (download.elasticsearch.org)... 54.243.77.158, 54.225.64.161, 54.225.133.195, ...
Connecting to download.elasticsearch.org (download.elasticsearch.org)|54.243.77.158|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 26326154 (25M) [application/x-redhat-package-manager]
Saving to: ‘elasticsearch-1.4.1.noarch.rpm’

elasticsearch-1.4.1.noarch.rpm        100%[===========================================================================>]  25.11M  7.20MB/s   in 3.5s   

2015-04-16 18:46:11 (7.20 MB/s) - ‘elasticsearch-1.4.1.noarch.rpm’ saved [26326154/26326154]



[ec2-user@ip-172-31-14-72 ~]$ sudo /sbin/chkconfig --add elasticsearch
error reading information on service elasticsearch: No such file or directory
[ec2-user@ip-172-31-14-72 ~]$ sudo rpm -ivh elasticsearch-1.4.1.noarch.rpm
Preparing...                          ################################# [100%]
Updating / installing...
   1:elasticsearch-1.4.1-1            ################################# [100%]
### NOT starting on installation, please execute the following statements to configure elasticsearch to start automatically using chkconfig
 sudo /sbin/chkconfig --add elasticsearch
### You can start elasticsearch by executing
 sudo service elasticsearch start
[ec2-user@ip-172-31-14-72 ~]$ 




```
Check whether elastic search is working or not


```
curl http://localhost:9200/
```

##Setting up Flume on collector/relay

```
[ec2-user@ip-172-31-13-251 ~]$ wget http://apache.arvixe.com/flume/1.5.2/apache-flume-1.5.2-bin.tar.gz
--2015-04-16 18:56:34--  http://apache.arvixe.com/flume/1.5.2/apache-flume-1.5.2-bin.tar.gz
Resolving apache.arvixe.com (apache.arvixe.com)... 198.58.87.82
Connecting to apache.arvixe.com (apache.arvixe.com)|198.58.87.82|:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 25323459 (24M) [application/x-gzip]
Saving to: ‘apache-flume-1.5.2-bin.tar.gz’

apache-flume-1.5.2-bin.tar.gz          100%[===========================================================================>]  24.15M  9.44MB/s   in 2.6s   

2015-04-16 18:56:37 (9.44 MB/s) - ‘apache-flume-1.5.2-bin.tar.gz’ saved [25323459/25323459]

[ec2-user@ip-172-31-13-251 ~]$  tar -zxf apache-flume-1.5.2-bin.tar.gz
[ec2-user@ip-172-31-13-251 ~]$ cd apache-flume-1.5.2-bin
[ec2-user@ip-172-31-13-251 apache-flume-1.5.2-bin]$ 

```

cd apache-flume-1.5.2-bin
    3  vi collector.conf
    4  ls
    5  vi conf/collector.conf
    6  ./bin/flume-ng agent -n collector -c conf -f conf/collector.conf -Dflume.root.logger=INFO,console
    7  wget https://download.elasticsearch.org/elasticsearch/elasticsearch/elasticsearch-1.4.1.noarch.rpm
    8  sudo rpm -ivh elasticsearch-1.4.1.noarch.rpm
    9  pwd
   10  mkdir -p plugins.d/elasticsearch/libext
   11  cp /usr/share/elasticsearch/lib/*.jar plugins.d/elasticsearch/libext/
   12  ./bin/flume-ng agent -n collector -c conf -f conf/collector.conf -Dflume.root.logger=INFO,console





#Back to webserver to create another agent to sink the data to flume agent











