#README.MD
Obligatorio del Taller de Linux 2021 -Ansible
Este proyecto se basa la instalación de LAMP SIMPLE Y NTP (CHRONY) utilizando Roles.
Contenido
Este proyecto está compuesto por 3 ramas:
1-Variables 
  •	group_vars:  Contiene dos ficheros:
	all: variables que contienen los paquetes necesarios para instalar servicios.
# Variables listadas para que apliquen al grupo all 
packages:
- apache2
- php
- php-mysql
- python3-selinux

packages_centos:
- httpd
- php
- php-mysql
- python3-libsemanage
- python3-libselinux
-
# dbservers: variables que contienen los paquetes necesarios para instalar el servicio mariadb
#Paquetes para mariadb Centos
mariadb_packages:
- mariadb-common
- mariadb-server
- python3-PyMySQL
- python3-libsemanage
- python3-libselinux
#Paquetes para mariadb ubuntu
mariadb_packages_ubuntu:
- mariadb-common
- mariadb-server
- python3-PyMySQL
#Iptables centos
mysqlservice: mysqld
mysql_port: '3306'
#Aplication db
dbuser: foouser
dbname: foodb
upassword: 'abc'
2-Roles
Common: Se encarga de instalar el servidor NTP-Chrony
•	handlers: Contiene el fichero mail.yml, que permite reiniciar el servicio
- name: restart_chrony_centos
  service:
    name: chronyd
    state: restarted

- name: restart_chrony_ubuntu
  service: 
    name: chrony
    state: restarted
•	tasks: Se encarga de instalar, configurar e iniciar el servicio, en Centos y Ubuntu.
- name: Install chrony Ubuntu
  apt:
    name: chrony
    state: present
  when: ansible_distribution == 'Ubuntu'
   

- name: install chrony centos
  yum:
    name: chrony
    state: present
  when: ansible_distribution == 'CentOS'
……

•	templates: Plantilla con la información, mediante el archivo  chrony.conf.j2
# Specify file containing keys for NTP authentication.
keyfile /etc/chrony.keys

# Get TAI-UTC offset and leap seconds from the system tz database.
leapsectz right/UTC

# Specify directory for log files.
logdir /var/log/chrony

•	vars: Contiene la variable de ntp_server
ntp_server: uy.pool.ntp.org
Db: Se encarga de Instalar y configurar el servidor de base de datos MariaDB
•	handlers: Se encarga de reiniciar el servicio mariadb, mediante mail.yml.
# Handler to handle DB tier notifications

- name: restart mariadb
  service:
    name: mariadb
    state: restarted

- name: restart iptables
  service:
    name: iptables
    state: restarted
•	tasks: Se encarga de instalar, configurar e iniciar el servicio en Centos.
- name: Install Mariadb package Centos
  yum:
    name: "{{ item }}"
    state: latest
  loop: "{{ mariadb_packages }}"
  when: ansible_distribution == "CentOS"

- name: Install Mariadb package Ubuntu
  apt:
    state: latest
    loop: "{{ mariadb_packages_ubuntu }}" 
  when: ansible_distribution == "Ubuntu"
……..
•	templates: Plantilla con la configuración del servicio.
[mysqld]
datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock
user=mysql
# Disabling symbolic-links is recommended to prevent assorted security risks
symbolic-links=0
port={{ mysql_port }}

[mysqld_safe]
log-error=/var/log/mysqld.log
pid-file=/var/run/mysqld/mysqld.pid
Web: Se encarga de Instalar y configurar el servidor Web 
•	handlers: Se encarga de reiniciar el servicio web, mediante mail.yml.
- name: restart iptables
  service:
    name: iptables
    state: restarted

- name: restart apache
  service: 
    name: apache2
    state: restarted
  
- name: restart httpd
  service:
    name: httpd
    state: resarted
•	tasks: Se encarga de instalar, configurar e iniciar el servicio, en Ubuntu
copy_code:
- name: Creates the inde.php file Ubuntu
  template:
    src: index.php.j2
    dest: /var/www/html/index.php
    owner: root
    group: root
  notify: restart apache   
  when: ansible_distribution == "Ubuntu"

- name: Creates the index.php file Centos
  template:
    src: inde.php.j2
    dest: /var/www/html/index.php
    owner: root
    group: root
  notify: restart httpd
  when: ansible_distribution == "CentOS"
	install_httpd:
- name: Install http and php etc Centos
  yum:
    name: "{{ item }}"
    state: present
  loop: "{{ packages_centos }}" 
  when: ansible_distribution == "CenOS" 

- name: install apache Ubuntu
  apt: 
    name: "{{ item }}"
    state: latest
  loop: "{{ packages }}"
  when: ansible_distribution == "Ubuntu"

- name: Insert rule httpd ufw
  ufw:
    rule: allow
    port: '80'
    proto: tcp    
  when: ansible_distribution == "Ubuntu"
	main.yml: 
- include: install_httpd.yml
- include: copy_code.yml

•	templates: Plantilla con el sitio de prueba.
Inde.php.j2:
<html>
 <head>
  <title> Prueba de  PHP </title>
 </head>
 <body>
 <?phpecho '<p>Hola Mundo</p>';?>
 </body>
</html>
 

3- Archivos de configuración:
•	ansible.cfg: archivo de configuración de ansible, donde se encuentran la dirección de los directorios.
inventory      = ./hosts
#library        = /usr/share/my_modules/
#module_utils   = /usr/share/my_module_utils/
#remote_tmp     = ~/.ansible/tmp
#local_tmp      = ~/.ansible/tmp
#plugin_filters_cfg = /etc/ansible/plugin_filters.yml
#forks          = 5
#poll_interval  = 15
#sudo_user      = root

•	host:  archivo con la ip y nombre de los servidores, donde se instalarán los servicios.
[webservers]
ubuntu1 ansible_host=192.168.56.104

[dbservers]
centos1 ansible_host=192.168.56.103

[all]
ubuntu1
centos1
•	site.yml: contiene el playbook con las ordenes de instalación de los servicios a los servidores, mediante los roles.
# This playbook deploys the whole application stack in this site.

- name: apply common configuration to all nodes
  become: yes
  hosts: all
  remote_user: ansible

  roles:
   - common

- name: configure and deploy the webservers and application code
  become: yes
  hosts: webservers
  remote_user: ansible

  roles:
   - web
- name: deploy MySQL and configure the databases
  become: yes
  hosts: dbservers
  remote_user: ansible

  roles:
   - db
Cómo clonar
git clone https://github.com/TomasBettanin/obligatorio202102
Instalación
Para instalar y ejecutar los playbooks se deben correr los siguientes comandos
ansible-playbook --syntax-check site.yml
ansible-playbook -C site.yml --ask-become-pass (Simula)
ansible-playbook site.yml --ask-become-pass

