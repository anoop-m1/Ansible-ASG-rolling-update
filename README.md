# Ansible-ASG-ELB-rolling-update

---
## Pre-Requests
- Install Ansible on your Master Machine
- Create an IAM user role under your AWS account and please enter the values once the playbook running time
##### Installation
[Ansible2](https://docs.ansible.com/ansible/2.3/index.html) (For your reference visit [How to install Ansible](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html))
##### IAM Role Creation
[IAM Role Creation](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_create.html)
##### Ansible Modules used
- [yum](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/yum_module.html) 
- [pip](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/pip_module.html)
- [ec2-key](https://docs.ansible.com/ansible/latest/collections/amazon/aws/ec2_key_module.html)
- [copy](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/copy_module.html)
- [ec2-group](https://docs.ansible.com/ansible/latest/collections/amazon/aws/ec2_group_module.html)
- [debug](https://www.google.com/search?q=debug+%2B+ansible&rlz=1C1ONGR_enIN928IN928&oq=debug+%2B+ansible&aqs=chrome..69i57.5092j0j4&sourceid=chrome&ie=UTF-8)
- [ec2_lc](https://docs.ansible.com/ansible/latest/collections/community/aws/ec2_lc_module.html)
- [ec2_elb_lb](https://docs.ansible.com/ansible/latest/collections/amazon/aws/ec2_elb_lb_module.html)
- [ec2_asg](https://docs.ansible.com/ansible/latest/collections/community/aws/ec2_asg_module.html)
- [ec2_instance_info](https://docs.ansible.com/ansible/latest/collections/community/aws/ec2_instance_info_module.html)
- [add_host](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/add_host_module.html)
- [git](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/git_module.html)
- [file](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/file_module.html)
- [pause](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/pause_module.html)
---

# CODE FLOW:

######    - name: "KeyPair Creation"
######    - name: "KeyPair_saving"
######    - name: "Creating Secutiry Group"
######    - name: "Creating Launch configuration"
######    - name: "Creating ELB"
######    - name: "Autoscaling-Group Creation"
######    - name: "Fetching Ec2 Instance Details"
######    - name: "Creating Dymaic Inventory"
######    - name: "Website From GitHub"
######    - name: "Installing Packages"
######    - name: "Cloning Repository from Git"
######    - name: "Disbaling ELB Health Check"
######    - name: "Off loading Ec2-Instance From ELB"
######    - name: "Copying New Content To documentRoot"
######    - name: "Restarting/enabling Application"
######    - name: "Enabling ELB Health Check"
######    - name: "Loading Ec2-Instance to ELB"


# ansible-playbook main.yml
```
---
---
- name: "Creating Aws Infrastructure"
  hosts: localhost
  connection: local
  vars_files:
     - new.vars
     - cred.vars

  tasks:

    - name: "KeyPair Creation"
      ec2_key:
        aws_access_key: "{{ access_key }}"
        aws_secret_key: "{{ secret_key }}"
        region: "{{ region }}"
        name: "{{ key_name }}"
        state: present
      register: keypair_data

    - name: "KeyPair_saving"
      when: keypair_data.changed == true
      copy:
        content: "{{ keypair_data.key.private_key }}"
        dest: "{{ key_name }}.pem"
        mode: 0400


    - name: "Creating Secutiry Group"
      ec2_group:
        aws_access_key: "{{ access_key }}"
        aws_secret_key: "{{ secret_key }}"
        name: "{{ sg_name }}"
        description: "SG created by ansible"
        region: "{{ region }}"
        rules:
          - proto: tcp
            from_port: 80
            to_port: 80
            cidr_ip: 0.0.0.0/0


          - proto: tcp
            from_port: 80
            to_port: 80
            cidr_ip: 0.0.0.0/0


          - proto: tcp
            from_port: 22
            to_port: 22
            cidr_ip: 0.0.0.0/0

      register: security_group
    - debug:
        var: "security_group.group_id"

    - name: "Creating Launch configuration"
      ec2_lc:
        aws_access_key: "{{ access_key }}"
        aws_secret_key: "{{ secret_key }}"
        region: "{{ region }}"
        name: "newlaunch_lc"
        image_id: ami-0233c2d874b811deb
        key_name: "{{ key_name }}"
        security_groups: "{{ security_group.group_id }}"
        instance_type: t2.micro
        user_data_path: userdata.sh
        state: present
      register: launch_status
    - debug:
        var: "launch_status"


    - name: "Creating ELB"
      ec2_elb_lb:
        name: "elb"
        aws_access_key: "{{ access_key }}"
        aws_secret_key: "{{ secret_key }}"
        region: "{{ region }}"
        connection_draining_timeout: 60
        cross_az_load_balancing: true
        security_group_ids: "{{ security_group.group_id }}"
        state: present
        zones:
          - "{{ region }}a"
          - "{{ region }}b"
          - "{{ region }}c"
        listeners:
          - protocol: http
            load_balancer_port: 80
            instance_port: 80
        health_check:
            ping_protocol: http
            ping_port: 80
            ping_path: "/health.html"
            response_timeout: 2
            interval: 10
            unhealthy_threshold: 2
            healthy_threshold: 2
      register: elb_status

    - debug:
        var: "elb_status.elb.dns_name"


    - name: "Autoscaling-Group Creation"
      ec2_asg:
        aws_access_key: "{{ access_key }}"
        aws_secret_key: "{{ secret_key }}"
        region: "{{ region }}"
        name: "devops_asg"
        load_balancers: "elb"
        launch_config_name: "newlaunch_lc"
        health_check_period: 30
        health_check_type: EC2
        replace_all_instances: no
        min_size: 2
        max_size: 2
        desired_capacity: 2
        state: present
        tags:
          - Name: devops_asg_servers
            propagate_at_launch: true
      register: autoscaling
    - debug:
        var: "autoscaling"


    - name: "Fetching Ec2 Instance Details"
      ec2_instance_info:
        aws_access_key: "{{ access_key }}"
        aws_secret_key: "{{ secret_key }}"
        region: "{{ region }}"
        filters:
          "tag:aws:autoscaling:groupName": "devops_asg"
          instance-state-name: [ "running"]
      register: ec2

    - name: "Creating Dymaic Inventory"
      add_host:
        name: "{{ item.public_ip_address }}"
        groups: "instances"
        ansible_host: "{{ item.public_ip_address }}"
        ansible_port: 22
        ansible_user: "ec2-user"
        ansible_ssh_private_key_file: "{{ key_name }}"
        ansible_ssh_common_args: "-o StrictHostKeyChecking=no"
      with_items:
        - "{{ ec2.instances }}"
- name: "ASG Rolling Update Start"
  hosts: instances
  become: true
  serial: 1
  vars_files:
    - new.vars
  tasks:

    - name: "Cloning Repository from Git"
      git:
        repo: "{{ git_url }}"
        dest: "{{ clone_dir }}"
      register: repo_status

    - name: "Disbaling ELB Health Check"
      when: repo_status.changed == true
      file:
        path: /var/www/html/health.html
        state: touch
        mode: 0000

    - name: "Off loading Ec2-Instance From ELB"
      when: repo_status.changed == true
      pause:
        seconds: "{{ health_time }}"


    - name: "Copying New Content To documentRoot"
      when: repo_status.changed == true
      copy:
        src: "{{ clone_dir }}"
        dest: /var/www/html/
        remote_src: true


    - name: "Restarting/enabling Application"
      when: repo_status.changed == true
      service:
        name: httpd
        state: restarted
        enabled: true

    - name: "Enabling ELB Health Check"
      when: repo_status.changed == true
      file:
        path: "/var/www/html/{{ health_page }}"
        state: touch
        mode: 0644

    - name: "Loading Ec2-Instance to ELB"
      when: repo_status.changed == true
      pause:
        seconds: "{{ health_time }}"


---


```
- credentials.vars
```sh
access_key: "<your-access-key>"     <------------------ Enter your IAM Access Key
secret_key: "<your-secret-key>"     <------------------ Enter your IAM Secret Key
```

- new.vars
```sh
region: "us-east-2"
key_name: "<<key-name>>"
sg_name: "<<security-group"
git_url: ""
clone_dir: "/var/website/"
health_time: 15
health_page: "health.html"
```
---
