<h3>### DNS ###</h3>

<p>настраиваем split-dns</p>

<h4>Описание домашнего задания</h4>

<ul>
<li>Между двумя виртуалками поднять vpn в режимах<br />
● tun;<br />
● tap;<br />
Прочувствовать разницу.</li>



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

<h4>1. TUN/TAP режимы VPN</h4>

<p>В домашней директории создадим директорию vpn, в котором будут храниться настройки виртуальных машин:</p>

<pre>[user@localhost otus]$ mkdir ./vpn
[user@localhost otus]$</pre>

<p>Перейдём в директорию vpn:</p>

<pre>[user@localhost otus]$ cd ./vpn/
[user@localhost vpn]$</pre>

<p>Создадим файл Vagrantfile:</p>

<pre>[user@localhost vpn]$ vi ./Vagrantfile</pre>

<p>1. Заполним следующим содержимым:</p>

<pre># -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure(2) do |config|
  config.vm.box = "centos/7"
  config.vm.define "server" do |server|
    server.vm.hostname = "server.loc"
    server.vm.network "private_network", ip: "192.168.10.10"
  end
  config.vm.define "client" do |client|
    client.vm.hostname = "client.loc"
    client.vm.network "private_network", ip: "192.168.10.20"
  end
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
