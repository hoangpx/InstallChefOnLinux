# Install Chef Server, Workstation, Client on Azure VM
Refer [https://www.digitalocean.com/community/tutorials/how-to-set-up-a-chef-12-configuration-management-system-on-ubuntu-14-04-servers]

## Server: 
server address on azure 
_10.44.62.11 chef/igs-chef!_


### Configuration  host name
```sudo nano /etc/hosts```

change to
```
127.0.1.1  10.44.62.11 igs-chef.daa.local
127.0.0.1 localhost
10.44.62.11 10.44.62.11 igs-chef.daa.local
```
check again

```hostname –f```

the output should be the ip _10.44.62.11_

#### This is script to install server automatically 

```
#!/bin/bash

# create staging directories
if [ ! -d /drop ]; then
  mkdir /drop
fi
if [ ! -d /downloads ]; then
  mkdir /downloads
fi

# download the Chef server package
if [ ! -f /downloads/chef-server-core-12.16.9-1.el7.x86_64.rpm ]; then
  echo "Downloading the Chef server package..."
  wget -nv -P /downloads https://packages.chef.io/files/stable/chef-server/12.16.9/el/7/chef-server-core-12.16.9-1.el7.x86_64.rpm
fi

# install Chef server
if [ ! $(which chef-server-ctl) ]; then
  echo "Installing Chef server..."
  rpm -Uvh /downloads/chef-server-core-12.16.9-1.el7.x86_64.rpm
  chef-server-ctl reconfigure

  echo "Waiting for services..."
  until (curl -D - http://localhost:8000/_status) | grep "200 OK"; do sleep 15s; done
  while (curl http://localhost:8000/_status) | grep "fail"; do sleep 15s; done

  echo "Creating initial user and organization..."
  chef-server-ctl user-create chefadmin Chef Admin admin@hoangpx.com insecurepassword --filename /drop/chefadmin.pem
  chef-server-ctl org-create hoangpx "hoangpx, Inc." --association_user chefadmin --filename hoangpx-validator.pem
fi

echo "Your Chef server is ready!"
```

### Install chef push job on server
```
chef-server-ctl install opscode-push-jobs-server
opscode-push-jobs-server-ctl reconfigure
chef-server-ctl reconfigure
```

Note : chefadmin.pem in /drop/

## Setup workstation 
_10.44.62.16 chef-workstation/igs-chef-workstation!_

Refer [https://docs.chef.io/install_dk.html]

### Generate chef repo
```chef generate repo chef-repo```
set git account
```
git config --global user.name "Your Name"
git config --global user.email username@domain.com
```
configuration git
```
echo ".chef" >> ~/chef-repo/.gitignore
cd ~/chef-repo
git add .
git commit -m "Excluding the ./.chef directory from version control"
cd ~
```
download chefdk

```wget https://packages.chef.io/files/stable/chefdk/2.3.1/el/7/chefdk-2.3.1-1.el7.x86_64.rpm```

install

```sudo rpm -Uvh chefdk-2.3.1-1.el7.x86_64.rpm```

verify

```chef verify
echo 'eval "$(chef shell-init bash)"' >> ~/.bash_profile
source ~/.bash_profile
which ruby
```

### Copy pem file from server to workstation

```scp chef@10.44.62.11:/home/chef/hoangpx-validator.pem ~/
scp chef@10.44.62.11:/home/chef/chefadmin.pem ~/

scp chefadmin.pem 192.168.20.151:/home/hpham@DAA.LOCAL/Desktop
scp hoangpx-validator.pem 192.168.20.151:/home/hpham@DAA.LOCAL/Desktop

scp chefadmin.pem chef-workstation@10.44.62.16:/home/chef-workstation/chef-repo/.chef/chefadmin.pem
scp hoangpx-validator.pem chef-workstation@10.44.62.16:/home/chef-workstation/chef-repo/.chef/hoangpx-validator.pem
```


### Create knife.rb in chef-repo/.chef
```
current_dir = File.dirname(__FILE__)
log_level                :info
log_location             STDOUT
node_name                "adminchef"
client_key               "#{current_dir}/chefadmin.pem"
validation_client_name   "hoangpx-validator"
validation_key           "#{current_dir}/hoangpx-validator.pem"
chef_server_url          "https://10.44.62.11/organizations/hoangpx"
syntax_check_cache_path  "#{ENV['HOME']}/.chef/syntaxcache"
cookbook_path ["#{current_dir}/../cookbooks"]
```

### Check if workstation can connect server
```cd ~/chef-repo
knife ssl fetch
knife client list
```

output should be ```hoangpx-validator```

### Install push job on Workstation
```chef gem install knife-push
knife cookbook site install push-jobs
```

change in push-job/attributes/default.rb
```
default['push_jobs']['package_url']                 = 'https://packages.chef.io/files/stable/push-jobs-client/2.4.1/el/7/push-jobs-client-2.4.1-1.el7.x86_64.rpm'
default['push_jobs']['package_checksum']            = '612d0482e347aa74c41aff3224e4a35dedf4dda21ff9cb8fe46c6393bce26be0'
default['push_jobs']['package_version']             = nil

# These variables control whether we validate ssl
default['push_jobs']['chef']['verify_api_cert']     = false
default['push_jobs']['chef']['ssl_verify_mode']     = :verify_none
```
upload push job
```
knife cookbook upload --all
```

## Install Chef client (This should be automatically installed when everything is ready)
```curl -L https://www.chef.io/chef/install.sh | sudo bash```

How to run push job
Run chef-client from client as:
```sudo chef-client –r "recipe[push-jobs]"```

Check status of current available node 
```
knife node status
knife node status <node-name>
```
Run your recipe from workstation as:

```knife job start 'chef-client –r recipe[push-job]' <node-name>```

Run your commands/script from workstation as:

```knife job start 'my_script.sh' <my_node>```

Note
```jmeter 10.44.62.15
chef 10.44.62.11
workstation 10.44.62.16
```