# Ansible
[How To Install and Configure Ansible on Ubuntu 20.04](https://www.digitalocean.com/community/tutorials/how-to-install-and-configure-ansible-on-ubuntu-20-04)

Prerequisites

Installing Ansible on Control Node

```text-plain
sudo add-apt-repository --yes --update ppa:ansible/ansible
sudo apt update
sudo apt install ansible
```

On Ansible Control Node

```text-plain
cd ~
mkdir ansible
```

```text-plain
cd ansible
nano inventory
```

inventory

```text-plain
[production_hosts]
prod-host1 ansible_host=172.31.12.63

[development_hosts]
dev-host1 ansible_host=172.31.15.40

[production_hosts:vars]
ansible_user=priv

[development_hosts:vars]
ansible_user=priv
```

Listing the configurations

```text-plain
ansible-inventory -i inventory --list -y
```

Testing connection

```text-plain
ansible all -i inventory -m ping -u priv
```