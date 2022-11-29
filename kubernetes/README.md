# Kubernetes [kubeadm] using Ansible
Step 1

Create 4 instances with Ubuntu 18.04 OS.

```text-plain
sudo hostnamectl set-hostname ansible
```

```text-plain
sudo hostnamectl set-hostname master
```

```text-plain
sudo hostnamectl set-hostname worker1
```

```text-plain
sudo hostnamectl set-hostname worker2
```

Step 2 (On ansible host as Ubuntu user)

```text-plain
sudo apt update
```

Step 3

```text-plain
sudo apt install software-properties-common -y
sudo apt install ansible -y
```

Step 4

```text-plain
ssh-keygen
```

Step 5 (or GitHub clone)

```text-plain
mkdir ~/kube-cluster
mkdir ~/kube-cluster/secrets
cd ~/kube-cluster
```

```text-plain
nano inventory-ubuntu
```

inventory-ubuntu

```text-plain
[masters]
 master ansible_host=[private_ip] ansible_user=ubuntu
 
[workers]
 worker1 ansible_host=[private_ip] ansible_user=ubuntu
 worker2 ansible_host=[private_ip] ansible_user=ubuntu
 
[all:vars]
 ansible_python_interpreter=/usr/bin/python3
 ansible_ssh_private_key_file=secrets/key.pem
```

```text-plain
nano secrets/key.pem
```

Copy the content in key.pem from local.

```text-plain
chmod 400 secrets/key.pem
```

```text-plain
ansible all -i inventory-ubuntu -m ping
```

Step 6

```text-plain
nano initial.yml
```

initial.yml

```text-plain
---
- hosts: allansible all -i inventory-ubuntu -m ping

  become: yes
  tasks:
    - name: create the 'kube' user
      user: name=kube append=yes state=present createhome=yes shell=/bin/bash
 
    - name: allow 'kube' to have passwordless sudo
      lineinfile:
        dest: /etc/sudoers
        line: 'kube ALL=(ALL) NOPASSWD: ALL'
        validate: 'visudo -cf %s'
  
    - name: set up authorized keys for the kube user
      authorized_key: user=kube key="{{item}}"
      with_file:
        - ~/.ssh/id_rsa.pub
```

```text-plain
ansible-playbook -i inventory-ubuntu initial.yml
```

Step 7

```text-plain
nano inventory-kube
```

inventory-kube

```text-plain
[masters]
 master ansible_host=[private_ip] ansible_user=kube
 
[workers]
 worker1 ansible_host=[private_ip] ansible_user=kube
 worker2 ansible_host=[private_ip] ansible_user=kube
 
[all:vars]
 ansible_python_interpreter=/usr/bin/python3
```

```text-plain
nano ~/kube-cluster/kube-dependencies.yml
```

kube-dependencies.yml

```text-plain
---
- hosts: all
  become: yes
  tasks:
  - name: install Docker
    apt:
      name: docker.io
      state: present
      update_cache: true

  - name: install APT Transport HTTPS
    apt:
      name: apt-transport-https
      state: present
  
  - name: add Kubernetes apt-key
    apt_key:
      url: https://packages.cloud.google.com/apt/doc/apt-key.gpg
      state: present
  
  - name: add Kubernetes APT repository
    apt_repository:
      repo: deb http://apt.kubernetes.io/ kubernetes-xenial main
      state: present
      filename: 'kubernetes'
    
  - name: install kubelet
    apt:
      name: kubelet
      state: present
      update_cache: true
  
  - name: install kubeadm
    apt:
      name: kubeadm
      state: present
  
- hosts: master
  become: yes
  tasks:
  - name: install kubectl
    apt:
      name: kubectl
      state: present
```

```text-plain
ansible-playbook -i inventory-kube kube-dependencies.yml
```

Step 8

```text-plain
nano ~/kube-cluster/master.yml
```

master.yml

```text-plain
---
- hosts: master
  tasks:
    - name: remove swap
      become: yes
      shell: "swapoff -a"

    - name: initialize the cluster
      become: yes
      shell: kubeadm init --pod-network-cidr=10.244.0.0/16 --ignore-preflight-errors=all

    - name: create .kube directory
      become: yes
      become_user: kube
      file:
        path: $HOME/.kube
        state: directory
        mode: 0755
  
    - name: copy admin.conf to user's kube config
      become: yes
      copy:
        src: /etc/kubernetes/admin.conf
        dest: /home/kube/.kube/config
        remote_src: yes
        owner: kube

    - name: install Pod network
      become: yes
      become_user: kube
      shell: kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```

```text-plain
ansible-playbook -i inventory-kube master.yml
```

Step 9

```text-plain
nano ~/kube-cluster/workers.yml
```

workers.yml

```text-plain
---
- hosts: master
  become: yes
  gather_facts: false
  tasks:
  
  - name: get join command
    shell: kubeadm token create --print-join-command
    register: join_command_raw
 
  - name: set join command
    set_fact:
      join_command: "{{ join_command_raw.stdout_lines[0] }}"
  
- hosts: workers
  become: yes
  tasks:
  - name: remove swap
    shell: "swapoff -a"

  - name: join cluster	
    shell: "{{ hostvars['master'].join_command }} >> node_joined.txt"
    args:
      chdir: $HOME
      creates: node_joined.txt
```

```text-plain
ansible-playbook -i inventory-kube workers.yml
```

Step 10

```text-plain
ansible master -i inventory-kube -a "kubectl get nodes"
```