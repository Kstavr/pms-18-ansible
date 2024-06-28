# pms-18-ansible

# Ansible Setup for Telegraf , InfluxDB , Grafana , Owncloud 

This project sets up Telegraf and InfluxDB using Ansible on a target server. The configuration ensures that Telegraf collects Docker metrics and sends them to InfluxDB.

## Prerequisites

- Ansible installed on the control node
- A target server 
- InfluxDB token and organization details

## Installation Steps

### 1. Install Ansible

#### On Ubuntu

```bash
sudo apt update
sudo apt install ansible -y

Create the following files in your project directory:
[defaults]
inventory = inventory
remote_user = vm2
private_key_file = /home/ansible/.ssh/id_rsa
host_key_checking = False

[targets]
target1 ansible_host=<host_ip> ansible_user=vm2 ansible_ssh_private_key_file=~/.ssh/id_rsa

Create a file named setup4.yml with the following content:
- name: Ensure Docker is installed and running
  hosts: targets
  become: yes
  become_user: root
  tasks:
    - name: Install Docker
      apt:
        name: docker.io
        state: present

    - name: Add user to docker group
      user:
        name: vm2
        groups: docker
        append: yes

    - name: Start Docker service
      service:
        name: docker
        state: started
        enabled: true

    - name: Gather system facts
      setup:
       gather_subset:
         - 'hardware'

    - name: Install Docker Compose
      get_url:
        url: "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-{{ ansible_system }}-{{ ansible_architecture }}"
        dest: /usr/local/bin/docker-compose
        mode: '0755'

    - name: Verify Docker Compose installation
      command: docker-compose --version

- name: Setup OwnCloud, Grafana, InfluxDB
  hosts: targets
  become: yes
  become_user: root
  tasks:
    - name: Create Docker Compose directory for OwnCloud
      file:
        path: /home/vm2/owncloud
        state: directory

    - name: Copy Docker Compose file for OwnCloud
      copy:
        dest: /home/vm2/owncloud/docker-compose.yml
        content: |
          version: '3.8'
          services:
            owncloud:
              image: owncloud/server
              ports:
                - "8080:8080"
              volumes:
                - owncloud_data:/mnt/data
              restart: always
              environment:
                - OWNCLOUD_DOMAIN=192.168.1.252:8080
                - OWNCLOUD_TRUSTED_DOMAINS=localhost,192.168.1.252
                - OWNCLOUD_OVERWRITE_CLI_URL=http://192.168.1.252:8080/
          volumes:
            owncloud_data:
      notify: Restart OwnCloud

    - name: Create Docker Compose directory for Grafana
      file:
        path: /home/vm2/grafana
        state: directory

    - name: Copy Docker Compose file for Grafana
      copy:
        dest: /home/vm2/grafana/docker-compose.yml
        content: |
          version: '3.8'
          services:
            grafana:
              image: grafana/grafana
              ports:
                - "3000:3000"
              restart: always
      notify: Restart Grafana

    - name: Create Docker Compose directory for InfluxDB
      file:
        path: /home/vm2/influxdb
        state: directory

    - name: Copy Docker Compose file for InfluxDB
      copy:
        dest: /home/vm2/influxdb/docker-compose.yml
        content: |
          version: '2.7'
          services:
            influxdb:
              image: influxdb:2.7
              ports:
                - "8086:8086"
              volumes:
                - influxdb_data:/var/lib/influxdb
              environment:
                - DOCKER_INFLUXDB_INIT_MODE=setup
                - DOCKER_INFLUXDB_INIT_USERNAME=admin
                - DOCKER_INFLUXDB_INIT_PASSWORD=adminpass
                - DOCKER_INFLUXDB_INIT_BUCKET=telegraf
                - DOCKER_INFLUXDB_INIT_ORG=myorg
                - DOCKER_INFLUXDB_INIT_ADMIN_TOKEN=mytoken
              restart: always
          volumes:
            influxdb_data:
      notify: Restart InfluxDB

- name: Install and Configure Telegraf natively
  hosts: targets
  become: yes
  become_user: root
  tasks:
    - name: Ensure curl is installed
      apt:
        name: curl
        state: present

    - name: Add InfluxData repository key
      apt_key:
        url: https://repos.influxdata.com/influxdb.key
        state: present

    - name: Add InfluxData repository
      lineinfile:
        path: /etc/apt/sources.list.d/influxdb.list
        line: "deb https://repos.influxdata.com/ubuntu jammy stable"
        create: yes

    #- name: Update apt cache
     # apt:
      #  update_cache: yes

    - name: Install Telegraf
      get_url:
        url: https://dl.influxdata.com/telegraf/releases/telegraf_1.21.4-1_amd64.deb
        dest: /tmp/telegraf_1.21.4-1_amd64.deb

    - name: Install Telegraf .deb package
      command: dpkg -i /tmp/telegraf_1.21.4-1_amd64.deb
      register: install_telegraf_result
      ignore_errors: yes

    - name: Debug install_telegraf_result
      debug:
        var: install_telegraf_result

    - name: Fix missing dependencies if any
      apt:
        update_cache: yes
        state: present
        name: -f
      when: install_telegraf_result is defined and install_telegraf_result.rc != 0


    - debug:
        msg: "install_telegraf_result: {{ install_telegraf_result }}"


    - name: Configure Telegraf
      copy:
        dest: /etc/telegraf/telegraf.conf
        content: |
          [global_tags]
          [agent]
            interval = "10s"
            round_interval = true
            metric_batch_size = 1000
            metric_buffer_limit = 10000
            collection_jitter = "0s"
            flush_interval = "10s"
            flush_jitter = "0s"
            precision = ""
            hostname = ""
            omit_hostname = false
          [[outputs.influxdb_v2]]
            urls = ["http://192.168.1.252:8086"]
            token = "rNzebb7YguBuwRX0qImURoac7sr_EUVx1k3NTAWqQj0EmTxnsJwwNBNRiWZZZeF6TsB1mP9I2OrBYUmJ90kuzg=="
            bucket = "telegraf"
            organization = "myorg"
          [[inputs.docker]]
            endpoint = "unix:///var/run/docker.sock"
            gather_services = false
            container_names = []
            timeout = "5s"
            perdevice_include = []
            total = true
            docker_label_include = []
            docker_label_exclude = []

    - name: Enable and start Telegraf service
      service:
        name: telegraf
        state: started
        enabled: true

    - name: Add telegraf user to docker group
      user:
        name: telegraf
        groups: docker
        append: yes

  handlers:
    - name: Restart OwnCloud
      command: docker-compose down && docker-compose up -d
      args:
        chdir: /home/vm2/owncloud

    - name: Restart Grafana
      command: docker-compose down && docker-compose up -d
      args:
        chdir: /home/vm2/grafana

    - name: Restart InfluxDB
      command: docker-compose down && docker-compose up -d
      args:
        chdir: /home/vm2/influxdb

Running the Playbook

ansible-playbook playbooks/setup4.yml -i inventory -K



Summary
In this project, we:

Installed Ansible on the control node.
Created necessary configuration files: ansible.cfg and inventory.
Created an Ansible playbook setup4.yml to install and configure Telegraf and InfluxDB.
Ensured the playbook is executed successfully and Telegraf is correctly sending metrics to InfluxDB.
This setup should provide a good starting point for monitoring Docker containers with Telegraf and InfluxDB.
