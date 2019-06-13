Vagrant Host Manager
====================

[![Gem](https://img.shields.io/gem/v/vagrant-hostmanager.svg)](https://rubygems.org/gems/vagrant-hostmanager)
[![Gem](https://img.shields.io/gem/dt/vagrant-hostmanager.svg)](https://rubygems.org/gems/vagrant-hostmanager)
[![Gem](https://img.shields.io/gem/dtv/vagrant-hostmanager.svg)](https://rubygems.org/gems/vagrant-hostmanager)
[![Twitter](https://img.shields.io/twitter/url/https/github.com/devopsgroup-io/vagrant-hostmanager.svg?style=social)](https://twitter.com/intent/tweet?text=Check%20out%20this%20awesome%20Vagrant%20plugin%21&url=https%3A%2F%2Fgithub.com%devopsgroup-io%2Fvagrant-hostmanager&hashtags=vagrant%hostmanager&original_referer=)

`vagrant-hostmanager` is a Vagrant plugin that manages the `hosts` file on guest machines (and optionally the host). Its goal is to enable resolution of multi-machine environments deployed with a cloud provider where IP addresses are not known in advance.

Do you like what we do? Consider supporting us through Patreon. All of the money goes directly back into growing our collection of open source and free software.
[![Patreon](https://img.shields.io/badge/patreon-donate-red.svg)](https://www.patreon.com/devopsgroup)

Installation
------------

    $ vagrant plugin install vagrant-hostmanager

Usage
-----
To update the `hosts` file on each active machine, run the following
command:

    $ vagrant hostmanager

The plugin hooks into the `vagrant up` and `vagrant destroy` commands
automatically.
When a machine enters or exits the running state , all active
machines with the same provider will have their `hosts` file updated
accordingly. Set the `hostmanager.enabled` attribute to `true` in the
Vagrantfile to activate this behavior.

To update the host's `hosts` file, set the `hostmanager.manage_host`
attribute to `true`.

To update the guests' `hosts` file, set the `hostmanager.manage_guest`
attribute to `true`.

A machine's IP address is defined by either the static IP for a private
network configuration or by the SSH host configuration. To disable
using the private network IP address, set `config.hostmanager.ignore_private_ip`
to true.

A machine's host name is defined by `config.vm.hostname`. If this is not
set, it falls back to the symbol defining the machine in the Vagrantfile.

If the `hostmanager.include_offline` attribute is set to `true`, boxes that are
up or have a private ip configured will be added to the hosts file.

In addition, the `hostmanager.aliases` configuration attribute can be used
to provide aliases for your host names.

Example configuration:

```ruby
Vagrant.configure("2") do |config|
  config.hostmanager.enabled = true
  config.hostmanager.manage_host = true
  config.hostmanager.manage_guest = true
  config.hostmanager.ignore_private_ip = false
  config.hostmanager.include_offline = true
  config.vm.define 'example-box' do |node|
    node.vm.hostname = 'example-box-hostname'
    node.vm.network :private_network, ip: '10.100.42.42'
    node.hostmanager.aliases = %w(example-box.localdomain example-box-alias)
  end
end
```

### Provisioner

Starting at version 1.5.0, `vagrant up` runs hostmanager before any provisioning occurs. 
If you would like hostmanager to run after or during your provisioning stage, 
you can use hostmanager as a provisioner.  This allows you to use the provisioning 
order to ensure that hostmanager runs when desired. The provisioner will collect
hosts from boxes with the same provider as the running box.

Example:

```ruby
# Disable the default hostmanager behavior
config.hostmanager.enabled = false

# ... possible provisioner config before hostmanager ...

# hostmanager provisioner
config.vm.provision :hostmanager

# ... possible provisioning config after hostmanager ...
```

Custom IP resolver
------------------

You can customize way, how host manager resolves IP address
for each machine. This might be handy in case of aws provider,
where host name is stored in ssh_info hash of each machine.
This causes generation of invalid /etc/hosts file.

Custom IP resolver gives you oportunity to calculate IP address
for each machine by yourself, giving You also access to the machine that is
updating /etc/hosts. For example:

```ruby
config.hostmanager.ip_resolver = proc do |vm, resolving_vm|
  if hostname = (vm.ssh_info && vm.ssh_info[:host])
    `host #{hostname}`.split("\n").last[/(\d+\.\d+\.\d+\.\d+)/, 1]
  end
end
```

Passwordless sudo
-----------------

To avoid being asked for the password every time the hosts file is updated,
enable passwordless sudo for the specific command that hostmanager uses to
update the hosts file.

  - Add the following snippet to the sudoers file (e.g.
    `/etc/sudoers.d/vagrant_hostmanager`):

    ```
    Cmnd_Alias VAGRANT_HOSTMANAGER_UPDATE = /bin/cp <home-directory>/.vagrant.d/tmp/hosts.local /etc/hosts
    %<admin-group> ALL=(root) NOPASSWD: VAGRANT_HOSTMANAGER_UPDATE
    ```

    Replace `<home-directory>` with your actual home directory (e.g.
    `/home/joe`) and `<admin-group>` with the group that is used by the system
    for sudo access (usually `sudo` on Debian/Ubuntu systems and `wheel`
    on Fedora/Red Hat systems).

  - If necessary, add yourself to the `<admin-group>`:

    ```
    usermod -aG <admin-group> <user-name>
    ```

    Replace `<admin-group>` with the group that is used by the system for sudo
    access (see above) and `<user-name>` with you user name.

Windows support
---------------

Hostmanager will detect Windows guests and hosts and use the appropriate
path for the ```hosts``` file: ```%WINDIR%\System32\drivers\etc\hosts```

By default on a Windows host, the ```hosts``` file is not writable without
elevated privileges. If hostmanager detects that it cannot overwrite the file,
it will attempt to do so with elevated privileges, causing the
[UAC](http://en.wikipedia.org/wiki/User_Account_Control) prompt to appear.

To avoid the UAC prompt, open ```%WINDIR%\System32\drivers\etc\``` in
Explorer, right-click the hosts file, go to Properties > Security > Edit
and give your user Modify permission.

### UAC limitations

Due to limitations caused by UAC, cancelling out of the UAC prompt will not cause any
visible errors, however the ```hosts``` file will not be updated.


Compatibility
-------------
This Vagrant plugin has been tested with the following host and guest operating system combinations.

Date Tested | Vagrant Version | vagrant-hostmanager Version | Host (Workstation) Operating System | Guest (VirtualBox) Operating System
------------|-----------------|-----------------------------|-------------------------------------|--------------------------------------
03/23/2016  | 1.8.1           | 1.8.1                       | Ubuntu 14.04 LTS                    | CentOS 7.2
03/22/2016  | 1.8.1           | 1.8.1                       | OS X 10.11.4                        | CentOS 7.2
05/03/2017  | 1.9.4           | 1.8.6                       | macOS 10.12.4                       | Windows Server 2012 R2


Troubleshooting
-------------
* Version 1.1 of the plugin prematurely introduced a feature to hook into
commands other than `vagrant up` and `vagrant destroy`. Version 1.1 broke support
for some providers. Version 1.2 reverts this feature until a suitable implementation
supporting all providers is available.

* Potentially breaking change in v1.5.0: the running order on `vagrant up` has changed
so that hostmanager runs before provisioning takes place.  This ensures all hostnames are 
available to the guest when it is being provisioned 
(see [#73](https://github.com/devopsgroup-io/vagrant-hostmanager/issues/73)).
Previously, hostmanager would run as the very last action.  If you depend on the old behavior, 
see the [provisioner](#provisioner) section.


Contribute
----------
To contribute, fork then clone the repository, and then the following:

**Developing**

1. Ideally, install the version of Vagrant as defined in the `Gemfile`
1. Install [Ruby](https://www.ruby-lang.org/en/documentation/installation/)
2. Currently the Bundler version is locked to 1.14.6, please install this version.
    * `gem install bundler -v '1.14.6'`
3. Then install vagrant-hostmanager dependancies:
    * `bundle _1.14.6_ install`

**Testing**

1. Build and package your newly developed code:
    * `rake gem:build`
2. Then install the packaged plugin:
    * `vagrant plugin install pkg/vagrant-hostmanager-*.gem`
3. Once you're done testing, roll-back to the latest released version:
    * `vagrant plugin uninstall vagrant-hostmanager`
    * `vagrant plugin install vagrant-hostmanager`
4. Once you're satisfied developing and testing your new code, please submit a pull request for review.

**Releasing**

To release a new version of vagrant-hostmanager you will need to do the following:

*(only contributors of the GitHub repo and owners of the project at RubyGems will have rights to do this)*

1. First, bump, commit, and push the version in ~/lib/vagrant-hostmanager/version.rb:
    * Follow [Semantic Versioning](http://semver.org/).
2. Then, create a matching GitHub Release (this will also create a tag):
    * Preface the version number with a `v`.
    * https://github.com/devopsgroup-io/vagrant-hostmanager/releases
3. You will then need to build and push the new gem to RubyGems:
    * `rake gem:build`
    * `gem push pkg/vagrant-hostmanager-1.6.1.gem`
4. Then, when John Doe runs the following, they will receive the updated vagrant-hostmanager plugin:
    * `vagrant plugin update`
    * `vagrant plugin update vagrant-hostmanager`

範例中文說明
====================
中文翻譯出處:
https://blog.skywebster.com/learning-ceph-part3-deploy-ceph-lab/


Installation
------------

```bash

$ vagrant up

$ vagrant ssh ansible


```

ceph-deploy工具
------------
ceph-deploy工具是官方的工具來佈署Ceph叢集，它工作的原理是用一個台管理節點透過SSH連線(不需要密碼)連線到叢集內所有的Ceph節點，然而ceph-deploy在佈署和管理大架構的Ceph叢集時會有很大個開銷(花時間和人力)，所以建議使用ceph-deploy在小型的環境，建議使用”編排工具orchestration tool” ex: Puppet, Chef, Salt, Ansible去更快速的佈署Ceph

Orchestration
------------
使用Orchestration Tool Ansible的好處

這是RedHat所使用的佈署工具，Redhat剛好擁有兩個專案Ceph和Ansible
1. 它是易於開發和成熟的設定Ceph角色和playbooks
2. 如果沒有使用過編排工具，Ansible比較容易學習
3. 並不需要去設定中央化的伺服器，意思就是說可以專注的使用這個工具而不是去安裝它
4. 可以透過設定檔清楚明瞭你的環境

安裝ANSIBLE
------------
先裝lab環境需要的三台

```bash

$ vagrant up ansible mon1 osd1

```

設定ubuntu上Ansible的套件庫ansible

```bash

vagrant@ansible:~$ sudo add-apt-repository --remove  ppa:ansible/ansible
vagrant@ansible:~$ sudo apt-add-repository ppa:ansible/ansible-2.6
```
透過apt更新套件和安裝ansible
Ansible version must be between 2.4.x and 2.6.x!"
```

vagrant@ansible:~$ sudo apt-get update && sudo apt-get install ansible -y

```

創建目錄查找檔案(INVENTORY FILE)
------------

Ansible inventory file是被Ansible用來參照所有已知的主機以及他們屬於的群組

``` 

Vagrant@ansible:~$ ssh-keygen

```

SSH無密鑰登入
------------
在增加主機到inventory file之前，我們必須實行遠端節點透過SSH無密鑰登入



複製公開金鑰到osd1節點
------------

```
Vagrant@ansible:~$ ssh-copy-id mon1

```

創建ANSIBLE INVENTORY FILE
------------

```

Vagrant@ansible:~$ sudo vi /etc/ansible/hosts

```

變數
------------
大多的playbook和role均會使用變數，變數可以用很多方式改寫，最簡單的方式是創建host_vars和groups_vars 資料夾，這些資料夾允許你去改寫變數基於主機和群組

創建一個目錄
------------

```

vagrant@ansible:~$ sudo mkdir /etc/ansible/group_vars
```
 
在group_vars內創建一個檔案mons，內容寫上a_variable: “foo”；創建一個檔案osds，內容寫上a_variable: “bar”


```

vagrant@ansible:~$ sudo vi /etc/ansible/group_vars/mons
a_variable: "foo"
vagrant@ansible:~$ sudo vi /etc/ansible/group_vars/osds
a_variable: "bar"

```

也可以創一個all檔案放到所有的群組中，它將會套用到所有的群組；然而變數一般都會明確對應到將被改寫的群組。Ceph Ansible modules允許你設定一些預設的參數也允許你定義不同的數值在特殊的角色中。

可以參照官方說明：

http://docs.ceph.com/ceph-ansible/stable-3.0/

```
vagrant@ansible:~$  sudo vi /etc/ansible/hosts

[mons] 
mon1 
mon2 
mon3

[mgrs]
mon1 
 
[osds] 
osd1 
osd2 
osd3 
 
[ceph:children] 
mons 
osds
mgrs

```

測試
------------
測試與遠端主機是否有連線成功，對mon1主機執行 ping module, 成功就會回覆 pong

語法:
ansible  對象  -m  模組
-m 是 module 的意思

``` 

vagrant@ansible:~$ ansible mon1 -m ping

```

在遠端主機mon1下指令uname -r ，返回核心版本號，其中-m command可以省略不寫
ansible  對象  -m 模組 -a  ‘參數’
-a  是 module arguments

```

vagrant@ansible:~$ ansible mon1 -m command -a 'uname -r'
vagrant@ansible:~$ ansible mon1 'uname -r'

```

一個非常簡單的PLAYBOOK

```
vagrant@ansible:~$ sudo vi /etc/ansible/playbook.yml
---
- hosts: mon1 osd1
  tasks:
    - name: Echo Variables
      debug: msg="I am a {{ a_variable }}"
      
```



執行在兩個節點mon1、osd1，並跑了之前設定的變數的任務(在group_vars內)，最後顯示所有的狀態ok

```

vagrant@ansible:~$ ansible-playbook /etc/ansible/playbook.yml

```

加入Ceph Ansible模組
------------

在加入之前重建六個測試環境虛擬機並安裝好ansible

使用git去github克隆Ansible的資源庫

```

vagrant@ansible:~$ git clone https://github.com/ceph/ceph-ansible.git

vagrant@ansible:~$ cd ceph-ansible

vagrant@ansible:~/ceph-ansible$ git checkout stable-3.2

``` 
複製下載的資源庫到本地的/etc/ansible目錄

```

vagrant@ansible:~$  sudo cp -a ceph-ansible/* /etc/ansible/
vagrant@ansible:~$  export LCi_ALL="en_US.UTF-8"
vagrant@ansible:~$  export LC_CTYPE="en_US.UTF-8"
vagrant@ansible:~$  sudo dpkg-reconfigure locales
vagrant@ansible:~$  sudo apt-get install -y  python-pip
vagrant@ansible:~$  sudo pip install notario netaddr

```

其中有三個重要的目錄

1. group_vars: 群組的變數，見上面。
1. infrastructure-playbooks: 此目錄內含預先寫好的playbooks去實行一些基本的任務。像是佈署叢集增加osd到已經存在的叢集。
1. roles: 此目錄下所有的角色為Ceph Ansible modules組成，可以看做是每個Ceph元件均有對應的一個角色，這些角色將透過playbook去安裝、設定、維護Ceph。

為了能夠佈署Ceph Cluster 透過Ansible，一些關鍵的變數必須設定在group_vars目錄下。下面的變數有些建議更改不要用預設值。

檢視全域的關鍵變量範例
------------

``` 

vagrant@ansible:~$  cd ~/ceph-ansible/group_vars
vagrant@ansible:~/ceph-ansible/group_vars$   cat all.yml.sample

    #mon_group_name: mons
    #osd_group_name: osds
    #rgw_group_name: rgws
    #mds_group_name: mdss
    #nfs_group_name: nfss
    ...
    #iscsi_group_name: iscsigws
```

表示控制哪些群組模組使用哪些Ceph主機:

```
#ceph_origin: 'upstream' # or 'distro' or 'local'
```

上圖表示可以選擇使用套件是要用社群、本地、或者是發行版

```

#fsid: "{{ cluster_uuid.stdout }}"
#generate_fsid: true

```

說明預設一個fsid會自動產生，而且存在一個檔案內，不應該改變它。除非你想要控制fsid透過硬體編碼

```

#monitor_interface: interface
#monitor_address: 0.0.0.0

```

上圖說明Monitor的選項，interface是mon主機顯示的網路介面名稱，還有IP和子網遮罩


```

#ceph_conf_overrides: {}

```

並不是所有變量均由ceph管理，但是這邊讓你可以定義任何額外的變量到ceph.conf

```

    ceph_conf_overrides:
      global:
        variable1: value
      mon:
        variable2: value
      osd:
        variable3: value

```
格式大概如上

檢視OSD變量範例
------------
 
```
vagrant@ansible:~/ceph-ansible/group_vars$ cat osds.yml.sample

#copy_admin_key: false

```

如果想要管理Ceph叢集中的OSD，不是只有管理Monitors，要設定true

```

 #devices: []
 #osd_auto_discovery: false
 #journal_collocation: false
 #raw_multi_journal: false
 #raw_journal_devices: []

```

控制什麼硬碟當做OSD，日誌要如何放置

可以使用ansible_device去查找未使用的設備，存在的分割表不會被使用

這些參數只能用在2.2不能用在 3.1,，journal_collocation 此變數設定你是否要將日誌和osd存在同的硬碟，不同的分割區 ，raw_journal_devices 允需你去定義日誌使用在哪個裝置上，通常一個SSD會被存放日誌並對應多個OSD硬碟，raw_multi_journal 簡單指定放日誌的裝置放幾份，不需要分割區

透過Ansible佈署測試叢集
------------

事前準備

1. 用vagrant建立乾淨的虛擬機mon3台 osd 3台佈署主機ansible 1台
1. ansible主機安裝ansible，且更新系統
1. 創建inverntory file
1. 所有節點無密鑰登入
1. 從github克隆Ceph Ansible模組
1. 開始安裝

group_vars

建立檔案 /etc/ansible/group_vars/ceph
```
ceph_origin: 'repository'
ceph_repository: 'community'
ceph_mirror: http://download.ceph.com
ceph_stable: true # use ceph stable branch
ceph_stable_key: https://download.ceph.com/keys/release.asc
ceph_stable_release: mimic # ceph stable release
ceph_stable_repo: "{{ ceph_mirror }}/debian-{{ ceph_stable_release }}"
monitor_interface: eth1 #Check ifconfig
public_network: 10.100.0.0/24
journal_size: 1024

```

建立檔案 /etc/ansible/group_vars/osds

```
osd_scenario: lvm
lvm_volumes:
- data: /dev/sdb

```

創建fetch資料夾在/etc/ansible下，改變擁有人為vagrant

```

vagrant@ansible:~$ sudo mkdir /etc/ansible/fetch
vagrant@ansible:~$ sudo chown vagrant /etc/ansible/fetch

```

用設好的site.yml.sample作為playbook佈署Ceph Cluster，-K 表示會做sudo 密碼驗證


```

vagrant@ansible:~$ cd /etc/ansible
vagrant@ansible:/etc/ansible$ sudo mv site.yml.sample site.yml
vagrant@ansible:/etc/ansible$ ansible-playbook -K site.yml

```

Once done, assuming Ansible completed without errors, SSH into mon1 and run the following code. If Ansible did encounter errors, scroll up and look for the part which errored, the error text should give you a clue as to why it failed.

```bash

vagrant@mon1:~$ sudo ceph -s

```

其他參考: https://github.com/ceph/ceph-ansible

# Testing

Documentation for writing functional testing scenarios for ceph-ansible.

* Testing with ceph-ansible[http://docs.ceph.com/ceph-ansible/stable-3.0/testing/index.html]
* Glossary[http://docs.ceph.com/ceph-ansible/stable-3.0/testing/glossary.html]

# Demos

## Vagrant Demos

Deployment from scratch on bare metal machines: https://youtu.be/E8-96NamLDo

## Bare metal demo

Deployment from scratch on bare metal machines: https://youtu.be/dv_PEp9qAqg


Ceph in containers
====================
We have seen previously that by using orchestration tools such as Ansible we can reduce the work required to deploy, manage, and maintain a Ceph cluster. We have also seen how these tools can help you discover available hardware resources and deploy Ceph to them.

However, using Ansible to configure bare-metal servers still results in a very static deployment, possibly not best suited for today's more dynamic workloads. Designing Ansible playbooks also needs to take into account several different Linux distributions and also any changes that may occur between different releases; systemd is a great example of this. Furthermore, a lot of development in orchestration tools needs to be customized to handle discovering, deploying, and managing Ceph. This is a common theme that the Ceph developers have thought about; with the use of Linux containers and their associated orchestration platforms, they hope to improve Ceph's deployment experience.

One such approach, which has been selected as the preferred option, is to join forces with a project called Rook. Rook works with the container management platform Kubernetes to automate the deployment, configuration, and consumption of Ceph storage. If you were to draw up a list of requirements and features which a custom Ceph orchestration and management framework would need to implement, you would likely design something which functions in a similar fashion to Kubernetes. So it makes sense to build functionality on top of the well-established Kubernetes project, and Rook does exactly that.

One major benefit of running Ceph in containers is that is allows collocation of services on the same hardware. Traditionally in Ceph clusters it was expected that Ceph monitors would run on dedicated hardware; when utilizing containers this requirement is removed. For smaller clusters, this can amount to a large saving in the cost of running and purchasing servers. If resources permit, other container-based workloads could also be allowed to run across the Ceph hardware, further increasing the Return on Investment for the hardware purchase. The use of Docker containers reserves the required hardware resources so that workloads cannot impact each other.

To better understand how these two technologies work with Ceph, we first need to cover Kubernetes in more detail and actual containers themselves.

# Containers
Although containers in their current form are a relatively new technology, the principle of isolating sets of processes from each other has been around for a long time. What the current set of technologies enhances is the completeness of the isolation. Previous technologies maybe only isolated parts of the filesystem, whereas the latest container technologies also isolate several areas of the operating system and can also provide quotas for hardware resources. One technology in particular, Docker, has risen to become the most popular technology when talking about containers, so much so that the two words are often used interchangeably. The word container describes a technology that performs operating system-level virtualization. Docker is a software product that controls primarily Linux features such as groups and namespaces to isolate sets of Linux processes.

It's important to note that, unlike full-blown virtualization solutions such as VMWare, Hyper-V, and KVM, which provides virtualized hardware and require a separate OS instance, containers utilize the operating system of the host. The full OS requirements of virtual machines may lead to several 10s of GB of storage being wasted on the operating system installation and potentially several GB of RAM as well. Containers typically consume overheads of storage and RAM measured in MB, meaning that a lot more containers can be squeezed onto the same hardware when compared to full virtualization technologies.

# Kubernetes

The ability to quickly and efficiently spin up 10's of containers in seconds soon makes you realize that. if VM sprawl was bad enough, with containers the problem can easily get a whole lot worse. With the arrival of Docker in the modern IT infrastructure, a need to manage all these containers arose. Enter Kubernetes.

Although several container orchestration technologies are available, Kubernetes has enjoyed wide-ranging success and, as it is the product on which Rook is built, this book will focus on it.

Kubernetes is an open source container-orchestration system for automating the deployment, scaling, and management of containerized applications. It was originally developed at Google to run their internal systems but has since been open sourced and seen its popularity flourish.

Although this chapter will cover deploying an extremely simple Kubernetes cluster to deploy a Ceph cluster with Rook, it is not meant to be a full tutorial and readers are encouraged to seek other resources in order to learn more about Kubernetes.

## Deploying a Ceph cluster with Rook

To deploy a Ceph cluster with Rook and Kubernetes, Vagrant will be used to create three VMs that will run the Kubernetes cluster.

The first task you'll complete is the deployment of three VMs via Vagrant. If you have followed the steps at that start of this chapter and used Vagrant to build an environment for Ansible, then you should have everything you require to deploy VMs for the Kubernetes cluster.

The following is the Vagrantfile to bring up three VMs; as before, place the contents into a file called Vagrantfile in a new directory and then run vagrant up:

```

nodes = [
  { :hostname => 'kube1',  :ip => '192.168.0.51', :box => 'xenial64', :ram => 2048, :osd => 'yes' },
  { :hostname => 'kube2',  :ip => '192.168.0.52', :box => 'xenial64', :ram => 2048, :osd => 'yes' },
  { :hostname => 'kube3',  :ip => '192.168.0.53', :box => 'xenial64', :ram => 2048, :osd => 'yes' }
]


Vagrant.configure("2") do |config|
  nodes.each do |node|
    config.vm.define node[:hostname] do |nodeconfig|
      nodeconfig.vm.box = "bento/ubuntu-16.04"
      nodeconfig.vm.hostname = node[:hostname]
      nodeconfig.vm.network :private_network, ip: node[:ip]


      memory = node[:ram] ? node[:ram] : 4096;
      nodeconfig.vm.provider :virtualbox do |vb|
        vb.customize [
          "modifyvm", :id,
          "--memory", memory.to_s,
        ]
        if node[:osd] == "yes"        
          vb.customize [ "createhd", "--filename", "disk_osd-#{node[:hostname]}", "--size", "10000" ]
          vb.customize [ "storageattach", :id, "--storagectl", "SATA Controller", "--port", 3, "--device", 0, "--type", "hdd", "--medium", "disk_osd-#{node[:hostname]}.vdi" ]
        end
      end
    end
    config.hostmanager.enabled = true
    config.hostmanager.manage_guest = true
  end
end

```
SSH in to the first VM, Kube1:

```bash

vagrant ssh kube1

```

Update the kernel to a newer version; this is required for certain Ceph features in Rook to function correctly:

```

sudo apt-get update && sudo apt-get install linux-generic-hwe-16.04

```

Install Docker, as follows:

```

sudo apt-get install docker.io

```

Enable and start the Docker service, as follows:

```

sudo systemctl start docker
sudo systemctl enable docker

```

Disable swap for future boots by editing /etc/fstab and commenting out the swap line:

And also disable swap now, as follows:

```

sudo swapoff -a

```

Add the Kubernetes repository, as follows:

sudo add-apt-repository “deb http://apt.kubernetes.io/ kubernetes-xenial main”

Add the Kubernetes GPG key, as follows:

```

sudo curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add

```

Install Kubernetes, as follows:

```

sudo apt-get update && sudo apt-get install -y kubeadm kubelet kubectl

```

Repeat the installation steps for Docker and Kubernetes on both the kube2 and kube3 VMs.

Once all the VMs have a working copy of Docker and Kubernetes, we can now initialize the Kubernetes cluster:

```

sudo kubeadm init --apiserver-advertise-address=10.100.0.51 --pod-network-cidr=10.1.0.0/16 --ignore-preflight-errors=NumCPU

```


At the end of the process, a command string is output; make a note of this as it is needed to join our additional nodes to the cluster. An example of this is as follows:


Now that we have installed Docker and Kubernetes on all our nodes and have initialized the master, let's add the remaining two nodes into the cluster. Remember that string of text you were asked to note down? Now we can run it on the two remaining nodes:

```

sudo kubeadm join 192.168.0.51:6443 --token c68o8u.92pvgestk26za6md --discovery-token-ca-cert-hash sha256:3954fad0089dcf72d0d828b440888b6e97465f783bde403868f098af67e8f073

```


```

$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config

```


We can now install some additional container networking support. Flannel, a simple networking add-on for Kubernetes, uses VXLAN as an overlay to enable container-to-container networking. First download the yaml file from GitHub:

```

$ wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

--2019-06-13 13:02:56--  https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
Resolving raw.githubusercontent.com (raw.githubusercontent.com)... 151.101.76.133
Connecting to raw.githubusercontent.com (raw.githubusercontent.com)|151.101.76.133|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 12306 (12K) [text/plain]
Saving to: 'kube-flannel.yml'

100%[================================================================================================================================================================================================================================================>] 12,306      --.-K/s   in 0s      

```

Before we install the Flannel networking component, we need to make a few changes to the YAML spec file:

```

$ nano kube-flannel.yml

```

We need to find the following lines and make the required changes, as follows:

Line 76: "Network": "10.1.0.0/16":

Line 126: - --iface=eth1:

Now we can issue the relevant Kubernetes command to apply the specification file and install Flannel networking:

```

$ kubectl apply -f kube-flannel.yml

podsecuritypolicy.extensions/psp.flannel.unprivileged created
clusterrole.rbac.authorization.k8s.io/flannel created
clusterrolebinding.rbac.authorization.k8s.io/flannel created
serviceaccount/flannel created
configmap/kube-flannel-cfg created
daemonset.extensions/kube-flannel-ds-amd64 created
daemonset.extensions/kube-flannel-ds-arm64 created
daemonset.extensions/kube-flannel-ds-arm created
daemonset.extensions/kube-flannel-ds-ppc64le created
daemonset.extensions/kube-flannel-ds-s390x created

```

After networking has been installed, we can confirm everything is working and that our Kubernetes worker nodes are ready to run workloads:

```

$ kubectl get nodes

NAME     STATUS   ROLES    AGE     VERSION
k8s-m1   Ready    master   2d21h   v1.14.0
k8s-n1   Ready    <none>   2d21h   v1.14.0
k8s-n2   Ready    <none>   2d21h   v1.14.0

```
Now let's also check that all containers that support internal Kubernetes services are running:

```

$ kubectl get pods --all-namespaces -o wide

NAMESPACE      NAME                                       READY   STATUS      RESTARTS   AGE     IP               NODE     NOMINATED NODE   READINESS GATES
kube-system    calico-etcd-zxf69                          1/1     Running     0          2d20h   192.168.57.160   k8s-m1   <none>           <none>
kube-system    calico-kube-controllers-866bb996d6-skhxr   1/1     Running     1          2d20h   192.168.57.160   k8s-m1   <none>           <none>
kube-system    calico-node-2fkw6                          2/2     Running     2          2d20h   192.168.57.160   k8s-m1   <none>           <none>
kube-system    calico-node-dxsvn                          2/2     Running     0          2d20h   192.168.57.162   k8s-n2   <none>           <none>
kube-system    calico-node-fdg58                          2/2     Running     1          2d20h   192.168.57.161   k8s-n1   <none>           <none>
kube-system    coredns-fb8b8dccf-dt9xk                    1/1     Running     0          2d20h   10.244.215.66    k8s-n1   <none>           <none>
kube-system    coredns-fb8b8dccf-fb5cm                    1/1     Running     0          2d20h   10.244.215.65    k8s-n1   <none>           <none>
kube-system    etcd-k8s-m1                                1/1     Running     0          2d20h   192.168.57.160   k8s-m1   <none>           <none>
kube-system    kube-apiserver-k8s-m1                      1/1     Running     0          2d20h   192.168.57.160   k8s-m1   <none>           <none>
kube-system    kube-controller-manager-k8s-m1             1/1     Running     1          2d20h   192.168.57.160   k8s-m1   <none>           <none>
kube-system    kube-proxy-klkjx                           1/1     Running     0          2d20h   192.168.57.161   k8s-n1   <none>           <none>
kube-system    kube-proxy-mxzwv                           1/1     Running     0          2d20h   192.168.57.160   k8s-m1   <none>           <none>
kube-system    kube-proxy-pfp84                           1/1     Running     0          2d20h   192.168.57.162   k8s-n2   <none>           <none>
kube-system    kube-scheduler-k8s-m1                      1/1     Running     1          2d20h   192.168.57.160   k8s-m1   <none>           <none>
kube-system    kubernetes-dashboard-5f7b999d65-l6cmh      1/1     Running     0          2d20h   10.244.42.131    k8s-m1   <none>           <none>


```
Note that the container networking service (Flannel) that we installed in the previous step has automatically been deployed  across all three nodes. At this point, we have a fully functioning Kubernetes cluster that is ready to run whatever containers we wish to run on it.

We can now deploy Rook into the Kubernetes cluster. First, let's clone the Rook project from GitHub:

```

$ git clone https://github.com/rook/rook.git

```
tutorials:
https://github.com/rook/rook/blob/master/Documentation/ceph-quickstart.md

Change to the examples directory, as follows:

```

$ cd rook/cluster/examples/kubernetes/ceph/

```

And now finally create the Rook-powered Ceph cluster by running the following two commands:

```bash
$ kubectl create -f common.yaml 

customresourcedefinition.apiextensions.k8s.io/cephclusters.ceph.rook.io created
customresourcedefinition.apiextensions.k8s.io/cephfilesystems.ceph.rook.io created
customresourcedefinition.apiextensions.k8s.io/cephnfses.ceph.rook.io created
customresourcedefinition.apiextensions.k8s.io/cephobjectstores.ceph.rook.io created
customresourcedefinition.apiextensions.k8s.io/cephobjectstoreusers.ceph.rook.io created
customresourcedefinition.apiextensions.k8s.io/cephblockpools.ceph.rook.io created
customresourcedefinition.apiextensions.k8s.io/volumes.rook.io created
clusterrole.rbac.authorization.k8s.io/rook-ceph-cluster-mgmt created
clusterrole.rbac.authorization.k8s.io/rook-ceph-cluster-mgmt-rules created
role.rbac.authorization.k8s.io/rook-ceph-system created
clusterrole.rbac.authorization.k8s.io/rook-ceph-global created
clusterrole.rbac.authorization.k8s.io/rook-ceph-global-rules created
clusterrole.rbac.authorization.k8s.io/rook-ceph-mgr-cluster created
clusterrole.rbac.authorization.k8s.io/rook-ceph-mgr-cluster-rules created
serviceaccount/rook-ceph-system created
rolebinding.rbac.authorization.k8s.io/rook-ceph-system created
clusterrolebinding.rbac.authorization.k8s.io/rook-ceph-global created
serviceaccount/rook-ceph-osd created
serviceaccount/rook-ceph-mgr created
role.rbac.authorization.k8s.io/rook-ceph-osd created
clusterrole.rbac.authorization.k8s.io/rook-ceph-mgr-system created
clusterrole.rbac.authorization.k8s.io/rook-ceph-mgr-system-rules created
role.rbac.authorization.k8s.io/rook-ceph-mgr created
rolebinding.rbac.authorization.k8s.io/rook-ceph-cluster-mgmt created
rolebinding.rbac.authorization.k8s.io/rook-ceph-osd created
rolebinding.rbac.authorization.k8s.io/rook-ceph-mgr created
rolebinding.rbac.authorization.k8s.io/rook-ceph-mgr-system created
clusterrolebinding.rbac.authorization.k8s.io/rook-ceph-mgr-cluster created


$ kubectl create -f operator.yaml

deployment.apps/rook-ceph-operator created


```

```bash

$ kubectl create -f cluster.yaml

cephcluster.ceph.rook.io/rook-ceph created

```

To confirm our Rook cluster is now working, let's check the running containers under the Rook namespace:

```

$ kubectl get pods --all-namespaces -o wide

NAMESPACE      NAME                                       READY   STATUS      RESTARTS   AGE     IP               NODE     NOMINATED NODE   READINESS GATES
kube-system    calico-etcd-zxf69                          1/1     Running     0          2d20h   192.168.57.160   k8s-m1   <none>           <none>
kube-system    calico-kube-controllers-866bb996d6-skhxr   1/1     Running     1          2d20h   192.168.57.160   k8s-m1   <none>           <none>
kube-system    calico-node-2fkw6                          2/2     Running     2          2d20h   192.168.57.160   k8s-m1   <none>           <none>
kube-system    calico-node-dxsvn                          2/2     Running     0          2d20h   192.168.57.162   k8s-n2   <none>           <none>
kube-system    calico-node-fdg58                          2/2     Running     1          2d20h   192.168.57.161   k8s-n1   <none>           <none>
kube-system    coredns-fb8b8dccf-dt9xk                    1/1     Running     0          2d20h   10.244.215.66    k8s-n1   <none>           <none>
kube-system    coredns-fb8b8dccf-fb5cm                    1/1     Running     0          2d20h   10.244.215.65    k8s-n1   <none>           <none>
kube-system    etcd-k8s-m1                                1/1     Running     0          2d20h   192.168.57.160   k8s-m1   <none>           <none>
kube-system    kube-apiserver-k8s-m1                      1/1     Running     0          2d20h   192.168.57.160   k8s-m1   <none>           <none>
kube-system    kube-controller-manager-k8s-m1             1/1     Running     1          2d20h   192.168.57.160   k8s-m1   <none>           <none>
kube-system    kube-flannel-ds-amd64-5kb47                1/1     Running     0          23m     192.168.57.161   k8s-n1   <none>           <none>
kube-system    kube-flannel-ds-amd64-7rfns                1/1     Running     0          23m     192.168.57.160   k8s-m1   <none>           <none>
kube-system    kube-flannel-ds-amd64-v5t98                1/1     Running     0          23m     192.168.57.162   k8s-n2   <none>           <none>
kube-system    kube-proxy-klkjx                           1/1     Running     0          2d20h   192.168.57.161   k8s-n1   <none>           <none>
kube-system    kube-proxy-mxzwv                           1/1     Running     0          2d20h   192.168.57.160   k8s-m1   <none>           <none>
kube-system    kube-proxy-pfp84                           1/1     Running     0          2d20h   192.168.57.162   k8s-n2   <none>           <none>
kube-system    kube-scheduler-k8s-m1                      1/1     Running     1          2d20h   192.168.57.160   k8s-m1   <none>           <none>
kube-system    kubernetes-dashboard-5f7b999d65-l6cmh      1/1     Running     0          2d20h   10.244.42.131    k8s-m1   <none>           <none>
rook-ceph      rook-ceph-agent-5ljpz                      1/1     Running     0          3m54s   192.168.57.161   k8s-n1   <none>           <none>
rook-ceph      rook-ceph-agent-wvlxj                      1/1     Running     0          3m53s   192.168.57.162   k8s-n2   <none>           <none>
rook-ceph      rook-ceph-operator-775cf575c5-6dqh4        1/1     Running     0          3m55s   10.244.215.83    k8s-n1   <none>           <none>
rook-ceph      rook-discover-bj9lr                        1/1     Running     0          3m53s   10.244.111.203   k8s-n2   <none>           <none>
rook-ceph      rook-discover-f2lt6                        1/1     Running     0          3m53s   10.244.215.84    k8s-n1   <none>           <none>
```

You will see that Rook has deployed a couple of mons and has also started some discover containers. These discover containers run a discovery script to locate storage devices attached to the Kubernetes physical host. Once the discovery process has completed for the first time, Kubernetes will then run a one-shot container to prepare the OSD by formatting the disk and adding the OSD into the cluster. If you wait a few minutes and re-run the get pods command, you should hopefully see that Rook has detected the two disks connected to kube2 and kube3 and created osd containers for them:

```bash

rook-ceph      rook-ceph-agent-5ljpz                      1/1     Running     0          18m     192.168.57.161   k8s-n1   <none>           <none>
rook-ceph      rook-ceph-agent-wvlxj                      1/1     Running     0          18m     192.168.57.162   k8s-n2   <none>           <none>
rook-ceph      rook-ceph-mgr-a-547bf56fdf-wvr9f           1/1     Running     0          13m     10.244.111.205   k8s-n2   <none>           <none>
rook-ceph      rook-ceph-mon-a-66d969795f-qvl6l           1/1     Running     0          14m     10.244.215.87    k8s-n1   <none>           <none>
rook-ceph      rook-ceph-mon-b-7557ffdd78-nsnhj           1/1     Running     0          13m     10.244.111.204   k8s-n2   <none>           <none>
rook-ceph      rook-ceph-mon-c-6569b484f7-zqtdk           1/1     Running     0          13m     10.244.215.88    k8s-n1   <none>           <none>
rook-ceph      rook-ceph-operator-775cf575c5-6dqh4        1/1     Running     0          18m     10.244.215.83    k8s-n1   <none>           <none>
rook-ceph      rook-ceph-osd-prepare-k8s-n1-bkxj4         0/2     Completed   1          12m     10.244.215.89    k8s-n1   <none>           <none>
rook-ceph      rook-ceph-osd-prepare-k8s-n2-vdfsg         0/2     Completed   0          12m     10.244.111.206   k8s-n2   <none>           <none>
rook-ceph      rook-discover-bj9lr                        1/1     Running     0          18m     10.244.111.203   k8s-n2   <none>           <none>
rook-ceph      rook-discover-f2lt6                        1/1     Running     0          18m     10.244.215.84    k8s-n1   <none>           <none>


```

To interact with the cluster, let's deploy the toolbox container; this is a simple container containing the Ceph installation and the necessary cluster keys:


```bash

$ kubectl create -f toolbox.yaml

deployment.apps/rook-ceph-tools created
```

Now execute bash in the toolbox container:

```bash

$ kubectl -n rook-ceph exec -it $(kubectl -n rook-ceph get pod -l "app=rook-ceph-tools" -o jsonpath='{.items[0].metadata.name}') bash

bash: warning: setlocale: LC_CTYPE: cannot change locale (en_US.UTF-8): No such file or directory
bash: warning: setlocale: LC_COLLATE: cannot change locale (en_US.UTF-8): No such file or directory
bash: warning: setlocale: LC_MESSAGES: cannot change locale (en_US.UTF-8): No such file or directory
bash: warning: setlocale: LC_NUMERIC: cannot change locale (en_US.UTF-8): No such file or directory
bash: warning: setlocale: LC_TIME: cannot change locale (en_US.UTF-8): No such file or directory

[root@k8s-n1 /]#
```

This will present you with a root shell running inside the Ceph toolbox container, where we can check the status of the Ceph cluster by running ceph –s and see the current OSDs with ceph osd tree:

```bash

[root@k8s-n1 /]# ceph -s
  cluster:
    id:     7f302c9e-e57d-486f-bef3-e489a227166f
    health: HEALTH_OK
 
  services:
    mon: 3 daemons, quorum a,b,c (age 29m)
    mgr: a(active, since 28m)
    osd: 0 osds: 0 up, 0 in
 
  data:
    pools:   0 pools, 0 pgs
    objects: 0 objects, 0 B
    usage:   0 B used, 0 B / 0 B avail
    pgs:     
 
[root@k8s-n1 /]#  ceph osd tree
ceph osd tree
ID CLASS WEIGHT TYPE NAME    STATUS REWEIGHT PRI-AFF 
-1            0 root default 

[root@k8s-n1 /]#  exit
```

You will notice that, although we built three VMs, Rook has only deployed OSDs on kube2 and kube3. This is because by default Kubernetes will not schedule containers to run on the master node; in a production cluster this is the desired behavior, but for testing we can remove this limitation.

Exit back to the master Kubernetes node and run the following:

```

kubectl taint nodes $(hostname) node-role.kubernetes.io/master:NoSchedule-node/kube1 untained

```

You will notice that Kubernetes will deploy a couple of new containers onto kube1, but it won't deploy any new OSDs; this is due to a current limitation to the effect that the rook-ceph-operator component only deploys new OSDs on first startup. In order to detect newly available disks and prepare them as OSDs, the rook-ceph-operator container needs to be deleted.

Run the following command, but replace the container name with the one that is listed from the get pods command:

```

kubectl -n rook-ceph-system delete pods rook-ceph-operator-775cf575c5-6dqh4    

```

Kubernetes will now automatically spin up a new rook-ceph-operator container and in doing so will kick-start the deployment of the new osd; this can be confirmed by looking at the list of running containers again:

```bash

rook-ceph      rook-ceph-agent-5ljpz                      1/1     Running     0          42m     192.168.57.161   k8s-n1   <none>           <none>
rook-ceph      rook-ceph-agent-wvlxj                      1/1     Running     0          42m     192.168.57.162   k8s-n2   <none>           <none>
rook-ceph      rook-ceph-mgr-a-547bf56fdf-wvr9f           1/1     Running     0          37m     10.244.111.205   k8s-n2   <none>           <none>
rook-ceph      rook-ceph-mon-a-66d969795f-qvl6l           1/1     Running     0          38m     10.244.215.87    k8s-n1   <none>           <none>
rook-ceph      rook-ceph-mon-b-7557ffdd78-nsnhj           1/1     Running     0          38m     10.244.111.204   k8s-n2   <none>           <none>
rook-ceph      rook-ceph-mon-c-6569b484f7-zqtdk           1/1     Running     0          37m     10.244.215.88    k8s-n1   <none>           <none>
rook-ceph      rook-ceph-operator-775cf575c5-6dqh4        1/1     Running     0          42m     10.244.215.83    k8s-n1   <none>           <none>
rook-ceph      rook-ceph-osd-prepare-k8s-n1-bkxj4         0/2     Completed   1          36m     10.244.215.89    k8s-n1   <none>           <none>
rook-ceph      rook-ceph-osd-prepare-k8s-n2-vdfsg         0/2     Completed   0          36m     10.244.111.206   k8s-n2   <none>           <none>
rook-ceph      rook-ceph-tools-b8c679f95-f8f6p            1/1     Running     0          10m     192.168.57.161   k8s-n1   <none>           <none>
rook-ceph      rook-discover-bj9lr                        1/1     Running     0          42m     10.244.111.203   k8s-n2   <none>           <none>
rook-ceph      rook-discover-f2lt6                        1/1     Running     0          42m     10.244.215.84    k8s-n1   <none>           <none>

```

You can see kube1 has run a rook-discover container, a rook-ceph-osd-prepare, and finally a rook-ceph-osd container, which in this case is osd number 2.

We can also check, by using our toolbox container as well, that the new osd has joined the cluster successfully:


Now that Rook has deployed our full test Ceph cluster, we need to make use of it and create some RADOS pools and also consume some storage with a client container. To demonstrate this process, we will deploy a CephFS filesystem.

Before we jump straight into deploying the filesystem, let's first have a look at the example yaml file we will be deploying. Make sure you are still in the ~/rook/cluster/examples/kubernetes/ceph directory and use a text editor to view the filesystem.yaml file:

You can see that the file contents describe the RADOS pools that will be created and the MDS instances that are required for the filesystem. In this example, three pools will be deployed, two replicated and one erasure-coded for the actual data. Two MDS servers will be deployed, one running as active and the other running as a standby-replay.

Exit the text editor and now deploy the CephFS configuration in the yaml file:


```

$ kubectl create -f filesystem.yaml
cephfilesystem.ceph.rook.io/myfs created
```

Now let's jump back into our toolbox container, check the status, and see what's been created:

```bash

[vagrant@k8s-m1 ceph]$ kubectl -n rook-ceph exec -it $(kubectl -n rook-ceph get pod -l "app=rook-ceph-tools" -o jsonpath='{.items[0].metadata.name}') bash
bash: warning: setlocale: LC_CTYPE: cannot change locale (en_US.UTF-8): No such file or directory
bash: warning: setlocale: LC_COLLATE: cannot change locale (en_US.UTF-8): No such file or directory
bash: warning: setlocale: LC_MESSAGES: cannot change locale (en_US.UTF-8): No such file or directory
bash: warning: setlocale: LC_NUMERIC: cannot change locale (en_US.UTF-8): No such file or directory
bash: warning: setlocale: LC_TIME: cannot change locale (en_US.UTF-8): No such file or directory
[root@k8s-n1 /]# ceph -s
  cluster:
    id:     7f302c9e-e57d-486f-bef3-e489a227166f
    health: HEALTH_OK
 
  services:
    mon: 3 daemons, quorum a,b,c (age 40m)
    mgr: a(active, since 39m)
    mds: myfs:1 {0=myfs-a=up:creating} 1 up:standby-replay
    osd: 0 osds: 0 up, 0 in
 
  data:
    pools:   2 pools, 200 pgs
    objects: 0 objects, 0 B
    usage:   0 B used, 0 B / 0 B avail
    pgs:     100.000% pgs unknown
             200 unknown
 
[root@k8s-n1 /]# ceph osd  lspools
1 myfs-metadata
2 myfs-data0


```

We can see that two pools have been created, one for the CephFS metadata and one for the actual data stored on the CephFS filesystem.

To give an example of how Rook can then be consumed by application containers, we will now deploy a small NGINX web server container that stores its HTML content on the CephFS filesystem.

Place the following inside a file called nginx.yaml:

```

apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app: web
spec:
  containers:
    - name: nginx
      image: nginx:1.7.9
      ports:
        - containerPort: 80
      volumeMounts:
        - name: www   
          mountPath: /usr/share/nginx/html
  volumes:
    - name: www
      flexVolume:
      driver: "ceph.rook.io/rook"
      fsType: "ceph"
      options:
        fsName: "myfs"
        clusterNamespace: "rook-ceph"rook-ceph

```
ps.  You can validate the yaml  https://kubeyaml.com/

And now use the kubectl command to create the pod/nginx:

```bash

[vagrant@k8s-m1 ~]$ kubectl get pods 
NAME                                READY   STATUS    RESTARTS   AGE
nginx                               1/1     Running   0          48s


```


After a while, the container will be started and will enter a running state; use the get pods command to verify this:



We can now start a quick Bash shell on this container to confirm the CephFS mount has worked:

```

[vagrant@k8s-m1 ~]$ kubectl exec -it nginx bash

root@nginx:/# df -H

Filesystem               Size  Used Avail Use% Mounted on
rootfs                    44G  6.5G   38G  15% /
overlay                   44G  6.5G   38G  15% /
tmpfs                     68M     0   68M   0% /dev
tmpfs                    1.6G     0  1.6G   0% /sys/fs/cgroup
/dev/mapper/centos-root   44G  6.5G   38G  15% /dev/termination-log
shm                       68M     0   68M   0% /dev/shm
/dev/mapper/centos-root   44G  6.5G   38G  15% /etc/resolv.conf
/dev/mapper/centos-root   44G  6.5G   38G  15% /etc/hostname
/dev/mapper/centos-root   44G  6.5G   38G  15% /etc/hosts
/dev/mapper/centos-root   44G  6.5G   38G  15% /var/cache/nginx
/dev/mapper/centos-root   44G  6.5G   38G  15% /usr/share/nginx/html
tmpfs                    1.6G   13k  1.6G   1% /run/secrets/kubernetes.io/serviceaccount
tmpfs                    1.6G     0  1.6G   0% /proc/acpi
tmpfs                     68M     0   68M   0% /proc/kcore
tmpfs                     68M     0   68M   0% /proc/keys
tmpfs                     68M     0   68M   0% /proc/timer_list
tmpfs                     68M     0   68M   0% /proc/timer_stats
tmpfs                     68M     0   68M   0% /proc/sched_debug
tmpfs                    1.6G     0  1.6G   0% /proc/scsi
tmpfs                    1.6G     0  1.6G   0% /sys/firmware
root@nginx:/# 

```

We can see that the CephFS filesystem has been mounted into /usr/share/nginx/html. This has been done without having to install any Ceph components in the container and without any configuration or copying of key rings. Rook has taken care of all of this behind the scenes; once this is understood and appreciated, the real power of Rook can be seen. If the simple NGINX pod example is expanded to become an auto-scaling service that spins up multiple containers based on load, the flexibility given by Rook and Ceph to automatically present the same shared storage across the web farm with no additional configuration is very useful.


##  Delete the Operator and related Resources
https://github.com/rook/rook/blob/master/Documentation/ceph-teardown.md

```bash

kubectl delete -f operator.yaml
kubectl delete -f common.yaml


```

##  Summary

In this chapter, you learned about Ceph's various deployment methods and the differences between them. You will now also have a basic understanding of how Ansible works and how to deploy a Ceph cluster with it. It would be advisable at this point to continue investigating and practicing the deployment and configuration of Ceph with Ansible, so that you are confident enough to use it in production environments. The remainder of this book will also assume that you have fully understood the contents of this chapter in order to manipulate the configuration of Ceph.

You have also learned about the exciting new developments in deploying Ceph in containers running on the Kubernetes platform. Although the Rook project is still in the early stages of development, it is clear it is already a very powerful tool that will enable Ceph to function to the best of its ability while at the same time simplifying the deployment and administration required. With the continued success enjoyed by Kubernetes in becoming the recommended container management platform, integrating Ceph with the use of Rook will result in a perfect match of technologies.

It is highly recommended that the reader should continue to learn further about Kubernetes as this chapter has only scratched the surface of the functionality it offers. There are strong signs across the industry that containerization is going to be the technology for deploying and managing applications and having an understanding of both Kubernetes and how Ceph integrates with Rook is highly recommended.

## Questions

1. What piece of software can be used to rapidly deploy test environments?
1. Should vagrant be used to deploy production environments?
1. What project enables the deployment of Ceph on top of Kubernetes?
1. What is Docker?
1. What is the Ansible file called which is used to run a series of commands?

