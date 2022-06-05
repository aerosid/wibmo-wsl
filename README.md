# wibmo-wsl
Setting up WSL at Wibmo

# Notes
```bash
# batch.sh will hold a bunch of commands to run in one go.
touch ~/batch.sh && chmod +x ~/batch.sh

# update and upgrade installed packages.
sudo tee ~/batch.sh<<EOF
#!/bin/bash
set -e
set -x
sudo apt update
sudo apt -y upgrade 
sudo systemctl reboot
EOF
~/batch.sh

# install mysql
sudo tee ~/batch.sh<<EOF
#!/bin/bash
set -e
set -x
sudo apt install -y mysql-server
sudo systemctl enable mysql
sudo systemctl start mysql
sleep 10s
sudo systemctl status mysql
sudo mysql --execute="select version();"
sudo systemctl reboot
EOF
~/batch.sh

# configure database user, ubuntu
sudo tee ~/batch.sh<<EOF
#!/bin/bash
set -e
set -x
sudo mysql --execute="create user 'ubuntu'@'localhost' identified by 'hello';"
sudo mysql --execute="grant all privileges on *.* to 'ubuntu'@'localhost' with grant option;"
sudo mysql --execute="flush privileges;"
EOF
~/batch.sh
mysql -u ubuntu -p

# install redis
# see https://redis.io/docs/getting-started/installation/install-redis-on-linux/
sudo tee ~/batch.sh<<EOF
#!/bin/bash
set -e
set -x
curl -fsSL https://packages.redis.io/gpg | sudo gpg --dearmor -o /usr/share/keyrings/redis-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/redis-archive-keyring.gpg] https://packages.redis.io/deb $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/redis.list
sudo apt-get update
sudo apt-get install -y redis
sudo systemctl enable redis-server
sudo systemctl start redis-server
sleep 10s
sudo systemctl status redis-server
redis-cli ping
EOF
~/batch.sh

# install kafka
# see https://tecadmin.net/how-to-install-apache-kafka-on-ubuntu-20-04/

sudo apt install -y openjdk-11-jdk

sudo tee ~/batch.sh<<EOF
#!/bin/bash
set -e
set -x
wget https://archive.apache.org/dist/kafka/2.7.0/kafka_2.13-2.7.0.tgz
tar -xzf kafka_2.13-2.7.0.tgz
sudo mv kafka_2.13-2.7.0 /usr/local/kafka
ll /usr/local/kafka/bin/*.sh | grep -e zookeeper-server -e kafka-server
EOF
~/batch.sh

tee ~/zookeeper.service<<EOF
[Unit]
Description=Apache Zookeeper server
Documentation=http://zookeeper.apache.org
Requires=network.target remote-fs.target
After=network.target remote-fs.target
[Service]
Type=simple
ExecStart=/usr/local/kafka/bin/zookeeper-server-start.sh /usr/local/kafka/config/zookeeper.properties
ExecStop=/usr/local/kafka/bin/zookeeper-server-stop.sh
Restart=on-abnormal
[Install]
WantedBy=multi-user.target
EOF
sudo mv ~/zookeeper.service /etc/systemd/system
sudo chown root:root /etc/systemd/system/zookeeper.service

sudo tee ~/kafka.service<<EOF
[Unit]
Description=Apache Kafka Server
Documentation=http://kafka.apache.org/documentation.html
Requires=zookeeper.service
[Service]
Type=simple
Environment="JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64"
ExecStart=/usr/local/kafka/bin/kafka-server-start.sh /usr/local/kafka/config/server.properties
ExecStop=/usr/local/kafka/bin/kafka-server-stop.sh
[Install]
WantedBy=multi-user.target
EOF
sudo mv ~/kafka.service /etc/systemd/system
sudo chown root:root /etc/systemd/system/kafka.service

sudo systemctl daemon-reload
sudo systemctl enable zookeeper
sudo systemctl start zookeeper
sudo systemctl enable kafka
sudo systemctl start kafka
sudo systemctl status kafka

#see https://stackoverflow.com/questions/54059408/error-replication-factor-1-larger-than-available-brokers-0-when-i-create-a-k
/usr/local/kafka/bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic testTopic
/usr/local/kafka/bin/kafka-topics.sh --list --zookeeper localhost:2181
/usr/local/kafka/bin/kafka-console-producer.sh --broker-list localhost:9092 --topic testTopic
/usr/local/kafka/bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic testTopic --from-beginning

# install envoy
# see https://nextgentips.com/2021/12/14/how-to-install-envoy-proxy-server-on-ubuntu-20-04/
# see https://songrgg.github.io/architecture/deeper-understanding-to-envoy/
sudo tee ~/batch.sh<<EOF
#!/bin/bash
set -e
set -x
sudo apt install -y apt-transport-https gnupg2 curl lsb-release
curl -sL 'https://deb.dl.getenvoy.io/public/gpg.8115BA8E629CC074.key' | sudo gpg --dearmor -o /usr/share/keyrings/getenvoy-keyring.gpg
echo a077cb587a1b622e03aa4bf2f3689de14658a9497a9af2c427bba5f4cc3c4723 /usr/share/keyrings/getenvoy-keyring.gpg | sha256sum --check
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/getenvoy-keyring.gpg] https://deb.dl.getenvoy.io/public/deb/ubuntu $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/getenvoy.list
sudo apt update
sudo apt install -y getenvoy-envoy
envoy --version
EOF
~/batch.sh

tee ~/envoy-demo.yaml<<EOF
static_resources:
  listeners:
  - name: listener_0
    address:
      socket_address:
        address: 0.0.0.0
        port_value: 10000
    filter_chains:
    - filters:
      - name: envoy.filters.network.http_connection_manager
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
          stat_prefix: ingress_http
          access_log:
          - name: envoy.access_loggers.stdout
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.access_loggers.stream.v3.StdoutAccessLog
          http_filters:
          - name: envoy.filters.http.router
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router
          route_config:
            name: local_route
            virtual_hosts:
            - name: local_service
              domains: ["*"]
              routes:
              - match:
                  prefix: "/"
                route:
                  host_rewrite_literal: www.envoyproxy.io
                  cluster: service_envoyproxy_io
  clusters:
  - name: service_envoyproxy_io
    type: LOGICAL_DNS
    # Comment out the following line to test on v6 networks
    dns_lookup_family: V4_ONLY
    load_assignment:
      cluster_name: service_envoyproxy_io
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address:
                address: www.envoyproxy.io
                port_value: 443
    transport_socket:
      name: envoy.transport_sockets.tls
      typed_config:
        "@type": type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.UpstreamTlsContext
        sni: www.envoyproxy.io
EOF
envoy -c ~/envoy-demo.yaml

# install ansible
# see https://www.digitalocean.com/community/tutorials/how-to-install-and-configure-ansible-on-ubuntu-20-04
```
