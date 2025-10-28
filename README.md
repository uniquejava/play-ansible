## Installation

### Debian/Ubuntu apt

```sh
# debian
sudo apt install ansible
```

### RedHat dnf(只能安装ansibile-core)

无法使用community功能(比如shutdown模块)
```
dnf install ansible-core
```

### RedHat/macOS上更好的选择


```sh
# 仅为用户ansible提供服务, 去掉--user并加上sudo可以给所有用户安装
python3 -m pip install --user ansible

# 然后根据提示把以下路径加入PATH
# ansible
export PATH=$PATH:/Users/cyper/Library/Python/3.11/bin
```

## config file

配置文件优先级:  
1) 命令行参数 $ANSIBLE_CONFIG=xxxx
2) 当前项目目录下的 /Users/cyper/code/play-ansible/ansible.cfg
3) 用户家目录.ansible.cfg
4) 内置的 /etc/ansible/ansible.cfg


## Ad-hoc快速命令行

注意 ansible 后面紧跟 group 名.

### 拷贝文件

简单 (copy是ansible.­builtin.copy的简写)
```sh
$ touch CopyMe.txt
$ ansible all -i inventory.yml -m copy -a "src=CopyMe.txt dest=/home/j/"
```

高级 (设置属主, 强制覆盖同名文件并且备份原文件 --> 文件名加时间戳)
```sh
ansible all -i inventory.yml -m copy -a "src=records.txt dest=/home/jerry/
owner=jerry group=scta-bni mode=0640 force=true backup=true"
```


### 执行命令 (command 和  shell)

简单
```sh
ansible all -i inventory.yml -m command -a "ls -lh"
```

带 pipe (会先创建新的shell, 然后执行命令)

```sh
ansible all -i inventory.yml -m shell -a "ls -lh | grep txt"
```

### 安装/卸载软件包

become 就是 sudo
ask-become-pass 提示输入 sudo 密码
m 使用内置或第三方提供的模块
a 指定参数

更新全部包的索引

```sh
ansible all --become --ask-become-pass -i inventory.yml -m apt -a "update_cache=true"
```

安装软件包

```sh
ansible all --become --ask-become-pass -i inventory.yml -m apt
 -a "update_cache=true name=nmap"
```

卸载软件

```sh
ansible all --become --ask-become-pass -i inventory.yml -m apt 
-a "name=nmap state=absent"
```

升级全部的包

```sh
ansible all --become --ask-become-pass -i inventory.yml -m apt 
-a "update_cache=true name='*' state=latest"
```

### 管理systemd 服务和重启服务器

重启 rsyslog 服务

```sh
ansible all --become --ask-become-pass -i inventory.yml -m systemd_service
-a "name=rsyslog state=restarted"
```

重启服务器 (ansible会等待10分钟, 然后检查服务器是否重启成功)

```sh
ansible kafka --become --ask-become-pass -i inventory.yml -m reboot
```

关机 (去掉-a参数可以立即关机)

```sh
ansible kafka --become --ask-become-pass -i inventory.yml -m
community.general.shutdown -a "delay=60"
```


## vault and playbook for task execution

*   `ansible-vault create <file>`：创建一个新文件并立即加密。
*   `ansible-vault encrypt <file>`：加密一个已存在的文件。
*   `ansible-vault edit <file>`：编辑一个已加密的文件（会要求输入密码）。
*   `ansible-vault view <file>`：查看一个已加密文件的内容。
*   `ansible-vault decrypt <file>`：解密一个文件（将其恢复为明文）。
*   `ansible-vault rekey <file>`：修改一个加密文件的密码。


```
$ cd ansible

$ ansible-vault create vars/vault.yml
New Vault password: 
Confirm New Vault password: 

$ touch playbook_newuser.yml
```

## inventory 和 group_vars 最佳组织方式
```
inventories/
├── production
├── staging
└── development

group_vars/
├── all/
│   └── common.yml
├── webservers/           # 跨环境 Web 服务器通用配置
│   └── nginx_base.yml
├── dbservers/            # 跨环境数据库通用配置  
│   └── postgresql_base.yml
├── production/           # 生产环境特定配置
│   ├── webservers/       # 生产环境 Web 服务器配置
│   │   └── app_config.yml
│   └── dbservers/        # 生产环境数据库配置
│       └── postgresql.yml
└── development/          # 开发环境特定配置
    ├── webservers/
    │   └── app_config.yml
    └── dbservers/
        └── postgresql.yml
```

inventory文件示例

```ini
# inventories/production
[webservers]
web-prod-01.example.com
web-prod-02.example.com

[dbservers]
db-prod-01.example.com

[production:children]
webservers
dbservers
```


## 执行 create user 任务

### 准备vault
```sh
$ mkdir -p ansible/{files,logs,output,tasks,templates,vars}

$ cd ansible

$ ansible-vault create vars/vault.yml
New Vault password: 
Confirm New Vault password: 

$ touch playbook_newuser.yml
```

vars/vars.yml

```
user: kafka
```

vars/vault.yml

```
加密后的user_password: xxxx
```

### 准备playbook

playbook_newuser.yml (其中模板中的user和user_password来自vars/vault.yml)

```yml
- hosts: all
  become: yes
  gather_facts: yes
  vars_files:
    - vars/vars.yml
    - vars/vault.yml
  tasks:
    - name: Create User and Set Password
      ansible.builtin.­user:
        name: "{{ user }}"
        password: "{{ user_password }}"
        home: "/home/{{ user }}"
        shell: /bin/bash

```

### 运行playbook

```sh
ansible-playbook --ask-become-pass --ask-vault-pass -i inventory.yml playbook_newuser.yml
```

## playbook 拆分子任务到独立文件

### 定义子任务

ansible/tasks/newuser.yml

```yml
- name: Create User and Set Password
  ansible.builtin.­user:
    name: "{{ user }}"
    password: "{{ user_password }}"
    home: "/home/{{ user }}"
    shell: /bin/bash
```


### playbook包含子任务

ansible/playbook_newuser.yml

```yml {9}
- hosts: all
  become: yes
  gather_facts: yes
  vars_files:
    - vars/vars.yml
    - vars/vault.yml
  tasks:
    - name: Create User and Set Password
      include_tasks: tasks/newuser.yml
```

### 执行playbook

- J 等价于 --ask-vault-password
- K 等价于 --ask-become-pass
- b 等价于 --become
- i  指定 inventory文件
- 可以合并为 -JKbi

```

ansible-playbook -J -K -i inventory.yml playbook_newuser.yml
```