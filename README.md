# Project:

## Deploy a Kubernetes cluster on AWS, deploy a Docker image with three replicas to the Kubernetes cluster with the REST API of the docker image exposed on a NodePort of 31000, and create a PVC bound to a PV with 5 GB storage using Ansible.

## Prerequisites for your local computer
- Ansible
- AWS CLI
- Boto3 (Boto is an AWS SDK for Python that can be used to create, configure, and manage AWS services.)


[Provisioning](#prov)

[Configuration](#config)

[Deployment of Docker image and creation of Persistent Volume and Persistent Volume Claim](#deploy)


## Introduction

Kubernetes clusters are used to run containerized applications. A kubernetes cluster is a set of hosts for running a container. These hosts are called nodes. For this project, I provisioned and configured one master node and two worker nodes all running centOS8.

<a name="prov">

## Provisioning 
The AWS instances were provisioned using Ansible. Follow these steps to reproduce:

### Preliminaries
1. Create a new directory and name it Project. This is where all files will be located.

2. Log in to your AWS account and create an IAM role with AWS credential type Access key - Programmatic access. This ensures that the IAM role is assigned an access key and secret access key which is used for AWS CLI configuration. Set the permissions boundary to AmazonEC2FullAccess.


3. Navigate to Key pairs and create a new RSA Key pair that would be used to create and ssh into instances from the AWS CLI. Download the .pem file and put it in Projects directory. The Key pair I created is called ansible.pem.


4. Open a terminal on your local computer and type `aws configure`. Supply the access key ID and secret access key gotten from step 2. Also provide your preferrred AWS region. I used **us-east-1** for this project.


5. Put the access key ID and secret access key in a file called credentials.yml.


6. Run `chmod 400 ansible.pem` to give the key pair the right file permission.


7. Create a **hosts** file and an **ansible.cfg** file (declare the location of the hosts file along with other configurations here).


### Role Creation
After you are done with the preliminaries above, you can now begin creating the Ansible roles. Roles contain configuration files, templates, and variables. They are easy to manage and provide clean directory structure. I created three roles for the Kubernetes cluster - **ec2**, **master_node**, and **worker_node**.

8. Run the following commands to create the roles -

`ansible-galaxy init roles/ec2` 

`ansible-galaxy init roles/master_node` 

`ansible-galaxy init roles/worker_node` 


9. Change into the **tasks** directory in the ec2 role and open the **main.yml** playbook.


10. Write tasks to create a VPC, a public subnet, a private subnet, a route table for the public subnet, a security group, and an IGW (Internet Gateway). 
 
 
 ```
 - name: Create VPC for ec2 instances
  ec2_vpc_net:
    aws_access_key: "{{ access_key }}"
    aws_secret_key: "{{ secret_key }}"
    name: ansibleVPC
    state: present
    cidr_block: "{{ vpcCidrBlock }}"
    region: "{{ region_name }}"
  register: ansibleVPC
      
- name: Create internet gateway for ansibleVPC
  ec2_vpc_igw:
    aws_access_key: "{{ access_key }}"
    aws_secret_key: "{{ secret_key }}"
    state: present
    region: "{{ region_name }}"
    vpc_id: "{{ ansibleVPC.vpc.id }}"
    tags:
      Name: ansibleVPC_IGW     
  register: ansibleVPC_igw

- name: Get all Availability Zones present in {{ region_name }}
  aws_az_facts:
    aws_access_key: "{{ access_key }}"
    aws_secret_key: "{{ secret_key }}"
    region: "{{ region_name }}"
  register: az_in_region
  
- name: Show all Availability Zones present in {{ region_name }}
  debug: 
    var: az_in_region

- name: Show the zones that will be used for the public and private subnets
  debug:
    msg:
      - "public subnet: {{ az_in_region.availability_zones[0].zone_name }}"
      - "private subnet: {{ az_in_region.availability_zones[1].zone_name }}"

- name: Create public subnet
  ec2_vpc_subnet:
    aws_access_key: "{{ access_key }}"
    aws_secret_key: "{{ secret_key }}"
    state: present
    cidr: "{{ subNetCidrBlock }}"
    az: "{{ az_in_region.availability_zones[0].zone_name }}"
    vpc_id: "{{ ansibleVPC.vpc.id }}"
    region: "{{ region_name }}"
    map_public: yes
    tags:
      Name: public subnet
  register: public_subnet

- name: Show public subnet details
  debug: 
    var: public_subnet

- name: Create private subnet
  ec2_vpc_subnet:
    aws_access_key: "{{ access_key }}"
    aws_secret_key: "{{ secret_key }}"
    state: present
    cidr: "{{ subNetCidrBlockPriv }}"
    az: "{{ az_in_region.availability_zones[1].zone_name }}"
    vpc_id: "{{ ansibleVPC.vpc.id }}"
    region: "{{ region_name }}"
    resource_tags:
      Name: private subnet
  register: private_subnet

- name: Show private subnet details
  debug: 
    var: private_subnet

- name: Create route table for public subnet
  ec2_vpc_route_table:
    aws_access_key: "{{ access_key }}"
    aws_secret_key: "{{ secret_key }}"
    state: present
    region: "{{ region_name }}"
    vpc_id: "{{ ansibleVPC.vpc.id }}"
    tags:
      Name: rt_ansibleVPC_PublicSubnet
    subnets:
      - "{{ public_subnet.subnet.id }}"
    routes:
      - dest: "{{ destinationCidrBlock }}"
        gateway_id: "{{ ansibleVPC_igw.gateway_id }}"
  register: rt_ansibleVPC_PublicSubnet
    
- name: display public route table
  debug: var=rt_ansibleVPC_PublicSubnet

- name: Create a security group
  ec2_group:
    aws_access_key: "{{ access_key }}"
    aws_secret_key: "{{ secret_key }}"
    state: present
    name: sg_ansibleVPC_publicsubnet_host
    description: security group for hosts within the public subnet of ansible VPC
    vpc_id: "{{ ansibleVPC.vpc.id }}"
    region: "{{ region_name }}"
    rules:
      - proto: all
        cidr_ip: 0.0.0.0/0
        rule_desc: allow all ports
    rules_egress:
      - proto: all
        cidr_ip: 0.0.0.0/0
  register: sg_ansibleVPC_publicsubnet_host

- name: Show details for host security group
  debug: 
    var: sg_ansibleVPC_publicsubnet_host 
  ```

11. Write tasks to create three instances using a CentOS8 AMI ID located in a region of your choosing. 

```
- name: Deploy ec2 instances
  ec2:
    aws_access_key: "{{ access_key }}"
    aws_secret_key: "{{ secret_key }}"
    key_name: "{{ keypair }}"
    wait: true
    image: "{{ ami_id }}"
    group_id: "{{ sg_ansibleVPC_publicsubnet_host.group_id }}"
    vpc_subnet_id: "{{ public_subnet.subnet.id }}"
    assign_public_ip: yes
    region: "{{ region_name }}"
    instance_type: "{{ instance_type }}"
    count: 1
    state: present
    instance_tags:
      Name: "{{ item }}"
  register: ec2
  loop: "{{ instance_tag }}"

- name: display details for servers
  debug: 
    var: ec2

- name: Add instance 1 to a host group called master_node
  add_host:
    hostname: "{{ ec2.results[0].instances[0].public_ip }}"
    groupname: master_node

- name: Add instance 2 to a host group called worker_node
  add_host:
    hostname: "{{ ec2.results[1].instances[0].public_ip }}"
    groupname: worker_node

- name: Add instance 3 to the worker_node host group 
  add_host:
    hostname: "{{ ec2.results[2].instances[0].public_ip }}"
    groupname: worker_node

- name: Wait for SSH to come up
  wait_for:
    host: "{{ ec2.results[2].instances[0].public_dns_name }}"
    port: 22
    state: started
 ```


12. All variables should be located in the **main.yml** playbook located in the **vars** directory for each role.


13. Create an ec2.yml playbook in the project directory to run the ec2 role. Set the host to localhost.

```
- hosts: localhost
  connection: local
  gather_facts: no
  tasks:
    - name: Running EC2 Role
      include_role:
        name: ec2
```


14. Run `ansible-playbook ec2.yml` to spin up the instances.
</a>
  
<a name="config">

## Configuraton
Now that the ec2 instances have been provisioined, you can configure them to create the Kubernetes cluster. I also used Ansible for the configuration. 

15. You should see the running instances when you log in to your AWS account. Copy the public IP addresses of the three nodes and put them in the their respective groups in the **hosts** file.

16. Change into the **tasks** directory in the master_node role and open the **main.yml** playbook. Write tasks to install the necessary packages and initialize a Kubernetes cluster here.

```
---
# defaults file for master_node


- name: Disabling swap
  shell: swapoff -a

- name: Commenting Swap entries in /etc/fstab
  replace:
    path: /etc/fstab
    regexp: '(.*swap*)'
    replace: '#\1'

- name: Configuring yum repo for k8s master node
  yum_repository:
    name: kubernetes
    description: Kubernetes repo
    baseurl: https://packages.cloud.google.com/yum/repos/kubernetes-el7-$basearch
    enabled: 1
    gpgcheck: 1
    repo_gpgcheck: yes
    gpgkey: https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg

-  name: Install iproute-tc
   yum:
    name: "{{ pkg }}"
    state: present

- name: Installing firewalld
  package:
    name:
      - firewalld
    state: present

-  name: Giving --no-best option
   replace:
    path: "/etc/dnf/dnf.conf"
    regexp: "True"
    after: "best="
    replace: "False"

- name: Get dnf
  command: dnf config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo

- name: Install Docker
  command: dnf install docker-ce --nobest -y

- name: Start Docker
  service:
    name: docker
    state: started
    enabled: true

- name: Install kubeadm, kubelet, and kubectl
  yum: 
    name: "{{ k8s_repo_pkg }}"
    state: present
  register: kubeadm

- name: Start Kubelet
  service:
    name: kubelet
    state: started
    enabled: true

- name: Pulling images required for setting up the kubernetes cluster
  command: kubeadm config images pull

- name: Updating docker cgroup on master Node
  copy:
    dest: /etc/docker/daemon.json
    src: daemon.json

- name: Restart Docker service
  service: 
    name: "{{ docker_svc }}"
    state: restarted

- name: Starting firewalld service
  service:
    name: firewalld
    state: started
    enabled: true

- name: Allow Network ports in Firewalld
  firewalld:
    port: "{{ item }}"
    state: enabled
    permanent: yes
    immediate: yes
  with_items: "{{ ports }}"

- name: Initializing cluster
  become: yes    
  command: kubeadm init --pod-network-cidr=10.244.0.0/16 --ignore-preflight-errors=NumCPU --ignore-preflight-errors=Mem

- name: Setting up kubectl (Creating the directory, setting up configuration files, and Changing the owner of .kube/config)
  shell:
    cmd: |
      mkdir -p $HOME/.kube
      sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
      sudo chown $(id -u):$(id -g) $HOME/.kube/config

- name: "Downloading FLannel"
  get_url:
      url: "{{ flannel_url }}"
      dest: "/k8s/"
  ignore_errors: yes

- name: "Installing Flannel"
  command: "kubectl apply -f kube-flannel.yml"
  args:
      chdir: /k8s/
  ignore_errors: yes

- name: Generate join token
  command: kubeadm token create  --print-join-command
  register: output
  
 ```

17. Change into the **tasks** directory in the worker_node role and open the **main.yml** playbook. Write tasks to configure the worker nodes and join them to the master node here.

```
---
# defaults file for worker_node

- name: Disabling swap 
  shell: swapoff -a

- name: Commenting Swap entries in /etc/fstab
  replace:
    path: /etc/fstab
    regexp: '(.*swap*)'
    replace: '#\1'

- name: Configuring yum repo for k8s master node
  yum_repository:
    name: kubernetes
    description: Kubernetes repo
    baseurl: https://packages.cloud.google.com/yum/repos/kubernetes-el7-$basearch
    enabled: 1
    gpgcheck: 1
    repo_gpgcheck: yes
    gpgkey: https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg

- name: Install iproute-tc
  yum:
    name: "{{ pkg }}"
    state: present

- name: Installing firewalld
  package:
    name:
      - firewalld
    state: present

- name: Giving --no-best option
  replace:
    path: "/etc/dnf/dnf.conf"
    regexp: "True"
    after: "best="
    replace: "False"

- name: Get dnf
  command: dnf config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo

- name: Install docker
  command: dnf install docker-ce --nobest -y

- name: Start Docker
  service:
    name: docker
    state: started
    enabled: true

- name: Install kubeadm, kubelet, and kubectl
  yum: 
    name: "{{ k8s_repo_pkg }}"
    state: present
  register: kubeadm

- name: Start Kubelet
  service:
    name: kubelet
    state: started
    enabled: true

- name: Updating docker cgroup on worker node
  copy:
    dest: /etc/docker/daemon.json
    src: daemon.json

- name: Restart Docker service
  service: 
    name: "{{ docker_svc }}"
    state: restarted

- name: Starting firewalld service
  service:
    name: firewalld
    state: started
    enabled: true

- name: Allow Network ports in Firewalld
  firewalld:
    port: "{{ item }}"
    state: enabled
    permanent: yes
    immediate: yes
  with_items: "{{ ports }}"

- name: Joining slaves with the master
  command: "{{ hostvars[groups['master_node'][0]]['output']['stdout'] }}"

- name: Cleaning Caches on RAM
  shell: echo 3 > /proc/sys/vm/drop_caches
  
 ```

18. Create a cluster.yml playbook in the project directory to run the master and worker roles. Set the host for the master_node role to master_node. Do the same for worker_node. 

```
- hosts: master_node
  become: true
  user: centos
  gather_facts: no
  tasks:
    - name: Running K8s Master Role
      include_role:
        name: master_node

- hosts: worker_node
  become: true
  user: centos
  gather_facts: no
  tasks:
    - name: Running K8s Worker Role
      include_role:
        name: worker_node

- hosts: worker_node
  become: true
  user: centos
  gather_facts: no
  tasks:
    - name: Running K8s Worker Role
      include_role:
        name: worker_node
```

19. Run `ansible-playbook ec2.yml` to create the Kubernetes cluster.


20. SSH into the master node using `ssh -i ansible.pem centos@<ip address of master node>`. Run `kubectl get nodes` on the master node to confirm that the cluster has been created. 
</a>
  
<a name="deploy">

## Deployment of Docker image and creation of Persistent Volume and Persistent Volume Claim
Kubernetes does not offer data persistence out of the box. This means that whenever a pod is restarted, whatever data was stored or updated would be gone. This is you need a storage that doesn't depend on the pod lifecycle. When you create a persistent volume, it remains after one pod dies and another gets created. The new pod can easily pick up where the previous one left off by reading the existing data from the storage to get up-to-date data.

I created a YAML file that:
- deployed a REST API of a Docker image
- exposed the REST API of the docker image on a NodePort of 31000 using a Kubernetes service
- created a PVC (my-test-pvc) and bind it to a PV (my-pv) with host path /data/kubernetes/persistent-volume/my-pv with 5 GB storage

To reproduce:

21. On your master node, create a Kubernetes YAML file called **k8s_deployments.yaml**. This file will contain all the deployment configuration

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-world-rest
  labels:
    app: hello-world-rest
spec:
  replicas: 3
  selector:
    matchLabels:
      app: hello-world-rest
  template:
    metadata:
      labels:
        app: hello-world-rest
    spec:
      containers:
      - name: hello-world-rest
        image: vad1mo/hello-world-rest        
        ports:
        - containerPort: 5050
      volumes:
            - name: data
              persistentVolumeClaim:
                claimName: my-test-pvc 
       
---
#service to expose the above deployment

apiVersion: v1
kind: Service
metadata:
  name: hello-world-rest
spec:
  ports:
    - targetPort: 5050
      nodePort: 31000 
      port: 5050          
  type:  NodePort
  selector:    
      app: hello-world-rest

---
#persistent volume for the deployment above

  apiVersion: v1
  kind: PersistentVolume
  metadata:
    name: my-pv 
    labels:
      type: local
  spec:
    storageClassName: manual 
    capacity:
      storage: 5Gi
    accessModes:
      - ReadWriteMany 
    persistentVolumeReclaimPolicy: Retain
    hostPath:
      path: "/data/kubernetes/persistent-volume/my-pv" 

---  
#persistent volume claim

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-test-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 5Gi
  storageClassName: manual
  
```

22. Run `kubectl create -f k8s_deployments.yaml` to deploy.

</a>
