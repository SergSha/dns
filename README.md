<h3>### DNS ###</h3>

<p>настраиваем split-dns</p>

<h4>Описание домашнего задания</h4>

<ul>
<li>взять стенд https://github.com/erlong15/vagrant-bind</li>
<li>добавить еще один сервер client2</li>
<li>завести в зоне dns.lab</li>
<li>имена</li>
<li>web1 - смотрит на клиент1</li>
<li>web2 смотрит на клиент2</li>
<li>завести еще одну зону newdns.lab</li>
<li>завести в ней запись</li>
<li>www - смотрит на обоих клиентов</li>
<li>настроить split-dns</li>
<li>клиент1 - видит обе зоны, но в зоне dns.lab только web1</li>
<li>клиент2 видит только dns.lab</li>
<li>настроить все без выключения selinux<br />
Формат сдачи ДЗ - vagrant + ansible</li>
</ul>

<h4>1. Работа со стендом и настройка DNS</h4>

<p>В домашней директории создадим директорию dns, в котором будут храниться настройки виртуальных машин:</p>

<pre>[user@localhost otus]$ mkdir ./dns
[user@localhost otus]$</pre>

<p>Перейдём в директорию dns:</p>

<pre>[user@localhost otus]$ cd ./dns/
[user@localhost dns]$</pre>

<p>Скачаем стенд https://github.com/erlong15/vagrant-bind и изучим содержимое файлов:</p>

<pre>[user@localhost dns]$ ls -l
total 12
drwxrwxr-x. 2 user user 4096 фев 16  2020 provisioning
-rw-rw-r--. 1 user user 3100 сен 29 00:00 README.md
-rw-rw-r--. 1 user user  820 фев 16  2020 Vagrantfile
[user@localhost dns]$</pre>


<p>Откроем Vagrantfile, добавим ВМ "client2" и внесём некоторые корректировки, такие как вместо "ansible.sudo = "true" запишем "ansible.<b>become</b> = "true"":</p>

<pre>[user@localhost vpn]$ vi ./Vagrantfile</pre>

<pre># -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure(2) do |config|
  config.vm.box = "centos/7"

  config.vm.provision "ansible" do |ansible|
    ansible.verbose = "vvv"
    ansible.playbook = "provisioning/playbook.yml"
    ansible.<b>become</b> = "true"
  end


  config.vm.provider "virtualbox" do |v|
	  v.memory = 256
  end

  config.vm.define "ns01" do |ns01|
    ns01.vm.network "private_network", ip: "192.168.50.10", virtualbox__intnet: "dns"
    ns01.vm.hostname = "ns01"
  end

  config.vm.define "ns02" do |ns02|
    ns02.vm.network "private_network", ip: "192.168.50.11", virtualbox__intnet: "dns"
    ns02.vm.hostname = "ns02"
  end

  config.vm.define "client" do |client|
    client.vm.network "private_network", ip: "192.168.50.15", virtualbox__intnet: "dns"
    client.vm.hostname = "client"
  end

  <b>config.vm.define "client2" do |client2|
    client2.vm.network "private_network", ip: "192.168.50.16", virtualbox__intnet: "dns"
    client2.vm.hostname = "client2"
  end</b>

end</pre>







<p>Запустим эти виртуальные машины:</p>

<pre>[user@localhost vpn]$ vagrant up</pre>

<p>Проверим состояние созданной и запущенной машины:</p>

<pre>[user@localhost vpn]$ vagrant status
Current machine states:

server                    running (virtualbox)
client                    running (virtualbox)

This environment represents multiple VMs. The VMs are all listed
above with their current state. For more information about a specific
VM, run `vagrant status NAME`.
[user@localhost vpn]$</pre>

<p>2. Заходим на ВМ server и зайдём под пользователем root:</p>

<pre>[user@localhost vpn]$ vagrant ssh server
[vagrant@server ~]$ sudo -i
[root@server ~]# </pre>

<p>Выполняем следующие действия:<br />
● устанавливаем epel репозиторий:</p>

<pre>[root@server ~]# yum install -y epel-release</pre>
