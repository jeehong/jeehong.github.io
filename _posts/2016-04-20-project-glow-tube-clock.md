---
layout:     post
title:      DIY-辉光管实时时钟
keywords:   project glow tube clock
categories: [diy, project]
tags:	    [diy, project]
---

## 2016/4/20 ##
<p>明天开始，这个项目就启动了，我们拭目以待</p>
![image1](/images/images/githubpages/glow-tube-clock-img01.jpg)<br />
<p>辉光管亦称“冷阴极离子管”或“冷阴极充电管”，利用气体辉光放电原理而工作的离子管。玻璃管中包括一个金属丝网制成的阳极和多个阴极。大部分数码管阴极的形状为数字。管中充以低压气体，通常大部分为氖加上一些汞和／或氩。给某一个阴极充电，数码管就会发出颜色光，视乎管内的气体而定，一般都是橙色或绿色。此次的diy项目就是利用辉光管的数显效果来做的，CPU采用STM32F103RCT6、软件搭载FreeRTOS实时系统、暂定使用LwIP协议栈，功能暂定温湿度显示日月年时分秒显示，网络校时网络获取当前温湿度的主体功能后续再考虑其他更有趣的功能。</p>
源码和硬件工程全部放在github上: [grow-tube-clock](https://github.com/jeehong/grow-tube-clock)

## 2016/4/23 ##

<p>今天把以前画的一款硬件拖过来裁剪了一下又加入新的硬件，原理图雏形已经做好啦</p>
![image1](/images/glow-tube-clock-img/glow-tube-clock-img02.png)<br />
<p>而且从某宝上采购的辉光管也已经在路上啦，全新的前苏联产的IN1 六只和两个冒号氖泡，有图为证：</p>
![image1](/images/glow-tube-clock-img/glow-tube-clock-img03.jpg)<br />
![image1](/images/glow-tube-clock-img/glow-tube-clock-img04.jpg)<br />
![image1](/images/glow-tube-clock-img/glow-tube-clock-img05.jpg)<br />
<p>特地问了老板，这批货生产时间在1975-1985年之间，真古老啊。其实感觉挺值的，毕竟这东西生产年份那么遥远，可以当做收藏品了。</p>
<p>行了不说了，睡觉去，晚安。</p>


## 2016/4/26 ##

<p>前两天测试了一下如何从网络上获取时间的功能，本来打算自己租一个云主机写一个C代码提供时间服务，后来从网络上查到了更简单的方法，跟大家分享一下:</p>
<p>网络授时服务是一些网络上的时间服务器提供的时间，一般用于本地时钟同步。 授时服务有很多种，一般我们选择RFC-868。这个协议的工作流程是：（S代表Server，C代表Client）</p>
- S: 检测端口37
- U: 连接到端口37
- S: 以32位二进制数发送时间
- U: 接收时间
- U: 关闭连接
- S: 关闭连接
<p>协议非常简单，用TCP连接上后，服务器直接把时间发送回来。发送的是从1900年1月1日零时到现在的秒数。测试可用的TCP服务器有:</p>
- 132.163.4.103
- 128.138.140.44
- 132.163.4.102

如下图示：
<br />![image1](/images/glow-tube-clock-img/glow-tube-clock-img06.png)<br />
当连接到服务器37号端口时，服务器返回一个32bits的值随即断开链接，这个数值即是从1900年1月1日午夜到现在的秒数。数据从左至右从数据高位至低位。

## 2016/4/27 ##
<p>今天移植了LwIP协议栈到FreeRTOS，在网络上找了一个例程以及拿来之前做过的LwIP裸奔代码整合到自己的系统上。整个过程会有编译出错的情况，移植主要点在于：</p>
- 纯净的FreeRTOS和LwIP需要一些文件支持，比如sys\_arch.c、sys\_arch.h；
- 操作系统配置文件FreeRTOSConfig.h需要配置支持网络协议库的一些选项；
- LwIP配置文件lwipconfig.h、lwipopts.h需要配置支持操作系统的配置选项；
- 网卡结合硬件配置；
- 操作系统初始化LwIP协议栈、网卡驱动并创建网络任务；

<p>在main中调用下列操作，创建一个进程 LwIPEntry</p>
<pre><code>sys_thread_new((void * )NULL, LwIPEntry, ( void * )NULL, 350, 1);
</code></pre>
<p>LwIPEntry函数初始化过程执行如下代码：</p>
<pre><code>IP4_ADDR(&ipaddr, serverIP[0], serverIP[1], serverIP[2], serverIP[3]);
IP4_ADDR(&netmask, maskIP[0], maskIP[1], maskIP[2], maskIP[3]);
IP4_ADDR(&gw, gateIP[0], gateIP[1], gateIP[2], 1);
/* 初始化DM9000AEP与LWIP的接口，参数为网络接口结构体、ip地址、子网掩码、网关、网卡信息指针、初始化函数、输入函数 */
netif_add(&DM9000AEP, &ipaddr, &netmask, &gw, NULL, &ethernetif_init, &ethernet_input);	
netif_set_default(&DM9000AEP);	
#if LWIP_DHCP	
	dhcp_start(&DM9000AEP);	
#endif
netif_set_up(&DM9000AEP);	
</code></pre>
<p>这段代码转换网络地址并赋值给相应变量，设置添加网络接口，设置默认网卡以及使能网卡，这些操作对于LwIP是必须的。</p>
<p>最后，完成了初始化操作，就可以提供网络服务了，接下来就是配置socket操作：</p>
<pre><code>__pstConn = netconn_new(NETCONN_TCP);
netconn_bind(__pstConn, NULL, 80);
netconn_listen(__pstConn);
while(1)
{
	__pstNewConn = netconn_accept(__pstConn);	
	if(__pstNewConn != NULL)
	{			
		__pstNetbuf = netconn_recv(__pstNewConn);
		if(__pstNetbuf != NULL)
		{
			netconn_write(__pstNewConn, "HTTP/1.1 200 OK\r\nContent-type: text/html\r\n\r\n", 44, NETCONN_COPY);
			netconn_write(__pstNewConn, "&lt;body&gt;&lt;h1&gt;这是LWIP TCP测试！&lt;/h1&gt;&lt;/body&gt;", 40, NETCONN_COPY);	
			netbuf_delete(__pstNetbuf);	
		}	
		netconn_close(__pstNewConn);
		while(netconn_delete(__pstNewConn) != ERR_OK)
			vTaskDelay(1);
	}
/* ethernetif_input(NULL); */
/* tcp_tmr(); */
/* etharp_tmr(); */ 
}		
</code></pre>
<p>这一段就是配置一个TCP控制块来提供一个TCP Web服务，首先需要创建一个TCP控制块，绑定端口，其次设置为监听状态，循环过程，该进程一直处于阻塞状态等待客户端连接。当有新的客户端接入时，向下进行接受客户端的连接返回一个网页数据，并关闭该链接。</p>



