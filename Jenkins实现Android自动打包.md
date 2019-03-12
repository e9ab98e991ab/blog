# Jenkins实现Android自动打包

## 环境搭建

### sdk-tools安装

[下载sdk-tools](https://developer.android.com/studio")

![](/pics/download_sdk_tools.png)

安装完成后使用sdkmanager安装platform-tools和ploatforms

> 构建不需要Android Studio，仅下载命令行工具

### 安装git
略

### Jenkins安装

 1. 安装：略
 2. 插件：Git、Gradle、post build taks(根据构建log执行构建后操作)、thinBackup(配置备份)
 3. 运行：java -jar jenkins.war
 4. 配置环境变量：
 
![](/pics/jenkins_git.jpg)

![](/pics/jenkins_env.jpg)

### shell脚本

 [教程](http://www.runoob.com/linux/linux-shell.html)

 > if[condition]是if test condition的另一种写法
 
 > [[expression]]提供针对字符串比较的高级特性,如
 
 ```
 [["$branchName" == release* || "$branchName" == hotfix*]]
 ```
 
### 项目配置
 
#### 配置

![](/pics/jenkins_oldcar.jpg)

#### 脚本
拉取分支，执行Android构建<br/>
build.sh

```
#!/bin/bash
branchName=$1
versioon=$2
debug=$3
if [[ "$branchName" == release* || "$branchName" == hotfix* ]]
then
	git fetch --all
    git checkout $branchName
    if [ $? -eq 0 ]
    then
        git pull
    else
        echo BUILD FAILED
        echo no branch:$branchName
        exit -1
    fi
elif [[ $version == *.*.* ]]
then
	git fetch --all
    git checkout $version
    if [ $? -eq 0 ]
    then
        git pull
    else
        echo BUILD FAILED
        echo no version:$version
        exit -1
    fi
else
	echo BUILD FAILED
    echo branch is not release or hotfix,version tag is not exit
    exit -1
fi

chmod +x gradlew
if( $debug )
then
	./gradlew clean assembleDebug
else
	./gradlew clean assembleRelease
fi

```

Android构建成功,把apk移到指定目录，生成此次构建的json文件，发送RTX通知给测试人员<br/>
postSuccess.sh

```
#!/bin/bash
echo '構建成功'
branch=$1
v=$2
debug=$3
commitId=$4
if [[ $branch == release* || $branch == hotfix* ]]
    then
    v=${branch#*/}
fi

msg=''
job=$JOB_NAME
dir=${job%%_*}
if $debug
then
	mkdir -p "/Users/addcn/pack/build/app/$job/debug/$v"
    path=$(find . -type f -name '*.apk')
	mv "$path" "/Users/addcn/pack/build/app/$job/debug/$v/$v-debug.apk"
    msg="/app/$job/debug/$v/$v-debug.apk"
    date=$(date +'%Y-%m-%d %H:%M:%S')
    ~/script/apkJson.sh "$v" "$date" "$4" "$msg" "" "$5"
else
	mkdir -p "/Users/addcn/pack/build/app/$job/release/$v"
    path=$(find . -type f -name '*.apk')
    mv "$path" "/Users/addcn/pack/build/app/$job/release/$v/$v-release.apk"
    msg="/app/$job/release/$v/$v-release.apk"
    date=$(date +'%Y-%m-%d %H:%M:%S')
    ~/script/apkJson.sh "$v" "$date" "$4" "" "$msg" "$5"
fi

name=''
msg="https://dl.8891.app"
shift 5
while [ -n "$1" ]
do
    name=$1
    curl "http://eagle.591.com.tw/jalert/alert_action/57/0/$name?msg=$msg"
    shift
done

```

生成json文件<br/>
apkJson.sh

```
#!/bin/bash
source ~/script/decode.sh
file=''
url=''
type=''
mkdir -p "$HOME/pack/build/app/$JOB_NAME/json"
if [ -n "$4" ]
then
    file="$HOME/pack/build/app/$JOB_NAME/json/$1-debug.json"
    url=$4
    type=debug
else
    file="$HOME/pack/build/app/$JOB_NAME/json/$1-release.json"
    url=$5
    type=release
fi
if [ -e $file ]
then
    > $file
fi
commitMsg=$(urldecode $6)
echo "{\"icon\":\"/app/$JOB_NAME/icon.png\",\"name\":\"$JOB_NAME\",\"version\":\"$1\",\"date\":\"$2\",\"git_sha1\":\"$3\",\"download_url\":\"$url\",\"env\":\"$type\",\"description\":\"$commitMsg\"}" > $file

```

对文本进行解码<br/>
decode.sh

```
#!/bin/bash
urldecode() { : "${*//+/ }"; echo -e "${_//%/\\x}"; }
```

> 引用其他脚本：source xxx.sh。之后可以使用这个脚本里定义的方法

构建失败，发送通知和邮件给开发人员<br/>
postFailed.sh

```
#!/bin/bash
echo 發送消息給開發人員
v=$2
branch=$1
msg=''
if [ -n "$branch" ]
then
    msg="$branch 构建失败"
elif [ -n "$v" ]
then
    msg="$v 构建失败"
else
    msg='构建失败:no branch and tag version'
fi
name=''
shift 2
while [ -n "$1" ]
do
    name=$1
curl -X GET -G --data-urlencode "msg=$msg" "http://eagle.591.com.tw/jalert/alert_action/57/0/$name"
    shift
done

```

> 使用curl，如果有特殊字符(中文等)需要转码

### 触发构建

#### 触发链接

如：http://XXX/job/OldCar/buildWithParameters?token=xxxxxx&version=0.0.0&debug=false&message=test

1. version：用于master分支，值为master当前最新tag
2. branchName：用于release或hotfix分支，值为分支名
3. version和branchName二选一
4. debug：release和hotfix分支为true，master分支为false
5. message:commit信息

#### 分支結構
release和hotfix提交后触发构建<br/>
![](/pics/branch.jpg)

## 注意 
如果脚本在windows下编写，在mac上可能出现no such file or directory的错误。<br/>
解决方法
1. vi xxx.sh
2. set fileformat=unix
3. :qw!