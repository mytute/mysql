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
version: '3.8'

networks:
  my_network:
    driver: bridge

services:
  mysql:
    image: mysql:5.7
    container_name: mysql
    restart: always
    environment:
      MYSQL_DATABASE: 'test'
      MYSQL_USER: 'sample'
      MYSQL_PASSWORD: 'password'
      MYSQL_ROOT_PASSWORD: 'password'
    ports:
      - '3307:3306'
      #command: --bind-address=0.0.0.0 #--port=3306
    volumes:
      - ./mysql-db:/var/lib/mysql
      #- ./mysql-config/my.cnf:/etc/my.cnf 
    networks:
      - my_network

  prometheus:
    image: prom/prometheus
    container_name: prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - 9090:9090
    networks:
      - my_network

  grafana:
    image: grafana/grafana
    container_name: grafana
    ports:
      - 3000:3000
    restart: unless-stopped
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - ./grafana:/etc/grafana/provisioning/datasources
    networks:
      - my_network

  mysql-exporter:
    image: quay.io/prometheus/mysqld-exporter
    container_name: mysql-exporter
    restart: unless-stopped
    depends_on:
      - mysql
      #environment:
      #DATA_SOURCE_NAME: "exporter:password@(mysql:3306)/"
    command:
      - "--mysqld.username=exporter:password"
      - "--mysqld.address=mysql:3306"
      - --config.my-cnf=/cfg/.my.cnf
      # - --mysqld.address=mysql:3306  # Connect to MySQL using the service name
    volumes:
      - "./.my.cnf:/cfg/.my.cnf"
    ports:
      - 9104:9104
    networks:
      - my_network

```
> prometheus.yml  
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
      #- mysql-exporter:9104            # MySQL exporter container name as the target
       - 188.166.227.124:9104

```
> /.my.cnf
```bash
[client]
user=exporter
password=password
host=mysql
port=3306
[client.servers]
user=exporter
password=password
host=mysql
port=3306
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

go to <localhost_or_vps_ip_adddress>:3000 for Grafana > (click) "Connections" on left menu > (click) "Data sources" on left menu > (click) "Add data source" button >  (click) "Prometheus" on list > (input) in "Prometheus server URL" value that you open on browser > (click) "save & test" button of bottom of page > (click) "Dashboards" on left menu > (click) "+ Create dashboard" > (click) "+ Add visualization" button > (select) the data source as "prometheus" 

debug   
```bash
# check ip address of docker container
$ docker inspect <container_name_or_id> | grep IPAddress   

# create user for exporter
mysql> CREATE USER 'exporter'@'%' IDENTIFIED BY 'password' WITH MAX_USER_CONNECTIONS 3;
mysql> GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'exporter'@'%';
mysql> FLUSH PRIVILEGES;

# check users and host of mysql
mysql> SELECT user, host FROM mysql.user;

# check port and bind address of mysql container   
$ sudo docker exec -it  mysql bash
sudo docker exec -it  mysql bash
mysql> SHOW VARIABLES LIKE 'port';
mysql> SHOW VARIABLES LIKE 'bind_address';

# check if a specific port on a remote host is open and reachable
$ nc -zv   172.18.0.2  3306

# check ip address of docker using netword
$ sudo docker inspect  prometheus_my_network

# check mysql port
$ sudo tcpdump -i any port 3306 -A -nn

# connect to mysql
$ mysql -h <your_host> -P 3306 -u root -p

# check bind address of mysql docker container
$ mysql -h 172.18.0.2 -P 3306 -u root -ppassword -e "SHOW VARIABLES LIKE 'bind_address';"

# to restart docker-compose with all
$ sudo docker-compose restart   

# to revoe firewall
$ sudo iptables -I DOCKER-USER -p tcp --dport 3306 -j ACCEPT
```

### queries   
```bash

# calculates the derivative of the query count, showing how the query rate is changing over time.
> deriv(mysql_global_status_queries[60s])

# calculates the rate of queries executed per second over the last 60 seconds
# Queries executed by both user connections and internal processes.
> rate(mysql_global_status_queries[60s])

# tracks the rate of queries and commands executed.
# Does not include internal queries generated by MySQL itself (e.g., queries generated by stored procedures or triggers).   
> rate(mysql_global_status_questions[60s])

# shows the number of currently open connections to MySQL.
> mysql_global_status_threads_connected

# 
> rate(mysql_global_status_bytes_received[60s])   
```











