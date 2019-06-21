# Jenkins和Android环境Docker化

## Docker
[Docker手册](https://yeasy.gitbooks.io/docker_practice/introduction/)

## Shell Script
[Shell教程](https://www.runoob.com/linux/linux-shell.html)

## DockerFile

```
FROM jenkins/jenkins:lts

MAINTAINER LM

ENV ANDROID_HOME=$JENKINS_HOME/android
ENV ACIS_HOME=$ANDROID_HOME/sh
ENV APK_DIR=$ANDROID_HOME/apks
ENV PATH="$ANDROID_HOME/tools/bin:$PATH"

COPY ./sh /usr/share/jenkins/ref/android/sh
COPY ./tools /usr/share/jenkins/ref/android/tools
COPY ./Android模板 /usr/share/jenkins/ref/jobs/Android模板
COPY plugins.txt /usr/share/jenkins/ref/plugins.txt

RUN install-plugins.sh < /usr/share/jenkins/ref/plugins.txt
```

## Problem

- 运行shell脚本去掉sh后缀
解决方法：文件名直接去掉.sh后缀。

- Docker直接copy文件到```$JENKINS_HOME/android/```目录,并且有把host目录挂载到容器的```$JENKINS_HOME```，发现android目录丢失

原因：被指定为挂载点的目录中的文件会隐藏掉，能显示看的是挂载的目录的内容。<br/>
解决方法：源镜像```jenkins/jenkins:lts```在启动过程```jenkins-support```脚本会把```/usr/share/jenkins/ref/```目录下的文件拷贝到```$JENKINS_HOME```。所以先把android目录拷贝到```/usr/share/jenkins/ref/```。

- git命令error后面脚本不再执行
解决方法：模拟try catch

```
tag=''
msg=''
{
    tag=$(git tag --points-at $commitId)
    msg="${tag} build fail"
} || {}

# 放在||后的{}不会执行
if [ -z "$tag"]; then
    msg="${commitId} build fail"
fi
```
 
- 以root身份进入容器

```
# 使用 -u 参数
docker exec -it -u 0 jenkins /bin/bash
```

- 重命名容器

```
docker rename 原容器名  新容器名
```

- 挂载卷Wrong volume permissions
原因：jenkins用户没有操作volume的权限<br/>
解决方法：先创建目录并改变用户所有者为jenkins
```
mkdir -p /jenkins/jenkins_home
chown 1000:1000 /jenkins
```

- unable to prepare context: unable to 'git clone' to temporary context directory: error initializing submodules
原因：centos git版本太低，升级到git2.x
```
# Install WANDisco repo package
yum install http://opensource.wandisco.com/centos/7/git/x86_64/wandisco-git-release-7-2.noarch.rpm

# install git2.x
yum install git
```

- 超过100M文件不能上传github
解决方法：使用glf(Git large Fil Storage)

```
# 安装gls
brew install git-gls

# 管理大文件
git lfs track "*.jar"

# 上面命令生成的.gitattributes 推送到远程仓库
git add .gitattributes
git commit -m "add gitattributes"
git push origin addcn

# 之后就像往常一样commit和push文件
# 使用SourceTree如果有问题，可以尝试Terminal
```

##[完整实现](https://github.com/wslaimin/JADocker)