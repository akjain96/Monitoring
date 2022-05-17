![Elastic Search, Kibana](https://images.contentstack.io/v3/assets/bltefdd0b53724fa2ce/blt280217a63b82a734/6202d3378b1f312528798412/elastic-logo.svg)
​
# Setting up Monitoring on NGINX Instances 
​
#### Objective :
This document is made to guide end-user to setup Application specific monitoring on NGINX High Performance web-server. 
​
#### Tech Stack Used : 
- Elasticsearch (Self-hosted)
- Kibana
- File beat (Shippers)
- Metric beat (Shippers)
- Cloud Instances / Virtual machines
## Features
​
##### Pre-requisites
1. Install Docker (refer to official docs if not installed https://docs.docker.com/engine/install/)
2. Install Docker Compose (refer to official docs if not installed https://docs.docker.com/compose/install/)
3. Creating 2 different cloud hosted instances or Self hosted virtualized environment (Running virtual machines on OracleBox, VMWare etc.) (for hosting NGINX and ELK Stack), Preferably Ubuntu 20.04 Instances. 
​
Minimum specs requirement : 
ELK Stack Instance : 8 GiB RAM, 30 GB of General Purpose SSD Storage
NGINX Instance : 4 GiB RAM, 10 GB of General Purpose SSD Storage
(Failing to use lower compute resources lower than above mentioned requirement might halt your instances)
##### Steps to Setup Elastic Search, Kibana and NGINX : 
                                                    
*Setting up Elastic Search and Kibana (MASTER Configuration)* :
For setting up Elastic Search and Kibana firstly we need to create 2 docker containers. 
​
##### Step 1: 
Create a directory on your machine (VM/Cloud Instance) for this project : 
​
`mkdir $HOME/elasticsearch7-docker`
`cd $HOME/elasticsearch7-docker`
​
##### Step 2 : 
Create docker-compose.yml file
​
Inside that directory (after running above commands) create a docker-compose.yml file with contents as shown below
​
#### docker-compose.yml
```
version: '3'
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.9.2-amd64
    env_file:
      - elasticsearch.env
    volumes:
      - ./data/elasticsearch:/usr/share/elasticsearch/data
​
  kibana:
    image: docker.elastic.co/kibana/kibana:7.9.2
    env_file:
      - kibana.env
    ports:
      - 5601:5601
```
​
**IMPORTANT NOTE** : Do not use docker hub images of **elasticsearch** and **kibana** <= **7.9.2** as they are giving issues in docker networking. This is tested and verified. 
​
##### Step 3 : 
After creating docker-compose.yml, create two different env files :
1. elasticsearch.env
2. kibana.env 
to pass environment variables to docker-compose. 
​
#### elasticsearch.env
```
cluster.name=elasticsearch-cluster
network.host=0.0.0.0
bootstrap.memory_lock=true
discovery.type=single-node
```
​
Note: 
1. With latest version of Elasticsearch* , it is necessary to set the option **discovery.type=single-node** for a single node cluster otherwise it won't start
2. By default kibana instance is running on port : 5601
3. Elasticsearch instance is running on port : 9200
​
#### kibana.env
```
SERVER_HOST="0"
ELASTICSEARCH_URL=http://elasticsearch:9200
XPACK_SECURITY_ENABLED=false
```
​
##### Step 4: 
Create Elasticsearch data directory
​
Navigate to the directory where you have created your docker-compose.yml file and create a subdirectory data. Then inside the data directory create another directory elasticsearch.
​
```
mkdir data
cd data
mkdir elasticsearch
```
​
This directories are created by us as : 
We will be mounting this directory to the data directory of elasticsearch container (for data persistence). So, In docker-compose.yml file there are these lines:
​
    volumes:
      - ./data/elasticsearch:/usr/share/elasticsearch/data
      
This again ensures that the data on your Elasticsearch container persists even when the container is stopped and restarted later. So, you won't lose your indices when you restart the containers.
​
##### Step 5: 
Run the setup
Now after doing this all we are ready to setup our docker containers. Open terminal and navigate to the folder containing docker-compose.yml file and run the command:
​
``` docker-compose up -d ```
​
This will start pulling the images from **docker.elastic.co**. Once the images are pulled, it will start the containers.
​
**Command to check docker compose stack logs :** 
docker-compose logs -f {serviceName}
​
**A common error that you might encounter is related to vm.max_map_count being too low. You can fix it by running the command**
​
```sysctl -w vm.max_map_count=262144```
​
If both the services are running fine, you should be able to see kibana console on http://[public/private IP]:5601 on your web browser. Give it a few minutes as it takes some time for Elasticsearch cluster to be ready and for Kibana to connect to it. 
​
You can get more info by inspecting the logs using docker-compose logs -f kibana command.
​
##### Installing NGINX Web Server (SLAVE Configuration): 
​
Description :
NGINX is a high performance web server that can hosts websites and can have another different usage such as Reverse Proxy. 
​
##### Steps to install : 
​
###### We need to do some additional modifications in order to send metrics and application logs from NGINX web server to Kibana dashboard : 
1. Some configuration changes in NGINX Instance : 
A. To send NGINX specific metrics to kibana dashboard, we need to check whether : 
**ngx_http_stub_status_module** is enabled or not (on NGINX).
​
To check whether nginx, stub status module is enabled or not, you can run below command : 
``` nginx -V 2>&1 | grep --color -- --with-http_stub_status_module ```
​
B. We need to enable "**/nginx_status**" route path on NGINX web server (most important). 
This part ultimately help nginx to send metrics to kibana using **metricbeat shipper**.
​
To enable /nginx_status route we need to : 
Go to : /etc/nginx/sites-available/default (Open this file)
**Edit location code snippet**
#### Before : 
```
location / {
                # First attempt to serve request as file, then
                # as directory, then fall back to displaying a 404.
                try_files $uri $uri/ =404;
        }
```
​
#### After :
```
location /nginx_status {
                # First attempt to serve request as file, then
                # as directory, then fall back to displaying a 404.
                #try_files $uri $uri/ =404;
            ```
            stub_status;
        }
```
​
C. Test the Nginx configuration:
``` nginx -t ```
​
D. Reload the Nginx configuration:
``` nginx -s reload ```
​
E . Test **stub_status**:
``` curl http://<localhost/public/privateIP>/nginx_status ```
​
===
After configuring this, we need to configure Metricbeat and filebeat packages inside slave (NGINX) Instance. 
​
##### Download and install Metricbeat
​
STEP 1:
Copy below snippet
``` curl -L -O https://artifacts.elastic.co/downloads/beats/metricbeat/metricbeat-7.9.2-amd64.deb 
sudo dpkg -i metricbeat-7.9.2-amd64.deb
```
​
STEP 2:
Edit the configuration
​
** Modify /etc/metricbeat/metricbeat.yml to set the connection information: **
​
Copy snippet
```
output.elasticsearch:
  hosts: ["<es_url>"]
  username: "elastic"
  password: "<password>"
setup.kibana:
  host: "<kibana_url>"
```
Where <password> is the password of the elastic user, <es_url> is the URL of Elasticsearch, and <kibana_url> is the URL of Kibana.
​
3. Enable and configure the nginx, metricbeat module
​
Copy snippet
​
```
sudo metricbeat modules enable nginx
```
Modify the settings in the /etc/metricbeat/modules.d/nginx.yml file.
​
(Modify according to below code snippet)
​
The configuration file of Metricbeat nginx module is
​
/etc/metricbeat/modules.d/nginx.yml
Modify the configuration file.
​
before fixing:
```
​
- module: nginx
  #metricsets:
  #  - stubstatus
  period: 10s
​
  # Nginx hosts
  hosts: ["http://127.0.0.1"]
​
  # Path to server status. Default server-status
  #server_status_path: "server-status"
​
  #username: "user"
  #password: "secret"
```
​
After modification:
```
- module: nginx
  metricsets: ["stubstatus"]
  enabled: true
  period: 10s
​
  # Nginx hosts
  hosts: ["http://<localhost/public/private IP of NGINX>"]
​
  # Path to server status. Default server-status
  server_status_path: "nginx_status"
```
​
4. Start Metricbeat
​
The setup command loads the Kibana dashboards. If the dashboards are already set up, omit this command.
​
Copy snippet
```
sudo metricbeat setup
sudo service metricbeat start
```
====
​
After completing all the above steps we have enabled sending RPS metrics from NGINX web server to Kibana. And we can see the same on kibana dashboard. Metrics such as System Metrics (CPU, Memory and others) + RPS Metrics. 
​
Additional Step : 
To send application and error logs (NGINX) to kibana : 
​
**1. Download and install Filebeat**
​
Copy snippet
```
curl -L -O https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-7.9.2-amd64.deb
sudo dpkg -i filebeat-7.9.2-amd64.deb
```
**2. Edit the configuration**
Modify /etc/filebeat/filebeat.yml to set the connection information:
​
Copy snippet
```
output.elasticsearch:
  hosts: ["<es_url>"]
  username: "elastic"
  password: "<password>"
setup.kibana:
  host: "<kibana_url>"
```
Where <password> is the password of the elastic user, <es_url> is the URL of Elasticsearch, and <kibana_url> is the URL of Kibana.
​
**3. Enable and configure the nginx (filebeat) module**
​
Copy snippet
```
sudo filebeat modules enable nginx
```
​
**Specify the Nginx log file path to be collected (optional)**
edit - /etc/filebeat/modules.d/nginx.yml
​
Specify the Nginx log file path to be collected, and support glob fuzzy matching.
​
Example:
```
before fixing:
​
- module: nginx
  # Access logs
  access:
    enabled: true
​
    # Set custom paths for the log files. If left empty,
    # Filebeat will choose the paths depending on your OS.
    #var.paths:
​
  # Error logs
  error:
    enabled: true
​
    # Set custom paths for the log files. If left empty,
    # Filebeat will choose the paths depending on your OS.
    #var.paths:
```
```
After modification:
​
- module: nginx
  # Access logs
  access:
    enabled: true
​
    # Set custom paths for the log files. If left empty,
    # Filebeat will choose the paths depending on your OS.
    var.paths: ["/var/log/nginx/*access.log"]
​
  # Error logs
  error:
    enabled: true
​
    # Set custom paths for the log files. If left empty,
    # Filebeat will choose the paths depending on your OS.
    var.paths: ["/var/log/nginx/*error.log"]
```
**4. Start Filebeat**
The setup command loads the Kibana dashboards. If the dashboards are already set up, omit this command.
​
##### Copy snippet
```
sudo filebeat setup
sudo service filebeat start
```
