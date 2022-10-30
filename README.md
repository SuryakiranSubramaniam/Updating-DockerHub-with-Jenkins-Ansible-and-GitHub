# Updating-DockerHub-with-Jenkins-Ansible-and-GitHub

## Description:

This is a DevOps jenkins freestyle project using Git, Jenkins, Ansible and Docker on AWS for deploying a python-flask application in a Docker container.The process should be initiated from a new commit to a specific branch of a GitHub repository. This event kicks off a process that begins building the Docker image. Jenkins supports this event-driven flow using the “GitHub hook trigger for GITScm polling".

Updating-DockerHub-with-Jenkins-Ansible-and-GitHub

![alt text](https://github.com/SuryakiranSubramaniam/Updating-DockerHub-with-Jenkins-Ansible-and-GitHub/blob/main/image/junkins2.png)

## Setup:
- jenkins/ansible Server
- Docker Image Build Server
- Production/Test Server
Here we use amazon linux on these 3 servers.

## Jenkins Server

> One server is with Jenkins and other two servers are test and build

## Jenkins Installation and configuration.

````
yum update –y
wget -O /etc/yum.repos.d/jenkins.repo     https://pkg.jenkins.io/redhat-stable/jenkins.repo
rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io.key
yum upgrade
amazon-linux-extras install java-openjdk11 -y
yum install jenkins -y
systemctl enable jenkins
systemctl start jenkins
````

#### After the instalation, we can test the jenkins using the below port:

````
http://<Jenkins-Ansible-server-IP>:8080/
````
## Setuping the playbook on ansible master server:
````

# cat hosts 
[build]
172.31.46.22  ansible_user="ec2-user"

[test]
172.31.40.36  ansible_user="ec2-user"
````

````
# cat hosts 
[build]
172.31.46.22  ansible_user="ec2-user"

[test]
172.31.40.36  ansible_user="ec2-user"
[root@ip-172-31-38-152 deployment]# ansible-vault decrypt main2.yml 
Vault password: 
Decryption successful
[root@ip-172-31-38-152 deployment]# cat hosts 
[build]
172.31.46.22  ansible_user="ec2-user"

[test]
172.31.40.36  ansible_user="ec2-user"
[root@ip-172-31-38-152 deployment]# cat main2.yml 
---
- name: "Building Docker Image From Grihub Repository"
  hosts: build
  become: true
  vars:
    packages:
      - git
      - pip
      - docker
    project_repo_url: "https://github.com/SuryakiranSubramaniam/flaskapp.git"
    clone_dir: "/var/flask_app/"
    docker_user: "suryakiransubramaniam"
    docker_password: "xxxx"
    image_name: "suryakiransubramaniam/flaskapp"
        
  tasks:
                  
    - name: "Build - Installing packages"
      yum:
        name: "{{ packages }}"
        state: present
    
    - name: "Build - Adding Ec2-user to docker group"
      user:
        name: "ec2-user"
        groups:
          - docker
        append: true
            
            
    - name: "Build - Installing Python Extension For Docker"
      pip:
        name: docker-py
            
    - name: "Build - Restarting/Enabling Docker Service"
      service:
        name: docker
        state: restarted
        enabled: true
            
    - name: "Build - Clonning Repo {{ project_repo_url }}"
      git:
        repo: "{{project_repo_url}}"
        dest: "{{ clone_dir }}"
      register: clone_status
        
        
    - name: "Build - Loging to Docker-hub Account"
      when: clone_status.changed == true
      docker_login:
        username: "{{ docker_user }}"
        password: "{{ docker_password }}"
        state: present
    
    
    - name: "Build - Creating Docker Image And Push To Docker-hub"
      when: clone_status.changed == true
      docker_image:
        source: build
        build:
          path: "{{ clone_dir }}"
          pull: yes
        name: "{{ image_name }}"
        tag: "{{ item }}"
        push: true
        force_tag: yes
        force_source: yes
      with_items:
        - "{{ clone_status.after }}"
        - latest  
      
    
    - name: "Build - Deleting Local Image From Build Server"
      when: clone_status.changed == true
      docker_image:
        state: absent
        name: "{{ image_name }}"
        tag: "{{ item }}"
      with_items:
        - "{{ clone_status.after }}"
        - latest
    
        
    - name: "Build - Logout to Docker-hub Account"
      when: clone_status.changed == true
      docker_login:
        username: "{{ docker_user }}"
        password: "{{ docker_password }}"
        state: absent
       
    

- name: "Running Image On The Test Server"
  hosts: test
  become: true
  vars:
    image_name: "suryakiransubramaniam/flaskapp"
    packages:
      - docker
      - pip
  tasks:
    

    - name: "Test - Installing Packages"
      yum:
        name: "{{ packages }}"
        state: present
    
    - name: "Test - Adding Ec2-user to docker group"
      user:
        name: "ec2-user"
        groups:
          - docker
        append: true
            
            
    - name: "Test - Installing Python Extension For Docker"
      pip:
        name: docker-py
            
            
    - name: "Test - Docker service restart/enable"
      service:
        name: docker
        state: started
        enabled: true
    
    
    - name: "Test - Pulling Docker Image"
      docker_image:
        name: "suryakiransubramaniam/flaskapp"
        source: pull
        force_source: true
      register: image_status      
    
    - name: "Test- Run Container"
      when: image_status.changed == true     
      docker_container:
        name: flaskapp
        image: "{{ image_name }}:latest"
        recreate: yes
        pull: yes
        published_ports:
          - "80:5000"
````

````
#chown jenkins deployment
#chmod o=--- /var/deployment/
#ls -la
drwxr-x---  2 jenkins root   39 Oct 30 16:35 deployment
# ansible-vault encrypt /var/deployment/main.yml
New Vault password:
Confirm New Vault password:
Encryption successful

````
## Configuring Jenkins

> After the jenkins installation take the admin password from jenkins server using the below path:

````
ec2-user]# cat /var/lib/jenkins/secrets/initialAdminPassword
<pass>
````
> Unlock the jenkins and install the suggested plugins:




