# repositoriotlx
Primer repositoriotlx para obligatorio
Playbook Instalación Mariadb2
---
- name: Configuro servidor base de datos en ubuntu
  hosts: srv1ubuntu
  become: true
  user: sysadmin

  tasks:
  #Instalación de firewall
  - name: UFW instalado
    ansible.builtin.apt:
      name: ufw
      state: present
  #Abrimos puerto 22 para conexiones
  - name: Permitir puerto 22 en ufw
    community.general.ufw:
      rule: allow
      name: OpenSSH
  #Definición de políticas entrantes
  - name: Defino politicas de tráfico entrante
    community.general.ufw:
      policy: allow
      direction: outgoing
      state: enabled
  
  - name: Defino politicas de tráfico entrante
    community.general.ufw:
      policy: deny
      direction: incoming
      state: enabled
  #Inicamos servicio firewall
  - name: servicio UFW levantado y activo
    ansible.builtin.systemd_service:
      name: ufw
      state: started
      enabled: true

## Instalación de base de datos
  - name: MariaDB instalado
    ansible.builtin.apt:
      name: "{{ item }}"
      state: present
      update-cache: true
    loop:
      - mariadb-server
      - mariadb-client
      - python3-pymysql

  - name: Cambiar la configuracion para escuchar en todas las interfaces
    ansible.builtin.lineinfile:
      path: /etc/mysql/mariadb.conf.d/50-server.cnf
      regexp: '^bind-address'
      line: 'bind-address         = 0.0.0.0'
    notify: Restart mariadb

  - name: Ejecuto el handler si cambió la configuración
    meta: flush_handlers
  
  - name: Servidor Mariadb levantado
    ansible.builtin.systemd_service: 
      name: mariadb
      state: started
      enabled: true

  - name: Habilitamos en ufw la conexión a mariadb
    community.general.ufw:
      rule: allow
      port: '3306'
      protocol: tcp
      direction: in

##  Configuracion de la base de datos de la aplicación

  - name: Copio el dump de la base de datos
    ansible.builtin.copy:
      src: todo.sql
      dest: /tmp/todo.sql

  - name: Creo la base de datos todo
    community.mysql.mysql_db:
#      check_implicit_admin: true
#      login_host: localhost 
      login_unix_socket: /run/mysqld/mysqld.sock
#      login_user: root
#      login_password: dbadmin
      name: todo
      state: import
      target: /tmp/todo.sql

  handlers:
  #Reseteo de la base ante cambios
  - name: Restart mariadb
    ansible.builtin.systemd_service: 
      name: mariadb
      state: restarted

Para ejecutar la instalación de este Playbook 
ansible-playbook -i hosts.ini install_mariadb2.yml --verbose --ask-become-pass

Playbook Instalacion de Tomcat
---
- name: Install Tomcat for ToDo application on CentOS
  hosts: srv2centos
  become: yes
  #Rut ade instalación a traves de variable
  vars:
    tomcat_url: "https://archive.apache.org/dist/tomcat/tomcat-9/v9.0.58/bin/apache-tomcat-9.0.58.tar.gz"

  tasks:
    #Instalación de tar
    - name: Install tar
      ansible.builtin.dnf:
        name: tar
        state: present
    #Descarga desde la url
    - name: Download Tomcat package
      ansible.builtin.get_url:
        url: "{{ tomcat_url }}"
        dest: /opt/apache-tomcat.tar.gz
    #Copia de paquetes de instalación
    - name: Extract Tomcat package
      ansible.builtin.unarchive:
        src: /opt/apache-tomcat.tar.gz
        dest: /opt
        remote_src: yes
    #Proporciona un nombre descriptivo para la tarea, indicando que se está estableciendo una variable para el directorio de Tomcat.
    - name: Set Tomcat directory variable
      ansible.builtin.set_fact:
        dir_tomcat: "/opt/{{ tomcat_url.split('/')[-1].replace('.tar.gz', '') }}"
    #Proporciona un nombre descriptivo a la tarea, indicando que se va a crear un enlace simbólico a la instalación de Tomcat.
    - name: Create symlink to Tomcat
      ansible.builtin.file:
        src: "{{ dir_tomcat }}"
        dest: /opt/tomcat
        state: link
        force: yes

Para ejecutar la instalación de este Playbook 
ansible-playbook -i hosts.ini tomcat.yml --verbose --ask-become-pass -vvvv

Playbook para la instalación webser
---
- name: Instalar y configurar un webserver
  hosts: srv2centos
  become: yes

  tasks:
    - name: Instalar apache
      ansible.builtin.dnf:
        name: httpd
        state: present

    - name: Configurar virtualhost
      ansible.builtin.copy:
        src: ./files/virtualhost.conf
        dest: /etc/httpd/conf.d/virtualhost.conf
      notify: Reiniciar apache
    #Asigna propiedad de directorio y da permisos en el mismo
    - name: Crear directorio document root
      ansible.builtin.file:
        path: /var/www/ejemplo/html
        state: directory
        mode: '0755'
        owner: apache
        group: apache
    #Copia de la pagina en el directorio de descarga y asigna permisos 
    - name: Copiar pagina indice del sitio
      ansible.builtin.template:
        src: ./templates/index.j2
        dest: /var/www/ejemplo/html/index.html
        owner: apache
        group: apache
        mode: '0644'
    #Lista los protocolos a utilizar y permite las conexiones en el firewall
    - name: Permitir conexiones HTTP y HTTPS
      ansible.posix.firewalld:
        service: "{{ item }}"
        state: enabled
        immediate: yes
        permanent: yes
      loop:
        - http
        - https
    #Levanta y deja activo el servidor
    - name: Apache levantado y habilitado
      ansible.builtin.service:
        name: httpd
        state: started
        enabled: true

  handlers:
    #En caso de cambios se reinicia el servidor
    - name: Reiniciar apache
      ansible.builtin.service:
        name: httpd
        state: restarted

Para ejecutar la instalación de este Playbook 
ansible-playbook -i hosts.ini webserver.yml --verbose --ask-become-pass -vvvv


Playbook para la instalación de Java 

- name: Install Java on Linux
  hosts: srv2centos
  become: yes
  tasks:
    - name: Install Java
      yum:
        name: java-1.8.0-openjdk
        state: present

Para la ejecución de este playbook
ansible-playbook -i hosts.ini java.yml --verbose --ask-become-pass -vvvv
