---
title: ubuntu 20.04裸机安装paperless-ngx 
description: 
published: true
date: 2023-10-08T13:57:50.160Z
tags: install, ubuntu, paperless
editor: markdown
dateCreated: 2023-10-08T13:57:50.160Z
---



## 说明

在官方文档中该安装方式是在debian/buster上安装测试过而已，所以我在ubuntu上安装其实稳定性还是有待考究，但是需要的包，ubuntu也并不缺少，在安装部署过程中并没有因此遇到什么问题。

## 安装

1. 安装依赖项

   ```
   sudo apt install -y python3 python3-pip python3-dev imagemagick fonts-liberation gnupg libpq-dev default-libmysqlclient-dev pkg-config libmagic-dev mime-support libzbar0 poppler-utils
   ```

   

2. 安装OCRmyPDF依赖项

   ```
   sudo apt install -y unpaper ghostscript icc-profiles-free qpdf liblept5 libxml2 pngquant zlib1g tesseract-ocr
   ```

   

3. 安装python依赖项

   ```
   sudo apt install -y build-essential python3-setuptools python3-wheel
   ```

4. 添加用户

   ```
   sudo adduser paperless --system --home /opt/paperless --group
   ```

5. 安装redis大于等于6.0.0版本以上，这里我安装的是最新版本

   ```
   sudo apt install lsb-release curl gpg
   ```

   ```
   curl -fsSL https://packages.redis.io/gpg | sudo gpg --dearmor -o /usr/share/keyrings/redis-archive-keyring.gpg
   ```

   ```
   echo "deb [signed-by=/usr/share/keyrings/redis-archive-keyring.gpg] https://packages.redis.io/deb $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/redis.list
   ```

   ```
   sudo apt update 
   sudo apt install -y redis
   ```

   ```
   sudo systemctl enable redis-server.service
   ```

6. 安装postgresql，数据库可以选用postgresql、mariadb和sqllite，使用sqlite需要启用json1 extension，所以我还是选择使用postgresql，因为没有版本要求，这里我使用官方仓库提供的postgresql 12版本

   ```
   sudo apt install postgresql
   ```

7. 创建对应的数据库和用户名和密码，这里我都是设置成`paperless`

   ```
   sudo -u postgres psql
   postgres-# create database paperless;
   postgres-# create user paperless with encrypted password 'paperless';
   postgres-# grant all privileges on database paperless to paperless;
   ```

8. 下载最新版本的release archive

   ```
   curl -O -L https://github.com/paperless-ngx/paperless-ngx/releases/download/v1.10.2/paperless-ngx-v1.10.2.tar.xz
   ```

9. 解压

   ```
   tar -xf paperless-ngx-v1.10.2.tar.xz
   ```

10. 将解压文件拷贝到/opt/paperless

    ```
    cp -r paperless-ngx/* /opt/paperless
    ```

11. 修改配置`paperless.conf`

    ```bash
    cd /opt/paperless
    vim paperless.conf
    ......
    # Required services
    
    PAPERLESS_REDIS=redis://localhost:6379
    PAPERLESS_DBENGINE=postgres
    PAPERLESS_DBHOST=localhost
    PAPERLESS_DBPORT=5432
    PAPERLESS_DBNAME=paperless
    PAPERLESS_DBUSER=paperless
    PAPERLESS_DBPASS=paperless
    #PAPERLESS_DBSSLMODE=prefer
    
    # Paths and folders
    
    PAPERLESS_CONSUMPTION_DIR=/opt/paperless/data
    #PAPERLESS_DATA_DIR=../data
    #PAPERLESS_TRASH_DIR=
    #PAPERLESS_MEDIA_ROOT=../media
    #PAPERLESS_STATICDIR=../static
    #PAPERLESS_FILENAME_FORMAT=
    #PAPERLESS_FILENAME_FORMAT_REMOVE_NONE=
    
    # Security and hosting
    
    #PAPERLESS_SECRET_KEY=change-me
    PAPERLESS_URL=https://0.0.0.0
    ......
    ```

12. 补全目录并赋予权限

    ```bash
    mkdir /opt/paperless/data
    mkdir /opt/paperless/media
    mkdir /opt/paperless/consume
    sudo chown paperless:paperless /opt/paperless
    ```

13. 使用pip安装依赖

    ```
    sudo -Hu paperless pip3 install -r requirements.txt
    ```

    

14. 执行下述命令

    ```
    cd /opt/paperless/src
    sudo -Hu paperless python3 manage.py migrate
    sudo -Hu paperless python3 manage.py createsuperuser
    sudo -Hu paperless python3 manage.py runserver
    ```

    