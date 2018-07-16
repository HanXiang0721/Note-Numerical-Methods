# 环境变量配置

```shell
# Linear Baroclinic Model
export LNHOME=/var/opt/ln_solver
```



# 设定模式框架、分辨率与编译模式lib

打开`/var/opt/ln_solver/Lmake.inc `，修改下面这几行

```shell
#  include file for Makefile for linear solver
#
# set environment LNHOME        in your .cshrc
#
###### Architecture          ################
ARC  = N000
#ARC  = sun 
#ARC  = alpha
#ARC  = sgi 
#ARC  = sr8000

###### Model type            ################

### time-advance linear model (incl. storm track model)
PROJECT          = tintgr

### standard, making linear matrix (incl. stationary wave model)
#PROJECT         = mkamat

### accerelated iterative solver (AIM)
#PROJECT         = aim 

### nonlinear, dynamical core
#PROJECT         = dcore

### barotropic model
#PROJECT         = baro

### coupled mLBM-CZ
#PROJECT         = cz

### orographic forcing
#PROJECT         = wvfrc.topo

###### Horizontal Resolution ################
#HRES  = t10 
#HRES     = t21 
HRES  = t42 

###### Vertical Resolution   ################
#VRES  = l1
#VRES  = l5
#VRES  = l8
#VRES  = l11 
VRES  = l20 

###### Zonal wave truncation ################
ZWTRN  =        
#ZWTRN  = m15 
#ZWTRN  = m10 
#ZWTRN  = m5
#ZWTRN  = m6

###### Model options         ################

### time-advance linear model (incl. storm track model)
######## dry model
MODELOPT = -DOPT_CLASSIC
```

参数的意义见文件中的注释，但是务必先选定垂直与水平分辨率。

然后，

```shell
cd $LNHOME/model/src
make clean
make lib
```



# 背景场制作



## 利用ncep资料制作

首先需要编译。

```shell
cd $LNHOME/solver/util
make bs
```

然后打开`$LNHOME/solver/util/SETPAR`，修改对应几行

```shell
 &nmbs  cbs0='/var/opt/ln_solver/qqf/bs/qqf.t42l20',
        cbs='/var/opt/ln_solver/qqf/bs/qqf.t42l20.grd'
 &end

 &nmncp cncep='/var/opt/ln_solver/bs/ncep/ncep.clim.y58-97.t42.grd',
        cncep2='/var/opt/ln_solver/bs/ncep/ncep.clim.y58-97.ps.t42.grd',
        calt='/var/opt/ln_solver/bs/gt3/grz.t42',
        kmo=12, navg=3, ozm=f, osw=f, ousez=t
 &end
```

上面这几行的参数的意义分别如下：

| 参数名 |                           含义                            |
| :----: | :-------------------------------------------------------: |
|  cbs0  |                  运行后的背景场输出文件                   |
|  cbs   |                  运行后的背景场输出文件                   |
| cncep  |                    需要读入的ncep资料                     |
| cncep2 |                    需要读入的ncep资料                     |
|  calt  |                    需要读入的地形文件                     |
|  kmo   | 计算平均场的开始月份，例如，要计算夏季的平均场，那么kmo=6 |
|  navg  |   需要平均几个月，例如，要计算夏季的平均场，那么navg=3    |
|  ozm   |          是否需要zonal mean？需要填t，不需要填f           |
|  osw   |        是否需要zonal asymmetry？需要填t，不需要填f        |
| ousez  |          use Z for p->sigma，需要填t，不需要填f           |

修改完成后，运行`ncepsbs`就可以获得背景场



## 利用ERA40资料制作

流程与ncep资料制作背景场的流程一致，只是修改的几行变为：

```shell
 &nmbs  cbs0='/var/opt/ln_solver/qqf/bs/qqf.t42l20',
        cbs='/var/opt/ln_solver/qqf/bs/qqf.t42l20.grd'
 &end

 &nmecm cecm='/var/opt/ln_solver/bs/ecmwf/ERA40.clim.t42.grd',
        calt='/var/opt/ln_solver/bs/gt3/grz.t42',
        kmo=6, navg=3, ozm=f, osw=f
 &end
```

然后运行`ecmsbs`就可以获得背景场



# 模式编译

```shell
cd $LNHOME/model/src
make clean.special
make lbm
```

编译是否成功，只要检查下`$LNHOME/model/bin/$ARC `下的binary文件能不能和之前设置的分辨率以及框架对应上即可。假设之前设置的分辨率是T42L20，框架选择的是tintgr，那么生成的binary文件的文件名为：`lbm2.t42ml20ctintgr`



# 强迫场制作

以自带的强迫场文件`$LNHOME/sample/frc.t21l20.classic.grd `的制作为例

修改`$LNHOME/solver/util/SETPAR`中的这几行：

```shell
 &nmfin cfm='/var/opt/ln_solver/data/frc/frc.t42l20.CNP.mat',
        cfg='/var/opt/ln_solver/data/frc/frc.t42l20.CNP.grd'
        fact=1.0,1.0,1.0,1.0,1.0
 &end

 &nmvar ovor=f, odiv=f, otmp=t, ops=f, osph=t
 &end

 &nmhpr khpr=1,
        hamp=1.,
        xdil=10.,
        ydil=20.,
        xcnt=180.,
        ycnt=0.
 &end

 &nmvpr kvpr=2,
        vamp=3.,
        vdil=15.,
        vcnt=0.5
 &end

 &nmall owall=t
 &end

 &nmcls oclassic=t
 &end
```

其中的参数含义如下：

|  参数名  |                             含义                             |
| :------: | :----------------------------------------------------------: |
|   cfm    |                  rhs matrix file，输出文件                   |
|   cfg    |                 Grads forcing file，输出文件                 |
|   fact   |                            factor                            |
|   ovor   |            是否用vorticity forcing？是用t，否用f             |
|   odiv   |            是否用divergent forcing？是用t，否用f             |
|   otmp   |             是否用thermal forcing？是用t，否用f              |
|   ops    |                是否用Ps forcing？是用t，否用f                |
|   osph   |             是否用humidity forcing？是用t，否用f             |
|   khpr   |   horizontal shape of forcing: 1.elliptic, 2.zonal uniform   |
|   hamp   |                 如果填1，amplitude in 1/day                  |
|   xdil   |              zonal extent from center longitude              |
|   ydil   |            meridional extent from center latitude            |
|   xcnt   |                   center longitude, 0-360                    |
|   ycnt   |                   center latitude, -90~90                    |
|   kvpr   | vertical profile of forcing: 1: sinusoidal, 2: gamma, 3: uniform |
|   vamp   |                 如果填1，amplitude in 1/day                  |
|   vdil   |             dilation parameter, only for kvpr=2              |
|   vcnt   |                    center level in sigma                     |
|  owall   |                      full matrix or PWM                      |
| oclassic |                         moist model?                         |

文件中的nmhpr和nmvpr两段分别定义强迫的水平和垂直结构