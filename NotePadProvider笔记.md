# NotePadProvider笔记

注：NotePadProvider是Google Samples中的一个示例，主要介绍Content Provider用法，之中有不少其他知识点，一并记下，用以日后参考。

-------------------

## Activity设置按键模式setDefaultKeyMode()

setDefaultKeyMode()的作用是设置系统对Activity键盘按键的处理模式，模式有一下几种：
DEFAULT_KEYS_DISABLE  //系统不处理（默认状态）
DEFAULT_KEYS_DIALER 	//键盘输入唤醒拨号界面(输入不管是数字还是其他字符，字符会转化成数字)
DEFAULT_KEYS_SHORTCUT  //menu有设置shortcut，执行对应menu action
DEFAULT_KEYS_SEARCH_LOCAL  //将键盘输入作为搜索内容，进行本地搜索，如果本地没有实现自定义搜索，则使用全局搜索
DEFAULT_KEYS_SEARCH_GLOBAL  //将键盘输入作为搜索内容，进行全局搜索

更详细介绍可参考：
http://blog.csdn.net/silenceburn/article/details/6069988
## Menu动态加载
如果有可以响应某个Intent的应用程序，可以通过addIntentOptions()来动态加载。
函数原型：
public int addIntentOptions(int groupId, int itemId, int order,
                                ComponentName caller, Intent[] specifics,
                                Intent intent, int flags, MenuItem[] outSpecificItems);
参数介绍，主要介绍specifics,intent,outSpecificItems:
specifics:有需要开启响应其他Intent的Activity可以使用这个参数，定义Intent数组，定义的Intent可以是任意的与参数intent无关，按照定义顺序显示。
intent：能够响应的intent
outSpecificItems:如果定义的specific有activity可以响应，那么会输出items
###示例       
``` java
Uri uri = ContentUris.withAppendedId(getIntent().getData(), getSelectedItemId());
// Creates an array of Intents with one element. This will be used to send an Intent
// based on the selected menu item.
Intent[] specifics = new Intent[1];
// Sets the Intent in the array to be an EDIT action on the URI of the selected note.
specifics[0] = new Intent("com.lm.view");
// Creates an array of menu items with one element. This will contain the EDIT option.
MenuItem[] items = new MenuItem[1];
// Creates an Intent with no specific action, using the URI of the selected note.
Intent intent = new Intent(null, getIntent().getData());
/*Adds the category ALTERNATIVE to the Intent, with the note ID URI as its
* data. This prepares the Intent as a place to group alternative options in the
* menu.
**/
intent.addCategory(Intent.CATEGORY_ALTERNATIVE);
/*Add alternatives to the menu*/
menu.addIntentOptions(
	Menu.CATEGORY_ALTERNATIVE,  // Add the Intents as options in the alternatives group.
	Menu.NONE,                  // A unique item ID is not required.
	Menu.NONE,                  // The alternatives don't need to be in order.
	getComponentName(),         // The caller's name is not excluded from the group.
	specifics,                  // These specific options must appear first.
	intent,                     // These Intent objects map to the options in specifics.
	Menu.NONE,                  // No flags are required.
	items                       // The menu items generated from the specifics-to-
                                // Intents mapping
);
``` 
一个Activity配置如下：

```
<activity
            android:name="com.example.android.network.sync.basicsyncadapter.EntryListActivity"
            android:label="@string/app_name" >
            <!-- This intent filter places this activity in the system's app launcher. -->
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>

            <intent-filter android:label="edit">
                <action android:name="com.lm.view"/>
                <category android:name="android.intent.category.DEFAULT"/>
                <category android:name="android.intent.category.ALTERNATIVE"/>
                <category android:name="android.intent.category.SELECTED_ALTERNATIVE"/>
            </intent-filter>
        </activity>
```
这个Activity没有匹配上面的intent，但是specific里指定了，因此也可匹配。
##隐式Intent匹配规则
1、有匹配action或data(至少有一个)
2、`<category android:name="android.intent.category.DEFAULT" />`必须
                       
更详细介绍参考：
http://blog.csdn.net/niu_gao/article/details/7100707

> Markdown 是一种轻量级标记语言，它允许人们使用易读易写的纯文本格式编写文档，然后转换成格式丰富的HTML页面。    —— <a href="https://zh.wikipedia.org/wiki/Markdown" target="_blank"> [ 维基百科 ]

使用简单的符号标识不同的标题，将某些文字标记为**粗体**或者*斜体*，创建一个[链接](http://www.csdn.net)等，详细语法参考帮助？。

本编辑器支持 **Markdown Extra** , 　扩展了很多好用的功能。具体请参考[Github][2].  

### 表格

**Markdown　Extra**　表格语法：

项目     | 价格
-------- | ---
Computer | $1600
Phone    | $12
Pipe     | $1

可以使用冒号来定义对齐方式：

| 项目      |    价格 | 数量  |
| :-------- | --------:| :--: |
| Computer  | 1600 元 |  5   |
| Phone     |   12 元 |  12  |
| Pipe      |    1 元 | 234  |

###定义列表

**Markdown　Extra**　定义列表语法：
项目１
项目２
:   定义 A
:   定义 B

项目３
:   定义 C

:   定义 D

	> 定义D内容

### 代码块
代码块语法遵循标准markdown代码，例如：
``` python
@requires_authorization
def somefunc(param1='', param2=0):
    '''A docstring'''
    if param1 > param2: # interesting
        print 'Greater'
    return (param2 - param1 + 1) or None
class SomeClass:
    pass
>>> message = '''interpreter
... prompt'''
```

###脚注
生成一个脚注[^footnote].
  [^footnote]: 这里是 **脚注** 的 *内容*.
  
### 目录
用 `[TOC]`来生成目录：

[TOC]

### 数学公式
使用MathJax渲染*LaTex* 数学公式，详见[math.stackexchange.com][1].

 - 行内公式，数学公式为：$\Gamma(n) = (n-1)!\quad\forall n\in\mathbb N$。
 - 块级公式：

$$	x = \dfrac{-b \pm \sqrt{b^2 - 4ac}}{2a} $$

更多LaTex语法请参考 [这儿][3].

### UML 图:

可以渲染序列图：

```sequence
张三->李四: 嘿，小四儿, 写博客了没?
Note right of 李四: 李四愣了一下，说：
李四-->张三: 忙得吐血，哪有时间写。
```

或者流程图：

```flow
st=>start: 开始
e=>end: 结束
op=>operation: 我的操作
cond=>condition: 确认？

st->op->cond
cond(yes)->e
cond(no)->op
```

- 关于 **序列图** 语法，参考 [这儿][4],
- 关于 **流程图** 语法，参考 [这儿][5].

## 离线写博客

即使用户在没有网络的情况下，也可以通过本编辑器离线写博客（直接在曾经使用过的浏览器中输入[write.blog.csdn.net/mdeditor](http://write.blog.csdn.net/mdeditor)即可。**Markdown编辑器**使用浏览器离线存储将内容保存在本地。

用户写博客的过程中，内容实时保存在浏览器缓存中，在用户关闭浏览器或者其它异常情况下，内容不会丢失。用户再次打开浏览器时，会显示上次用户正在编辑的没有发表的内容。

博客发表后，本地缓存将被删除。　

用户可以选择 <i class="icon-disk"></i> 把正在写的博客保存到服务器草稿箱，即使换浏览器或者清除缓存，内容也不会丢失。

> **注意：**虽然浏览器存储大部分时候都比较可靠，但为了您的数据安全，在联网后，**请务必及时发表或者保存到服务器草稿箱**。

##浏览器兼容

 1. 目前，本编辑器对Chrome浏览器支持最为完整。建议大家使用较新版本的Chrome。
 3. IE９以下不支持
 4. IE９，１０，１１存在以下问题
    1. 不支持离线功能
    1. IE9不支持文件导入导出
    1. IE10不支持拖拽文件导入

---------

[1]: http://math.stackexchange.com/
[2]: https://github.com/jmcmanus/pagedown-extra "Pagedown Extra"
[3]: http://meta.math.stackexchange.com/questions/5020/mathjax-basic-tutorial-and-quick-reference
[4]: http://bramp.github.io/js-sequence-diagrams/
[5]: http://adrai.github.io/flowchart.js/
[6]: https://github.com/benweet/stackedit