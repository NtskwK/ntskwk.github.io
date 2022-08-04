---
title: 'AutoX下的一键打卡[补档]'
date: 2022-07-14 15:00:18
update: 2022-08-04 13:49:18
tags:
- 补档
- 实践
categories:
- 实践
keywords: 
- Auto.js
highlight_shrink: false
---
相关链接：

Hooの小窝：[http://blog.jiyehoo.com:81/index.php/archives/288/]()

项目地址：[https://github.com/NtskwK/AutoDailyClock-GUT]()

# 前提概要
**Auto-Daily-Clock**是[JiyeHoo](https://github.com/JiyeHoo)学长发起的，通过`auto.js`实现每日钉钉打卡的脚本。因为脚本的更新状况已经落后于钉钉数个版本，所以部分控件实例已无法正常识别。另一方面，由于学长本人现已从学校毕业，不方便继续维护这个项目。于是我决定 *暂时接手* 推进项目的维护工作。

[**AutoX**](https://github.com/kkevsekk1/AutoX)是一个基于[auto.js](https://github.com/hyb1996/Auto.js)的开源项目,它同样是一个支持无障碍服务的Android平台上的JavaScript IDE，其发展目标是JsBox和Workflow。使用该项目需要同时遵守[GPL-V2](https://opensource.org/licenses/GPL-2.0)和[MPL-2](https://www.mozilla.org/MPL/2.0)

# 项目背景
`auto.js`原作者现已停止对开源版本的维护工作，而付费版本的售价对于一个业余爱好者来说并不便宜，加上最后获取开源版本的打包插件方式对于新手来说并不友好，便放弃了这个平台。毕竟为了一个脚本去编译旧版IDE，确实过于繁琐了。 ~~（这个夏宇就是菜啦）~~

所以我选用了基于`auto.js`开发的`AutoX`作为继续维护的平台。主要原因有以下两点：

1. 支持旧版`auto.js`原有的基础函数，兼容性较好。
2. `AutoX`作开发仍在推进的开源项目，后续更方便维护安全，且安全具有保障

# 改进路线
原项目中使用`toast`事件来打印消息，这样不利于排查出现问题的原因，所以改用`log()`来集中打印日志信息。经过逐步调试可以发现，原本预定识别控件的样式都发生了变化，有的部分甚至已经无法识别当前所处的位置界面。 ~~可以直接准备全部重构了。~~

## 添加界面识别
在钉钉版本更新后，底部Tab栏的`工作台`按钮的图标会在`校徽`和“钉钉原有样式”之间变换。那么我们就不要去管它具体变成什么，只要通过其他方式确定页面已经正确加载再直接盲点即可。

```javascript
    className("android.widget.ListView").findOne();
    sleep(3000);
    var mWidth = device.width / 2;
    var mHeight = device.height - 150;
    click(mWidth, mHeight);
    sleep(500);
```

## 点击指定控件
通过`AutoX`自带的界面分析工具我们可以得知，填写打卡信息的页面中，获取定位一栏处，`text("获取")`对应控件以及其父级并不具备`点击`事件，所以改为获取控件坐标，然后尝试点击控件中心的位置。

```javascript
var hq = className("android.view.View").text("获取").findOne();
    if (hq) {
        log("开始获取定位");
        click(hq.bounds().centerX(), hq.bounds().centerY());
        sleep(2000);
    } else {
        log("自动获取定位");
    }
```

紧接着是最后的提交，因为钉钉版本更新后该按钮也无法获取，所以要用`gesture`滑倒最下面，进行盲点。这里可以看到原作的旧版本就是这样处理的，那么就把现有的代码注释掉，把注释代码前的注释符删掉就大功告成了。

## 加个GUI
为了方便其他人使用或进一步了解这个项目，给启动界面做了个简单的GUI

```javascript
"ui";
events.observeKey();
events.onKeyDown("volume_down",function(event){
            threads.shutDownAll();
        });
ui.layout(
    <vertical gravity="center_vertical" padding="16">
        <text margin="8" textSize="27sp" textStyle="bold" lines="3">
        欢迎使用AutoDailyClock
        </text>
        <text margin="8" textStyle="bold" textColor="red">
        本程序只适用于正常的健康日常打卡。如果身体发生异常状况请立即停止使用本程序，并及时向辅导员或其他负责人报告身体状况！
        </text>
        <button id="startButton"  text="开始打卡" style="Widget.AppCompat.Button.Colored" w="*"  h="70" margin="20"/>
        <button id="checkButton" text="检测权限" style="Widget.AppCompat.Button.Colored" w="*" h="70" margin="20"/>
        <text margin="8">
        使用本应用需要启用的权限：
        </text>
        <text margin="8">
        悬浮窗口、无障碍功能
        </text>
        <text margin="8" textColor="red">
        1.AutoDailyClock-GUT是一个开源项目，它是免费的。
        </text>
        <text margin="8" lines="5">
        2.本项目的源代码是公开的。如果对软件内容存在质疑，欢迎前往github下载源码。
        </text>
        <text marginBottom="5">
        反馈及咨询：natsukawa247@outlook.com
        </text>
        <text marginBottom="5">
        项目地址：
        </text>
        <text marginBottom="5">
        https://github.com/NtskwK/AutoDailyClock-GUT
        </text>
        <text marginBottom="5">
        Autojs相关文档:
        </text>
        <text marginBottom="5">
        https://pro.autojs.org/
        </text>
    </vertical>
);
```

~~不会用javascript画，所以随便找了个有GUI的脚本依葫芦画瓢给搬了过来。~~ 总之，得把注意事项给写上，滥用工具还出事了的话，我可不背锅。

## 细枝末节
减少`sleep()`的时间，并在`findOne()`中补上  延时等待。这样可以同时防止手机性能太差页面加载赶不上脚本执行速度，和点击过快
会导致卡住的问题。

```javascript
    className("android.widget.ListView").findOne(3000);
    sleep(500);
```

## 可有可无
专门单独写了个函数来报错 ~~（异常越处理越多，干脆不要处理好了）~~
```javascript
function end(step){
    log("无法" + step);
    log("请手动完成本次工作");
    toast("任务失败！详情请看日志内容");
    exit();
}
```

# 完事后话
`Auto-Daily-Clock`是这个项目一开始的名称，现更名为[**AutoDailyClock-GUT**](https://github.com/NtskwK/AutoDailyClock-GUT)是为了防止和其他项目产生冲突 ~~（但是我完全不想用AutoDailyDingClockGUT这种太长的名字）~~正如前文所说，我**不保证**会对它进行长期维护。一方面是相信这片人才辈出的天地一定后继有人，另一方面是希望持续已久的疫情早日解除。

最后，`AutoDailyClock-GUT`作为我的第一个auto.js(AutoX)项目给我带来了前所未有的经验，同时，它也作为我博客里完成的第一篇正文，纪念我开始迷茫的那段时光。

{% hideBlock 随时可能会被删掉的碎碎念 %}
我在不会用git的时候一套骚操作把更新push到上游去了，结果还合并成功了。我为什么不直接原地开release重新开版本号啊。这样混不到Star了啊（；´д｀）ゞ
{% endhideBlock %}