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

源码和硬件工程全部放在github上: [glow-tube-clock](https://github.com/jeehong/glow-tube-clock)

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
<pre><code>struct netconn  *__pstConn, *__pstNewConn;
	struct netbuf	*__pstNetbuf;

	char *web_page = "&lt;html&gt;&lt;body&gt;&lt;center&gt;&lt;hl&gt;hello,World!&lt;/hl&gt;&lt;br&gt;&lt;hr&gt;&lt;font size=15&gt;hello,world!&lt;/font&gt;&lt;/center&gt;&lt;/body&gt;&lt;/html&gt;";
	__pstConn = netconn_new(NETCONN_TCP);
	netconn_bind(__pstConn, NULL, 80);
	netconn_listen(__pstConn);
   /* Initilaize the HelloWorld module */
 	while(1)
	{
		__pstNewConn = netconn_accept(__pstConn);
		
		if(__pstNewConn != NULL)
		{			
			__pstNetbuf = netconn_recv(__pstNewConn);
			if(__pstNetbuf != NULL)
			{
				netconn_write(__pstNewConn, web_page, strlen(web_page), NETCONN_COPY);
				
				netbuf_delete(__pstNetbuf);	
			}
			
			netconn_close(__pstNewConn);
			while(netconn_delete(__pstNewConn) != ERR_OK)
				vTaskDelay(1);
		}
	}	
</code></pre>
<p>这一段就是配置一个TCP控制块来提供一个TCP Web服务，首先需要创建一个TCP控制块，绑定端口，其次设置为监听状态，循环过程，该进程一直处于阻塞状态等待客户端连接。当有新的客户端接入时，向下进行接受客户端的连接返回一个网页数据，并关闭该链接。</p>

## 2016/5/6 ##
<p>很高兴又来写博文，这次终于把网络部分调试成功了，通过浏览器输入IP地址可以看到：</p>
<br />![image1](/images/glow-tube-clock-img/glow-tube-clock-img07.png)<br />
<p>这个东西苦恼了很多天，我有运行正常裸奔的LwIP的源码，最开始移植进入FreeRTOS时网络总是不通畅。后来又从网络上下载了一个FreeRTOS+LwIP的例程，各种比较编译调试，都不行，调试结果DM9000根本接收不到任何数据，最后想起来参考FreeRTOS官方的例程，抱着尝试的态度将他的硬件初始化的代码加入到本项目下，奇迹般的可以ping通了，那这部分代码接口函数是：</p>
<code>	void prvSetupHardware(void); </code>
<p>在bsp.c中有定义，所以结论还是需要系统需要配合适当的硬件初始化过程，简简单单调用STM32库提供的初始化过程并不可取。</p>

## 2016/5/8 ##
<p>这个周末又要结束了，两天时间把PCB布置做好啦</p>
<br />![image1](/images/glow-tube-clock-img/glow-tube-clock-img08.png)<br />
<p>做IN-4封装的时候没仔细看，结果布好板子才发现封装画反了，六个管子需要做X镜像然后再布线，害我差不多花了两遍，很大的工作量都是在管子周边三极管和电阻排布和布线。</p>

## 2016/5/10 ##
今天是个重要的日子，那就是PCB发工厂做板啦，庆幸的是我工位旁坐着几个给力的硬件高手，才不至于原理图发生原理性错误。不过也不好说，最终结果还的等板子回来测试才能下结论哇，哈哈，硬件工程师们，考验你们实力的时候到了。关于新的硬件已经发布到github,并且发布了一个新的版本[V1.0.1](https://github.com/jeehong/glow-tube-clock/releases/tag/V1.0.1)，版本含义Va.b.c，a：发布的硬件版本，b：软件发布版本，c：软件测试版本。软件部分暂时停留在功能开发阶段，硬件完成后，会集中做软件开发的。

## 2016/5/22 ##
<p>今天将辉光管显示功能使用的api写了一下。具体来说，就是将需要显示的数字写入到74HC595当中，这之中就有一个转换关系的问题就，即如何将6个管子需要显示的内容转换到规律性不强的锁存器当中。</p>
<p>首先介绍一下锁存器电路，如下图，这是个局部视图，实际上网络标号S0-S59对应于辉光管秒低字节0-时高字节9；</p>
<br />![image1](/images/glow-tube-clock-img/glow-tube-clock-img09.png)<br />
<p>因为每个锁存器只输出8个位，导致锁存器与辉光管不是一一对应，为了得出每个管子显示内容与锁存器输出对应关系绘出下图：</p>
<br />![image1](/images/glow-tube-clock-img/glow-tube-clock-img10.jpg)<br />
<p>该视图，横线以上0-9就是要显示的内容，横线以下则是对应于当对应辉光管显示该数字时，需要传输到8个锁存器对应的8字节内容，该数组为map[8],通过观察和计算，得出了最下方的公式，即给出所在辉光管序号Q和待显示的内容V，则可以得出map[8]中对应字节所需要设置的内容；</p>
最有整理给出了api函数，在文件[app_display.c](https://github.com/jeehong/glow-tube-clock/blob/master/software/APP/app_display.c)中，如下所示：
<pre><code>/*
 * desc:指向向锁存器输入的8字节
 * src:指向待显示的6个辉光管数字[0,9],当超出给定范围时，认定为不显示任何内容
 * 小时高  小时低  分钟高  分钟低  秒高    秒低
 * src[0] src[1] src[2] src[3] src[4] src[5]
 */
static void app_display_set_map(char *desc, char *src)
{
	unsigned char tube;		/* 当前操作的管子序号 */
	const unsigned char tube_num = 5;	/* 管子总数 */
	unsigned char calc;
	memset(desc, 0, sizeof(char) * 8);
	for(tube = 0; tube <= tube_num; tube++)
	{
		if((unsigned char)src[tube] <= 9)
		{
			calc = tube * 10 + src[tube_num - tube];
			desc[calc / 8] = 0x01 << (calc % 8);
		}
		else		/* 当满足这个条件时，则默认为不显示任何内容 */
		{}
	}	
}
</code></pre>
这个函数被标记了static属性，也就是虽然称为一个api，我就是觉得这个函数在实现display功能中起到关键作用。但我想这还不具备可以开放给多任务的一个合理接口，对外开放还需要考虑到更多的因素，比如资源的互斥性。这个后面会斟酌并给出符合单片机高效性和实时性的接口。
