# Mac配置Android环境变量
##背景
Mac上在Android Studio控制台输入adb会发现adb:command not found的问题。原因是没有配置Android环境变量。
##配置环境变量
打开mac的terminal终端，输入 cd ~/ 【进入当前用户的home目录】

输入 touch .bash_profile 【如果没有.bash_profile这个文件，则创建一个文件】

输入 open .bash_profile 【用文本编辑器打开文件】
在打开的文本编辑器中写入如下代码：
export ANDROID_HOME=android sdk所在目录(如：/Users/lm/Library/Android/sdk)
export PATH=`${PATH}:${ANDROID_HOME}/tools`
export PATH=`${PATH}:${ANDROID_HOME}/platform-tools`

输入 source .bash_profile 【立即生效】


