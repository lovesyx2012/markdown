[TOC]



#### 1. 背景

目前互联网公司大都采用前后端分离的架构，前端基于vue或者react工程化管理项目，通常通过pm2来管理进程。

##### 1.1 开始管理项目

```shell
cd  /home/data/web-app
sudo -i pm2 start npm --name "web-app" -- run space
```

##### 1.2 常用命令

```shell
sudo -i pm2 list
sudo -i pm2 stop web-app
sudo -i pm2 start web-app
sudo -i pm2 restart web-app
sudo -i pm2 show web-app
```



#### 2. 通过端口定位项目

有时候开发只知道端口号（因为端口号可能会在项目代码里面体现），需要确认这个端口号是否是指定的项目在使用，可以通过以下方式判断：

##### 2.1 通过端口定位到进程

```shell
sudo netstat -atpln | grep 18100
```

##### 2.2 通过进程ID定位到项目

```shell
sudo lsof -np 1878
```

##### 2.3 示例

![](https://www.yunsom.com/storage/api/file/200014202102081443580-99ffa2382e)



#### 3 nginx 配置

```shell
upstream space_web_app {
    server 127.0.0.1:18100;
}

server {
    listen 80;
    server_name web-app.baidu.space;
    index index.html;

    location / {
        proxy_pass_header Server;
        proxy_set_header Host $http_host;
        proxy_redirect off;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Scheme $scheme;
        proxy_pass http://space_web_app;
    }
}
```



#### 4. 发布

##### 4.1 package.json scripts脚本配置

```json
"scripts": {
    "space_build": "NODE_ENV=space NODE_OPTIONS='--max_old_space_size=4096' next build",
    .....
}
```

##### 4.2 jekins发布脚本

###### 4.2.1 打包脚本

```shell
echo $PATH
rm -f web-app.tar.gz
npm config set registry http://registry.npm.taobao.org/repository/npm-group/
yarn config set registry http://registry.npm.taobao.org/repository/npm-group/
rm -rf ./build/
yarn install
npm run space_build
tar -zcf web-app.tar.gz * .[!.]* --exclude=web-app.tar.gz --exclude=.git
```

###### 4.2.2 启动脚本

```shell
#!/bin/bash

source /etc/profile
PROJECT_DIR=/home/data/scm3/web-app
PKG_FILENAME=web-app.tar.gz

if [ ! -d $PROJECT_DIR ];then
  mkdir $PROJECT_DIR
else
  rm -rf $PROJECT_DIR/*
fi
cd  $PROJECT_DIR
mv ../$PKG_FILENAME ./
tar zxvf $PKG_FILENAME
#rm -f $PKG_FILENAME
chown www:www -R $PROJECT_DIR
#yarn start
pm2 delete web-app
pm2 start npm --name "web-app" -- run space
```

##### 4.3 系统服务脚本/etc/init.d/pm2

[脚本地址](https://github.com/lovesyx2012/centosfile/blob/master/pm2-root)

```
https://github.com/lovesyx2012/centosfile/blob/master/pm2-root
```