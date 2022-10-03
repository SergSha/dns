<h3>### DNS ###</h3>

<p>настраиваем split-dns</p>

<h4>Описание домашнего задания</h4>

<ol>
<li>взять стенд https://github.com/erlong15/vagrant-bind<br />
добавить еще один сервер client2<br />
завести в зоне dns.lab имена:
<ul type="disc">
<li>web1 - смотрит на клиент1</li>
<li>web2 смотрит на клиент2</li>
</ul>
завести еще одну зону newdns.lab<br />
завести в ней запись
<ul type="disc">
<li>www - смотрит на обоих клиентов</li>
</ul></li>

<li>настроить split-dns
<ul type="disc">
<li>клиент1 - видит обе зоны, но в зоне dns.lab только web1</li>
<li>клиент2 видит только dns.lab</li>
</ul>
</ol>

<p>Дополнительное задание<br />
* настроить все без выключения selinux</p>

<p>Формат сдачи ДЗ - vagrant + ansible</p>


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

<pre>[user@localhost dns]$ vi ./Vagrantfile</pre>

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

<ul>Рассмотрим требуемые нам файлы:
<li>playbook.yml — это Ansible-playbook, в котором содержатся инструкции по настройке нашего стенда</li>
<li>client-motd — файл, содержимое которого будет появляться перед пользователем, который подключился по SSH</li>
<li>named.ddns.lab и named.dns.lab — файлы описания зон ddns.lab и dns.lab соответсвенно</li>
<li>master-named.conf и slave-named.conf — конфигурационные файлы, в которых хранятся настройки DNS-сервера</li>
<li>client-resolv.conf и servers-resolv.conf — файлы, в которых содержатся IP-адреса DNS-серверов</li></ul>

<p>Содержимое файла playbook.yml:</p>

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
#Владелец файла — root, група файла — named
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

<pre>[user@localhost dns]$ vagrant up</pre>

<p>Проверим состояние созданной и запущенной машины:</p>

<pre>[user@localhost dns]$ vagrant status
Current machine states:

ns01                      running (virtualbox)
ns02                      running (virtualbox)
client                    running (virtualbox)
client2                   running (virtualbox)

This environment represents multiple VMs. The VMs are all listed
above with their current state. For more information about a specific
VM, run `vagrant status NAME`.
[user@localhost dns]$</pre>

<p>Перед выполнением следующих заданий, нужно обратить внимаение, на каком адресе и порту работают наши DNS-сервера.</p>

<p>Проверить это можно двумя способами:<br />
● Посмотреть с помощью команды SS:</p>

<p>2. Заходим на ВМ server и зайдём под пользователем root:</p>

<pre>[user@localhost dns]$ vagrant ssh ns01
[vagrant@ns01 ~]$ sudo -i
[root@ns01 ~]# </pre>

<pre>[root@ns01 ~]# <b>ss -tulpn</b>
Netid  State      Recv-Q Send-Q Local Address:Port               Peer Address:Po                                                                                                                                   rt
udp    UNCONN     0      0         *:969                   *:*                                                                                                                                                      users:(("rpcbind",pid=400,fd=7))
udp    UNCONN     0      0      192.168.50.10:53                    *:*                                                                                                                                                      users:(("named",pid=4423,fd=512))
udp    UNCONN     0      0      127.0.0.1:323                   *:*                                                                                                                                                      users:(("chronyd",pid=3468,fd=5))
udp    UNCONN     0      0         *:68                    *:*                                                                                                                                                      users:(("dhclient",pid=2364,fd=6))
udp    UNCONN     0      0         *:111                   *:*                                                                                                                                                      users:(("rpcbind",pid=400,fd=6))
udp    UNCONN     0      0      [::]:969                [::]:*                                                                                                                                                      users:(("rpcbind",pid=400,fd=10))
udp    UNCONN     0      0         [::1]:53                 [::]:*                                                                                                                                                      users:(("named",pid=4423,fd=513))
udp    UNCONN     0      0         [::1]:323                [::]:*                                                                                                                                                      users:(("chronyd",pid=3468,fd=6))
udp    UNCONN     0      0      [::]:111                [::]:*                                                                                                                                                      users:(("rpcbind",pid=400,fd=9))
tcp    LISTEN     0      128       *:111                   *:*                                                                                                                                                      users:(("rpcbind",pid=400,fd=8))
<b>tcp    LISTEN     0      10     192.168.50.10:53                    *:*                                                                                                                                                      users:(("named",pid=4423,fd=21))</b>
tcp    LISTEN     0      128       *:22                    *:*                                                                                                                                                      users:(("sshd",pid=611,fd=3))
tcp    LISTEN     0      128    192.168.50.10:953                   *:*                                                                                                                                                      users:(("named",pid=4423,fd=23))
tcp    LISTEN     0      100    127.0.0.1:25                    *:*                                                                                                                                                      users:(("master",pid=700,fd=13))
tcp    LISTEN     0      128    [::]:111                [::]:*                                                                                                                                                      users:(("rpcbind",pid=400,fd=11))
tcp    LISTEN     0      10        [::1]:53                 [::]:*                                                                                                                                                      users:(("named",pid=4423,fd=22))
tcp    LISTEN     0      128    [::]:22                 [::]:*                                                                                                                                                      users:(("sshd",pid=611,fd=4))
tcp    LISTEN     0      100       [::1]:25                 [::]:*                                                                                                                                                      users:(("master",pid=700,fd=14))
[root@ns01 ~]#</pre>


<p>● Посмотреть информацию в настройках DNS-сервера (/etc/named.conf)<br />
✔ На хосте ns01:</p>

<pre>[root@ns01 ~]# cat /etc/named.conf
...

    // network
        <b>listen-on port 53 { 192.168.50.10; };</b>
        listen-on-v6 port 53 { ::1; };
...
[root@ns01 ~]#</pre>

<p>✔ На хосте ns02:</p>

<pre>// network
listen-on port 53 { 192.168.50.11; };
listen-on-v6 port 53 { ::1; };</pre>



<p>Аналогично на сервере ns02:</p>

<pre>[student@pv-homeworks1-10 dns]$ vagrant ssh ns02
Last login: Fri Sep 30 10:04:26 2022 from 10.0.2.2
[vagrant@ns02 ~]$ sudo -i
[root@ns02 ~]# ss -tulpn
Netid State      Recv-Q Send-Q                                                          Local Address:Port                                                                         Peer Address:Port
udp   UNCONN     0      0                                                                   127.0.0.1:323                                                                                     *:*                   users:(("chronyd",pid=3467,fd=5))
udp   UNCONN     0      0                                                                           *:68                                                                                      *:*                   users:(("dhclient",pid=2359,fd=6))
udp   UNCONN     0      0                                                                           *:111                                                                                     *:*                   users:(("rpcbind",pid=387,fd=6))
udp   UNCONN     0      0                                                                           *:975                                                                                     *:*                   users:(("rpcbind",pid=387,fd=7))
udp   UNCONN     0      0                                                               192.168.50.11:53                                                                                      *:*                   users:(("named",pid=4079,fd=512))
udp   UNCONN     0      0                                                                       [::1]:323                                                                                  [::]:*                   users:(("chronyd",pid=3467,fd=6))
udp   UNCONN     0      0                                                                        [::]:111                                                                                  [::]:*                   users:(("rpcbind",pid=387,fd=9))
udp   UNCONN     0      0                                                                        [::]:975                                                                                  [::]:*                   users:(("rpcbind",pid=387,fd=10))
udp   UNCONN     0      0                                                                       [::1]:53                                                                                   [::]:*                   users:(("named",pid=4079,fd=513))
<b>tcp   LISTEN     0      128                                                             192.168.50.11:953                                                                                     *:*                   users:(("named",pid=4079,fd=23))</b>
tcp   LISTEN     0      100                                                                 127.0.0.1:25                                                                                      *:*                   users:(("master",pid=875,fd=13))
tcp   LISTEN     0      128                                                                         *:111                                                                                     *:*                   users:(("rpcbind",pid=387,fd=8))
tcp   LISTEN     0      10                                                              192.168.50.11:53                                                                                      *:*                   users:(("named",pid=4079,fd=21))
tcp   LISTEN     0      128                                                                         *:22                                                                                      *:*                   users:(("sshd",pid=663,fd=3))
tcp   LISTEN     0      100                                                                     [::1]:25                                                                                   [::]:*                   users:(("master",pid=875,fd=14))
tcp   LISTEN     0      128                                                                      [::]:111                                                                                  [::]:*                   users:(("rpcbind",pid=387,fd=11))
tcp   LISTEN     0      10                                                                      [::1]:53                                                                                   [::]:*                   users:(("named",pid=4079,fd=22))
tcp   LISTEN     0      128                                                                      [::]:22                                                                                   [::]:*                   users:(("sshd",pid=663,fd=4))
[root@ns02 ~]#</pre>

<pre>[root@ns02 ~]# cat /etc/named.conf
...
    // network
        listen-on port 53 { 192.168.50.11; };
        listen-on-v6 port 53 { ::1; };
...
[root@ns02 ~]#</pre>

<p>Исходя из данной информации, нам нужно подкорректировать файл /etc/resolv.conf для DNS-серверов: на хосте ns01 указать nameserver 192.168.50.10, а на хосте ns02 — 192.168.50.11</p>

<pre>[root@<b>ns01</b> ~]# vi /etc/resolv.conf
domain dns.lab
search dns.lab
nameserver 192.168.50.10</pre>

<pre>[root@<b>ns02</b> ~]# vi /etc/resolv.conf
domain dns.lab
search dns.lab
nameserver 192.168.50.11</pre>

<p>В Ansible для этого воспользуемся шаблоном с Jinja. Изменим имя файла servers-resolv.conf на servers-resolv.conf.j2 и укажем там следующие условия:</p>

<pre>domain dns.lab
search dns.lab
#Если имя сервера ns02, то указываем nameserver 192.168.50.11
{% if ansible_hostname == 'ns02' %}
nameserver 192.168.50.11
{% endif %}
#Если имя сервера ns01, то указываем nameserver 192.168.50.10
{% if ansible_hostname == 'ns01' %}
nameserver 192.168.50.10
{% endif %}</pre>

<p>После внесение измений в файл, внесём измения в ansible-playbook:<br />
Используем вместо модуля copy модуль template:</p>

<pre>- name: copy resolv.conf to the servers
template: src=servers-resolv.conf.j2 dest=/etc/resolv.conf owner=root
group=root mode=0644</pre>

<h4>Добавление имён в зону dns.lab</h4>

<p>Проверим, что зона dns.lab уже существует на DNS-серверах:<br />
Фрагмент файла /etc/named.conf на сервере ns01:</p>

<pre>// Имя зоны
zone "dns.lab" {
  type master;
  // Тем, у кого есть ключ zonetransfer.key можно получать копию файла зоны
  allow-transfer { key "zonetransfer.key"; };
  // Файл с настройками зоны
  file "/etc/named/named.dns.lab";
};</pre>

<p>Похожий фрагмент файла /etc/named.conf находится на slave-сервере ns02:</p>

<pre>// Имя зоны
zone "dns.lab" {
  type slave;
  // Адрес мастера, куда будет обращаться slave-сервер
  masters { 192.168.50.10; };
};</pre>

<p>Также на хосте ns01 мы видим файл /etc/named/named.dns.lab с настройкой зоны:</p>

<pre>$TTL 3600
$ORIGIN dns.lab.
@               IN      SOA     ns01.dns.lab. root.dns.lab. (
                            2711201407 ; serial
                            3600       ; refresh (1 hour)
                            600        ; retry (10 minutes)
                            86400      ; expire (1 day)
                            600        ; minimum (10 minutes)
                        )

                IN      NS      ns01.dns.lab.
                IN      NS      ns02.dns.lab.

; DNS Servers
ns01            IN      A       192.168.50.10
ns02            IN      A       192.168.50.11</pre>

<p>Именно в этот файл нам потребуется добавить имена. Допишем в конец файла следующие строки:</p>

<pre>; Web
web1 IN A 192.168.50.15
web2 IN A 192.168.50.16</pre>

<p>Если изменения внесены вручную, то для применения настроек нужно:<br />
● Перезапустить службу named: systemctl restart named<br />
● Изменить значение Serial (добавить +1 к числу 2711201407), изменение значения serial укажет slave-серверам на то, что были внесены изменения и что им надо обновить свои файлы с зонами.</p>

<p>После внесения изменений, выполним проверку с клиента:</p>

<pre>[root@<b>client</b> ~]# <b>dig @192.168.50.10 web1.dns.lab</b>

; &lt;&lt;&gt;&gt; DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.9 &lt;&lt;&gt;&gt; @192.168.50.10 web1.dns.lab
; (1 server found)
;; global options: +cmd
;; Got answer:
;; -&gt;&gt;HEADER&lt;&lt;- opcode: QUERY, status: NOERROR, id: 19915
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 2, ADDITIONAL: 3

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;web1.dns.lab.                  IN      A

;; ANSWER SECTION:
<b>web1.dns.lab.           3600    IN      A       192.168.50.15</b>

;; AUTHORITY SECTION:
dns.lab.                3600    IN      NS      ns01.dns.lab.
dns.lab.                3600    IN      NS      ns02.dns.lab.

;; ADDITIONAL SECTION:
ns01.dns.lab.           3600    IN      A       192.168.50.10
ns02.dns.lab.           3600    IN      A       192.168.50.11

;; Query time: 6 msec
;; SERVER: 192.168.50.10#53(192.168.50.10)
;; WHEN: Mon Oct 03 13:22:05 UTC 2022
;; MSG SIZE  rcvd: 127

[root@client ~]#</pre>

<pre>dig @192.168.50.11 web2.dns.lab</pre>

<pre>[root@<b>client</b> ~]# <b>dig @192.168.50.11 web2.dns.lab</b>

; &gt;&gt;&gt;&gt; DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.9 &gt;&gt;&gt;&gt; @192.168.50.11 web2.dns.lab
; (1 server found)
;; global options: +cmd
;; Got answer:
;; -&gt;&gt;HEADER&gt;&gt;- opcode: QUERY, status: NOERROR, id: 19040
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 2, ADDITIONAL: 3

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;web2.dns.lab.                  IN      A

;; ANSWER SECTION:
<b>web2.dns.lab.           3600    IN      A       192.168.50.16</b>

;; AUTHORITY SECTION:
dns.lab.                3600    IN      NS      ns02.dns.lab.
dns.lab.                3600    IN      NS      ns01.dns.lab.

;; ADDITIONAL SECTION:
ns01.dns.lab.           3600    IN      A       192.168.50.10
ns02.dns.lab.           3600    IN      A       192.168.50.11

;; Query time: 7 msec
;; SERVER: 192.168.50.11#53(192.168.50.11)
;; WHEN: Mon Oct 03 13:28:13 UTC 2022
;; MSG SIZE  rcvd: 127

[root@client ~]#</pre>

<p>В примерах мы обратились к разным DNS-серверам с разными запросами.</p>

<h4>Добавление имён в зону dns.lab с помощью Ansible</h4>

<p>В существующем Ansible-playbook менять ничего не потребуется. Нам потребуется добавить стоки с новыми именами в файл named.dns.lab и снова запустить Ansible.</p>

<p>Модули, которые внесут данные изменения (переписаны в YAML формате):</p>

<pre>#Настройка хоста ns01
- hosts: ns01
become: yes
tasks:

#Копирование конфигурации DNS-сервера
  - name: copy named.conf
    copy:
      src: master-named.conf
      dest: /etc/named.conf
      owner: root
      group: named
      mode: 0640

#Копирование файлов с настроками зоны.
#Будут скопированы все файлы, в имя которых начинается на «named.d»
  - name: copy zones
    copy:
      src: "{{ item }}"
      dest: /etc/named/
      owner: root
      group: named
      mode: 0660
    loop:
    - named.ddns.lab
    - named.dns.lab
    - named.dns.lab.client
    - named.dns.lab.rev
    - named.newdns.lab

#Перезапуск службы Named и добавление её в автозагрузку
  - name: ensure named is running and enabled
    service:
      name: named
      state: restarted
      enabled: yes</pre>

<h4>Создание новой зоны и добавление в неё записей</h4>

<p>Для того, чтобы прописать на DNS-серверах новую зону нам потребуется:<br />
● На хосте ns01 добавить зону в файл /etc/named.conf:</p>

// lab's newdns zone
zone "newdns.lab" {
  type master;
  allow-transfer { key "zonetransfer.key"; };
  allow-update { key "zonetransfer.key"; };
  file "/etc/named/named.newdns.lab";
};

<p>● На хосте ns02 также добавить зону и указать с какого сервера запрашивать информацию об этой зоне (фрагмент файла /etc/named.conf):</p>

<pre>// lab's newdns zone
zone "newdns.lab" {
  type slave;
  masters { 192.168.50.10; };
  file "/etc/named/named.newdns.lab";
};</pre>

<p>● На хосте ns01 создадим файл /etc/named/named.newdns.lab</p>


$TTL 3600
$ORIGIN dns.lab.
@               IN      SOA     ns01.dns.lab. root.dns.lab. (
                            2711201407 ; serial
                            3600       ; refresh (1 hour)
                            600        ; retry (10 minutes)
                            86400      ; expire (1 day)
                            600        ; minimum (10 minutes)
                        )

                IN      NS      ns01.dns.lab.
                IN      NS      ns02.dns.lab.

; DNS Servers
ns01            IN      A       192.168.50.10
ns02            IN      A       192.168.50.11


<pre>vi /etc/named/named.newdns.lab
$TTL 3600
$ORIGIN newdns.lab.
@               IN      SOA     ns01.dns.lab. root.dns.lab. (
                            2711201007 ; serial
                            3600       ; refresh (1 hour)
                            600        ; retry (10 minutes)
                            86400      ; expire (1 day)
                            600        ; minimum (10 minutes)
                        )

                IN      NS      ns01.dns.lab.
                IN      NS      ns02.dns.lab.

; DNS Servers
ns01            IN      A       192.168.50.10
ns02            IN      A       192.168.50.11

; WWW
www             IN      A       192.168.50.15
www             IN      A       192.168.50.16</pre>

<p>В конце этого файла добавим записи www. У файла должны быть права 660, владелец — root, группа — named.</p>

<p>После внесения данных изменений, изменяем значение serial (добавлем +1 к значению 2711201007) и перезапускаем named:</p>

<pre>systemctl restart named</pre>

<h4>Создание новой зоны и добавление в неё записей с помощью Ansible</h4>

<p>Для создания зоны и добавления в неё записей, добавляем зону в файл /etc/named.conf на хостах ns01 и ns02, а также создаем файл named.newdns.lab, который далее отправим на сервер ns01.</p>

<p>Добавим в модуль copy наш файл named.newdns.lab:</p>

<pre>- name: copy zones
  copy: src={{ item }} dest=/etc/named/ owner=root group=named mode=0660
  loop:
  - named.d*
  - named.newdns.lab</pre>

<p>Соответственно файл named.newdns.lab будет скопирован на хост ns01 по адресу /etc/named/named.newdns.lab</p>

<p>Остальную часть playbook оставим без изменений.</p>

<h4>2. Настройка Split-DNS</h4>





