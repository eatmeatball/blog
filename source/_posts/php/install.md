---
title: curl
date: 2018-11-04 15:25:38
tags: php
categories: php 
---
ubuntu

ubuntu 中软件的卸载方法：

1.apt方式安装：(相应的文件夹有相关的软件，先进入到软件的文件夹)
  普通安装： sudo apt-get install 软件名
  修复安装： sudo apt-get -f install 软件名
  重新安装： sudo apt-get --rreinstall install 软件名
　(一般是 .deb　格式)
2.dpkg 方式：
  1.普通安装：sudo dpkg -i package_name.deb
=====================================
如果软件的格式为 .rpm 格式包时，则
1.先安装　alien 和　fakeroot 这两个工具，
 sudo apt-get install alien fakeroot
alien 把　.rpm 为　.deb 格式的文件
1. 将　.rpm 格式的文件转为 同文件名的　.deb
 fakeroot alien xx.rpm
这样就可以按上面的方法安装了

 1.apt 方式：
  a. 移除式卸载： apt-get remove 软件名
  b. 清除式卸载： apt-get --purge remove 软件名 (同时清除配置)
  c. 清除式卸载： apt-get --purge 软件名  (同时清除配置)

 2.dpkg 方式：
   a. 移除式卸载：sudo dpkg dpkg_name
   b. 清除式卸载：sudo -P dpkg_name

查看已经安装的软件名称：

dpkg -l

查找软件库中的软件
apt-cache search 正则表达式
或者
aptitude search 软件包（部分）
可以查看相关软件的名称　标志 i　表示已经安装


apt-cache search php7
```


./configure \
--prefix=/usr/local/php73 \
--exec-prefix=/usr/local/php73 \
--bindir=/usr/local/php73/bin \
--sbindir=/usr/local/php73/sbin \
--includedir=/usr/local/php73/include \
--libdir=/usr/local/php73/lib/php \
--mandir=/usr/local/php73/php/man \
--with-config-file-path=/usr/local/php73/etc \
--with-openssl \
--enable-mbstring \
--enable-fpm

sudo make && sudo make install