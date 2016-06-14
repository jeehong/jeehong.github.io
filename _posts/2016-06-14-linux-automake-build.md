---
layout:     post
title:      automake工程编译与发布至多级目录结构处理(c语言工程)
keywords:   programming
categories: [linux, programming]
tags:	    [linux, programming]
---

<p>存在多级目录的代码工程，一般是各个功能模块分布在各自的文件夹下，对于代码管理是一种比较明晰合理的管理方式，由此给Makefile对于源码结构管理带来了挑战。这种情况下，我们使用automake软件来生成最终的Makefile文件来编译连接，automake要求各个源码文件夹都存在各自的Makefile.am，用来管理源代码，在顶级目录中的Makefile.am文件，其中参数SUBDIRS指明了这个目录下有多少个**直接下级**目录要编译；</p>

<p>下面举例说明如何使用该工具：</p>
我测试的例子源码目录结构如下，main.c分别调用了add/ sub/ 文件夹下的模块，并生成可执行文件。
<code>.
├── main.c
├── add
│   ├── add.c
│   ├── add.h
└── sub
    ├── sub.c
    └── sub.h</code>
<p>下列给出使用automake生成Makefile的详细步骤：</p>

<p>顶级目录下运行命令生成configure.scan</p>
<code>autoscan</code>
<p>修改configure.scan文件名为configure.ac</p>
<code>mv configure.scan configure.ac</code>
<p>vim修改configure.ac文件中AC_INIT内容为AC_INIT(main 1.0, jeehong2015@gmail.com)</p>
<p>vim添加行AM_INIT_AUTOMAKE(main, 1.0)</p>
<p>vim添加行AC_PROG_CXX和AC_PROG_RANLIB</p>
<p>AC_PROG_RANLIB表示使用了静态库编译,需要此宏定义</p>
<p>vim修改行AC_OUTPUT内容为AC_OUTPUT(Makefile)</p>
<p>vim将AC_CONFIG_FILES替换为AC_OUTPUT</p>
<p>删除最低行AC_OUTPUT</p>

<p>运行命令生成如下命令生成aclocal.m4文件和autom4te.cache,该文件主要处理本地的宏定义。</p>
<code>aclocal</code>
<p>运行如下命令生成configure</p>
<code>autoconf</code>
<p>运行命令autoheader，负责生成config.h.in文件。该工具通常会从“acconfig.h”文件中复制用户附加的符号定义，因此此处没有附加符号定义，所以不需要创建“acconfig.h”文件。</p>
<code>autoheader</code>
<p>在automake命令调用前需要手动创建文件Makefile.am，此文件在源码工程中各级目录都要存在，且针对相应目录下的文件做了不同的配置。</p>

<p>**顶级目录**下的Makefile.am</p>
<code>vim Makefile.am</code>
<p>添加内容如下：</p>
<code>AUTOMAKE_OPTIONS=foreign

SUBDIRS=add sub
bin_PROGRAMS=main
main_SOURCES=main.c
main_LDADD=sub/libsub.a add/libadd.a</code>

<p>其中的AUTOMAKE_OPTIONS为设置automake的选项。由于GNU（在第1章中已经有所介绍）对自己发布的软件有严格的规范，比如必须附 带许可证声明文件COPYING等，否则automake执行时会报错。automake提供了三种软件等级：foreign、gnu和gnits，让用 户选择采用，默认等级为gnu。在本例使用foreign等级，它只检测必须的文件。 </p>
<p>bin_PROGRAMS定义要产生的执行文件名。如果要产生多个执行文件，每个文件名用空格隔开。 main_SOURCES定义“main”这个执行程序所需要的原始文件。如果”main”这个程序是由多个原始文件所产生的，则必须把它所用到的所有原 始文件都列出来，并用空格隔开。例如：若目标体“main”需要“main.c”、“sunq.c”、“main.h”三个依赖文件，则定义 main_SOURCES=main.c sunq.c main.h。要注意的是，如果要定义多个执行文件，则对每个执行程序都要定义相应的file_SOURCES。 其次 
使用automake对其生成“configure.ac”文件，在这里使用选项“—adding-missing”可以让automake自动添加有一些必需的脚本文件。</p>

<p>进入add/目录下：</p>
<p>创建Makefile.am并写入以下内容：</p>
<code>AUTOMAKE_OPTIONS=foreign                                                         
noinst_LIBRARIES=libadd.a
libadd_a_SOURCES=add.h add.c</code>

<p>noinst_LIBRARIES 表示了本目录下的代码编译成libxxx.a库。不需要发布。如果需要发布，则写成bin_LIBRARIES。注意，库的名称格式必需为 libxxx.a。因为编译静态库，configure.ac需要定义。</p>
<p>libxxx_a_SOURCES 编译libxxx.a需要的源文件。注意将库名称中的'.'号改成'_'号。</p>

<p>同理进入sub/目录下做同样操作：</p>
<p>创建Makefile.am并写入以下内容：</p>
<code>AUTOMAKE_OPTIONS=foreign                                                         
noinst_LIBRARIES=libsub.a
libsub_a_SOURCES=sub.h sub.c</code>

<p>至此，需要配置的环境已经全部结束，依次运行如下命令来生成Makefile：</p>
<code>automake --add-missing</code>
<code>./configure</code>
<p>此时已经生成了Makelfile，最后执行如下命令得到我们的目标代码并产生执行结果。</p>
<code>make</code>
<code>./main</code>
<p>执行结果：</p>
<code>jeehong@dev:~/project/study/2.3.8$ ./main 
int a+b IS:22
int a-b IS:-2</code>

<p>通过学习知道了automake的大致流程，但是对于细节性的内容需要我们在实际能够碰到需求再做扩展，比如如何配置编译器等等。</p>