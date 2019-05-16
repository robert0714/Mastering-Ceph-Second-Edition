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

vagrant@ansible:~$ sudo apt-add-repository ppa:ansible/ansible
```
透過apt更新套件和安裝ansible

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

```
vagrant@ansible:~$  sudo vi /etc/ansible/hosts
[mon1]
10.100.0.41

[osd1]
10.100.0.51

[all]
10.100.0.4[1:3]
10.100.0.5[1:3]

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

```
 
複製下載的資源庫到本地的/etc/ansible目錄

```

vagrant@ansible:~$ sudo cp -a ceph-ansible/* /etc/ansible/

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

```

表示控制哪些群組模組使用哪些Ceph主機

上圖表示可以選擇使用套件是要用社群、本地、或者是發行版

說明預設一個fsid會自動產生，而且存在一個檔案內，不應該改變它。除非你想要控制fsid透過硬體編碼

上圖說明Monitor的選項，interface是mon主機顯示的網路介面名稱，還有IP和子網遮罩

並不是所有變量均由ceph管理，但是這邊讓你可以定義任何額外的變量到ceph.conf

格式大概如上

檢視OSD變量範例
------------
 
```
vagrant@ansible:~/ceph-ansible/group_vars$ cat osds.yml.sample

```

如果想要管理Ceph叢集中的OSD，不是只有管理Monitors，要設定true

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

```

vagrant@ansible:~$ sudo vi /etc/ansible/group_vars/ceph.yml

ceph_orign: 'upstream'
ceph_stable: true # use ceph stable branch
ceph_stable_key: https://download.ceph.com/key/release.asc
ceph_stable_release: jewel # ceph stable release
ceph_stable_repo: "http://download.ceph.com/debian-{{ ceph_stable_release }}"
monitor_interface: enp0s8 # Check ifconfig
public_network: 10.100.0.0/24
journal_size: 1024
vagrant@ansible:~$  sudo vi /etc/ansible/group_vars/osds.yml
</span>---
journal_collocation: True
devices:
 - /dev/sdb

```

創建fetch資料夾在/etc/ansible下，改變擁有人為vagrant

```

vagrant@ansible:~$ sudo mkdir /etc/ansible/fetch
vagrant@ansible:~$ sudo chown vagrant /etc/ansible/fetch

```

用設好的site.yml.sample作為playbook佈署Ceph Cluster，-K 表示會做sudo 密碼驗證


```

vagrant@ansible:~$ cd /etc/ansible
vagrant@ansible:~$ sudo mv site.yml.sample site.yml
vagrant@ansible:~$ ansible-playbook -K site.yml

```
其他參考: https://github.com/ceph/ceph-ansible