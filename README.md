## Following is the procedure I followed to configure "Reverse proxy" using ansible playbooks :
Firstly Reverse proxy needs proxy server and webserver. Reverse proxy is used to balance the load on the server. In accomplishing this task I have used httpd and php for the webserver and haproxy for the proxy server

To work with ansible we need to create an ansible.cfg file in the present working directory as ansible.cfg or .ansible.cfg file in the home directory of the user. Following is how my ansible.cfg(I have created this in my present working directory) file looks
```conf
[defaults]
inventory=./inventory/hosts
remote_user=root
host_key_checking=false
```
as I have mentioned inventory file path in the ansible.cfg file as ./inventory/hosts I have created inventory file as follows in the mentioned path
```conf
#Enter the IP addresses and the passwords of the name node below
[servers]
# Examples : 
# 192.168.0.7 ansible_ssh_pass=password
# 192.168.0.9 ansible_ssh_pass=password
#Enter the IP addresses and the passwords of the data nodes below
[proxy]
# Examples : 
# 192.168.0.10 ansible_ssh_pass=password
```
The group names for the hosts are used to configure the hosts and also we use the groups and get the server ip addresses and use them in haproxy.cfg file in the above file under servers we need to give the ip addresses and passwords of the web servers and under proxy we need to give the ip addresses and passwords of the reverse proxy server

## Common Task which will configure the yum in both server and proxy server :
To use the httpd, haproxy and php softwares firstly we need to install them, as I am using a RedHat operating system it needs yum to be configured to install httpd or php or any software, as I am using a configuration file that can install most of the softwares, here is the configuration file for the most of the softwares I am using that for the repositories of softwares

In the configuration file the baseurl is where the software is present and gpgcheck is kept 0 as we don't want key checking during installation now after configuring the yum we need to install the packages httpd and php

After creating the yum configuration file we need to copy this file to the webserver to install the softwares using yum as the repositories of yum will be in /etc/yum.repos.d I have copied the repository file to this folder with the ansible copy module. Following is the task that I have used to copy the yum configuration file
```yaml
- hosts : servers
  tasks :
          
          - name : "configuring yum for all hosts to install httpd, php and  haproxy"
            copy :
                    src : "yum.repo"
                    dest : "/etc/yum.repos.d/reverseproxy.repo"
```
## Configuring the webserver :
After copying the configuration file for yum I installed the packages using package module of ansible it will decide which command to be used based on the operating system and installs the software, in my case it chooses yum/dnf in background as I am using RedHat, as we need httpd and php softwares in webserver we can install them using the following tasks
```yaml
          - name : "installing httpd package for server"
            package :
                  name : "httpd"
                  state : present
          - name : "installing php package for server"
            package :
                  name : "php"
                  state : present      
```
After installing the packages copying the webpages to the webserver using the following tasks
```yaml
          - name : "removing the index.html file on webserver"
            file :
                  path : "/var/www/html/index.html"
                  state : absent
          - name : "copying the php file to the webserver"
            copy :
                  src : "./index.php"
                  dest : "/var/www/html/index.php"
```
The webpage content is as follows
```php
<pre>
<?php
print `/usr/sbin/ifconfig`;
?>
</pre>
```

After copying the content we need to configure the webserver using the configuration file here I have used the the variables stored in the vars.yml file to know on which the webserver must be launched. In the httpd configuration file I have used the server_port variable to launch the web server. The complete httpd.conf file can be found here

To copy the httpd configuration file I have used the following
```yaml
          - name : "copying the httpd configuration file to server"
            template :
                  src : "./httpd.conf"
                  dest : "/etc/httpd/conf/httpd.conf"

            notify : "restarting httpd service"
```
The vars.yml file having the variables is as follows
```yaml
- server_port : <webserver_port>
- proxy_port : <proxy_server_port>
```
I also checked if any process is running on these ports before starting the webserver and haproxy server and killed the process using the following task for webserver
```yaml
          - name : "killing the service running port on server_port"
            shell : "kill `netstat -tnlp | grep :{{server_port}} | awk '{print $7}' | awk -F/ '{print $1}'`"

            ignore_errors : yes
```
I have opened the port on the firewall using the following
```yaml
          - name : "restarting firewall"
            service :
                  name : "firewalld"
                  state : restarted
          - name : "changing settings for firewall"
            firewalld :
                  port : "{{server_port}}/tcp"
                  state : enabled
                  immediate : yes
```
I have used handlers to restart the httpd service such that it will restart whenever there is a change in configuration file following is the handler I have used
```yaml
  handlers :
          - name : "restarting httpd service"
            service :
                  name : "haproxy"

                  state : restarted
```
## Configuring haproxy service :
After copying the configuration file for yum I installed the packages using package module of ansible it will decide which command to be used based on the operating system and installs the software in my case it chooses yum/dnf in background as I am using RedHat, as we need haproxy software for reverse proxy hence we need to do the following to install the haproxy software
```yaml
          - name : "installing haproxy"
            package :
                  name : "haproxy"
                  state : present    
```
We need to configuring the haproxy using the configuration file I have used the the variables that are storing the port on which the webserver must be launched to know the port of the webserver and ports on which the haproxy server must be launched to know the port of the proxy server. To copy the haproxy configuration file I have used the following
```yaml
          - name : "copying the haproxy configuration file"
            template :
                  src : "./haproxy.cfg"
                  dest : "/etc/haproxy/haproxy.cfg"
            notify : "restarting haproxy service"
```
In the haproxy.cfg file I have used jinja templating to dynamically add the hosts of the webserver for the proxy. I have used the following for adding the webservers. The complete haproxy.cfg file can be found here
```cfg
{% for i in groups['servers'] %}

    server  app{{loop.index}} {{i}}:{{server_port}} check

{% endfor %}
```
The vars.yml file where I have stored the port numbers is as follows
```yaml
- server_port : <server_port>
- proxy_port : <proxy_port>
```

I also checked if any process is running on these ports before starting the webserver and haproxy server and killed the process using the following task for proxy server
```yaml
          - name : "killing the service running port on proxy_port"
            shell : "kill `netstat -tnlp | grep :{{proxy_port}} | awk '{print $7}' | awk -F/ '{print $1}'`"
            ignore_errors : yes
```
I have opened the port on the firewall using the following
```yaml
          - name : "restarting firewall"
            service :
                  name : "firewalld"
                  state : restarted
          - name : "changing settings for firewall"
            firewalld :
                  port : "{{proxy_port}}/tcp"
                  state : enabled
                  immediate : yes
```
I have used handlers to restart the haproxy service such that it will restart whenever there is a change in configuration file it will restart the haproxy server
```yaml
  handlers :
          - name : "restarting haproxy service"
            service :
                  name : "haproxy"
                  state : restarted
```
