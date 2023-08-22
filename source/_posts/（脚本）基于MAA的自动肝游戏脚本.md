---
title: （脚本）基于MAA的自动肝游戏脚本
date: 2023-08-22 01:02:36
update:
tags:
- 实践
- Python
- 脚本
categories:
keywords:
- Python
- MAA
- 明日方舟
top_img:
cover:
---
# 什么是MAA

[Meo Assistant Arknights](https://github.com/MaaAssistantArknights/MaaAssistantArknights)（现改名为Maa Assistant Arknights）是一款由[MistEO](https://github.com/MistEO)制作的基于**图像识别**技术，可以一键完成全部日常任务的明日方舟游戏小助手！

<img src="https://s2.loli.net/2023/08/22/3nq7EP9YQOmlJLW.png" alt="MAA GUI" style="zoom: 33%;"/>

`MAA`已经封装了简单易用的[接口](https://github.com/MaaAssistantArknights/MaaAssistantArknights/tree/dev/src)，我们可以通过查看附带的范例和阅读[集成文档](https://maa.plus/docs/3.1-%E9%9B%86%E6%88%90%E6%96%87%E6%A1%A3.html)中的介绍可以进一步了解它的工作方式。

# Python初调试
安装好`MAA`后目录内已经附带了一个Python的demo，一般是在这个位置。

> MAA\Python\sample.py

ps: 为了保险起见，个人建议最好还是先在外面通过`MAA.exe`直接启动GUI检查`MAA`是否能正常正常工作再开始脚本调试。

打开脚本后进行初配置。

```python
# 请设置为存放 dll 文件及资源的路径
path = pathlib.Path(__file__).parent.parent

# 设置更新器的路径和目标版本并更新
Updater(path, Version.Stable).update()

# 外服需要再额外传入增量资源路径，例如
# incremental_path=path / 'resource' / 'global' / 'YoStarEN'
Asst.load(path=path)

# 若需要获取详细执行信息，请传入 callback 参数
# 例如 asst = Asst(callback=my_callback)
asst = Asst()

# 设置额外配置
# 触控方案配置
asst.set_instance_option(InstanceOptionType.touch_type, 'maatouch')
# 暂停下干员
# asst.set_instance_option(InstanceOptionType.deployment_with_pause, '1')

# 启动模拟器。例如启动蓝叠模拟器的多开Pie64_1，并等待30s
Bluestacks.launch_emulator_win(r'C:\Program Files\BlueStacks_nxt\HD-Player.exe', 30, "Pie64_1")

# 获取Hyper-v蓝叠的adb port
# port = Bluestacks.get_hyperv_port(r"C:\ProgramData\BlueStacks_nxt\bluestacks.conf", "Pie64_1")

# 请自行配置 adb 环境变量，或修改为 adb 可执行程序的路径
if asst.connect('adb.exe', '127.0.0.1:5555'):
    print('连接成功')
else:
    print('连接失败')
    exit()
```

这部分除了`模拟器启动参数`和`adb环境变量`需要检查外，其他地方如果没有特殊需求可以不用动。*（脚本如果要移动到位置记得改`dll path`）*

然后是下面就是一组添加到任务列表的结构。

```python
asst.append_task(AsstHandle handle, const char* type, const char* params)

asst.append_task(...)
asst.append_task(...)
...

asst.start()

while asst.running():
    time.sleep(0)

# AsstHandle handle
# 实例句柄
# const char* type
# 任务类型
# const char* params
# 任务参数，json string

# 返回值：AsstTaskId
# 若添加成功，返回该任务 ID, 可用于后续设置任务参数；
# 若添加失败，返回 0
```

为了方便理解，此处展示的是接口原型。在demo中已经写好了`一键长草`的任务流程，初调试的时候不用动。改造的流程后面会介绍，现在只要确定整个流程能正常执行就可以了。

一切准备就绪直接启动运行

```cmd
python sample.py
```

**（为保证资源正常释放，中止程序运行会连带关闭模拟器！）**

不出意外的话能跑通整个流程的话就可以开始根据自己的需求修改脚本了。

# 魔改思路

在前面加个判断，这样写还能在模拟器已经开启时节省时间，还能解决模拟器连不上这种小概率事件，

```python
while not asst.connect('adb.exe', '127.0.0.1:5555'):
    print('没有检测到可用的连接！正在尝试启动模拟器……')
    Bluestacks.launch_emulator_win(r'C:\Program Files\BlueStacks_nxt\HD-Player.exe', 10, "Pie64_1")
    port = Bluestacks.get_hyperv_port(r"C:\ProgramData\BlueStacks_nxt\bluestacks.conf", "Pie64_1")        

print('连接成功')
```

因为我只在`官服`和`B服`各有一个号，所以只需要遍历添加每个任务，不涉及模拟器多开。

```python
servers = ["Official","Bilibili"]

...

for select_server in servers:
    asst.append_task('StartUp',{
        "client_type": select_server,
        "start_game_enabled": True
    })
    asst.append_task(...)
    asst.append_task(...)
```

为了方便使用，这里按时间给刷体力任务配置不同的参数。

```python
weekday = time.localtime(time.time()).tm_wday
weekday_str = ["周一", "周二", "周三", "周四", "周五", "周六", "周日"][weekday]
print("今天是{}".format(weekday_str))

# 肝活动
campaign = True

# 钱本开放日
money = [2,4,6,7]
money_campain = False

# 先判断是否要肝活动
if campaign:
    fight_config = {
        # 留空为继续刷上次的关卡
        'stage': 'SL-7',
        'expiring_medicine': 9,
        'report_to_penguin': True,
        # 'penguin_id': '1234567'
    }
elif weekday == 1:
    fight_config = {
        # 剿灭
        'stage': 'Annihilation',
        'expiring_medicine': 9,
        'report_to_penguin': True,
    }
elif weekday in money or money_campain:
    fight_config = {
        'stage': 'CE-6',
        'expiring_medicine': 9,
        'report_to_penguin': True,
    }
else:
    fight_config = {
        # 其他时间都去刷狗粮本
        'stage': 'LS-6',
        'expiring_medicine': 9,
        'report_to_penguin': True,
    }           


...

asst.append_task('Fight',fight_config)
```

在改到这个地方的时候有一个插曲。我在文档里没有找到剿灭任务的参数，只看见`请参考 C# 集成示例`这段话。可是这个东西在哪是完全不知道的，因为仓库里的集成示例只有 C、Python、Golang、Dart、Java、Java HTTP、Rust、TypeScript 根本没有 C# ！

<img src="https://s2.loli.net/2023/08/22/z2sGPtuvABbqdQJ.jpg"/>

既然没有那就只能自己找解决方法了。经过检索后发现，由于项目重构，这部分内容现在已经不存在了。但是好消息是，我又在这个项目的其他位置找到了其他可以替代这部分的文档。于是更新了这里的指引说明 pr 上去了。（最后一节会展开说说这里）

~~简繁日英的文档改起来还是比较容易的，但是韩语我是真的一点都看不懂，这一块还是留给大佬吧。~~

把脚本改完之后尝试运行一次，没问题的话就可以添加到定时计划中了。

# 添加到定时计划任务

直接在`windows`搜索框找到并打开任务计划程序
。
<img src="https://s2.loli.net/2023/08/23/oDlsJaRqEOSK1PH.png" alt="任务计划程序" style="zoom: 20%;"/>

选择创建基本任务，配置任务描述和触发条件。在`启动程序`内填入`Python`的绝对路径，在`参数`内填入脚本的绝对路径和其他附加参数（没有特殊需求的可以不用填附加参数）。

<img src="https://s2.loli.net/2023/08/23/Us9KOl1pCocFtzr.png" style="zoom: 30%;" />

完成之后就MAA就能自动按计划执行任务了。

# 题外话

其实在更改文档的时候又遇到了插曲的小插曲。在操作系统环境设定成选择不同的语言时，系统会变更默认使用的编码方案，导致**相同的字**打印出来的样子是**不同的**。举例来说就是同样是使用汉字的国家在简化繁体字选择的途径不一样，导致了在不同编码中某些看起来“一样”的字实际上用的是完全不同的机内码。大家可以把系统的语言设定分别调成`zh-CN`、`zh-TW`、`zh-SG`、`zh-HK`，然后放大仔细观察一下文字的细节，就会发现这个问题真正麻烦的地方是，它并不可以直接通过更换字体或者更换输入法就可以解决的！

庆幸的是这次要修改的文档内容不是很多，而且大多都有可以参考的内容，只需要将别处的文字复制过来。在拼接的时候有一个很重要的细节是需要注意的，对`zh-TW`的处理并不是直接把简体字的内容直接转为繁体字就好了，而是要使用更符合地方语言习惯的语义用词去描述对应的内容，在复制的时候也要尽量采用之前的编辑者使用的措辞。

{% hideBlock 还有一些碎碎念 %}
我真的是服了啊，在维护日语文档的到底都有几批人啊，两个段落我能看见三种不同的逗号和四种不同的引号。
{% endhideBlock %}