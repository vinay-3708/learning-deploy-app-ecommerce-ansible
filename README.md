# Introduction


This is <a href=https://github.com/kodekloudhub/learning-app-ecommerce.git>Kodekloud's</a> simple e-commerce application which is going to deploy in httpd webserver and configure to get the product data from the mariadb database using anisble playbook.


# Pre-Requisites to install in the ansible controller

1. Install Anisble using the below command or follow the <a href=https://docs.ansible.com>official_docs</a>.

```
sudo yum install epel-release -y
sudo yum install ansible -y
```

2. Install all required modules mentioned below. To check the modules installed in the server, use following command:


```
anisble-doc --list
```

If you want to install any module, use the following command:


```
ansible-galaxy install <MODULE_NAME>
```

3. Ensure to disable the host_key_checking in /etc/ansible/ansible.cfg or make sure to login atleast once into the remote server using ssh commands before running the ansible commands.


# Ansible modules used

1. yum package manager.
2. service module.
3. ansible.posix.firewalld module.
4. command and shell module.
5. local_action module.
6. community.mysql module.
7. ansible.builtin.git module.


# Ansible Playbook Explanation:

I have configured the 11 tasks as one play. 

In this, The first task is to install the required packages using yum package manager with help of looping.


```
    - name: 'Install FIrewallD & MariaDB & HTTPD & GIT & php & phpy-mysql'
      become: true
      yum:
        name: '{{ item }}'
        state: present
      with_items:
        - git
        - firewalld
        - mariadb
        - httpd
        - php
        - php-mysqlnd
```

In the second task, I have started the firewalld, mariadb & httpd services.

```
- name: 'Start all services'
  become: true
  service:
    name: '{{ item }}'
    state: started
  with_items:
    - firewalld
    - mariadb
    - httpd
```

In the third task, I have added the 3306 & 80 ports to allow the traffic. In this module, the attirbute 'immediate' will make the rules enabled instantly, if it's value is yes.


```
  - name: 'Configure FirewallD for DB & httpd'
    become: yes
    ansible.posix.firewalld:
        permanent: yes
        immediate: yes
        port: "{{item.port}}/{{item.proto}}"
        state: "{{item.state}}"
        zone: "{{item.zone}}"
    with_items:
        - {port: "3306", proto: "tcp", state: "enabled", zone: "public" }
        - {port: "80", proto: "tcp", state: "enabled", zone: "public" }
```

In the fourth & fifth task, I have used command module, register module to store the stdoutput to see the status of firewalld rules as a log file "firewall-rules.log".


```
- name: 'Check the firewalld rules'
    become: yes
    command: firewall-cmd --list-all
    register: firewall_rules
- name: 'Print Firewall Rules to firewall-rules.txt'
    local_action:
        copy content="{{ firewall_rules.stdout }}"
        dest="/home/mvk-admin/Learning/Ansible/firewall-rules.log"
```

In the sixth & seventh task, I have created a db "ecomdb" and a user "ecomuser". I have used the root privilages to login as root user using the login_ unix_socket attribute.

```
    - name: 'Create a ecomdb in mariadb'
      become: true
      community.mysql.mysql_db:
        name: ecomdb
        state: present
        login_unix_socket: /var/lib/mysql/mysql.sock
    - name: 'Create a DB user ecomuser'
      become: true
      community.mysql.mysql_user:
        name: ecomuser
        password: ecompassword
        priv: '*.*:ALL'
        state: present
        login_unix_socket: /var/lib/mysql/mysql.sock
```

In the Eigth task, Run sql queries to insert the data.


```
    - name: Run several insert queries against db ecomdb in single transaction
      become: yes
      community.mysql.mysql_query:
        login_unix_socket: /var/lib/mysql/mysql.sock
        login_db: ecomdb
        query:
        - USE ecomdb;
        - CREATE TABLE products (id mediumint(8) unsigned NOT NULL auto_increment,Name varchar(255) default NULL,Price varchar(255) default NULL, ImageUrl varchar(255) default NULL,PRIMARY KEY (id)) AUTO_INCREMENT=1;
        - INSERT INTO products (Name,Price,ImageUrl) VALUES ("Laptop","100","c-1.png"),("Drone","200","c-2.png"),("VR","300","c-3.png"),("Tablet","50","c-5.png"),("Watch","90","c-6.png"),("Phone Covers","20","c-7.png"),("Phone","80","c-8.png"),("Laptop","150","c-4.png");
        single_transaction: true
```

In the Ninth & Tenth task, I have cloned the repository from kodekloud's github repo to /var/www/html and replacing the host value to localhost using SED command.

```
    - name: 'CLone Github code'
      become: yes
      ansible.builtin.git:
        repo: 'https://github.com/kodekloudhub/learning-app-ecommerce.git'
        dest: /var/www/html/
        single_branch: yes
        version: master
    - name: 'Run sed command'
      become: yes
      command: sed -i 's/172.20.1.101/localhost/g' /var/www/html/index.php
```


In the last task, Running the curl command to the http://localhost:80/ and redirecting the output to "website-log.txt"


```
    - name: 'Test the site'
      shell: "curl http://localhost:80/ > /home/mvk-admin/Learning/Ansible/website-log.txt"
      ignore_errors: yes
```


# Running Ansible commands

Run the below command to run playbook.

```
ansible-playbook playbook.yaml -i inventory
```

Once you run the playbook, All the tasks will be executed and able to see the following output in console and log (firewall-rules.txt & website-log.txt) files will be created.

```
[mvk-admin@mylinuxvm Ansible]$ ansible-playbook playbook.yaml -i inventory 

PLAY [Deployment of ecom app throug LAMP Stack] *****************************************************************************************************************************************************

TASK [Gathering Facts] ******************************************************************************************************************************************************************************
[WARNING]: Unhandled error in Python interpreter discovery for host local: unexpected output from Python interpreter discovery
[WARNING]: sftp transfer mechanism failed on [localhost]. Use ANSIBLE_DEBUG=1 to see detailed information
[WARNING]: scp transfer mechanism failed on [localhost]. Use ANSIBLE_DEBUG=1 to see detailed information
[WARNING]: Platform unknown on host local is using the discovered Python interpreter at /usr/bin/python, but future installation of another Python interpreter could change the meaning of that
path. See https://docs.ansible.com/ansible-core/2.15/reference_appendices/interpreter_discovery.html for more information.
ok: [local]

TASK [Install FIrewallD & MariaDB & HTTPD & GIT & php & phpy-mysql] *********************************************************************************************************************************
[WARNING]: sftp transfer mechanism failed on [localhost]. Use ANSIBLE_DEBUG=1 to see detailed information
[WARNING]: scp transfer mechanism failed on [localhost]. Use ANSIBLE_DEBUG=1 to see detailed information
ok: [local] => (item=git)
ok: [local] => (item=firewalld)
ok: [local] => (item=mariadb)
ok: [local] => (item=httpd)
ok: [local] => (item=php)
ok: [local] => (item=php-mysqlnd)

TASK [Start all services] ***************************************************************************************************************************************************************************
[WARNING]: sftp transfer mechanism failed on [localhost]. Use ANSIBLE_DEBUG=1 to see detailed information
[WARNING]: scp transfer mechanism failed on [localhost]. Use ANSIBLE_DEBUG=1 to see detailed information
ok: [local] => (item=firewalld)
ok: [local] => (item=mariadb)
ok: [local] => (item=httpd)

TASK [Configure FirewallD for DB & httpd] ***********************************************************************************************************************************************************
[WARNING]: sftp transfer mechanism failed on [localhost]. Use ANSIBLE_DEBUG=1 to see detailed information
[WARNING]: scp transfer mechanism failed on [localhost]. Use ANSIBLE_DEBUG=1 to see detailed information
ok: [local] => (item={'port': '3306', 'proto': 'tcp', 'state': 'enabled', 'zone': 'public'})
ok: [local] => (item={'port': '80', 'proto': 'tcp', 'state': 'enabled', 'zone': 'public'})

TASK [Check the firewalld rules] ********************************************************************************************************************************************************************
[WARNING]: sftp transfer mechanism failed on [localhost]. Use ANSIBLE_DEBUG=1 to see detailed information
[WARNING]: scp transfer mechanism failed on [localhost]. Use ANSIBLE_DEBUG=1 to see detailed information
changed: [local]

TASK [Print Firewall Rules to firewall-rules.txt] ***************************************************************************************************************************************************
changed: [local -> localhost]

TASK [Create a ecomdb in mariadb] *******************************************************************************************************************************************************************
[WARNING]: sftp transfer mechanism failed on [localhost]. Use ANSIBLE_DEBUG=1 to see detailed information
[WARNING]: scp transfer mechanism failed on [localhost]. Use ANSIBLE_DEBUG=1 to see detailed information
ok: [local]

TASK [Create a DB user ecomuser] ********************************************************************************************************************************************************************
[WARNING]: sftp transfer mechanism failed on [localhost]. Use ANSIBLE_DEBUG=1 to see detailed information
[WARNING]: scp transfer mechanism failed on [localhost]. Use ANSIBLE_DEBUG=1 to see detailed information
ok: [local]

TASK [Run several insert queries against db ecomdb in single transaction] ***************************************************************************************************************************
[WARNING]: sftp transfer mechanism failed on [localhost]. Use ANSIBLE_DEBUG=1 to see detailed information
[WARNING]: scp transfer mechanism failed on [localhost]. Use ANSIBLE_DEBUG=1 to see detailed information
changed: [local]

TASK [CLone Github code] ****************************************************************************************************************************************************************************
[WARNING]: sftp transfer mechanism failed on [localhost]. Use ANSIBLE_DEBUG=1 to see detailed information
[WARNING]: scp transfer mechanism failed on [localhost]. Use ANSIBLE_DEBUG=1 to see detailed information
changed: [local]

TASK [Run sed command] ******************************************************************************************************************************************************************************
[WARNING]: sftp transfer mechanism failed on [localhost]. Use ANSIBLE_DEBUG=1 to see detailed information
[WARNING]: scp transfer mechanism failed on [localhost]. Use ANSIBLE_DEBUG=1 to see detailed information
changed: [local]

TASK [Test the site] ********************************************************************************************************************************************************************************
[WARNING]: sftp transfer mechanism failed on [localhost]. Use ANSIBLE_DEBUG=1 to see detailed information
[WARNING]: scp transfer mechanism failed on [localhost]. Use ANSIBLE_DEBUG=1 to see detailed information
changed: [local]

PLAY RECAP ******************************************************************************************************************************************************************************************
local                      : ok=12   changed=6    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
```


# Testing

1. Login into DB and run the following commands:

```
sudo su
mysql
SHOW databases;
USE ecomdb;
SHOW tables;
SELECT * FROM products;
```

You will see the similar output page.

<img src="Ansible/DB_OUTPUT.PNG>


2. Open the browser and hit http://localhost:80/ URL. You will see the following page:

<img src="Ansible/Capture_web_site.PNG">
