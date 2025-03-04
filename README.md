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


### show VMs
```
vagrant global-status
```

```
62d572f  master  virtualbox running /home/mgr/Practice/vagrant
563dabd  worker1 virtualbox running /home/mgr/Practice/vagrant
```

### Modify the VM's Vagrant SSH settings in the host's SSH configuration.:
```
vagrant ssh-config >> ~/.ssh/config
```

### restart ssh.service
```
sudo systemctl restart ssh.service
```
### Modify the master and worker IPs in the Ansible configuration files
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

**You should modify the inventory file.:**
```
vim ansible/hosts

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

[IP] master
[IP] worker
```

### run ansible playbook files:

#### install k8s all dependencies :
```
ansible-playbook -i hosts -f 50 -v site.yml --tags k8s-installation
```

#### init master nodes and join workers to master node:

```
ansible-playbook -i hosts -v -f 50 site.yml --tags k8s-init
```

#### join worker:

```
ansible-playbook -i hosts -v -f 50 site.yml --tags k8s-join
```

#### Deploy a simple web app on the cluster and configure an Ingress manifest.:

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

## HPA test
go to traffic-generator container
```
kubectl exec -it traffic-generator -- /bin/sh
```
we use of wrk. it's a HTTP benchmarking tool
```
apk add wrk
```
for start testing we should use of hello-app-service so pay attention to below command
```
wrk -c 5 -t 5 -d 300s -H "Connection: Close" http://hello-app-service:8080
```
-c 5 => 5 Connections
-t 5 => 5 treads
-d 300s => duration
http://hello-app-service:8080 => our service

you should see

Running 5m test @ http://isc-service:80
  5 threads and 5 connections
You will see that the number of pods increases as resource consumption increases and after finish requests the number of pods will be decries.

## Test

### test URLs :
```
curl sample.ir:[ingress port]
curl grafana.sample.ir:[ingress port]
curl prom.sample.ir:[ingress port]
curl alert.sample.ir:[ingress port]
```

