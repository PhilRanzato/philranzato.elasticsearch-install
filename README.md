pllr.elasticsearch
=========

Install Elasticsearch on a CentOS 7 or Rhel 7 OS.

Requirements
------------

None.

Role Variables
--------------

```yaml
---
# Lock elasticsearch version with yum versionlock
lock_es_version: true
```

Dependencies
------------

None.

Example Playbook
----------------

```yaml
---
- name: "Install Elasticsearch"
  hosts: elasticsearch-master
  roles:
  - { role: pllr.elasticsearch-install }
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

`02-host-configuration.yml`

```yaml
---
# disable swapoff (swapoff -a)
- name: "Disable swapoff"
  shell: swapoff -a
  become: true

# cat << EOF >> /etc/security/limits.conf
# elasticsearch  -  nofile  65535
# EOF
- name: "Configure Limits"
  pam_limits:
    domain: elasticsearch
    limit_type: hard
    limit_item: nofile
    value: '65535'
  become: true

# mkdir -p /etc/systemd/system/elasticsearch.service.d
- name: "Create oveeride folder for systemd elaticsearch service"
  file:
    state: directory
    path: /etc/systemd/system/elasticsearch.service.d
    recurse: yes
  become: true

# cat << EOF > /etc/systemd/system/elasticsearch.service.d/override.conf
# [Service]
# LimitMEMLOCK=infinity
# EOF
- name: "Copy the override configuration"
  copy:
    src: override.conf
    dest: /etc/systemd/system/elasticsearch.service.d/override.conf
  become: true
  notify: "Reload daemons"

- name: "Enforce ulimits"
  shell: ulimit -n 65535

# systemctl daemon-reload
- name: "Flush Handlers"
  meta: flush_handlers
```

`03-install-java.yml`

```yaml
---
# sudo yum install java-11-openjdk-devel
- name: "Install java"
  yum: 
    name: java-11-openjdk-devel
    state: latest
  become: true
```

`04-yum-versionlock.yml`

```yaml
---
# yum install -y yum-plugin-versionlock
- name: "Install yum versionlock plugin"
  yum:
    name: yum-plugin-versionlock
    state: present
  become: true

- name: "Lock current version for elasticsearch"
  shell: yum versionlock elasticsearch
  # expecting a module, disable warnings for not using yum module
  args:
    warn: false
  become: true
```
