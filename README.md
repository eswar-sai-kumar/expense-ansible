# Expense project using Ansible

## EC2 instances creation
Create 3 EC2 instances (use RHEL-9 AMI, key-pair, allow-everything security group)

Name them db, backend, frontend

## Creating records 
Go to Route53 -> Hosted zones -> click on eswarsaikumar.site -> Create 3 records 

Name them db,backend,do not give frontend(make name as blank) 

Keep values as their respective public IP addresses

![Screenshot 2025-04-30 081626](https://github.com/user-attachments/assets/fbff9e22-c29b-45bc-ad28-d156a53df0cf)

![Screenshot 2025-04-30 081644](https://github.com/user-attachments/assets/45f6395b-2078-473e-abf2-e29e1ef23177)

![Screenshot 2025-04-30 081858](https://github.com/user-attachments/assets/11535301-40c3-4c59-a45b-f791d007e062)

## inventory.ini
```
[db]
db.eswarsaikumar.site

[backend]
backend.eswarsaikumar.site

[frontend]
eswarsaikumar.site
```

## Servers in gitbash (Follow same process for 3 servers)   
```
ssh -i daws.pem ec2-user@3.85.221.230
```

After running command, Enter password "DevOps321"

###### Install Ansible
```
sudo dnf install ansible -y
```
###### Install Git
```
sudo dnf install git -y
```
###### Clone GitHub Repo
```
git clone "https://github.com/eswar-sai-kumar/expense-ansible.git"
```
```
cd expense-ansible
```
###### Don't forgot to pull after pushing to github
```
git pull
```

## db.yaml
```
- name: configure DB server
  hosts: db
  become: yes
  vars:
    login_host: db.eswarsaikumar.site
  vars_prompt:
  - name: mysql_root_password
    prompt: please enter mysql root password
    private: no
  tasks:
  - name: install mysql-server
    ansible.builtin.dnf:
      name: mysql-server
      state: latest

  - name: start mysql
    ansible.builtin.service:
      name: mysqld
      state: started
      enabled: yes

  - name: install python mysql dependencies
    ansible.builtin.pip:
      name:
      - PyMySQL
      - cryptography
      executable: pip3.9

  # check password is already is setup or not
  - name: check db connection
    community.mysql.mysql_info: # ansible community module
      login_user: root
      login_password: "{{mysql_root_password}}"
      login_host: "{{login_host}}"
      filter: version # it only fetches version from the mysql information, if version came that means password already set 
    ignore_errors: yes # errors will come if password not set yet, so we ignore errors and set up password
    register: mysql_connection_output # saves mysql info here

  - name: print output
    ansible.builtin.debug:
      msg: "output: {{mysql_connection_output}}" 
  
  - name: setup root password
    ansible.builtin.command: "mysql_secure_installation --set-root-pass {{mysql_root_password}}"
    when: mysql_connection_output.failed is true  # don't give {{ }} in when conditon
```
```
ansible-playbook -i inventory.ini -e ansible_user=ec2-user -e ansible_password=DevOps321 db.yaml
```
## backend.yaml
```
- name: configure backend server
  become: yes
  hosts: backend
  vars:
    login_host: db.eswarsaikumar.site
    login_password: ExpenseApp@1
  tasks:
  - name: disable default nodejs
    ansible.builtin.command: dnf module disable nodejs -y

  - name: enable nodejs:20
    ansible.builtin.command: dnf module enable nodejs:20 -y

  - name: install nodejs
    ansible.builtin.dnf: # only works for Redhat family
      name: "{{ item }}"
      state: installed
    loop:
    - nodejs
    - mysql

  - name: create expense user
    ansible.builtin.user:
      name: expense

  - name: create app directory
    ansible.builtin.file:
      path: /app
      state: directory

  - name: download backend code
    ansible.builtin.get_url:
      url: https://expense-builds.s3.us-east-1.amazonaws.com/expense-backend-v2.zip
      dest: /tmp/backend.zip

  - name: unzip backend code
    ansible.builtin.unarchive:
      src: /tmp/backend.zip # by default checks in ansible server
      dest: /app
      remote_src: yes

  - name: install npm dependencies
    ansible.builtin.command: npm install
    args:
      chdir: /app

  - name: copy backend service
    ansible.builtin.copy:
      src: backend.service
      dest: /etc/systemd/system/backend.service

  - name: install python mysql dependencies
    ansible.builtin.pip: 
      name:
      - PyMySQL # to connect MySQL DB from Ansible
      - cryptography
      executable: pip3.9 # pip3.11 if python version is 3.11

  - name: import data into mysql
    community.mysql.mysql_db:
      login_user: root
      login_password: "{{ login_password }}"
      login_host: "{{ login_host }}"
      target: /app/schema/backend.sql
      state: import
      name: all

  - name: daemon reload
    ansible.builtin.systemd_service:
      daemon_reload: true

  - name: start and enable backend service
    ansible.builtin.service:
      name: backend
      state: started
      enabled: yes
```
## backend.service
```
[Unit]
Description = Backend Service

[Service]
User=expense
Environment=DB_HOST="db.eswarsaikumar.site"
ExecStart=/bin/node /app/index.js
SyslogIdentifier=backend

[Install]
WantedBy=multi-user.target
```
```
ansible-playbook -i inventory.ini -e ansible_user=ec2-user -e ansible_password=DevOps321 backend.yaml
```
## frontend.yaml
```
- name: configure frontend server
  become: yes
  hosts: frontend
  tasks:
  - name: install nginx server
    ansible.builtin.dnf:
      name: nginx
      state: latest
  
  - name: start and enable nginx
    ansible.builtin.service:
      name: nginx
      state: started
      enabled: yes

  - name: delete default website
    ansible.builtin.file:
      path: /usr/share/nginx/html
      state: absent

  - name: create html directory
    ansible.builtin.file:
      path: /usr/share/nginx/html
      state: directory

  - name: download and extract frontend code
    ansible.builtin.unarchive:
      src: https://expense-builds.s3.us-east-1.amazonaws.com/expense-frontend-v2.zip
      dest: /usr/share/nginx/html
      remote_src: yes

  - name: copy backend conf
    ansible.builtin.copy:
      src: expense.conf
      dest: /etc/nginx/default.d/expense.conf

  - name: restart nginx
    ansible.builtin.service:
      name: nginx
      state: restarted
```
## expense.conf
```
proxy_http_version 1.1;

location /api/ { proxy_pass http://backend.eswarsaikumar.site:8080/; }

location /health {
  stub_status on;
  access_log off;
}
```
```
ansible-playbook -i inventory.ini -e ansible_user=ec2-user -e ansible_password=DevOps321 frontend.yaml
```

