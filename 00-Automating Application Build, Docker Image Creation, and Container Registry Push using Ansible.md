

# Automating Application Build, Docker Image Creation, and Container Registry Push using Ansible


## Project Architecture:
```

                              +----------------------+
                              |   Git Repository     |
                              | (GitHub / GitLab)    |
                              +----------+-----------+
                                         |
                                  git clone / pull
                                         |
                                         |
                        +----------------v----------------+
                        |      Ansible Control Node       |
                        |          Ubuntu Server          |
                        +----------------+----------------+
                                         |
                -------------------------------------------------
                |               |               |               |
                |               |               |               |
                v               v               v               v
        +---------------+ +--------------+ +--------------+ +--------------+
        | Build Project | | Run Testing  | | Code Quality | | Security Scan|
        | npm install   | | npm test     | | ESLint       | | SAST / SCA   |
        | npm run build | |              | |              | | Secret Scan  |
        +-------+-------+ +------+-------+ +------+-------+ +------+-------+
                |                 |                |                |
                +-----------------+----------------+----------------+
                                          |
                                          v
                               +----------------------+
                               | Build Docker Image   |
                               | docker build         |
                               +----------+-----------+
                                          |
                                          v
                               +----------------------+
                               | Docker Image Scan    |
                               | Trivy / Grype        |
                               +----------+-----------+
                                          |
                                          v
                               +----------------------+
                               | Push Docker Image    |
                               | Docker Hub / GHCR    |
                               +----------+-----------+
                                          |
                                          v
                               +----------------------+
                               | Generate Reports     |
                               | Build, Test, Scan    |
                               +----------+-----------+
                                          |
                                          v
                               +----------------------+
                               | Optional Deployment  |
                               | Docker / Kubernetes  |
                               +----------------------+
							   
```
							   
## Create the Directory Structure

```

ansible-ci-project/
│
├── ansible.cfg
├── inventory
├── site.yml
├── README.md
│
├── group_vars/
│   └── all.yml
│
├── host_vars/
│
├── roles/
│
│   ├── build/
│   │   ├── tasks/
│   │   │   └── main.yml
│   │   ├── vars/
│   │   ├── defaults/
│   │   ├── handlers/
│   │   ├── files/
│   │   └── templates/
│   │
│   ├── docker/
│   │   ├── tasks/
│   │   │   └── main.yml
│   │   ├── vars/
│   │   ├── defaults/
│   │   ├── handlers/
│   │   ├── files/
│   │   └── templates/
│   │
│   ├── push/
│   │   ├── tasks/
│   │   │   └── main.yml
│   │   ├── vars/
│   │   ├── defaults/
│   │   ├── handlers/
│   │   ├── files/
│   │   └── templates/
│   │
│   └── cleanup/
│       ├── tasks/
│       │   └── main.yml
│       ├── vars/
│       ├── defaults/
│       ├── handlers/
│       ├── files/
│       └── templates/
│
├── playbooks/
│   ├── build.yml
│   ├── docker.yml
│   ├── push.yml
│   └── deploy.yml
│
└── files/
    └── Dockerfile

```
Create Using Terminal


```
mkdir -p roles/{build,docker,push,cleanup}/{tasks,vars,defaults,handlers,files,templates}

mkdir playbooks
mkdir group_vars
mkdir host_vars
mkdir files

touch inventory
touch ansible.cfg
touch site.yml
touch README.md

touch group_vars/all.yml

touch playbooks/build.yml
touch playbooks/docker.yml
touch playbooks/push.yml
touch playbooks/deploy.yml

touch roles/build/tasks/main.yml
touch roles/docker/tasks/main.yml
touch roles/push/tasks/main.yml
touch roles/cleanup/tasks/main.yml

touch files/Dockerfile
```


## Step: Create ansible.cfg

```
[defaults]
inventory=inventory
host_key_checking=False
retry_files_enabled=False
roles_path=roles
stdout_callback=yaml	
```

## Create Inventory hosts.ini file 

under inventory folder

```
[buildserver]
localhost ansible_connection=local

```

## Step 7: Create site.yml

```
- hosts: buildserver
  become: yes

  roles:
    - build
    - docker
    - push

```

## Step 8: Create group_vars/all.yml
```
project_name: nodejs-apps

docker_image: nodejs-apps

docker_tag: latest

docker_username: anisur81

docker_password: dckr_pat_lKpBAiulBIaogOEXd0t8zPdlp7I

docker_registry: docker.io

project_path: "{{ playbook_dir }}/nodejsapp"
```


## create the file for Build Role :  

roles/build/tasks/main.yml

```

- name: Install Node.js dependencies
  ansible.builtin.command: npm install
  args:
    chdir: "{{ project_path }}"

- name: Build Node.js application
  ansible.builtin.command: npm run build
  args:
    chdir: "{{ project_path }}"
	```

## Step 10: Docker Role
```
- name: Build Docker Image
  community.docker.docker_image:
    name: "{{ docker_username }}/{{ docker_image }}"
    tag: "{{ docker_tag }}"
    source: build
    build:
      path: /opt/project
```	  

## Step 11: Push Role
```
- name: Login Docker Hub
  community.docker.docker_login:
    username: "{{ docker_username }}"
    password: "{{ docker_password }}"

- name: Push Image
  community.docker.docker_image:
    name: "{{ docker_username }}/{{ docker_image }}"
    tag: "{{ docker_tag }}"
    push: yes
    source: local

```

## Install  Ansible
```
sudo apt install -y ansible-core
ansible --version
ansible-playbook --version
ansible-galaxy --version

Install the Docker Collection

Since your playbooks use community.docker, install the required collection:

ansible-galaxy collection install community.docker

Verify:

ansible-galaxy collection list

```

## Now Ansible will automatically use your ansible.cfg.
```
administrator@WIN-RJ32TRRFOFP:~/projects/ansible$ ansible -i inventory/hosts.ini buildserver -m ping
localhost | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}

```

## Step 12: Run the Project
ansible-playbook site.yml	--ask-become-pass

```
## Finally run the ansible ci pipeline command
root@jaspertestsvr:/opt/ansible# ansible-playbook site.yml        --ask-become-pass
BECOME password:

PLAY [buildserver] *********************************************************************************************************************************

TASK [Gathering Facts] *****************************************************************************************************************************
[WARNING]: Host 'localhost' is using the discovered Python interpreter at '/usr/bin/python3.14', but future installation of another Python interpreter could cause a different interpreter to be discovered. See https://docs.ansible.com/ansible-core/2.20/reference_appendices/interpreter_discovery.html for more information.
ok: [localhost]

TASK [build : Install Node.js dependencies] ********************************************************************************************************
changed: [localhost]

TASK [docker : Build Docker Image] *****************************************************************************************************************
ok: [localhost]

TASK [push : Login Docker Hub] *********************************************************************************************************************
ok: [localhost]

TASK [push : Push Image] ***************************************************************************************************************************
changed: [localhost]

PLAY RECAP *****************************************************************************************************************************************
localhost                  : ok=5    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

```




