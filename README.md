pllr.elasticsearch
=========

Install and configure Elasticsearch on a CentOS 7 or Rhel 7 virtual machine.

Requirements
------------

A CA certificate and key when bootstrapping with `xpack` security.

Role Variables
--------------

```yaml
---
## Custom variables
# This variable defaults to the hostname without the domain
node_name: "{{ inventory_hostname | regex_replace('\\..+', '') }}"

elasticsearch:
  dir:
    # Elasticsearch data directory
    data: /var/lib/elasticsearch
    # Elasticsearch logs directory
    logs: /var/logs/elasticsearch
  certificates: 
    # Elasticsearch certificates directory (empty if not using xpacx security)
    dir: /opt/private/ssl
    # Elasticsearch http certificate (empty if not using xpacx security)
    http_certificate_name: "{{ node_name }}-http-certs.p12"
    # Elasticsearch transport certificate (empty if not using xpacx security)
    transport_certificate_name: "{{ node_name }}-transport-certs.p12"

## /etc/elasticsearch/lasticsearch.yml
# Cluster name
cluster.name: "default"

# When bootstrapping using different zones
multiple_zones: false

node:
  name: "{{ node_name }}"
  # Whether the elasticsearch host is a data node
  data: true
  # Whether the elasticsearch host is a master node
  master: true

path:
  # uses the variable above for directories of data and logs
  data: "{{ elasticsearch.dir.data }}"
  logs: "{{ elasticsearch.dir.logs }}"

# Lock the memory on startup
bootstrap.memory_lock: true

# Set the bind address to a specific IP
network.host: "_eth0_,_local_"

# Set a custom port for HTTP
http.port: "9200"

# xpack security and monitoring
xpack:
  security:
    enabled: true
    http.ssl:
      enabled: true
      verification_mode: full
      keystore.path:  "{{ elasticsearch.certificates.dir }}/{{ elasticsearch.certificates.http_certificate_name }}"
      truststore.path: "{{ elasticsearch.certificates.dir }}/{{ elasticsearch.certificates.http_certificate_name }}"
    transport.ssl:
      enabled: true
      verification_mode: full
      keystore.path: "{{ elasticsearch.certificates.dir }}/{{ elasticsearch.certificates.transport_certificate_name }}"
      truststore.path: "{{ elasticsearch.certificates.dir }}/{{ elasticsearch.certificates.transport_certificate_name }}"
  monitoring.collection:
    enabled: true
```

Dependencies
------------

None.

Example Playbook
----------------

```yaml
---
- name: "Install and configure Elasticsearch"
  hosts: elasticsearch-master
  roles:
  - { role: pllr.elasticsearch }
```

License
-------

Apache-2

Author Information
------------------

Check me on [LinkedIn](www.linkedin.com/in/phil-ranzato-47b8bb194)

Tasks Analysis
------------------

`01-install.yml`

```yaml
---
# rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch
- name: "Import elasticsearch GPG key"
  rpm_key:
    state: present
    key: https://artifacts.elastic.co/GPG-KEY-elasticsearch

# cat << EOF > /etc/yum.repos.d/elasticsearch.repo
# [elasticsearch]
# name=Elasticsearch repository for 7.x packages
# baseurl=https://artifacts.elastic.co/packages/7.x/yum
# gpgcheck=1
# gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
# enabled=0
# autorefresh=1
# type=rpm-md
# EOF
- name: "Configure Elasticsearch repository"
  copy:
    src: "elasticsearch.repo"
    dest: "/etc/yum.repos.d/elasticsearch.repo"

# yum install -y --enablerepo=elasticsearch elasticsearch 
- name: "Install Elasticsearch"
  yum:
    name: elasticsearch
    enable_repo: elasticsearch
    state: latest
  # systemctl daemon-reload
  # systemctl enable elasticsearch
  # systemctl start elasticsearch
  notify: "Start Elasticsearch"
```
