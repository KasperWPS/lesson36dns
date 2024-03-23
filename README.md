# Домашнее задание № 22 по теме: "Мосты, туннели и VPN". К курсу Administrator Linux. Professional

## Задание

- Изменить впредоставленный стенд
 - Добавить client2
 - Добавить имена:
   - web1 - ассоциирован с хостом client
   - web2 - ассоциирован с хостом client2
 - Добавить зону newdns.lab
   - Добавить имя www, которое ассоциировано с обоими хостами - client и client2

- Настроить split-dns, где
 - client - видит обе зоны, но в зоне dns.lab только web1
 - client2 - видит только dns.lab

- \* SELinux не отключать

### Выполнение

- Виртуальные машины разворачиваются и настраиваются автоматически с применением provisioner Ansible
- Проверка работы:

```bash
sestatus
```
```
SELinux status:                 enabled
SELinuxfs mount:                /sys/fs/selinux
SELinux root directory:         /etc/selinux
Loaded policy name:             targeted
Current mode:                   enforcing
Mode from config file:          enforcing
Policy MLS status:              enabled
Policy deny_unknown status:     allowed
Max kernel policy version:      31
```

**clietn**

```bash
vagrant ssh client -c 'ping -c1 www.newdns.lab'
```
```
PING www.newdns.lab (192.168.56.15) 56(84) bytes of data.
64 bytes from client (192.168.56.15): icmp_seq=1 ttl=64 time=0.006 ms

--- www.newdns.lab ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.006/0.006/0.006/0.000 ms
```

```bash
vagrant ssh client -c 'ping -c1 web1.dns.lab'
```
```
PING web1.dns.lab (192.168.56.15) 56(84) bytes of data.
64 bytes from client (192.168.56.15): icmp_seq=1 ttl=64 time=0.008 ms

--- web1.dns.lab ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.008/0.008/0.008/0.000 ms
```

```bash
vagrant ssh client -c 'ping -c1 web2.dns.lab'
```
```
ping: web2.dns.lab: Name or service not known
```

**client2**

```bash
vagrant ssh client2 -c 'ping -c1 www.newdns.lab'
```
```
ping: www.newdns.lab: Name or service not known
```

```bash
vagrant ssh client2 -c 'ping -c1 web1.dns.lab'
```
```
PING web1.dns.lab (192.168.56.15) 56(84) bytes of data.
64 bytes from 192.168.56.15 (192.168.56.15): icmp_seq=1 ttl=64 time=0.343 ms

--- web1.dns.lab ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.343/0.343/0.343/0.000 ms
```

```bash
vagrant ssh client2 -c 'ping -c1 web2.dns.lab'
```
```
PING web2.dns.lab (192.168.56.16) 56(84) bytes of data.
64 bytes from client2 (192.168.56.16): icmp_seq=1 ttl=64 time=0.007 ms

--- web2.dns.lab ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.007/0.007/0.007/0.000 ms
```

Конфигурация стенда (config.json)
```json
[
  {
    "name": "ns01",
    "cpus": 1,
    "gui": false,
    "box": "centos/7",
    "private_network":
    [
      { "ip": "192.168.56.10", "adapter": 2, "netmask": "255.255.255.0" }
    ],
    "memory": "640",
    "no_share": true
  },
  {
    "name": "ns02",
    "cpus": 1,
    "gui": false,
    "box": "centos/7",
    "private_network":
    [
      { "ip": "192.168.56.11", "adapter": 2, "netmask": "255.255.255.0" }
    ],
    "memory": "640",
    "no_share": true
  },
  {
    "name": "client",
    "cpus": 1,
    "gui": false,
    "box": "centos/7",
    "private_network":
    [
      { "ip": "192.168.56.15", "adapter": 2, "netmask": "255.255.255.0" }
    ],
    "memory": "640",
    "no_share": true
  },
  {
    "name": "client2",
    "cpus": 1,
    "gui": false,
    "box": "centos/7",
    "private_network":
    [
      { "ip": "192.168.56.16", "adapter": 2, "netmask": "255.255.255.0" }
    ],
    "memory": "640",
    "no_share": true
  }
]
```

Playbook
```yaml
---
- hosts: all
  become: yes
  tasks:

  - name: Install epel-release
    ansible.builtin.yum:
      name: epel-release
      state: present

  - name: Install packages
    ansible.builtin.yum:
      name:
      - bind
      - bind-utils
      - vim
      state: latest

  - name: Set timezone
    community.general.timezone:
      name: Europe/Moscow
    notify:
      - Restart Chrony service

  - name: Accept login with password from sshd
    ansible.builtin.lineinfile:
      path: /etc/ssh/sshd_config
      regexp: '^PasswordAuthentication no$'
      line: 'PasswordAuthentication yes'
      state: present
    notify:
      - Restart sshd

  - name: Copy transferkey to all servers and clients
    ansible.builtin.copy:
      src: named.zonetransfer.key
      dest: /etc/named.zonetransfer.key
      owner: root
      group: named
      mode: '0644'

  - name: Copy resolv.conf to the servers
    ansible.builtin.template:
      src: servers-resolv.conf.j2
      dest: /etc/resolv.conf
      owner: root
      group: root
      mode: '0644'
    when: (ansible_hostname == 'ns01' or ansible_hostname == 'ns02')

  handlers:

  - name: Restart Chrony service
    ansible.builtin.service:
      name: chronyd
      state: restarted

  - name: Restart sshd
    ansible.builtin.service:
      name: sshd
      state: restarted
      enabled: true


- hosts: ns01
  become: yes
  tasks:

  - name: Copy named.conf
    ansible.builtin.copy:
      src: files/master-named.conf
      dest: /etc/named.conf
      owner: root
      group: named
      mode: '0640'
    notify: Restart named

  - name: Copy zones
    ansible.builtin.copy:
      src: "{{ item }}"
      dest: /etc/named/
      owner: root
      group: named
      mode: '0660'
    with_fileglob:
      - 'named.d*'
      - named.newdns.lab
    notify: Restart named

  - name: Set /etc/named permissions
    ansible.builtin.file:
      path: /etc/named
      owner: root
      group: named
      mode: '0670'

  handlers:

  - name: Restart named
    service:
      name: named
      state: restarted
      enabled: yes


- hosts: ns02
  become: yes
  tasks:

  - name: Copy named.conf
    ansible.builtin.copy:
      src: files/slave-named.conf
      dest: /etc/named.conf
      owner: root
      group: named
      mode: '0640'
    notify: Restart named

  - name: Set /etc/named permissions
    ansible.builtin.file:
      path: /etc/named
      owner: root
      group: named
      mode: '0670'

  handlers:

  - name: Restart named
    service:
      name: named
      state: restarted
      enabled: yes

- hosts: client,client2
  become: yes
  tasks:
  - name: Copy resolv.conf to the client
    copy:
      src: files/client-resolv.conf
      dest: /etc/resolv.conf
      owner: root
      group: root
      mode: '0644'

  - name: Copy rndc conf file
    copy:
      src: files/rndc.conf
      dest: /home/vagrant/rndc.conf
      owner: vagrant
      group: vagrant
      mode: '0644'

  - name: Copy motd to the client
    copy:
      src: files/client-motd
      dest: /etc/motd
      owner: root
      group: root
      mode: '0644'
```

