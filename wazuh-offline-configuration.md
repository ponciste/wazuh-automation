# Configuration steps after changing Wazuh IP address.

This is a step-by-step guide in order to have Wazuh working corrently on a different IP address.

## Changes to Config Files

First of all, the configuration files containing the old IP address need to be adjusted.

### Change `opensearch.yml` configuration

```console
nano /etc/wazuh-indexer/opensearch.yml
```

Adjust the following line: 

```console
...
network.host: localhost
...
```

### Change `wazuh.yml` configuration

```console
nano /usr/share/wazuh-dashboard/data/wazuh/config/wazuh.yml
```

Adjust the following line: 

```console
...
url: https://localhost
...
```

### Change `filebeat.yml` configuration

```console
nano /etc/filebeat/filebeat.yml
```

Adjust the following line: 

```console
...
output.elasticsearch.hosts:
  - 127.0.0.1:9200
...
```

### Change `opensearch_dashboards.yml` configuration

```console
nano /etc/wazuh-dashboard/opensearch_dashboards.yml
```

Adjust the following line: 

```console
...
server.host: 0.0.0.0
opensearch.hosts: https://localhost:9200
...
```

## Changes to SSL Certificates

The second necessary step, is to generate new SSL certificate and move them over the right Wazuh components folder.

### Delete current SSL certificates

```console
 rm -rf /etc/wazuh-indexer/certs/
 rm -rf /etc/filebeat/certs/
 rm -rf /etc/wazuh-dashboard/certs/
```

### Generate new SSL certificates

First, get the installation files to generate the certificates.

```console
cd /home/ironhack
curl -sO https://packages.wazuh.com/4.7/wazuh-certs-tool.sh
curl -sO https://packages.wazuh.com/4.7/config.yml
```

Edit the `config.yml` file by adjusting the IP address for the indexer, manager and dashboard.

It should look exactly like below, of course will the correct IP address.

```console
nodes:
  # Wazuh indexer nodes
  indexer:
    - name: node-1
      ip: "127.0.0.1"
    #- name: node-2
    #  ip: "<indexer-node-ip>"
    #- name: node-3
    #  ip: "<indexer-node-ip>"

  # Wazuh server nodes
  # If there is more than one Wazuh server
  # node, each one must have a node_type
  server:
    - name: wazuh-1
      ip: "127.0.0.1"
    #  node_type: master
    #- name: wazuh-2
    #  ip: "<wazuh-manager-ip>"
    #  node_type: worker
    #- name: wazuh-3
    #  ip: "<wazuh-manager-ip>"
    #  node_type: worker

  # Wazuh dashboard nodes
  dashboard:
    - name: dashboard
      ip: "127.0.0.1"
```

Run the certs tool, compress and delete the output wazuh-certificates folder:

```console
bash ./wazuh-certs-tool.sh -A
tar -cvf ./wazuh-certificates.tar -C ./wazuh-certificates/ .
rm -rf ./wazuh-certificates
```

**NOTE:** at this points the certificates are generated and they need to be copied over the Wazuh components below. Basically the steps needs to be repeated.

### Indexer SSL certificates

```console
NODE_NAME=node-1
mkdir /etc/wazuh-indexer/certs
tar -xf ./wazuh-certificates.tar -C /etc/wazuh-indexer/certs/ ./$NODE_NAME.pem ./$NODE_NAME-key.pem ./admin.pem ./admin-key.pem ./root-ca.pem
chmod 500 /etc/wazuh-indexer/certs
chmod 400 /etc/wazuh-indexer/certs/*
chown -R wazuh-indexer:wazuh-indexer /etc/wazuh-indexer/certs
```

Restart the indexer service:

```console
systemctl restart wazuh-indexer
```

Test the indexer service:

```console
systemctl status wazuh-indexer
```

### Filebeat SSL certificates

```console
NODE_NAME=wazuh-1
mkdir /etc/filebeat/certs
tar -xf ./wazuh-certificates.tar -C /etc/filebeat/certs/ ./$NODE_NAME.pem ./$NODE_NAME-key.pem ./root-ca.pem
chmod 500 /etc/filebeat/certs
chmod 400 /etc/filebeat/certs/*
chown -R root:root /etc/filebeat/certs
```

Restart the filebeat service:

```console
systemctl restart filebeat
```

Test the filebeat service:

```console
filebeat test output
```

### Dashboard SSL certificates

```console
NODE_NAME=dashboard
mkdir /etc/wazuh-dashboard/certs
tar -xf ./wazuh-certificates.tar -C /etc/wazuh-dashboard/certs/ ./$NODE_NAME.pem ./$NODE_NAME-key.pem ./root-ca.pem
chmod 500 /etc/wazuh-dashboard/certs
chmod 400 /etc/wazuh-dashboard/certs/*
chown -R wazuh-dashboard:wazuh-dashboard /etc/wazuh-dashboard/certs
```

Restart the dashboard service:

```console
systemctl restart wazuh-dashboard
```

Test the dashboard service:

```console
systemctl status wazuh-dashboard
```

## Contact

In case extra clarifications on the procedure need to be provided feel free to contact me:

**Name:** Stefano Poncini

**E-mail:** ponciste@gmail.com 
