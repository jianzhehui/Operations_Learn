# ELADMIN CI/CD 搭建保姆级教程（Gitee + Jenkins + 腾讯企业邮）

> 适用范围：业务服务器（Ubuntu 22.04，已跑 ELADMIN）+ 独立 CI/CD 服务器（CentOS 7.9）。
> 代码仓库：Gitee；CI/CD：Jenkins；通知：腾讯企业邮。
> 网络：CI/CD 与业务服务器内网互通（可 SSH 直连）；CI/CD 无公网 IP（Gitee webhook 打不进来，自动打版用**轮询**）。

---

## 0. 架构总览与信息规划表

```
开发机(Windows,能上外网) ──push──▶ Gitee 云仓库(eladmin / eladmin-web)
                                       ▲
                                       │ 轮询(SCM poll：git ls-remote)
CI/CD 服务器(CentOS7.9, Jenkins) ───────┘
        │ ssh/scp（内网直连）
        ▼
业务服务器(Ubuntu22.04)  ELADMIN 后端(8000) + 前端(Nginx 80)
```

| 项 | 值（示例，按实际替换） |
|---|---|
| Gitee 组织/账号 | `yourname` |
| 后端仓库 | `eladmin`（JPA 版，Spring Boot 2.7） |
| 前端仓库 | `eladmin-web` |
| 主分支 | `master` |
| CI/CD 服务器 | CentOS 7.9，内网 IP `192.168.1.20` |
| 业务服务器 | Ubuntu 22.04，内网 IP `192.168.1.10` |
| Jenkins 端口 | `8080`（只对内网，不暴露公网） |
| JDK | Temurin **17**（Jenkins 与后端构建共用） |
| Maven | 3.9.x（阿里云镜像） |
| Node.js | 16.x（前端构建，官方要求 12~16） |
| 腾讯企业邮 SMTP | `smtp.exmail.qq.com:465`（SSL） |

> **本次先交付**（第 1~4 阶段）：Gitee 能拉代码 + CI/CD 机能手工编译打包。
> **长期**（第 5 阶段）：自动打版 / 手工打版 / 秒级回滚 / 邮件通知。

---

## 1. 阶段一：Gitee 代码仓库

### 1.1 建仓库

Gitee 上新建两个**私有**仓库：`eladmin`（后端）、`eladmin-web`（前端）。主分支默认 `master`。

### 1.2 开发机上传代码

在**开发机（Windows）**上，代码源就是本地这份 `eladmin_vip`（`eladmin` 后端、`eladmin-web` 前端）。

**后端：**

```bash
cd eladmin
git init
git remote add origin https://gitee.com/yourname/eladmin.git
git add .
git commit -m "init: eladmin backend"
git branch -M master
git push -u origin master
```

**前端：**

```bash
cd eladmin-web
git init
git remote add origin https://gitee.com/yourname/eladmin-web.git
git add .
git commit -m "init: eladmin web"
git branch -M master
git push -u origin master
```

### 1.3 上传前必须处理：配置与敏感信息

1. **前端 `.env.production` 先改对再提交**（走 Nginx 代理路线）：

   ```ini
   ENV = 'production'
   VUE_APP_BASE_API = '/'
   VUE_APP_WS_API = '/'
   ```

   > 不要提交 `https://eladmin.vip` 或 `/api` 这类错误值（前面的坑）。

2. **`.gitignore`**：不要把 `target/`、`node_modules/`、`dist/`、`nohup.out`、IDE 文件提交进去。

3. **敏感信息不入库**：后端 `application-prod.yml` 里的数据库密码、Redis 密码，建议改成环境变量占位符（如 `${DB_PASSWORD}`），真实值只放在业务服务器本机。学习阶段嫌麻烦也要心里有数：**一旦仓库公开，密码即泄露**。

### 1.4 为 CI/CD 建只读密钥（部署公钥）

CI/CD 只拉代码，不推送 → 用 Gitee「部署公钥」的**只读**权限，比账号密码更安全、更细粒度。

在 **CI/CD 服务器**生成密钥（哪个用户执行 git 就用哪个用户的 home，Jenkins 用 `jenkins` 用户）：

```bash
ssh-keygen -t ed25519 -C "cicd-deploy-key" -f ~/.ssh/id_ed25519_gitee -N ""
cat ~/.ssh/id_ed25519_gitee.pub   # 复制输出的公钥
```

Gitee 上：进入**每个仓库 → 管理 → 部署公钥 → 添加部署公钥** → 粘贴公钥 → **不勾选「允许写入」**（即只读）→ 确定。两个仓库都加。

测试（SSH 协议，22 端口）：

```bash
git clone git@gitee.com:yourname/eladmin.git
git clone git@gitee.com:yourname/eladmin-web.git
```

> 能 clone 成功 = 「仓库可拉代码」这一项完成。部署公钥是**单仓库**的：同一把公钥要分别加到两个仓库。

---

## 2. 阶段二：CI/CD 服务器基础环境（CentOS 7.9）

> 提示：CentOS 7 已于 2024-06 EOL，安全补丁停止。生产建议后续迁移到 Rocky/AlmaLinux 9；本次按你的要求仍用 CentOS 7.9。

### 2.1 系统准备

```bash
timedatectl set-timezone Asia/Shanghai
systemctl status firewalld   # 防火墙先开着，后面只放行内网段
```

### 2.2 安装 JDK 17（Temurin）

Jenkins 当前 LTS 需要 Java 17/21；后端 Spring Boot 2.7 也支持 17，**一套 JDK 两边共用**，最省资源。

```bash
mkdir -p /opt/java
cd /tmp
curl -L -o jdk17.tar.gz "https://api.adoptium.net/v3/binary/latest/17/ga/linux/x64/jdk/hotspot/normal/eclipse"
tar -xzf jdk17.tar.gz -C /opt/java
mv /opt/java/jdk-17* /opt/java/jdk-17

# 注册到系统，让 `java` 指向 17
alternatives --install /usr/bin/java java /opt/java/jdk-17/bin/java 1
alternatives --install /usr/bin/javac javac /opt/java/jdk-17/bin/javac 1
alternatives --set java /opt/java/jdk-17/bin/java

cat > /etc/profile.d/java.sh <<'EOF'
export JAVA_HOME=/opt/java/jdk-17
export PATH=$JAVA_HOME/bin:$PATH
EOF
source /etc/profile.d/java.sh
java -version   # 应显示 17.x
```

### 2.3 安装 Maven（阿里云镜像）

```bash
cd /opt
curl -L -o maven.tar.gz "https://mirrors.aliyun.com/apache/maven/maven-3/3.9.9/binaries/apache-maven-3.9.9-bin.tar.gz"
tar -xzf maven.tar.gz
mv apache-maven-3.9.9 maven
cat > /etc/profile.d/maven.sh <<'EOF'
export MAVEN_HOME=/opt/maven
export PATH=$MAVEN_HOME/bin:$PATH
EOF
source /etc/profile.d/maven.sh
```

阿里云镜像写入 `settings.xml`（加速依赖下载，国内必做）：

```xml
<settings>
  <mirrors>
    <mirror>
      <id>aliyunmaven</id>
      <mirrorOf>central</mirrorOf>
      <name>aliyun public</name>
      <url>https://maven.aliyun.com/repository/public</url>
    </mirror>
  </mirrors>
</settings>
```

### 2.4 安装 Node.js 16（系统级，root 和 jenkins 用户都能用）

```bash
cd /opt
curl -L -o node.tar.xz "https://mirrors.aliyun.com/nodejs-release/v16.20.2/node-v16.20.2-linux-x64.tar.xz"
tar -xf node.tar.xz
mv node-v16.20.2-linux-x64 node
ln -sf /opt/node/bin/node /usr/local/bin/node
ln -sf /opt/node/bin/npm  /usr/local/bin/npm
npm config set registry https://registry.npmmirror.com   # 国内镜像
node -v && npm -v
```

### 2.5 安装 Git

```bash
yum install -y git
git --version
```

---

## 3. 阶段三：安装 Jenkins（省资源 + 安全）

### 3.1 安装（官方 LTS 仓库）

```bash
wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key
yum install -y jenkins
```

> 若因 CentOS 7 EOL 导致仓库拉不到，备用方案：直接下 LTS war 用 `java -jar jenkins.war` 跑，其余配置相同。

### 3.2 启动与初始密码

```bash
systemctl start jenkins
systemctl enable jenkins
cat /var/lib/jenkins/secrets/initialAdminPassword   # 初始密码
```

浏览器打开 `http://<CI/CD内网IP>:8080`，输入初始密码 → 创建管理员账号。

### 3.3 插件：只装必要的（关键省资源点）

首次安装选 **「选择插件来安装」**，只勾：

- Git（拉代码）
- Email Extension Plugin（发腾讯企业邮通知）
- Publish Over SSH（部署到业务机）
- （可选）Build Timestamp Plugin（版本号带时间）

其余全部不装。装完后到「系统管理 → 插件管理 → 已安装」把没用的插件删掉。

### 3.4 安全与省资源清单（对照执行）

| 项 | 位置 | 操作 |
|---|---|---|
| 禁止匿名访问 | 系统管理 → 安全 → 授权策略 | 改成「登录用户可以做任何事」，取消匿名 |
| 关闭 Agent 端口 | 系统管理 → 安全 → Agent | TCP 入站端口选「**禁用**」（单机跑，不需要 agent） |
| 限制 JVM 内存 | `/etc/sysconfig/jenkins` | `JENKINS_JAVA_OPTIONS="-Xms256m -Xmx1024m -Djava.awt.headless=true"` |
| 8080 只对内网 | firewalld | 只放行内网网段访问 8080，不对外网 |
| 丢弃旧构建 | 每个 Job 配置 | 「丢弃旧的构建」→ 保留最近 10 次，删旧产物（省磁盘） |
| 关闭使用统计/遥测 | 系统管理 → 系统配置 | 取消「向 Jenkins 发送使用统计」 |
| 关闭插件自动更新提示 | 系统管理 → 插件管理 | 不勾选自动检查更新（按需手动更） |

改完 `systemctl restart jenkins`。

### 3.5 配置腾讯企业邮 SMTP（发通知）

前置：登录腾讯企业邮后台 → 该邮箱账号开启 **SMTP 服务** → 生成**客户端专用密码（授权码）**，记下来。

Jenkins：**系统管理 → 系统配置 → Extended E-mail Notification**：

```
SMTP server:   smtp.exmail.qq.com
SMTP Port:     465
Use SSL:       勾选
用户名:        cicd@yourcompany.com        (发件邮箱)
密码:          <客户端专用密码/授权码>      (不是邮箱登录密码)
Default Recipients: 运维收件人@yourcompany.com
```

同时在「系统管理 → 系统配置」顶部设置 **Jenkins URL** 和 **系统管理员邮件地址**（发件人）。点「通过发送测试邮件测试配置」验证，收到即成功。

---

## 4. 阶段四（当前交付）：手工编译打包

目标：在 CI/CD 机上能拉代码、手动把后端打成 jar、前端打成 dist。

### 4.1 后端手动构建

```bash
mkdir -p /data/build && cd /data/build
git clone git@gitee.com:yourname/eladmin.git
cd eladmin
mvn clean package -DskipTests
ls -l eladmin-system/target/eladmin-system-2.7.jar   # 产物
```

> 产物就是 `eladmin-system-2.7.jar`，和业务服务器上跑的是同一个。

### 4.2 前端手动构建

```bash
cd /data/build
git clone git@gitee.com:yourname/eladmin-web.git
cd eladmin-web
# 确认 .env.production 为 VUE_APP_BASE_API='/'
npm install
npm run build
ls -d dist/    # 产物
```

> 到这一步，「仓库能拉 + 机器能手工编译打包」两项当前要求即完成。

---

## 5. 阶段五（长期）：Jenkins 流水线 + 秒级回滚

### 5.1 部署到业务机的准备（SSH 免密 + 版本化目录）

**① CI/CD → 业务机 SSH 免密：**

```bash
# CI/CD 机，jenkins 用户
sudo -u jenkins ssh-keygen -t ed25519 -C "jenkins-deploy" -f /var/lib/jenkins/.ssh/id_ed25519_deploy -N ""
cat /var/lib/jenkins/.ssh/id_ed25519_deploy.pub
```

业务机上建**最小权限部署账号**，把上面公钥加到它的 `~/.ssh/authorized_keys`：

```bash
# 业务机（Ubuntu）
useradd -m -s /bin/bash deploy
mkdir -p /home/deploy/.ssh && chmod 700 /home/deploy/.ssh
echo "<CI/CD的公钥>" >> /home/deploy/.ssh/authorized_keys
chmod 600 /home/deploy/.ssh/authorized_keys && chown -R deploy:deploy /home/deploy/.ssh
```

**② 业务机版本化目录（秒级回滚的基石）：**

```
/opt/eladmin/
├── releases/
│   ├── <版本号1>/  (eladmin-system-2.7.jar + dist/)
│   └── <版本号2>/  ...
├── current -> releases/<版本号2>      # 软链指向当前生效版本
```

- Nginx `root` 改为 `/opt/eladmin/current/dist`
- 后端 systemd `ExecStart=/usr/bin/java -jar /opt/eladmin/current/eladmin-system-2.7.jar --spring.profiles.active=prod`

> 版本号规则：`v$(date +%Y%m%d%H%M%S)-${GIT_COMMIT前7位}`。数据（MySQL/Redis）不在版本目录里，切换版本不影响数据。

### 5.2 自动打版 + 手工打版（Pipeline）

Jenkins 新建 Pipeline 任务，`Jenkinsfile` 放仓库根目录（或直接写在任务里）。要点：

- **自动触发**：无公网 IP，用「轮询 SCM」，`H/5 * * * *`（每 5 分钟 git ls-remote 检查 master 是否有新提交）
- **手工触发**：同任务加参数（`BUILD NOW` 或 `Build with Parameters` 选分支/版本号）

示例（示意，路径按实际改）：

```groovy
pipeline {
  agent any
  triggers { pollSCM('H/5 * * * *') }          // 自动：轮询 master
  parameters {
    choice(name: 'BRANCH', choices: ['master'], description: '分支')
    booleanParam(name: 'DEPLOY', defaultValue: true, description: '是否部署到业务机')
  }
  environment {
    VERSION = "v${new Date().format('yyyyMMddHHmmss')}-${env.GIT_COMMIT?.take(7)}"
    RELEASE = "/opt/eladmin/releases/${env.VERSION}"
  }
  stages {
    stage('拉代码') {
      steps { checkout scmGit(branches: [[name: "*/${params.BRANCH}"]],
              userRemoteConfigs: [[url: 'git@gitee.com:yourname/eladmin.git']]) }
    }
    stage('后端打包') {
      steps { sh 'mvn clean package -DskipTests' }
    }
    stage('前端打包') {
      steps {
        dir('web') {
          checkout scmGit(branches: [[name: "*/${params.BRANCH}"]],
                  userRemoteConfigs: [[url: 'git@gitee.com:yourname/eladmin-web.git']])
          sh 'npm install && npm run build'
        }
      }
    }
    stage('归档') {
      steps {
        sh "mkdir -p ${RELEASE}"
        sh "cp eladmin-system/target/eladmin-system-2.7.jar ${RELEASE}/"
        sh "cp -r web/dist ${RELEASE}/dist"
      }
    }
    stage('部署') {
      when { expression { params.DEPLOY } }
      steps {
        sh """
          ssh deploy@192.168.1.10 "mkdir -p ${RELEASE}"
          scp -r ${RELEASE}/ deploy@192.168.1.10:/opt/eladmin/releases/
          ssh deploy@192.168.1.10 "ln -sfn ${RELEASE} /opt/eladmin/current \\
            && sudo systemctl restart eladmin && sudo nginx -s reload"
        """
      }
    }
  }
  post {
    success { emailext(subject: "构建成功 ${env.VERSION}",
              body: "仓库 master 更新，版本 ${env.VERSION} 已构建${params.DEPLOY ? '并部署' : ''}。",
              to: 'ops@yourcompany.com') }
    failure { emailext(subject: "构建失败 ${env.JOB_NAME}",
              body: "详见 ${env.BUILD_URL}", to: 'ops@yourcompany.com') }
  }
}
```

> 说明：CI/CD 只有**只读**部署公钥，所以「打版」= 本地按版本号归档产物；如需在 Gitee 上打 tag，需另配一个**独立的写权限凭证**，与只读密钥分开（安全起见建议不打 tag 或单独账号）。

### 5.3 秒级回滚

单独建一个「回滚」Pipeline 任务，参数选目标版本：

```groovy
pipeline {
  agent any
  parameters {
    string(name: 'VERSION', description: '要回滚到的版本号，例如 v20250101120000-abc1234')
  }
  stages {
    stage('回滚') {
      steps {
        sh "ssh deploy@192.168.1.10 'ln -sfn /opt/eladmin/releases/${params.VERSION} /opt/eladmin/current && sudo systemctl restart eladmin && sudo nginx -s reload'"
      }
    }
  }
  post {
    success { emailext(subject: "已回滚到 ${params.VERSION}", body: "回滚完成", to: 'ops@yourcompany.com') }
  }
}
```

> 「秒级」原理：切换版本只是 `ln -sfn`（软链指向旧版本）+ 重启服务，不重新构建、不迁移数据。

### 5.4 版本清理（省磁盘）

定期只保留最近 N 个版本，防止 releases 目录无限增长：

```bash
# 业务机 crontab：保留最近 10 个版本
0 3 * * 0  ls -1dt /opt/eladmin/releases/*/ | tail -n +11 | xargs -r rm -rf
```

---

## 6. 常见问题速查

| 现象 | 原因/处理 |
|---|---|
| Jenkins 起不来，日志报 Java 版本 | 必须 Java 17/21；`alternatives --set java /opt/java/jdk-17/bin/java` |
| clone 报 `Permission denied (publickey)` | 部署公钥没加对仓库，或 key 放在了错误的用户 home 下（Jenkins 用 `jenkins` 用户） |
| 邮件测试失败 | 用「客户端专用密码/授权码」不是登录密码；端口 465 勾 SSL |
| 轮询不触发构建 | 触发器用 `pollSCM`，且 Git 插件凭据要能 ssh 拉到 Gitee |
| npm 安装慢 | `npm config set registry https://registry.npmmirror.com` |
| 回滚后前端还是旧的 | nginx 缓存/没 reload：`sudo nginx -s reload`，浏览器强刷 |
