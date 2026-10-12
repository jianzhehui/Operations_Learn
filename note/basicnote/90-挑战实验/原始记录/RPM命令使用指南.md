# RPM命令使用指南
## RPM 安装操作
```bash
rpm -i 需要安装的包名
```
示例
```bash
rpm -i example.rpm 
#安装 example.rpm 包

rpm -iv example.rpm 
#安装 example.rpm 包并在安装过程中显示正在安装的文件信息

rpm -ivh example.rpm 
#安装 example.rpm 包并在安装过程中显示正在安装的文件信息及安装进度
```
## RPM 查询操作
```bash
rpm -q
#附加查询命令
a 查询所有已经安装的包

以下两个附加命令用于查询安装包的信息；
i 显示安装包的信息；
l 显示安装包中的所有文件被安装到哪些目录下；
s 显示安装版中的所有文件状态及被安装到哪些目录下

以下两个附加命令用于指定需要查询的是安装包还是已安装后的文件；
p 查询的是安装包的信息；
f 查询的是已安装的某文件信息；
```
示例
```bash
rpm -qa | grep tomcat4 
#查看 tomcat4 是否被安装；

rpm -qip example.rpm 
#查看 example.rpm 安装包的信息；

rpm -qif /bin/df 
#查看/bin/df 文件所在安装包的信息；

rpm -qlf /bin/df 
#查看/bin/df 文件所在安装包中的各个文件分别被安装到哪个目录下；
```
## RPM 卸载操作
```bash
rpm -e 需要卸载的安装包

rpm -e tomcat4
#卸载tomcat4
```
## RPM 升级操作
```bash
rpm -U

rpm -Uvh example.rpm
#升级软件包
```