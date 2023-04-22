---
title: ansible服务
date: 2023-04-21 23:55:04
tags:
---



安装Ansible和设置控制节点：

1. 在linux1上安装Ansible，可以使用以下命令：

   ```
   sudo apt update
   sudo apt install ansible
   ```

2. 配置Ansible的主机清单文件。在控制节点上使用vim编辑文件/etc/ansible/hosts，添加受控节点的IP地址或主机名。例如：

   ```
   [web_servers]
   linux2 ansible_host=192.168.0.2
   linux3 ansible_host=192.168.0.3
   linux4 ansible_host=192.168.0.4

   [database_servers]
   linux5 ansible_host=192.168.0.5
   linux6 ansible_host=192.168.0.6
   linux7 ansible_host=192.168.0.7

   [all:vars]
   ansible_user=your_user_name
   ansible_ssh_private_key_file=/path/to/your/private/key
   ```

   注意：将your_user_name和/path/to/your/private/key替换为实际的用户名和私钥路径。

3. 验证Ansible是否可以与所有受控节点通信。可以使用以下命令：

   ```
   ansible all -m ping
   ```

   如果所有节点都响应pong，则表示成功。

   注意：如果您使用的是不同的SSH端口号，可以在清单文件中使用ansible_ssh_port变量指定端口号。

使用Ansible进行自动化运维：

现在您已经设置好了Ansible控制节点和受控节点，可以开始使用Ansible进行自动化运维任务。以下是一些例子：

1. 运行命令

   ```
   ansible all -a "ls -l /var/log"
   ```

   将在所有受控节点上运行命令“ls -l /var/log”。

2. 复制文件

   ```
   ansible all -m copy -a "src=/path/to/local/file dest=/path/to/remote/file"
   ```

   将在所有受控节点上复制本地文件到远程目录。

3. 安装软件包

   ```
   ansible all -m apt -a "name=nginx state=present"
   ```

   将在所有受控节点上安装Nginx软件包。

以上只是一些简单的例子，Ansible可以完成更复杂的任务，包括配置管理、自动化部署、容器编排等等。建议您参考官方文档学习更多操作。
