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


<p>Откроем Vagrantfile, добавим ВМ client2 и внесём некоторые корректировки, такие как вместо <i>ansible.sudo = "true"</i> запишем <i>ansible.<b>become</b> = "true"</i>:</p>

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

<p>После того, как добавили виртуальную машину client2, разберём содержимое каталога provisioning:</p>

<pre>ls -l ./provisioning</pre>

<p>Рассмотрим требуемые нам файлы:<br />
● playbook.yml — это Ansible-playbook, в котором содержатся инструкции по настройке нашего стенда<br />
● client-motd — файл, содержимое которого будет появляться перед пользователем, который подключился по SSH<br />
● named.ddns.lab и named.dns.lab — файлы описания зон ddns.lab и dns.lab соответсвенно<br />
● master-named.conf и slave-named.conf — конфигурационные файлы, в которых хранятся настройки DNS-сервера<br />
● client-resolv.conf и servers-resolv.conf — файлы, в которых содержатся IP-адреса DNS-серверов</p>

<ul>Рассмотрим требуемые нам файлы:
<li>playbook.yml — это Ansible-playbook, в котором содержатся инструкции по настройке нашего стенда</li>
<li>client-motd — файл, содержимое которого будет появляться перед пользователем, который подключился по SSH</li>
<li>named.ddns.lab и named.dns.lab — файлы описания зон ddns.lab и dns.lab соответсвенно</li>
<li>master-named.conf и slave-named.conf — конфигурационные файлы, в которых хранятся настройки DNS-сервера</li>
<li>client-resolv.conf и servers-resolv.conf — файлы, в которых содержатся IP-адреса DNS-серверов</li></ul>

<p>содержимое файла playbook.yml:</p>

<pre>---
- hosts: all
  become: yes
  tasks:

#Установка пакетов bind, bind-utils и ntp
  - name: install packages
    yum: name={{ item }} state=latest
    with_items:
    - bind
    - bind-utils
    - ntp

#Копирование файла named.zonetransfer.key на хосты с правами 0644
#Владелец файла — root, група файла — named</pre>
  - name: copy transferkey to all servers and the client
    copy: src=named.zonetransfer.key dest=/etc/named.zonetransfer.key owner=root group=named mode=0644

#Настройка хоста ns01
- hosts: ns01
  become: yes
  tasks:

#Копирование конфигурации DNS-сервера
  - name: copy named.conf
    copy: src=master-named.conf dest=/etc/named.conf owner=root group=named mode=0640

#Копирование файлов с настроками зоны.
#Будут скопированы все файлы, в имя которых начинается на «named.d»
  - name: copy zones
    copy: src={{ item }} dest=/etc/named/ owner=root group=named mode=0660
    with_fileglob:
    - named.d*

#Копирование файла resolv.conf
  - name: copy resolv.conf to the servers
    copy: src=servers-resolv.conf dest=/etc/resolv.conf owner=root group=root mode=0644

#Изменение прав каталога /etc/named
#Права 670, владелец — root, группа — named
  - name: set /etc/named permissions
    file: path=/etc/named owner=root group=named mode=0670

#Перезапуск службы Named и добавление её в автозагрузку
  - name: ensure named is running and enabled
    service: name=named state=restarted enabled=yes

- hosts: ns02
  become: yes
  tasks:
  - name: copy named.conf
    copy: src=slave-named.conf dest=/etc/named.conf owner=root group=named mode=0640

  - name: copy resolv.conf to the servers
    copy: src=servers-resolv.conf dest=/etc/resolv.conf owner=root group=root mode=0644

  - name: set /etc/named permissions
    file: path=/etc/named owner=root group=named mode=0670

  - name: ensure named is running and enabled
    service: name=named state=restarted enabled=yes

- hosts: client
  become: yes
  tasks:
  - name: copy resolv.conf to the client
    copy: src=client-resolv.conf dest=/etc/resolv.conf owner=root group=root mode=0644

#Копирование конфигруационного файла rndc
  - name: copy rndc conf file
    copy: src=rndc.conf dest=/home/vagrant/rndc.conf owner=vagrant group=vagrant mode=0644

#Настройка сообщения при входе на сервер
  - name: copy motd to the client
    copy: src=client-motd dest=/etc/motd owner=root group=root mode=0644</pre>

<p>Так как мы добавили ещё одну виртуальную машину (client2), нам потребуется её настроить. Так как настройки будут совпадать с ВМ client, то мы просто добавим хост в модуль по настройке клиента:</p>

<pre>- hosts: client,client2
  become: yes
  tasks:
  - name: copy resolv.conf to the client
    copy: src=client-resolv.conf dest=/etc/resolv.conf owner=root group=root mode=0644

#Копирование конфигруационного файла rndc
  - name: copy rndc conf file
    copy: src=rndc.conf dest=/home/vagrant/rndc.conf owner=vagrant group=vagrant mode=0644

#Настройка сообщения при входе на сервер
  - name: copy motd to the client
    copy: src=client-motd dest=/etc/motd owner=root group=root mode=0644</pre>






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
