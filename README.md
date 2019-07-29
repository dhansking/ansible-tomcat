# ansible-tomcat

(Last Updated On: June 26, 2019)
Apache Tomcat is a free and open-source HTTP server designed to serve Java web pages. Tomcat is an implementation of the Java Servlet, JavaServer Pages, Java Expression Language, and Java WebSocket technologies. It is widely deployed and powers various mission-critical web applications around the world.

The standard way of installing Tomcat on a Linux system such as Ubuntu/CentOS/Debian is manual and time-consuming. This guide will discuss a better way, which is automated and can be reproduced easily.

Environment Setup
I assume you have a CentOS 7+, Ubuntu 16.04+ system with Systemd service manager. This Ansible installation won’t work for Upstart or Sysvinit.


 
Step 1: Install Ansible
The main dependency on your Workstation is Ansible. Install Ansible on your Linux system using the commands shared below.

###### CentOS / Fedora / RHEL ######
sudo yum install ansible
sudo dnf install ansible

###### Ubuntu/Debian/Linux Mint ######
sudo apt-get install -y  software-properties-common
sudo apt-add-repository --yes --update ppa:ansible/ansible
sudo apt-get update
sudo apt-get install -y ansible

###### Arch/Manjaro ######
$ sudo pacman -S ansible

###### macOS ######
sudo easy_install pip
sudo pip install ansible
Step 2: Create Ansible folders
Let’s create a directory for our project.

mkdir -p ~/projects/ansible/roles
cd ~/projects/ansible/roles
Our Ansible YAML files will be located in this parent directory.

Step 3: Create Ansible playbook files
We now need to create ansible role and tasks as YAML files to orchestrate steps of the manual ordered processes.

cd ~/projects/ansible/roles
mkdir -p tomcat/{tasks,handlers,defaults,vars,templates}
The tomcat role has the following folders:

tasks – contain task files
handlers – contain handlers file
defaults – the default lower priority variables for this role
vars – has variables associated with the role
templates – holds files for use with the template resource
You should have something like this.

$ tree
.
├── roles
│   └── tomcat
│       ├── defaults
│       ├── handlers
│       ├── tasks
│       └── vars
└── tomcat

7 directories, 0 files
3.1 Create setup tasks
Let’s create tasks which will setup complete Tomcat environment.

3.11 – For Debian family.
$ vim tomcat/tasks/tomcat-setup-Debian.yml
Add:

- name: Ensure the system can use the HTTPS transport for APT.
  stat:
    path: /usr/lib/apt/methods/https
  register: apt_https_transport

- name: Install APT HTTPS transport.
  apt:
    name: "apt-transport-https"
    state: present
    update_cache: yes
  when: not apt_https_transport.stat.exists

- name: Install basic packages
  package:
    name: ['vim','aptitude','bash-completion','tmux','tree','htop','wget','unzip','curl','git']
    state: present
    update_cache: yes

- name: Install Default Java (Debian/Ubuntu)
  apt:
    name: default-jdk
    state: present

- name: Add tomcat group
  group:
    name: tomcat

- name: Add "tomcat" user
  user:
    name: tomcat
    group: tomcat
    home: /usr/share/tomcat
    createhome: no
    system: yes

- name: Download Tomcat
  get_url:
    url: "{{ tomcat_archive_url }}"
    dest: "{{ tomcat_archive_dest }}"

- name: Create a tomcat directory
  file:
    path: /usr/share/tomcat
    state: directory
    owner: tomcat
    group: tomcat

- name: Extract tomcat archive
  unarchive:
    src: "{{ tomcat_archive_dest }}"
    dest: /usr/share/tomcat
    owner: tomcat
    group: tomcat
    remote_src: yes
    extra_opts: "--strip-components=1"
    creates: /usr/share/tomcat/bin

- name: Copy tomcat service file
  template:
    src: templates/tomcat.service.j2
    dest: /etc/systemd/system/tomcat.service
  when: ansible_service_mgr == "systemd"

- name: Start and enable tomcat
  service:
    daemon_reload: yes
    name: tomcat
    state: started
    enabled: yes
  when: ansible_service_mgr == "systemd"
3.12 – For RedHat family.
$ vim tomcat/tasks/tomcat-setup-RedHat.yml
with:

- name: Add EPEL repository
  yum:
    name: epel-release
    state: present

- name: Install basic packages
  package:
    name: ['vim','bash-completion','tmux','tree','htop','wget','unzip','curl','git']
    state: present

- name: Install Java 8 CentOS
  yum:
    name: java-1.8.0-openjdk
    state: present

- name: Add tomcat group
  group:
    name: tomcat

- name: Add "tomcat" user
  user:
    name: tomcat
    group: tomcat
    home: /usr/share/tomcat
    createhome: no
    system: yes

- name: Download Tomcat
  get_url:
    url: "{{ tomcat_archive_url }}"
    dest: "{{ tomcat_archive_dest }}"

- name: Create a tomcat directory
  file:
    path: /usr/share/tomcat
    state: directory
    owner: tomcat
    group: tomcat

- name: Extract tomcat archive
  unarchive:
    src: "{{ tomcat_archive_dest }}"
    dest: /usr/share/tomcat
    owner: tomcat
    group: tomcat
    remote_src: yes
    extra_opts: "--strip-components=1"
    creates: /usr/share/tomcat/bin

- name: Copy tomcat service file
  template:
    src: templates/tomcat.service.j2
    dest: /etc/systemd/system/tomcat.service
  when: ansible_service_mgr == "systemd"

- name: Start and enable tomcat
  service:
    daemon_reload: yes
    name: tomcat
    state: started
    enabled: yes
  when: ansible_service_mgr == "systemd"

- name: Start and enable firewalld
  service:
    name: firewalld
    state: started
    enabled: yes
  when: ansible_service_mgr == "systemd"

- name: Open tomcat port on the firewall
  firewalld:
    port: 8080/tcp
    permanent: true
    state: enabled
    immediate: yes
  when: ansible_service_mgr == "systemd"
Create main.yml file.

$ vim tasks/main.yml
---
- name: Add the OS specific variables
  include_vars: "{{ item }}"
  with_first_found:
    - "{{ ansible_distribution }}{{ ansible_distribution_major_version }}.yml"
    - "{{ ansible_os_family }}.yml"

- include_tasks: "tomcat-{{ ansible_os_family }}.yml"
3.2 – Create vars file.
$ vim defaults/main.yml
---
tomcat_ver: 9.0.21
tomcat_archive_url: https://archive.apache.org/dist/tomcat/tomcat-9/v{{ tomcat_ver }}/bin/apache-tomcat-{{ tomcat_ver }}.tar.gz
tomcat_archive_dest: /tmp/apache-tomcat-{{ tomcat_ver }}.tar.gz
Check tomcat releases page for latest Tomcat 9.x.

Then add OS family specific variables.

$ vim tomcat/vars/Debian.yml
---
JAVA_HOME: /usr/lib/jvm/default-java

$ vim tomcat/vars/RedHat.yml
JAVA_HOME: /usr/lib/jvm/jre
3.3 – Create service systemd template
We also need to add Tomcat systemd service as a template.

vim templates/tomcat.service.j2
Add:

[Unit]
Description=Tomcat
After=syslog.target network.target

[Service]
Type=forking

User=tomcat
Group=tomcat

Environment=JAVA_HOME={{ JAVA_HOME }}
Environment='JAVA_OPTS=-Djava.awt.headless=true'

Environment=CATALINA_HOME=/usr/share/tomcat
Environment=CATALINA_BASE=/usr/share/tomcat
Environment=CATALINA_PID=/usr/share/tomcat/temp/tomcat.pid

ExecStart=/usr/share/tomcat/bin/catalina.sh start
ExecStop=/usr/share/tomcat/bin/catalina.sh stop

[Install]
WantedBy=multi-user.target
3.4 – Add handler.
$ vim handlers/main.yml
- name: restart tomcat
  service:
    name: tomcat
    state: restarted
3.5 – Define playbook execution file.
Playbook file defines roles to be executed, and on which instances.

cd ~/ansible/
vim tomcat-setup.yml
Paste:

- name: Tomcat deployment playbook
  hosts: app-servers
  become: yes
  become_method: sudo
  remote_user: vagrant
  roles:
    - tomcat
Replace app-servers with the server IP address or hostname or server group defined in the inventory file. This can be a custom file such as hosts in the project directory, or global inventory file – /etc/ansible/hosts.

$ sudo vim /etc/ansible/hosts
Example:

$ cat /etc/ansible/hosts

192.168.10.11
servera.example.com

[app-servers]
server1
server2
Where there is a DNS mapping for names used on /etc/hosts or records resolvable via DNS server in the network.

$ cat /etc/hosts
192.168.121.52 server1
192.168.121.108 server2
server1 is CentOS 7 machine and server2 runs Ubuntu 18.04.

This is the final folder structure:

$ cd ~/projects/ansible
$ tree                            
.
├── roles
│   └── tomcat
│       ├── defaults
│       │   └── main.yml
│       ├── handlers
│       │   └── main.yml
│       ├── tasks
│       │   ├── main.yml
│       │   ├── tomcat-Debian.yml
│       │   └── tomcat-RedHat.yml
│       ├── templates
│       │   └── tomcat.service.j2
│       └── vars
│           ├── Debian.yml
│           └── RedHat.yml
├── setup-tomcat.yml
└── tomcat
Step 4: Execute Playbook
We are ready to run tomcat playbook for the action to begin.

$ cd ~/projects/ansible
$ ansible-playbook setup-tomcat.yml
My execution output sample:

PLAY [Tomcat deployment playbook] *****************************************************************************************************************

TASK [Gathering Facts] ****************************************************************************************************************************
ok: [server1]
ok: [server2]

TASK [tomcat : include_tasks] *********************************************************************************************************************
included: /home/jmutai/projects/ansible/roles/tomcat/tasks/tomcat-RedHat.yml for server1
included: /home/jmutai/projects/ansible/roles/tomcat/tasks/tomcat-Debian.yml for server2

TASK [tomcat : Set Server timezone] ***************************************************************************************************************
changed: [server1]

TASK [tomcat : Add EPEL repository to CentOS] *****************************************************************************************************
changed: [server1]

TASK [tomcat : Install basic packages] ************************************************************************************************************
changed: [server1]

TASK [tomcat : Install Java 8 CentOS] *************************************************************************************************************
changed: [server1]

TASK [tomcat : Add tomcat group] ******************************************************************************************************************
changed: [server1]

TASK [tomcat : Add "tomcat" user] *****************************************************************************************************************
changed: [server1]

TASK [tomcat : Download Tomcat] *******************************************************************************************************************
changed: [server1]

TASK [tomcat : Create a tomcat directory] *********************************************************************************************************
changed: [server1]

TASK [tomcat : Extract tomcat archive] ************************************************************************************************************
changed: [server1]

TASK [tomcat : Copy tomcat service file] **********************************************************************************************************
changed: [server1]

TASK [tomcat : Start and enable tomcat] ***********************************************************************************************************
changed: [server1]

TASK [tomcat : Start and enable firewalld] ********************************************************************************************************
changed: [server1]

TASK [tomcat : Open tomcat port on the firewall] **************************************************************************************************
changed: [server1]

TASK [tomcat : Set Server timezone] ***************************************************************************************************************
changed: [server2]

TASK [tomcat : Ensure the system can use the HTTPS transport for APT.] ****************************************************************************
ok: [server2]

TASK [tomcat : Install APT HTTPS transport.] ******************************************************************************************************
skipping: [server2]

TASK [tomcat : Install basic packages] ************************************************************************************************************
 [WARNING]: Could not find aptitude. Using apt-get instead

changed: [server2]
TASK [tomcat : Install Default Java (Debian/Ubuntu)] **********************************************************************************************
changed: [server2]

TASK [tomcat : Add tomcat group] ******************************************************************************************************************
changed: [server2]

TASK [tomcat : Add "tomcat" user] *****************************************************************************************************************
changed: [server2]
TASK [tomcat : Download Tomcat] *******************************************************************************************************************
changed: [server2]

TASK [tomcat : Create a tomcat directory] *********************************************************************************************************
changed: [server2]

TASK [tomcat : Extract tomcat archive] ************************************************************************************************************
changed: [server2]

TASK [tomcat : Copy tomcat service file] **********************************************************************************************************
changed: [server2]

TASK [tomcat : Start and enable tomcat] ***********************************************************************************************************
changed: [server2]

PLAY RECAP ****************************************************************************************************************************************
server1                    : ok=14   changed=0    unreachable=0    failed=0    skipped=1    rescued=0    ignored=0   
server2                    : ok=13   changed=5    unreachable=0    failed=0    skipped=1    rescued=0    ignored=0   
Test your installation and configuration by visiting server URL on port 8080.


This is a green light for a successful installation of Tomcat on Ubuntu/Debian/CentOS using Ansible.
