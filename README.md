# Installing mod_jk on apache httpd in CentOS 7.x

The Apache Tomcat Connectors project is part of the Tomcat project and provides web server plugins to connect web servers with Tomcat and other backends. 
The supported web servers are:
  - the Apache HTTP Server with a plugin (module) named mod_jk.
  - Microsoft IIS with a plugin (extension) named ISAPI redirector (or simply redirector).
  - the iPlanet Web Server with a plugin named NSAPI redirector. The iPlanet Web Server was previously known under various names, including Netscape Enterprise Server, SunOne Web Server and Sun Enterprise System web server.

In all cases the plugin uses a special protocol named Apache JServ Protocol or simply AJP to connect to the backend. Backends known to support AJP are Apache Tomcat, Jetty and JBoss. Although there exist 3 versions of the protocol, ajp12, ajp13, ajp14, most installations only use ajp13. The older ajp12 does not use persistent connections and is obsolete, the newer version ajp14 is still experimental. Sometimes ajp13 is called AJP 1.3 or AJPv13, but we will mostly use the name ajp13.

Most features of the plugins are the same for all web servers. Some details vary on a per web server basis. The documentation and the configuration is split into common parts and web server specific parts.

down to the more detailed documentation that is available. Each available manual is described in more detail below.

For more info: http://tomcat.apache.org/connectors-doc/
# Let's start


### Installation

Install the dependencies server.

```sh
$ yum install httpd-devel apr apr-devel apr-util apr-util-devel gcc gcc-c++ make autoconf libtool
```

Download the connector from http://www-us.apache.org...

```sh
$ mkdir -p /opt/mod_jk/
$ cd /opt/mod_jk
$ wget http://www-us.apache.org/dist/tomcat/tomcat-connectors/jk/tomcat-connectors-1.2.42-src.tar.gz
$ tar -xvzf tomcat-connectors-1.2.42-src.tar.gz
$ cd tomcat-connectors-1.2.42-src/native/
```
For production environments...
```sh
$ ./configure --with-apxs=/usr/bin/apxs --enable-api-compatibility
$ make
$ libtool --finish /usr/lib64/httpd/modules
$ make install
```
If all goes well you're going to have the mod_jk.so installed on your /etc/httpd/modules folder.

### Configuration

```sh
$ vim $TOMCAT_HOME/conf/server.xml
```
Usually $TOMCAT_HOME is 
```sh
$ vim /usr/share/tomcat/conf/server.xml
```
Add under the <Service name="Catalina"> tag:
```sh
<!-- Define an AJP 1.3 Connector on port 8009 -->
<Connector port="8009" protocol="AJP/1.3" redirectPort="8443" />
```
And modify the Engine tag so its looks like:
```sh
<!-- Define an AJP 1.3 Connector on port 8009 -->
<Engine name="Catalina" defaultHost="localhost" jvmRoute="jvm1">
```

Observation 1: for each tomcat instance linked to your httpd server, you need to define a different jvmRoute parameter. For example, for a second instance you could use:

```sh
<!-- Define an AJP 1.3 Connector on port 8009 -->
<Engine name="Catalina" defaultHost="localhost" jvmRoute="jvm2">
```
Now, lets go with the httpd configuration. First, create a mod_jk.conf file in your conf.d folder:

```sh
$ vim /etc/httpd/conf.d/mod_jk.conf
```
Fill with the following:

```sh
LoadModule jk_module "/etc/httpd/modules/mod_jk.so"
JkWorkersFile /etc/httpd/conf/workers.properties
# Where to put jk shared memory
JkShmFile     /var/run/httpd/mod_jk.shm
# Where to put jk logs
JkLogFile     /var/log/httpd/mod_jk.log
# Set the jk log level [debug/error/info]
JkLogLevel    info
# Select the timestamp log format
JkLogStampFormat "[%a %b %d %H:%M:%S %Y] "
#JkRequestLogFormat "%w %V %T"
#JkEnvVar SSL_CLIENT_V_START worker1
```
Before continuing, create the folder to store the shared memory of the module:
```sh
$ mkdir -p /var/run/mod_jk
$ chown apache:apache /var/run/mod_jk
```
Now, create the workers.properties file: (look at the JkWorkersFile property on mod_jk.conf file):
```sh
$ vim /etc/httpd/conf/workers.properties
```
With the next content:
```sh
workers.apache_log=/var/log/httpd
worker.list=app1Worker
worker.stat1.type=status
 
worker.app1Worker.type=ajp13
worker.app1Worker.host=app1.myhost.com #put your app host here
worker.app1Worker.port=8009
```
For every app server from tomcat to httpd you're going to have a specific worker. Don't forget to define the worker first in the worker.list property. For example, lets assume we're going to add another app from tomcat:
```sh
workers.apache_log=/var/log/httpd
worker.list=app1Worker,app2Worker
worker.stat1.type=status
 
worker.app1Worker.type=ajp13
worker.app1Worker.host=app1.myhost.com #put your app host here
worker.app1Worker.port=8009
 
worker.app2Worker.type=ajp13
worker.app2Worker.host=app2.myhost.com #put your app host here
worker.app2Worker.port=8009
```
Well, everything looks good now. The final step is to configure the VirtualHost for every app on httpd:
```sh
vim /etc/httpd/conf.d/app1.conf
```
It's a good practice to maintain your VirtualHosts in separated files. Now, in your recently created app1.conf file:
```sh
<VirtualHost *:80>
    ServerName app1.myhost.com
    ServerAdmin admin@myhost.com
    LogFormat "%h %l %u %t \"%r\" %>s %b \"%{Referer}i\" \"%{User-agent}i\"" combined
    CustomLog /var/log/httpd/app1_access.log combined
    ErrorLog /var/log/httpd/app1_error.log
    <IfModule mod_jk.c>
       JkMount /* app1Worker
    </IfModule>
</VirtualHost>
```
We are connecting httpd with tomcat using the JkMount directive in the VirtualHost configuration. If for example you'are adding a VirtualHost for your second app use the app2Worker configured previously and so on for other apps.

```sh
systemctl restart httpd.service
```

### References

 - http://www.diegoacuna.me/installing-mod_jk-on-apache-httpd-in-centos-6-x7-x/
 - https://www.digitalocean.com/community/tutorials/how-to-install-apache-tomcat-7-on-centos-7-via-yum



[//]: # (These are reference links used in the body of this note and get stripped out when the markdown processor does its job. There is no need to format nicely because it shouldn't be seen. Thanks SO - http://stackoverflow.com/questions/4823468/store-comments-inins-markdown-syntax)
