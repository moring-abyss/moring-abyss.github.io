---
title: jenkins安装
---

# yum安装jenkins

## 添加jenkins repo
> sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo

## 导入jenkins公钥
> sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io.key

## 更新yum
> sudo yum upgrade

## 安装jdk
> sudo yum install fontconfig java-11-openjdk

## 安装jenkins
> yum install jenkins

## 启动
sudo systemctl enable jenkins
sudo systemctl start jenkins
sudo systemctl status jenkins

## 安装git
> yum install git

## 安装nodejs
> rm -fv /etc/yum.repos.d/nodesource*
> curl --silent --location https://rpm.nodesource.com/setup_14.x | sudo bash
> yum install -y nodejs

## 插件安装
这里不是很熟悉，所以选择了推荐的插件安装

### su无法切换jenkins用户
> 执行 - vi /etc/passwd
> 找到jenkins那行，把/bin/false改成/bin/bash

### 无法clone代码
> jenkins用户加入root组
> gpasswd -a root jenkins
> vim /etc/sysconfig/jenkins; 修改配置：JENKINS_USER="root"    JENKINS_GROUP="root"
> 修改文件夹权限：chomd 777 <文件夹名称>

### 给jenkins用户添加sudo权限
添加到sudo组或者wheel组
> usermod -a -G wheel jenkins
> usermod -a -G sudo jenkins
> sudo visudo -f /etc/sudoers
找到root    ALL=(ALL)       ALL，这行下添加代码
> jenkins    ALL=(ALL)       NOPASSWD:ALL 