# ELADMIN部署（传统方式）

##  1、源码打包准备

#### 1.1、项目最低要求

JDK 1.8+

Redis 3.0+

Maven 3.0+（开发机后端打包要求）

```
https://maven.apache.org/download.cgi
```

windows下载.zip版本，解压到任意位置

MySQL 5.5+

Node 10+，推荐 12～16（开发机前端打包要求）

使用nvm管理不同node版本

```
https://www.nvmnode.com/guide/download.html
```

将nvm配置到环境变量中

```JavaScript
nvm instll 16
#安装对应版本的node
```

```
nvm use 版本号
#切换到对应版本的node
```

npm换源

```
npm config set registry https://registry.npmmirror.com
```

Spring Boot 2.7.18

#### 1.2、准备源码

```bash
git clone https://gitee.com/elunez/eladmin-mp.git
```

#### 1.3、打包前后端

打包后端

```
cd eladmin
#进入后端项目
```

修改生产配置

```
eladmin/eladmin-system/src/main/resources/config/application-prod.yml
```

```yaml
spring:
  datasource:
	  url: allowPublicKeyRetrieval=true
      username: ${DB_USER:root}
      password: ${DB_PWD:Ifcfg-ens224}
#修改数据库密码
```

```powershell
mvn clean package -DskipTests
#执行打包
```

```
eladmin/eladmin-system/target/eladmin-system-版本号.jar
#jar位置
```

打包前端

```
cd eladmin-web
#进入后端
```

修改生产配置

```
.env.production
```

```
VUE_APP_BASE_API  = '/'
VUE_APP_WS_API = '/'
```

```
npm install
#安装依赖

npm run build:prod
#执行打包
```

```
eladmin-web/dist
#文件夹位置
```

## 2、服务器环境部署

#### 2.1、配置yum源

```bash
curl -o /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-7.repo

curl -o /etc/yum.repos.d/epel.repo https://mirrors.aliyun.com/repo/epel-7.repo

sed -i.bak \
  -e 's|^mirrorlist=|#mirrorlist=|g' \
  -e 's|^#baseurl=http://mirror.centos.org/centos|baseurl=https://mirrors.ustc.edu.cn/centos-vault/centos|g' \
  /etc/yum.repos.d/CentOS-Base.repo

yum clean all
yum makecache
```

#### 2.2、安装JDK 8

```bash
yum install -y java-1.8.0-openjdk
```

#### 2.3、安装MySQL

```bash
wget -i -c https://dev.mysql.com/get/mysql80-community-release-el7-10.noarch.rpm

yum install -y mysql80-community-release-el7-10.noarch.rpm
yum install mysql-community-server --nogpgcheck
```

修改密码

```bash
vim /etc/my.cnf
```

```bash
skip-grant-tables
#跳过登录检查
```

```sql
mysql -uroot -p

flush privileges;

alter user 'root'@'localhost' identified by 'Ifcfg-ens224'
set password for 'root'@'localhost' = 'Ifcfg-ens224';
#修改密码

create database eladmin;

flush privileges;
exit;
```

修改密码后将跳过检查注释掉

```bash
systemctl start mysqld
systemctl enable mysqld
```

#### 2.4、安装nginx与redis

```bash
yum install -y redis nginx
```

```bash
systemctl start nginx redis
systemctl enable nginx redis
```

验证redis正常运行

```bash
redis-cli
```

```bash
ping
```

#### 2.5、创建部署目录

```bash
mkdir -p /opt/eladmin/backend
mkdir -p /opt/eladmin/sql
mkdir -p /var/www/eladmin
```

上传文件

```javascript
后端 Jar    ->   /opt/eladmin/backend/eladmin.jar
sql/eladmin.sql  ->  /opt/eladmin/sql/eladmin.sql
dist/     ->    /var/www/eladmin
```

导入数据库

```bash
mysql -uroot -p'Ifcfg-ens224' eladmin  < /opt/eladmin/sql/eladmin.sql
```

## 3、项目启动

#### 3.1、Nginx配置

```bash
vim /etc/nginx/conf.d/eladmin.conf
```

```nginx
server{
        listen 80;
        server_name 100.100.129.100;
    
        index index.html;
        root  /var/www/eladmin/dist;  

        location / {
          try_files $uri $uri/ @router;
          index  index.html;
        }

        location @router {
          rewrite ^.*$ /index.html last;
        }

        location /api {
          proxy_pass http://127.0.0.1:8000;
          proxy_set_header Host $host;
          proxy_set_header X-Real-IP $remote_addr;
          proxy_set_header X-Forwarded-Proto $scheme;
          proxy_set_header X-Forwarded-Port $server_port;
          proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
          proxy_set_header Upgrade $http_upgrade;
          proxy_set_header Connection "upgrade";
        }

        location /auth {
          proxy_pass http://127.0.0.1:8000;
          proxy_set_header Host $host;
          proxy_set_header X-Real-IP $remote_addr;
          proxy_set_header X-Forwarded-Proto $scheme;
          proxy_set_header X-Forwarded-Port $server_port;
          proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
          proxy_set_header Upgrade $http_upgrade;
          proxy_set_header Connection "upgrade";
        }

        location /webSocket {
          proxy_redirect off;
          proxy_pass http://127.0.0.1:8000/webSocket;
          proxy_http_version 1.1;
          proxy_set_header Upgrade $http_upgrade;
          proxy_set_header Connection "upgrade";
          proxy_set_header Host $http_host;
          proxy_set_header X-Real-IP $remote_addr;
          proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
          proxy_connect_timeout 60s;
          proxy_read_timeout 86400s;
          proxy_send_timeout 60s;
        }

        location /avatar {
          proxy_pass http://127.0.0.1:8000;
        }

        location /file {
          proxy_pass http://127.0.0.1:8000;
        }
    
    	location /druid/ {
          proxy_pass http://127.0.0.1:8000;
          proxy_set_header Host $host;
          proxy_set_header X-Real-IP $remote_addr;
          proxy_set_header X-Forwarded-Proto $scheme;
          proxy_set_header X-Forwarded-Port $server_port;
          proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    	}
    	
}

```

#### 3.2、脚本编写

启动脚本

```bash
vim /opt/eladmin/start.sh
```

```bash
#!/bin/bash
nohup java -jar /opt/eladmin/backend/eladmin-system-1.1.jar --spring.profiles.active=prod > nohup.out 2>&1 &
```

停止脚本

```bash
vim /opt/eladmin/stop.sh
```

```bash
#!/bin/bash
PID=$(ps -ef | grep /opt/eladmin/backend/eladmin-system-1.1.jar | grep -v grep | awk '{ print $2 }')
if [ -z "$PID" ]
then
echo Application is already stopped
else
echo kill -9 $PID
kill -9 $PID
fi
```

查看日志脚本

```
vim /opt/eladmin/log.sh
```

```bash
#!/bin/bash
tail -f nohup.out
```

赋予权限

```bash
chmod 755 start.sh stop.sh log.sh
```

## 4、实现服务开机自启

配置环境变量

```bash
vim /etc/eladmin/eladmin.env
```

```bash
DB_HOST=127.0.0.1
DB_NAME=eladmin
DB_PORT=3306
DB_USER=root
DB_PWD=Ifcfg-ens224

REDIS_host=127.0.0.1
REDIS_PORT=6379
REDIS_DB=1
```

配置服务文件

```bash
vim/systemd/system/eladmin.service
```

```bash
[Unit]
Description=Eladmin Backend
After=network.target mysqld.service redis.service
Wants=network.target mysqld.service redis.service

[Service]
Type=simple
User=root
WorkingDirectory=/opt/eladmin

EnvironmentFile=/etc/eladmin/eladmin.env

ExecStart=/usr/bin/java \
		  -Xms256m -Xmx1024m \
		  -Dfile.encoding=UTF-8 \
		  -jar /opt/eladmin/backend/eladmin-system-1.1.jar \
		  --spring.profiles.active=prod
		  
Restart=on-failure
RestartSec=10
SuccessExitStatus=143

[Install]
WantedBy=multi-user.target
```

检查语法错误

```bash
systemd-analyze verify /etc/systemd/system/eladmin.service
```

```bash
systemd start eladmin
```

如有错误查看报错信息

```bash
journalctl -u eladmin -n 200 --no-pager
```

查看日志

## 5、验证测试

测试前端

```bash
curl -I http://127.0.0.1/
```

应返回200

测试后端

```bash
curl -i http://127.0.0.1/api/auth/code
```

浏览器访问`http://100.100.129.100/`



