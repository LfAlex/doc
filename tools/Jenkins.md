# Jenkins 

版本号：2.277.4

## 安装

### Ubuntu/Debian

```bash
wget -q -O - https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo apt-key add -
```

Then add the following entry in your `/etc/apt/sources.list`:

```tex
deb https://pkg.jenkins.io/debian-stable binary/
```

install jenkins

```bash
sudo apt-get update
sudo apt-get install jenkins
```

### CentOS/Fedora/Red Hat

```bash
 sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
 sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io.key
 yum install jenkins
```

### Offline

安装包与启动脚本run.sh放在同一目录

```shell
export JAVA_HOME=/usr/local/jdk-11.0.11+9
export JENKINS_HOME=/home/jenkins/jenkins-home
JENKINS_WAR=/home/jenkins/jenkins-app/jenkins.war
user=`whoami`
if [ "$user" != "jenkins" ]; then
   echo "must run as user: jenkins"
   exit 1
fi
export LC_ALL=en_US.UTF-8
export LANG=en_US.UTF-8

exec $JAVA_HOME/bin/java -Dfile.encoding=UTF-8  -jar $JENKINS_WAR
```

## 配置

- 将Jenkins设置为启动时启动的守护进程。查看`/etc/init.d/jenkins`获取更多细节
- 创建一个'`jenkins`'用户来运行此服务
- 直接将控制台日志输出到文件`/var/log/jenkins/jenkins.log`。如果您正在解决Jenkins问题，请检查此文件
- /etc/default/jenkins`为启动填充配置参数，例如JENKINS_HOME
- 将Jenkins设置为在端口8080上进行监听。使用浏览器访问此端口以开始配置

## 插件安装

将jenkins-plugins.tar.gz解压到工作主目录plugins下（默认主目录路径:/var/lib/jenkins/plugins），重新启动jenkins。

若为离线安装，在$JENKINS_HOME目录下plugins，解压插件包

注：登录时不要安装plugin，方法：选择安装插件-》点击已选择-》移除掉所有已选项-》点击安装。

## 系统配置

### JDK配置

系统管理-》全局工具配置-》Jdk-》新增jdk-》取消自动安装-》填写jdk路径配置

### Maven配置

系统管理-》全局工具配置-》Maven-》文件系统中的settings文件-》填写文件路径

### Git配置

系统管理-》全局工具配置-》Git-》path to git executable 填写：git，名字自定义

## 常见问题

一、如果安装成功后无法访问，排查问题思路

1. 查看防火墙是否关闭或端口是否暴露
2. 查看请求地址域名是否映射正确，jenkins默认