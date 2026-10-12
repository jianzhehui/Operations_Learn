## 代码仓库构建与CI/CD

#### 1、环境部署

使用ubuntu 22.04server构建

网卡配置
```yaml
network:
  ethernets:
    ens34:
      addresses: [100.100.129.101/24]
      nameservers:
        addresses: [223.5.5.5,223.6.6.6]
      routes:
        - to: default
          via: 100.100.129.1
  version: 2
```

```bash
netplan apply
systemctl start systemd-networkd
systemctl enable systemd-networkd
```

apt源设置

```bash
cp /etc/apt/source.list /etc/apt/source.list.bak
```

```bash
cat > /etc/apt/source.list
```

```bash
deb http://mirrors.pku.edu.cn/ubuntu jammy main restricted universe multiverse
# deb-src http://mirrors.pku.edu.cn/ubuntu jammy main restricted universe multiverse

deb http://mirrors.pku.edu.cn/ubuntu jammy-updates main restricted universe multiverse
# deb-src http://mirrors.pku.edu.cn/ubuntu jammy-updates main restricted universe multiverse

deb http://mirrors.pku.edu.cn/ubuntu jammy-backports main restricted universe multiverse
# deb-src http://mirrors.pku.edu.cn/ubuntu jammy-backports main restricted universe multiverse

deb http://mirrors.pku.edu.cn/ubuntu jammy-security main restricted universe multiverse
# deb-src http://mirrors.pku.edu.cn/ubuntu jammy-security main restricted universe multiverse

## Uncomment the following two lines to add software from Canonical's
## 'partner' repository.
## This software is not part of Ubuntu, but is offered by Canonical and the
## respective vendors as a service to Ubuntu users.
# deb http://archive.canonical.com/ubuntu jammy partner
# deb-src http://archive.canonical.com/ubuntu jammy partner
```

```bash
apt update
```

jdk安装
```bash
apt install -y openjdk-21-jdk
```



```bash
cd /opt
wget https://mirrors.huaweicloud.com/openjdk/21/openjdk-21_linux-x64_bin.tar.gz

tar -zxf openjdk-21_linux-x64_bin.tar.gz
```

```bash
vim /etc/profile.d/jdk21.sh
```

```bash
export JAVA_HOME=/opt/jdk-21
export PATH=$JAVA_HOME/bin:$PATH
```

```bash
source /etc/profile.d/jdk21.sh
update-alternatives --install /usr/bin/java java /opt/jdk-21/bin/java 2100
```

Maven安装

```bash
cd /opt
wget https://mirrors.huaweicloud.com/apache/maven/maven-3/3.9.16/binaries/apache-maven-3.9.16-bin.tar.gz

tar -xzf apache-maven-3.9.16-bin.tar.gz 
```

```bash
vim /etc/profile.d/mvn.sh
```

```bash
export MVN_HOME=/opt/apache-maven-3.9.16
export PATH=$MVN_HOME/bin:$PATH
```

```bash
source /etc/profile.d/mvn.sh
```

更改mvn源

```bash
cp /opt/apache-maven-3.9.16/conf/setting.xml /opt/apache-maven-3.9.16/conf/setting.xml.bak
```

```bash
cat > /opt/apache-maven-3.9.16/conf/setting.xml
```

```bash
<mirror>
    <id>huaweicloud</id>
    <mirrorOf>*</mirrorOf>
    <url>https://mirrors.huaweicloud.com/repository/maven/</url>
</mirror>
```

安装Git

```bash
apt install -y git
```

```bash
git config --global user.name cicd
git config --global user.email cicd@jianzhehui.club
```

安装nodejs

```bash
cd /opt
wget https://registry.npmmirror.com/-/binary/node/v16.20.2/node-v16.20.2-linux-x64.tar.gz

tar -xzf node-v16.20.2-linux-x64.tar.gz
```

```bash
vim /etc/profile.d/node16.sh
```

```bash
export NODE16_HOME=/opt/node16
export PATH=$NODE16_HOME/bin:$PATH
```

```bash
source /etc/profile.d/node16.sh
```

```bash
npm config set registry https://registry.npmmirror.com
```

安装gitlab

```bash
wget https://mirrors.tuna.tsinghua.edu.cn/gitlab-ce/apt/packages.gitlab.com/gitlab/gitlab-ce/ubuntu/jammy/pool/main/g/gitlab-ce/gitlab-ce_19.3.3-ce.0_amd64.deb
```

```bash
apt install gitlab-ce_19.3.3-ce.0_amd64.deb
```

修改配置

```bash
vim /etc/gitlab/gitlab.rb
```

```bash
发信配置
external_url 'http://100.100.129.101:8888' 
gitlab_rails['time_zone'] = 'Asia/Shanghai'

gitlab_rails['smtp_enable'] = true
gitlab_rails['smtp_address'] = "smtp.qiye.aliyun.com"
gitlab_rails['smtp_port'] = 465
gitlab_rails['smtp_user_name'] = "cicd@jianzhehui.club"
gitlab_rails['smtp_password'] = 'password'
gitlab_rails['smtp_domain'] = "jianzhehui.club"      
gitlab_rails['smtp_authentication'] = "login"
gitlab_rails['smtp_enable_starttls_auto'] = false
gitlab_rails['smtp_tls'] = true
gitlab_rails['smtp_pool'] = true

gitlab_rails['gitlab_email_enabled'] = true
gitlab_rails['gitlab_email_from'] = 'cicd@jianzhehui.club'
gitlab_rails['gitlab_email_display_name'] = 'GitLab'
gitlab_rails['gitlab_email_reply_to'] = '2060375680@qq.com'

puma['port'] = 8081
```

内存优化

```bash
prometheus_monitoring['enable'] = false
puma['worker_processes'] = 2
sidekiq['concurrency'] = 4
postgresql['shared_buffers'] = "64MB"
postgresql['max_connections'] = 100
```



```bash
gitlab-ctl reconfigure
gitlab-ctl start
```

查看初始密码

```bash
systemctl start gitlab-runsvdir
systemctl enable gitlab-runsvdir

cat cat /etc/gitlab/initial_root_password
```

在loaclhost:8888上进行登录

修改为中文

头像 - > 偏好设置 - > 语言

修改默认分支为master

管理员 - > 仓库 - > 默认分支

创建项目、群组与用户

推送代码

配置ssh密钥

安装Jenkins

```bash
wget https://mirrors.huaweicloud.com/jenkins/debian/jenkins_2.580_all.deb

apt install /opt/jenkins_2.580_all.deb
```

查看密码

```bash
systemctl start jenkins
systemctl enable jenkins

cat /var/lib/jenkins/secrets/initialAdminPassword
```

在localhost:8080上进行登录

初始化不安装任何插件

配置插件url(插件高级设置)

```bash
https://mirrors.huaweicloud.com/jenkins/update-center.json
```

修改default.json

```bash
cd /var/lib/jenkins/updates
sed -i 's|https://updates.jenkins.io/download|https://mirrors.huaweicloud.com/jenkins|g' /var/lib/jenkins/updates/default.json
```

安装chinese，git，publish over ssh，pipeline

配置邮箱通知

修改系统管理员邮件地址

```bash
cicd@jianzhehui.club
```

E-mail Notification配置

```bash
SMTP server				SMTP Port
smtp.qiye.aliyun.com	 465
```

```bash
Credentials(jenkins)

Domain
全局凭证

类型
Username with password

范围
全局

用户名
cicd@jianzhehui.club

密码
由密码生成器生成
```

生成jenkins用户的公钥

```
su - jenkins
mkdir .ssh

ssh-keygen -t ed25519 -C 'jenkins-deploy-key' -f /var/lib/jenkins/.ssh/id_ed25519_gitee -N ""
cat /var/lib/jenkins/.ssh/id_ed25519_gitee.pub
```

```bash
vim /var/lib/jenkins/.ssh/config
```

```bash
Host gitlab-cicd
	HostName http://100.100.129.101
	User git
	IdentityFile /var/lib/jenkins/.ssh/id_ed25519_gitee
	IdentitiesOnly yes
```

```bash
cat /var/lib/jenkins/.ssh/id_ed25519_gitee.pub
```

将jenkins的公钥也放入仓库

配置publish over ssh

配置实现拉取代码和手工编译打包

添加jenkins凭证
凭据类型：SSH Username with private key
```
Username git
Private Key 生成的密钥
```
新建jenkins任务
选择自由风格软件项目
源码管理选择git
```
Repository URL：git@gitee.com:jianzhehui/eladmin.git
Credentials：gitee-eladmin-readonly
Branches to build：*/master
```
勾选丢弃旧的构建，关闭并发构建
构建环境中选择


```bash
#!/bin/bash
set -euo pipefail

export JAVA_HOME=/opt/jdk-21
export MVN_HOMAE=/opt/maven
export NODE_HOME=/opt/node
export PATH="$JAVA_HOME/bin:$MVN_HOME/bin:$NODE_HOME:$PATH"
export MAVEN_OPTS="-Xmx1024m"

COMMIT_ID=$(git rev-parse HEAD)
OUTPTU_DIR="$WORKSPACE/artifacts/$BUILD_NUMBER"
mkdir -p "$OUTPTU_DIR"

cd "$WORKSPACE/eladmin"
mvn -B -ntp -T 1C -Dmaven.test.skip=true clean package

cd "$WORKSPACE/eladmin-web"
npm config set registry https://registry.npmmirror.com
npm install
npm run build:prod

cd "$WORKSPACE"
cp eladmin/eladmin-system/target/eladmin-system-1.1.jar "$OUTPUT_DIR/eladmin.jar"
cp -r  "$OUTPUT_DIR/dist"

DATE=$(date)
printf "release = %s\ncommit = %s\nbuild_time = %s\n" "$BUILD_NUMBER" "$COMMIT_ID" "$DATE" > "$OUTPUT_DIR/buildinfo.txt"
```

构建后操作
增加构建后操作步骤 -> Editable Email Notification
在Project Recipient List中填入收件邮箱
高级设置中添加trigger，选择Failure - Any,Fixed

增加构建后操作步骤 -> publish over ssh

```bash
Source files
artifacts/$BUILD_NUMBER/**

Remove prefix
artifacts/$BUILD_NUMBER

Remote directory
$BUILD_NUMBER

Exec command
ln -sfn /opt/eladmin/artifacts/$BUILD_NUMBER /opt/eladmin/artifacts/current
```

