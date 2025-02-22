# MySQL monitoring using Prometheus and Grafana and MySQL-Exporter  

for scalable system need monitoring for any anomalies or any issues in production environment.   

for this you need to install "docker" and "docker-compose" into your vps   

MySQL           : port 3306:3306  > config : .my.cnf    
MySQL Exporter  : port 9104:9104    
Prometheus      : port 9090:9090  > config : prometheus.yml    
Grafana         : port 3000:3000      


docker for linux(Ubuntu)   

> /docker-compose.yml  
```yml
version: '3'
services:

  mysql:
    image: mysql
    container_name: mysql
    restart: always
    environment:
      MYSQL_DATABASE: 'test'              # Name of the database
      MYSQL_USER: 'sample'                # Username
      MYSQL_PASSWORD: 'password'          # Password for 'sample' user
      MYSQL_ROOT_PASSWORD: 'password'     # Password for root user
    ports:
      - '3307:3306'                       # Map Docker MySQL port 3306 to host port 3307
    volumes:
      - ./mysql-db:/var/lib/mysql         # Persist MySQL data

  prometheus:
    image: prom/prometheus
    container_name: prometheus
    volumes:
      - "./prometheus.yml:/etc/prometheus/prometheus.yml"  # Mount prometheus.yml
    ports:
      - 9090:9090                         # Expose Prometheus on port 9090

  grafana:
    image: grafana/grafana
    container_name: grafana
    ports:
      - 3000:3000                         # Expose Grafana on port 3000
    restart: unless-stopped
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - ./grafana:/etc/grafana/provisioning/datasources # Persist Grafana configurations

  mysql-exporter:
    image: prom/mysqld-exporter
    container_name: mysql-exporter
    depends_on:
      - mysql
    command: 
      - --config.my-cnf=/cfg/.my.cnf
      - --mysqld.address=mysql:3306       # Connect to MySQL using its container name
    volumes:
      - "./.my.cnf:/cfg/.my.cnf"          # Mount .my.cnf for credentials
    ports:
      - 9104:9104                         # Expose MySQL exporter on port 9104

```
> /prometheus.yml
```bash
global:
  scrape_interval: 2s

scrape_configs:
 - job_name: prometheus
   static_configs:
    - targets:
       - prometheus:9090                 # Prometheus container name as the target
       
 - job_name: mysql_exporter
   static_configs:
    - targets:
       - mysql-exporter:9104            # MySQL exporter container name as the target
```
> /.my.cnf
```bash
[client]
user=root
password=password
host=mysql                        # Connect using the MySQL container name
```

By default, Docker directly manipulates iptables rules to allow traffic to containers. This behavior can bypass UFW, meaning Docker's port mappings (e.g., 3306 mapped to the MySQL container) are open to external connections even if UFW doesn't explicitly allow the port.  

 inspect the iptables rules directly to see if Docker has added rules for port 3306   
 ```bash
$ sudo iptables -L -n  
```
configure to prevent Docker from bypassing UFW.  
> /etc/docker/daemon.json
```bash
{
  "iptables": false
}
```
restart docker  
```bash
$ sudo systemctl restart docker
```

go to localhost:3000 for Grafana > (click) "Connections" on left menu > (click) "Data sources" on left menu > (click) "Add data source" button >  (click) "Prometheus" on list > (input) in "Prometheus server URL" value that you open on browser > (click) "save & test" button of bottom of page > (click) "Dashboards" on left menu > (click) "+ Create dashboard" > (click) "+ Add visualization" button > (select) the data source as "prometheus" 




