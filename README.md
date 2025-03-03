## install Vrtualbox & Vagrant

#### Import GPG key for VirtualBox repository packages.

```
#Download
curl https://www.virtualbox.org/download/oracle_vbox_2016.asc | gpg --dearmor > oracle_vbox_2016.gpg
curl https://www.virtualbox.org/download/oracle_vbox.asc | gpg --dearmor > oracle_vbox.gpg

#Install on system
sudo install -o root -g root -m 644 oracle_vbox_2016.gpg /etc/apt/trusted.gpg.d/
sudo install -o root -g root -m 644 oracle_vbox.gpg /etc/apt/trusted.gpg.d/
```

```
wget -O - https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install vagrant
```

#### Next we add the actual packages repository into the system.
```
echo "deb [arch=amd64] http://download.virtualbox.org/virtualbox/debian $(lsb_release -sc) contrib" | sudo tee /etc/apt/sources.list.d/virtualbox.list
```

##### If you do not have VirtualBox already installed on your Ubuntu, run the below commands to install
```
sudo apt update
sudo apt install linux-headers-$(uname -r) dkms
sudo apt install virtualbox-7.0
```

```
VBoxManage --version
vagrant --version
```


## Vagrant

run Vagrantfile:
```
cd vagrant
vagrant up
```


### show vm's
```
vagrant global-status
```

```
62d572f  master  virtualbox running /home/mgr/Practice/vagrant
563dabd  worker1 virtualbox running /home/mgr/Practice/vagrant
```

### add vm's vagrant ssh to host ssh config:
```
vagrant ssh-config >> ~/.ssh/config
```

### restart ssh.service
```
sudo systemctl restart ssh.service
```
### change master ip
```
NEW_MASTER_IP=$(vagrant ssh master -c "ip -4 addr show | grep '192.168' | awk '{print \$2}' | cut -d'/' -f1" | tr -d '\r')
sed -i "s/^    MASTER_IP: \".*\"/    MASTER_IP: \"$NEW_IP\"/" ../ansible/site.yml
```

```
NEW_WORKER_IP=$(vagrant ssh worker -c "ip -4 addr show | grep '192.168' | awk '{print \$2}' | cut -d'/' -f1" | tr -d '\r')
sed -i "/block: |/,+2 s|.*master|          $NEW_MASTER_IP   master|" ../ansible/roles/k8s-installation/tasks/main.yml
sed -i "/block: |/,+2 s|.*worker|          $NEW_WORKER_IP   worker|" ../ansible/roles/k8s-installation/tasks/main.yml

```

## Ansible

### install `Ansible`

```
sudo apt update
sudo apt install software-properties-common
sudo add-apt-repository --yes --update ppa:ansible/ansible
sudo apt install ansible
ansible --version
```

### k8s installation with ansible:

**you should change your inventory:**
```
sudo vim /etc/ansible/hosts

[k8s-nodes]
master
worker

[masters]
master

[workers]
worker
```

### add vm's name to /etc/hosts
```
sudo vim /etc/hosts

192.168.56.12 master
192.168.56.13 worker
```

### run ansible playbook file:

#### install k8s all dependencies :
```
ansible-playbook -v -f 50 site.yml --tags k8s-installation
```

#### init master nodes and join workers to master node:

```
ansible-playbook -v -f 50 site.yml --tags k8s-init
```

#### join worker:

```
ansible-playbook -v -f 50 site.yml --tags k8s-join
```

#### run a simple web app on cluster and set ingress manifest:

```
cd app
kubectl apply -f deployment.yml -f svc.yml -f ing.yml -f traffic-generator.yaml -f hpa.yml
```

#### install prometheus & grafana for monitoring cluster:

```
ansible-playbook -v -f 50 site.yml --tags k8s-monitoring
```
### apply monitoring ingress
```
cd monitoring
kubectl apply -f grafana-ing.yml -f prom-ing.yml -f alert-ing.yml
```

#### you should check your cluster
```
kubectl cluster-info
```


## Test

### test URLs on local browser:
```
sample.ir
grafana.sample.ir
prom.sample.ir
alert.sample.ir
```

