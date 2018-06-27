# MPAS使用指南

> MPAS(The Model for Prediction Across Scales)是由美国Los Alamos国家实验室与NCAR联合开发的下一代数值模式。这个数值模式主要可以用于大气，海洋或者其他地球系统的模拟，同时也可以用于全球气候，区域气候以及天气的研究。MPAS主要利用Voronoi六边形网格(Spherical Centriodal Voronoi Tesselations)，以Arakawa C网格类型以及Hybrid垂直坐标作为其动力框架。MPAS的动力框架目前已经接入CAM，而MPAS的物理方案则是规划从WRF里移入。MPAS目前已经拥有一个集合卡尔曼滤波的同化系统，主要是与DART系统对接，与此同时，将MPAS和GSI对接的工作也已经在进行。MPAS是一个变网格模式，从理论上讲，它可以消除有限区域模式由于边界条件带来的大部分问题。MPAS由于其六边形网格，给用户处理模式输出带来了较大的困难。

## 编译模式主体

> 基本的环境可以与WRF，CESM一同搭建

进入MPAS的源码包，找到并打开Makefile。可以发现Makefile中列出了很多编译选项，笔者使用ifort与icc。MPAS的编译选择主要是利用下面这个格式：<br/>
`make compiler CORE=core`<br/>
compiler：使用什么编译器，目前支持ifort，gfortran，xlf，pgi<br/>
core=用户需要什么类型的动力框架，目前选择有：atmosphere，init_atmosphere，landice，ocean，sw，test<br/>
笔者这里编译两个：<br/>
`make ifort CORE=atmosphere`<br/>
`make ifort CORE=init_atmosphere`

## 编译METIS
> METIS可以用于加密网格，即让MPAS在某个区域网格分辨率提升，而其他地区网格分辨率保持不变。

直接进入目录，修改Makefile中的编译器选项，然后使用命令：<br/>
`make`<br/>
进行编译。当然，METIS的运行比较耗费资源，MPAS的官网也给出了若干已经处理好的加密网格，只需要做次旋转平移，将加密的位置旋转平移到目标位置即可。

## 运行
在编译完成后就可以开始运行了。运行流程非常简单，第一步先修改好stream文件，然后做好初始条件，最后就可以运行模式了(`./atmosphere_model`)。

## 后处理
MPAS的后处理比较麻烦，因为是六边形网格，不管是搜索某个经纬度的点还是插值，传统工具都失效了。具体的处理官网给出了部分工具，请看[这里](https://mpas-dev.github.io/)。


学会了实际模拟，理想试验就很简单了，具体可以参阅手册。根据笔者的经验看，MPAS的编译使用是几大广泛应用的数值模式中最简单的。