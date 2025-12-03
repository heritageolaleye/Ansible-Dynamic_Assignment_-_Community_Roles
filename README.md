## Table of Content
- [overview](#overview)
- [Key Concepts](#key-concepts)
- [prerequisites](#prerequisites)
- [Dynamics Assignments Implementation](#dynamic-assignments-implementation)
- [Community Roles](#community-roles)
- [Load Balancer Implementation](#load-balancer-roles)
- [Conclusion](#conclusion)
## Overview
This guide outlines the implementation of dynamic assignments in Ansible using the include module and demonstrates the integration of community roles for configuring UAT environments. The implementation covers database setup, load balancer configuration, and web server deployment.

## Key Concepts
- Dynamic assignments using include module
- Community roles integration
- Multi-environment configuration management
- Load balancer failover setup

## Prerequisites

## Required Infrastructure
- Ansible control node
- UAT web servers (2 instances)
- Load balancer server
- Database server

## Required Tools
- Ansible (version 2.8 or higher)
- Git
- Visual Studio Code (optional)

## Initial Directory Structure

```bash
ansible-config-mgt/
├── dynamic-assignments/
├── inventory/
│   ├── dev/
│   ├── prod/
│   ├── staging/
│   └── uat/
├── playbooks/
└── static-assignments/
```

## Dynamic Assignments Implementation

## Directory Configuration

1. Create the dynamic assignments structure:

```bash
mkdir dynamic-assignments
touch dynamic-assignments/env-vars.yml
```
2. Create environment variable directories:

```bash
mkdir env-vars
touch env-vars/{dev,stage,uat,prod}.yml
```
![ansible](/images/ansible.jpg)

# Environment Variables Configuration

1. Configure env-vars.yml:

```bash
---
- name: Load environment variables
  include_vars: "{{ item }}"
  with_first_found:
    - files:
        - dev.yml
        - stage.yml
        - prod.yml
        - uat.yml
      paths:
        - "{{ playbook_dir }}/../env-vars"
  tags:
    - always
```

# Update site.yml with dynamic assignments

Update site.yml file to make use of the dynamic assignment. (At this point, we cannot test it yet. We are just setting the stage for what is yet to come. So hang on to your hats)

*site.yml* should now look like this.

```bash
---
- hosts: all
  name: Include dynamic variables
  become: yes
  tasks:
    - include_tasks: ../dynamic-assignments/env-vars.yml
      tags:
        - always

- import_playbook: ../static-assignments/common.yml

- import_playbook: ../static-assignments/uat-webservers.yml

- import_playbook: ../static-assignments/loadbalancers.yml
```

## Community Roles
Utilizing community roles can significantly enhance productivity. For instance, to install MySQL, leverage the geerlingguy.mysql role from Ansible Galaxy.

## Download Mysql Ansible Role

You can browse available community roles **[here](https://galaxy.ansible.com/)**. We will be using a **[MySQL role developed by geerlingguy](https://galaxy.ansible.com/geerlingguy/mysql)**.

*Configure VSCode to ssh into your Ansible server. Once this is done and you are connected to the ansible server through VSCode, the next step is to make sure that git is installed with git --version, then go to ansible-config-mgt directory and run*

```bash
git init
git pull https://github.com/<your-name>/ansible-config-mgt.git
git remote add origin https://github.com/<your-name>/ansible-config-mgt.git
git branch roles-feature
git switch roles-feature
```
Inside roles directory create your new MySQL role with ansible-galaxy install geerlingguy.mysql

Rename the folder to mysql

![ansible2](/images/ansible2.jpg)

Read README.md file, and edit roles configuration to use correct credentials for MySQL required for the tooling website.

Create Database and mysql user (roles/mysql/vars/main.yml)

```bash
mysql_root_password: ""
mysql_databases:
  - name: tooling
    encoding: utf8
    collation: utf8_general_ci
mysql_users:
  - name: webaccess
    host: "172.31.18.0/20" # The webserver's subnet cidr
    password: testR00t.3
    priv: "tooling.*:ALL"
```
![ansible3](/images/ansible3.jpg)

Create a new playbook inside static-assignments folder and name it db-servers.yml , update it with mysql roles.

```bash
- hosts: db_servers
  become: yes
  vars_files:
    - vars/main.yml
  roles:
    - { role: mysql }
```
![ansible4](/images/ansible4.jpg)

![ansible20](/images/ansible20.jpg)

Now it is time to upload the changes into your GitHub:

```bash
git add .
git commit -m "Commit new role files into GitHub"
git push --set-upstream origin roles-feature
```
![ansible5](/images/ansible5.jpg)

![ansible6](/images/ansible6.jpg)

Now, if you are satisfied with your codes, you can create a Pull Request.
Merge it to main branch on GitHub

![ansible7](/images/ansible7.jpg)

## Load Balancer roles
We want to be able to choose which Load Balancer to use, Nginx or Apache, so we need to have two roles respectively:

1. Nginx
2. Apache

With our experience on Ansible so far you can:
- Decide if you want to develop your own roles, or find available ones from the community

I would be using the community route. I can do so using these commands:

```bash
ansible-galaxy role install geerlingguy.nginx

ansible-galaxy role install geerlingguy.apache
```

![ansible8](/images/ansible8.jpg)

Rename the installed Nginx and Apache roles

```bash
mv geerlingguy.nginx nginx

mv geerlingguy.apache apache
```
![ansible9](/images/ansible9.jpg)

# For nginx

```bash
# roles/nginx/defaults/main.yml
enable_nginx_lb: false
load_balancer_is_required: false
```
![ansible10](/images/ansible10.jpg)

# For apache

```bash
# roles/apache/defaults/main.yml
enable_apache_lb: false
load_balancer_is_required: false
```

## Update assignment

This role (nginx or apache) is applied if both conditions specified in the when statement are met:

*loadbalancers.yml file*

```bash
#loadbalancers.yml
---
- hosts: lb
  become: yes
  roles:
    - role: nginx
      when: enable_nginx_lb | bool and load_balancer_is_required | bool
    - role: apache
      when: enable_apache_lb | bool and load_balancer_is_required | bool
```
![ansible12](/images/ansible12.jpg)

## Update site.yml files respectively

```bash
---
- hosts: all
  name: Include dynamic variables
  become: yes
  tasks:
    - include_tasks: ../dynamic-assignments/env-vars.yml
      tags:
        - always

- import_playbook: ../static-assignments/common.yml

- import_playbook: ../static-assignments/uat-webservers.yml

- import_playbook: ../static-assignments/loadbalancers.yml

- import_playbook: ../static-assignments/db-servers.yml
```
![ansible13](/images/ansible13.jpg)

Now you can make use of env-vars\uat.yml file to define which loadbalancer to use in UAT environment by setting respective environmental variable to true.

You will activate load balancer, and enable nginx by setting these in the respective environment's env-vars file.

```bash
enable_nginx_lb: true
enable_apache_lb: false
load_balancer_is_required: true
```
![ansible14](/images/ansible14.jpg)

## Set up for Nginx Load Balancer

# Update roles/nginx/defaults/main.yml

Configure Nginx virtual host

```bash
nginx_vhosts:
  - listen: "80"
    server_name: "oikiacc.duckdns.org"
    root: "/var/www/html"
    index: "index.php index.html index.htm"
    locations:
      - path: "/"
        proxy_pass: "http://myapp1"
    server_name_redirect: "oikiacc.duckdns.org"
    template: "{{ nginx_vhost_template }}"
    state: "present"

nginx_upstreams:
  - name: myapp1
    strategy: "ip_hash"
    keepalive: 16
    servers:
      - "10.128.0.18 weight=5"
      - "10.128.0.19 weight=5"

nginx_log_format: |-
  '$remote_addr - $remote_user [$time_local] "$request" '
  '$status $body_bytes_sent "$http_referer" '
  '"$request_time" "$upstream_response_time" "$pipe" "$http_user_agent"'
```
![ansible15](/images/ansible15.jpg)

## Update inventory/uat.ini

```bash
[uat-webservers]
10.128.0.18 ansible_user=ennoiaconcept ansible_ssh_private_key_file=~/.ssh/gcp-key
10.128.0.19 ansible_user=ennoiaconcept ansible_ssh_private_key_file=~/.ssh/gcp-key

[lb]
10.128.0.20 ansible_user=ennoiaconcept ansible_ssh_private_key_file=~/.ssh/gcp-key

[db_servers]
10.128.0.15 ansible_user=ennoiaconcept ansible_ssh_private_key_file=~/.ssh/gcp-key
```
![ansible21](/images/ansible21.jpg)

## Update Webservers Role in roles/webservers/tasks/main.yml to install Epel, Remi's repoeitory, Apache, PHP and clone the tooling website from your GitHub repository

```bash
---
- name: Install Apache
  become: true
  ansible.builtin.apt:
    name: "apache2"
    state: present

- name: Install Git
  become: true
  ansible.builtin.apt:
    name: "git"
    state: present

- name: Install PHP and required modules
  become: true
  ansible.builtin.apt:
    name:
      - php
      - libapache2-mod-php
      - php-mysql
      - php-cli
      - php-curl
      - php-zip
      - php-mbstring
      - php-xml
    state: present
    update_cache: yes

- name: Enable PHP module in Apache
  become: true
  ansible.builtin.command: a2enmod php*
  args:
    warn: false

- name: Restart Apache to apply PHP configuration
  become: true
  ansible.builtin.service:
    name: apache2
    state: restarted
```
## Update roles/nginx/tasks/main.yml with the code below to create a task that check and stop apache if it is running

```bash
---
- name: Check if Apache is running
  ansible.builtin.service_facts:

- name: Stop and disable Apache if it is running
  ansible.builtin.service:
    name: apache2
    state: stopped
    enabled: no
  when: "'apache2' in services and services['apache2'].state == 'running'"
  become: yes
```
## Configuration Testing

Now run the playbook against your uat inventory

```bash
ansible-playbook -i inventory/uat.ini playbooks/site.yml --extra-vars "@env-vars/uat.yml"
```
![ansible16](/images/ansible16.jpg)

![ansible17](/images/ansible17.jpg)

## Confirm that Nginx is enabled and running and Apache is disabled

![ansible22](/images/ansible22.jpg)

## Check ansible configuration for Nginx

```bash
sudo vi /etc/nginx/nginx.conf
```
![ansible23](/images/ansible23.jpg)

## Access the tooling website using the Public IP address on a browser

![ansible18](/images/ansible18.jpg)

![ansible19](/images/ansible19.jpg)

## Conclusion

In this guide, we have successfully navigated the complexities of implementing dynamic assignments and community roles in Ansible for configuring a UAT environment. We looked at the essential components required for setting up infrastructure, including web servers, load balancers, and database servers, etc... while utilizing community roles to beef-up our productivity.

When we were carrying out this project, the following was touched on:

- Dynamic Assignments: We learned how to utilize the include module to manage dynamic variables effectively, thus allowing for a more flexible and scalable configuration management approach.
- Community Roles: When we integrated community roles from Ansible Galaxy, we simplified the installation and configuration of services like MySQL, Nginx, and Apache, thus reducing the time and effort required for setup.
- Load Balancer Configuration: We then explored how to configure both Nginx and Apache as load balancers, which assisted us to switch between them based on our environment's needs, thus ensuring high availability and efficient traffic management.
- Best Practices: When carrying out this kind of project it is important to note that pre-defining key variables, testing dynamically loaded tasks, and maintaining backups of configuration files are non-negotiables to prevent potential issues during deployment.



