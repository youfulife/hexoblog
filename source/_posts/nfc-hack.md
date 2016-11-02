title: nfc hack 初探
date: 2014-12-26 15:55:34
tags: nfc
categories: 编程
---
前端时间研究了一把Mifare Classic tag的读取，复制，破解。
之前没有接触过RFID的东西，这两天的主要补充了一下基础知识，看了一些论坛上的破解文章，对RFID tag的通信流程，破解方法有了一定的认识，有点意思。
目前国内大部分的小区门禁卡，校园一卡通都是MIFARE Classic 1K类型的卡，固定频率13.56MHZ，而且不是全卡加密。

两天的时间这里看看，那里逛逛，思路有点凌乱了，发现了不少RFID卡破解相关的软件和硬件工具，先记录在此，后续慢慢研究：
###1. MCT
Android上的一款好用的tag工具Mifare Classic Tool (MCT)，界面简洁，交互性较好，开源。
支持的功能也比较全，读取tag信息，修改tag信息，复制tag信息，基于字典的暴力破解，编辑上传key文件，数据高亮，内置详细使用手册等等。
github源代码：https://github.com/ikarus23/MifareClassicTool
APK下载地址：http://publications.icaria.de/mct/
<!-- more -->
###2. ACR122U
读卡器ACR122U，价格便宜，淘宝200块钱左右，支持各种操作系统和libnfc，但是只能读取13.56MHZ的tag，新手入门必备。

###3. Proxmark3
RFID破解神器Proxmark3，土豪玩的玩意儿，价格大概2000RMB。

    (1)号称可以读取任何的RFID tag
    (2)可以作为一个读卡器，也可以作为一个tag
    (3)可以窃听读卡器和tag之间的通信
    (4)只要有个USB电源就可以单独工作，不需要PC机器
    (5) 开源，可以买回来自己DIY

所以说有钱什么事都好办。
官网：http://proxmark.org/
官方购买网站：http://www.proxmark3.com/
国内RadioWar团队是目前比较知名的相关方向研究团队http://radiowar.org/

这些个破解技术研究研究自己玩玩就好，不要去尝试实践一把，尤其是现在的饭卡，水卡之类的一卡通都有后台数据库定时对账的。不小心抓住了，你懂的。
