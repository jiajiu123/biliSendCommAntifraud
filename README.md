## 功能介绍
使用软件之前，如果你不了解阿瓦隆系统，请先阅读这篇文档：[深挖b站如何控评-对阿瓦隆系统探究](https://github.com/freedom-introvert/Research-on-Avalon-System-in-Bilibili-Comment-Area)  
### 软件截图
![IMG_20230323_211608](IMG_20230323_211608.jpg)
![IMG_20230323_213024](IMG_20230323_213024.jpg)
![IMG_20230323_213007](IMG_20230323_213007.jpg)
![Screenshot_2023-06-04-16-12-59-915_icu.freedomIntrovert.biliSendCommAntifraud](Screenshot_2023-06-04-16-12-59-915_icu.freedomIntrovert.biliSendCommAntifraud.jpg)

### 登录
点击右上角“cookie”图标，可以直接填写cookie（列如从Chrome抓取到的），也可以直接使用内置网页浏览器进行登录。在浏览器登录你的b站账号，会抓取到cookie，设置并返回以完成cookie设置。  
![IMG_20230325_090006](IMG_20230325_090006.jpg)

### 使用方法
配置随机测试评论池，将会随机抽取一个用于评论被ban时回复该评论测试评论状态以及评论区是否戒严。同一个评论区抽取到的测试的评论不会重复。一行一个，最少4个，如果你没配置默认为古诗3首。  

先输入BV号、cv号或者链接，支持bilibili\.com 标准链接和 b23\.tv 短链接。支持视频、专栏、动态类型的评论区。  

在评论输入框输入要发的评论，点击“发送并检测”，发出去会自动帮你检查一下，评论是否正常显示，或者被ShandowBan或者是系统秒删。如果被ban，你可以进一步操作，检查评论区将会去检查评论区是否戒严，如果没有戒严继续检查是否该评论仅在此评论区被ban；更多评论选项可以[扫描敏感词]、[申诉]、[删除发布的评论]这里不多介绍  。

如果你觉得输入表情之类的不方便，你可以在b站的评论输入框打好评论然后复制粘贴到这里。不过你在这里直接打“[doge]”也无伤大雅。
## Xposed功能
在Xposed模块里启用并勾选哔哩哔哩（国际版以后适配），在哔哩哔哩APP里评论发送成功后，会调起本发评反诈进行检查，方便快捷！  
Xposed支持对评论回复的检查！  
（图中评论区为戒严评论区）  
![IMG_20230511_020917](IMG_20230511_020917.jpg)
### 注意事项
- 哔哩发评反诈必须登录账号（设置cookie），而且登录的账号要与哔哩哔哩APP里登录的一致！不然一定会出现异想天开的问题🤣
- 请允许哔哩哔哩打开“哔哩发评反诈”！由于context问题，Xposed hook到的context，是哔哩哔哩的，统计只能插入哔哩哔哩APP的数据库，所以我使用了透明activity显示在哔哩哔哩之上，当评论发送成功时启动本应用的这个透明activity并传入参数，这样就可以使用自己的context，将统计数据插入自己的数据库。
## 发送并检查评论
```mermaid
graph TB
    A[发送评论] --> B(无账号环境下获取最新评论列表) --> C{是否找到该评论}
    C --找到了--> D[评论正常显示]
    C --没找到--> E(用测试内容回复该评论)
    E --"提示“已经被删除了”"--> F[评论被系统秒删]
    E --回复成功--> H(删除测试评论) --> G[评论被shadowban]
```
### 发送后的等待时间
为什么发送后要等待一段时间再检查（流程图中略）？因为发布之后，系统需要一定的时间来处理评论，发送了立即获取评论列表导致获取到的最新评论没有该评论但是评论其实没有问题的，从而导致误判。当然你可以设置发送后要等待多久。设置成5秒（5000ms）最好，默认5秒，可在菜单中设置。  
### 新检查方法   
哔哩哔哩还有一个获取评论回复列表的api，有以下特性
- 可以显示出根评论的内容等信息
- 游客获取shadowban评论的回复提示“已经被删除了”
- 登录了账号获取shadowban评论的回复正常
- 评论真的被系统秒删了，登录账号获取该评论的回复也提示“已经被删除了”
- up主发的评论被shadowban，游客也能正常获取该评论的回复列表  

根据此特性我改进了检查程序，不对评论发送回复就可以检测状态，减少测试评论的发送，可以减少这些测试评论对申诉所造成影响！2.2.0版本已更新该方法。

```mermaid
graph TB
    10[评论发送] --> 20(无账号下翻找评论/评论回复列表)
    20 --找到了--> 400[评论正常显示]
    20 --没找到 --> 
    22{是否为回复评论} 
    22 --否--> 60(有账号获取该评论的回复列表) --> 70(获取成功) --> 80[评论被shadowban]
    22 --是--> 50(有账号翻遍评论回复列表) --没找到--> 52[评论被系统秒删]
    50 --找到了--> 54[评论被shadowBan]
    60 --> 90(提示已经被删除了) --> 100("回复该评论(用于判断cookie是否过期)") --> 110(还是提示已经被删除了) --> 105[评论被系统秒删]
    100 --> 120(提示未登录等错误) --> 130["cookie已过期，请重新登陆获取"]
    100 --> 140("回复成功（几乎不可能）") --> 150[评论被shadowban]
    
```

## 检查评论区
```mermaid
graph TB
    A[在该评论区发布测试评论]  --> B(无账号环境下获取最新评论列表) --> C{是否找到该评论}
    C --找到了--> Z(删除测试评论)--> D[评论区正常] --> J[测试是否仅在此评论区被ban] --> K(在你的评论区发送一样内容的评论) --> L{翻找评论列表}
    C --没找到--> E[评论区被戒严] --> F(有账号下获取评论回复列表)
    F --"提示“已经被删除了”" --> AC(删除测试评论) --> G[评论默认立即删除]
    F --获取成功--> H(删除测试评论) --> Y[评论默认shadowban] 
    L --找到了--> AA(删除发的评论)--> X[仅在此评论区被ban] 
    L --没找到--> AB(删除发的评论) --> V[在其他评论区也被ban]
```
## 统计
开启自动记录即可使用统计。搜集数据，汇总统计，有助于分析他ban什么评？喜欢戒严什么内容下的评论区？  
支持导出导入，点击右上角按钮即可。导出为csv格式。导入数据将插入到列表最前面。

### 统计被ban评论
![IMG_20230326_210709](IMG_20230326_210709.jpg)  
评论4种状态

- 仅自己可见
- 被系统秒删
- 包含敏感词
- 评论被隐身（invisible）
- 评论疑似正常（申诉时提示无可申诉评论时状态会切换为此）
- 未知状态（直接去申诉无法得知状态）
  

评论区id颜色示意，意义为你是否检查过评论区，比如是否戒严与是否仅在此评论区被ban
- 灰色：没有检测戒严之类的
- 蓝色：只知道评论区没有戒严
- 绿色：评论区没有戒严且评论在其他评论区也被ban
- 红色：评论仅在此评论区被ban
### 统计戒严评论区
![IMG_20230326_210311](IMG_20230326_210311.jpg)  
## 复查评论
### 已被Ban的评论
如果评论被ban，还想关心后续的情况，比如你自前的申诉了，或者是等系统给你解禁。前提是你不要删除评论或评论已被删哦。在被ban评论列表里，点击被ban评论信息下的“复查状态”即可。要复制评论可以长按文字进行复制或点击“评论内容”复制全文。

**注：在被Ban评论列表里复查评论不会更新评论状态**

### 历史评论记录
可开启记录历史评论，无论是否被ban都收录。对于正常的评论，也有可能出现后续被shdowBan或被删除的情况，甚至有传言说识别到了故意正常显示一段时间，然后一段时间后再shadowBan回去。有一位群友有真实的案例：评论发出后，甚至有很多的点赞与回复，一天后点赞停止，也没有人再回复了。检查之后发现评论还在，但是切小号就没有了，明显shadowBan了。所以添加评论记录让大家能再检查。



有一种特殊的状态我不知道该怎么称呼，暂且叫它“疑似审核中”，此状态下游客获取评论区获取不到此条评论，但是去申诉会提示“无可申诉评论”（大概率），此时再去检查发现游客通过评论的rpid拉取回复列表成功（可通过root属性获取根评论信息），只是列表里翻不到此条评论。这种情况可以过几个小时再来历史评论记录里检查。

如果评论发送出去正常，但是后期再检查时发现被shadowban了，软件会添加此评论到「被ban评论列表」，状态为：“**仅自己可见（秋后算账）**”。  
  
历史评论记录可记录评论的点赞与回复数（回复的评论没有回复数），在您更新评论状态后，此信息也会更新。



评论的状态将以字体的形式展现

**正常：** 示例

**ShadowBan（仅自己可见）：** <span style="color:red"> 示例 </span>（红色字）

**被删除：** <span style="color:red">~~示例~~</span>（红色字）

**invisible：**<span style="color:#CCCCCC">示例</span>（灰色字）  


### 快捷编辑重发
在历史评论记录里，长按评论item，即可编辑重发，重发后不影响你原来的评论。不支持发送图片（即使你的评论以前有图片）。如果你要编辑的评论是回复评论，后面发送的评论也是某评论的回复评论
![IMG_20230923_165329](IMG_20230923_165329.jpg)
### 评论再检查逻辑
``` mermaid
graph TB
00[有账号拉取评论回复] 
00 --> 10(root评论的rpid=当前评论的rpid)
00 --> 02["收到“已经被删除了”"] --> 04["评论被删除，或回复的根评论被删除"]
10 --"是，这是主评论"--> 20["无账号拉取评论回复，快速判断"] 
20 --成功--> 30["评论正常……吗？"] -.-> 100["无账号翻遍评论区"] 
100 --找到了--> 110["评论正常"]
100 --没找到--> 120["评论shadowBan-，或审核中"]
20 --"“已经被删除了”"--> 40["评论被shadowBan"]
10 --"否，这是评论的回复"--> 50["无账号翻找回复列表"] 
50 --找到了--> 60["评论正常"]
50 --没找到--> 70["有账号翻找评论回复列表"] 
70 --找到了--> 80["评论shadowBan"]
70 --没找到--> 90["评论被删除"]
```
## 后台等待
等待时间过长时，你可能不愿意一直盯着这个dialog，直到它等待完成去检查评论。此时你可以点击“后台等待”按钮，它就会转至通知栏，下拉通知栏可以查看等待进度。等待完成后，点击通知可进行下一步的检查。  
警告：如果你把通知划掉了，将丢失待检查评论，历史评论记录也不会记录你的评论！
![IMG_20230910_180229](IMG_20230910_180229.jpg)
![IMG_20230910_180248](IMG_20230910_180248.jpg)
## 扫描敏感词
在评论被ban后，在[更多评论选项]里选择[扫描敏感词]，即可开始扫描。请注意，扫描之前必须先设置你的评论区！
### 扫描方法如图，★为敏感词位置。
![Screenshot_2023-03-18-08-24-12-966_top.weixiansen574.biliSendCommAntifraud](二分展开发查找敏感词.png)
如上图扫描步骤，只到评论二分到设置的最小单位，停止扫描。敏感词就在红块内外。

### 疑问
- 红块就是敏感词所在域吗？不完全是。假设敏感词是“我是敏感词”，会出现：[<font color="#00dd00">我是敏</font><font color="#dd0000">感词</font>]的情况。因为发送“我是敏”没事，但是继续展开到“我是敏感词”才会出问题。红块没有完全盖住敏感词，有时候只是盖住了敏感词的一部分，也会导致敏感词不成立。
- 如果出现两个敏感词？展开法原本是从头开始一点一点展开到不能发送为止，如果有两个敏感词，只要展开到第一个敏感词处就不能正常显示了，二分展开法也是如此，所以只会扫出第一个敏感词，接着把敏感词改掉可以继续扫描下一个。
- 如果出现类似“唯唯诺诺，重拳出击”这样同时出现才会生效的敏感词？既然是同时出现才有效，那么直到扫描到它最后一个词语时才会异常，那么结果也是最后一个词语。当然这难以检测出具体是哪两个组合词，如果你把扫出来的词单独发送能正常显示，说明为组合词。如果你很想找出具体是哪两个组合词的话，我可以给你个方法，按以上二分法查找，在每次测试的评论末尾加上那个词语，就可以找出来，这我就不写到程序里了：）。
### 为什么要在自己的评论区扫描敏感词？
因为查重黑名单机制（详见阿瓦隆系统研究文档），这个机制可能会ban掉与你之前发布大致相同的评论（即使你删除了之前发的），因为相似而被ban的会直接影响到扫描结果，导致扫偏，甚至扫描过后，那段话都没法发了。不过，如果你是该评论区的up主，你将不会受到该机制的影响，所以扫描就在你所设置的评论区进行。当然，扫描敏感词要求的是全站通用的敏感词，仅在某评论区被ban的评论是没有扫描的价值的，故程序在扫描之前会检测是否评论区戒严以及是否仅在此评论区被ban。  

你的评论区可以设置视频和动态还有专栏，尽量的选择一些很无聊没有人看的（观看数<500）。你可以随便发一条很无聊的游戏录像或者是你瞎发条动态都行，只要够无聊到没有人观看。
### 其他评价
突然发现某些游戏（网易我的世界除外）算仁慈了😂，“文本文本敏感词文本”，变“文本文本***文本”，至少你可以知道敏感词是啥。

## 发送后直接申诉（高级玩法）
参见「申诉评论」  
### 由来
如果发发的评论正常显示，然后去申诉会发生什么🤔  
答案是：该bv号或链接下无可申诉评论  
![IMG_20230320_194931](IMG_20230320_194931.jpg)

``` json
{
  "code": 12082,
  "message": "该bv号或链接下无可申诉评论",
  "ttl": 1,
  "data": null
}
```
### 实现流程


``` mermaid
graph TB
    10[发送评论] --> 20("去申诉，提交评论区bv号或URL") 
    20 --该bv号或链接下无可申诉评论--> 30[评论正常显示]
    20 --申诉提交成功--> 40[评论被ban] .-> 50["等待系统通知，听天由命🙃"]

```
利用该方法，可以最精确的判断评论是否被ban。如果被ban了，然后就坐等系统通知（申诉成功or失败），非常方便。缺点也很大，每天只有三次的申诉机会，如果评论被ban，申诉提交成功就会使用一次，用完了就只能等明天了；无法得知评论具体状态，shadowban还是秒删？
## 关于申诉评论
在评论被ban后可在[更多评论选项]里去申诉该评论，与网页版申诉流程是一样的。  

发送申诉后，人机不会看你的申诉理由，你的申诉会先过一遍人机，如果人机对申诉不痛不痒才交给人工，真人才会去理解申诉理由。如果有多条评论，机器审核通过的会先恢复，然后才到人工审核的。  

人机申诉会通过你发的一系列被ban状态的评论，也就是如果你发了多条，他会全部恢复，无论评论是多久前发的。他不会看你的申诉理由，你在申述理由里填要指定恢复那条是没用的，除了人工会看。在shadowban评论上回复的任何评论都属于shadowban评论，一样会加入待申诉列表里。还有一个特性（bug？）是：你删除掉了shadowban评论，但是申诉过后依然会系统通知你“您的评论申诉已处理”，然后下面跟着你评论的略写，不过都是“无法恢复”，很少情况会恢复成功。  

评论不能正常显示时判断评论状态会发送测试回复评论、测试戒严的评论区会发送测试评论，请注意：申述后，这些自己已经删除的测试评论可能会被恢复，如果通知是“无法恢复”，那么不用管他，如果是“无违规”，请注意去删除测试评论！ 
## Xposed-显示隐身评论
突然发现评论有一个属性叫“invisible”，意为不可见的，如果为true，评论将不展示。这是前端的行为，下载到了评论，但是不展示罢了。  
本模块将把隐身评论显示出来，并将“[隐藏的评论]”注入在IP属地的后面作为提示（由于国际版没有IP属地，国际版可显示隐身评论但不标明）。 
![IMG_20230604_202429](IMG_20230604_202429.jpg) 
## 浏览器油猴脚本
有很多人想要浏览器的脚本，但是我不怎么会前端，我用ChatGPT简单生成了一个脚本并自己修改了下。拦截发送评论请求并获取评论ID进行检查。  

仅简简单单实现了检查功能，并使用alert()显示结果。没有检查评论区、申诉、扫描敏感词等功能，有意者可以帮我实现完整版脚本。  
脚本：[哔哩发评反诈-油猴.js](./哔哩发评反诈-油猴.js)

## 关于
### 讨论交流
Telegram: [@biliSendCommAntifraud](https://t.me/biliSendCommAntifraud)
### LOGO含义
来自：Never Gonna Give You Up - Rick Astley  
意为“发送成功”但是你被骗了🤪

### icon使用
部分来源于[iconfont-阿里巴巴矢量图标库](https://www.iconfont.cn/)