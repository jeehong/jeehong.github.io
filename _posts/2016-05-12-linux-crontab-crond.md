---
layout:     post
title:      Linux下执行定时任务的方法
keywords:   linux，crontab,crond
categories: [linux]
tags:	    [linux]
---

<p>前段时间将Linux开发板上的符合AC97音频规范的WM9714声卡配起来了，并将ALSA库编译拷贝到文件系统中。本来想做一个基于DLNA网络协议的数字媒体播放器，想通过手机控制嵌入式设备播放音乐的功能，但是基于网络上资料较少便搁浅了。不过今天突然想在基于现有音频的功能上加闹钟的功能，于是在网上查了一些关于闹钟的功能，发现了一个新功能，那就是在Linux系统下如何创建基于RTC的定时任务，答案就是使用组合命令crontab以及crond！</p>

<p>cron是一个Linux下的定时执行工具，可以在无需人工干预的情况下运行作业。由于cron是Linux的内置服务，但它不自动起来，我想在Linux上电是便随系统启动，即在/etc/init.d/rcS文件最后一行添加<code> crond & </code>的内容，Linux系统在启动时会调用该功能。</p>
<p>可以使用crontab命令的用户是有限制的。如果/etc/cron.allow文件存在，那么只有其中列出的用户才能使用该命令；如果该文件不存 在但cron.deny文件存在，那么只有未列在该文件中的用户才能使用crontab命令；如果两个文件都不存在，那就取决于一些参数的设置，可能是只 允许超级用户使用该命令，也可能是所有用户都可以使用该命令。</p>
<p>cron服务提供 crontab命令来设定cron服务的，以下是这个命令的一些参数与说明：<br />
　　<code>crontab -u </code> >>> 设定某个用户的cron服务，一般root用户在执行这个命令的时候需要此参数 
<p>-u 如果使用该选项，也就是指定了是哪个具体用户的crontab 文件将被修改。如果不指定该选项，crontab 将默认是操作者本人的crontab ，也就是执行该crontab 命令的用户的crontab 文件将被修改。但是请注意，如果使用了su命令再使用crontab 命令很可能就会出现混乱的情况。所以如果是使用了su命令，最好使用-u选项来指定究竟是哪个用户的crontab文件。</p>
　　<code>crontab -l </code> >>> 列出某个用户cron服务的详细内容 <br />
　　<code>crontab -r </code> >>> 删除某个用户的cron服务 <br />
　　<code>crontab -e </code> >>> 编辑某个用户的cron服务 </p>
<p>由于我的嵌入式文件系统缺少关于crontab的运行文件夹，所以执行<code>mkdir -p /var/spool/cron/crontabs</code>创建相关目录</p>
运行<code> crontab -e </code>来配置定时任务的，编辑完成后会在文件夹/var/spool/cron/crontabs下生成一个文件，该文件以当前用户名命名，其描述了所有任务的配置记录。而编写规则是每一行编辑一条定时任务，该任务格式是：
<br />min hour day month week command
<br />比如我要在每月12日晚上11点钟往文件a中写入当前时间则编辑：<br />
<code>0 23 12 * * date > /home/root/a.txt</code>
<br />
<br />**一些比较实用的例子：**
<br />	#每晚的21:30重启apache。
<br />	<code>30 21 * * * /usr/local/etc/rc.d/lighttpd restart</code>
<br />	#每月1、10、22日
<br />	<code>45 4 1,10,22 * * /usr/local/etc/rc.d/lighttpd restart</code>
<br />	#每两个小时
<br />	<code>0 */2 * * * date</code>
<br />	#晚上11点到早上8点之间每两个小时，早上8点
<br />	<code>0 23-7/2，8 * * * date</code>
<br />	#每个月的4号和每个礼拜的礼拜一到礼拜三的早上11点
<br />	<code>0 11 4 * mon-wed date</code>
<br />	#每小时执行/etc/cron.hourly内的脚本,注意"run-parts"这个参数，如果去掉这个参数，后面就可以写要运行的某个脚本名，而不是文件夹名
<br />	<code>01 * * * * root run-parts /etc/cron.hourly</code>
<br />	#任意天任意月，其实就是每天的下午4点、5点、6点的5 min、15 min、25 min、35 min、45 min、55 min时执行命令
<br />	<code>5,15,25,35,45,55 16,17,18 * * * command</code>
<br />	#每天早晨三点二十分执行用户目录下如下所示的两个指令（每个指令以;分隔）
<br />	<code>20 3 * * * /bin/rm -f expire.ls logins.bad;bin/expire$#@62;expire.1st</code>
<br />
<br />经过上面的举例，想必已经掌握了他的规则，下面我就把早上的闹钟启动起来了，配置文件如下：
<br /><code>0       1       *       *       *       ntpdate 133.100.11.8 && hwclock -w</code>
<br /><code>0       8       *       *       1-5     madplay /home/lonely.mp3 -m -a -30 -G</code>
<br />功能在于每天凌晨1点钟校准时间并写入硬件；早上八点播放音乐lonely.mp3一遍；
