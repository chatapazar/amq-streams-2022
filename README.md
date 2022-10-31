# AMQ Streams Training 2022

This repository includes the demos and labs for the AMQ Steams 2022.
Following demos are included:

* [Introduction to Kafka](./kafka-introduction)
    * Running Kafka locally
    * Console consumer
    * Console producer
    * Management tools
    * Connect standalone
* [Kafka Architecture](./kafka-architecture)
    * Topics
    * Partitions
    * Replicas
    * Consumers
    * Producers
    * Consumer Groups

## Prepare Lab
* secure shell to lab terminal (receive user/password from instructor)
* for first time login type 'yes' to confirm login with password, Are you sure you want to continue connecting (yes/no/[fingerprint])?
```bash
ssh <username@lab>
student@bastion.k5mnq.sandbox322.opentlc.com's password: <your passowrd>
```
* install openjdk
sudo yum install java-11-openjdk-devel
* openssl (rhel already install)
* install cfssl
VERSION=$(curl --silent "https://api.github.com/repos/cloudflare/cfssl/releases/latest" | grep '"tag_name"' | sed -E 's/.*"([^"]+)".*/\1/')
VNUMBER=${VERSION#"v"}
wget https://github.com/cloudflare/cfssl/releases/download/${VERSION}/cfssl_${VNUMBER}_linux_amd64 -O cfssl
chmod +x cfssl
sudo mv cfssl /usr/local/bin

* clone git from URL: https://github.com/chatapazar/amq-streams-2022
```bash
cd ~
git clone https://github.com/chatapazar/amq-streams-2022
```

* a