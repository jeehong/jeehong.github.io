---
layout:     post
title:      Linux下执行定时任务的方法
keywords:   linux，crontab,crond
categories: [linux]
tags:	    [linux]
---

<p>前段时间将Linux开发板上的符合AC97音频规范的WM9714声卡配起来了，并将ALSA库编译拷贝到文件系统中。本来想做一个基于DLNA网络协议的数字媒体播放器，想通过手机控制嵌入式设备播放音乐的功能，但是基于网络上资料较少便搁浅了。不过今天突然想在基于现有音频的功能上加闹钟的功能，于是在网上查了一些关于闹钟的功能，发现了一个新功能，那就是在Linux系统下如何创建基于RTC的定时任务，答案就是使用组合命令crontab以及crond！</p>
<p>cron是一个Linux下的定时执行工具，可以在无需人工干预的情况下运行作业。由于cron是Linux的内置服务，但它不自动起来，我想在Linux上电是便随系统启动，即在/etc/init.d/rcS文件最后一行添加<code> crond & </code>的内容，Linux系统在启动时会调用该功能。</p>
<p>cron服务提供 crontab命令来设定cron服务的，以下是这个命令的一些参数与说明：<br />
　　<code>crontab -u </code> >>> 设定某个用户的cron服务，一般root用户在执行这个命令的时候需要此参数 <br />
　　<code>crontab -l </code> >>> 列出某个用户cron服务的详细内容 <br />
　　<code>crontab -r </code> >>> 删除某个用户的cron服务 <br />
　　<code>crontab -e </code> >>> 编辑某个用户的cron服务 </p>

运行<code> crontab -e </code>来配置定时任务的，编写规则是每一行编辑一条定时任务，该任务格式是：
<br />min hour day month week command
<br />比如我要在每月12日晚上11点钟往文件a中写入当前时间则编辑：0 23 12 * * date > /home/root/a.txt
<br />**一些比较实用的例子：**
<br />	#每晚的21:30重启apache。
<br />	<code>30 21 * * * /usr/local/etc/rc.d/lighttpd restart</code>
<br />	#每月1、10、22日
<br />	<code>45 4 1,10,22 * * /usr/local/etc/rc.d/lighttpd restart</code>
<br />	#每天早上6点10分
<br />	<code>10 6 * * * date</code>
<br />	#每两个小时
<br />	<code>0 */2 * * * date</code>
<br />	#晚上11点到早上8点之间每两个小时，早上8点
<br />	<code>0 23-7/2，8 * * * date</code>
<br />	#每个月的4号和每个礼拜的礼拜一到礼拜三的早上11点
<br />	<code>0 11 4 * mon-wed date</code>
<br />经过上面的举例，想必已经掌握了他的规则，下面我就把早上的闹钟启动起来了，配置文件如下：
<br /><code >0       1       *       *       *       ntpdate 133.100.11.8 && hwclock -w</code>
<br /><code> 0       8       *       *       1-5     madplay /home/lonely.mp3 -m -a -30 -G</code>
<br /><code> 30      8       *       *       1-5     madplay /home/lonely.mp3 -m -a -30 -G</code>
<br />功能在于每天凌晨1点钟校准时间并写入硬件；早上八点播放音乐lonely.mp3一遍；八点半播放音乐lonely.mp3一遍；