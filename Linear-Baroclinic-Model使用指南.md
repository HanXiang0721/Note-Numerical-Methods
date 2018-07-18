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



# 模式运行

在`$LNHOME/model/sh/tintgr`下，新建脚本`run.csh`

```shell
#!/bin/csh -f
#
#      sample script for linear model run (dry model)
#
# NQS command for mail
#@$-q   b
#@$-N   1
#@$-me
#
setenv LNHOME   /var/opt/ln_solver                 # ROOT of model
setenv LBMDIR   $LNHOME/model                          # ROOT of LBM 
setenv SYSTEM   N000                                 # execute system
setenv RUN      $LBMDIR/bin/$SYSTEM/lbm2.t21ml20ctintgr # Excutable file
setenv TDIR     $LNHOME/solver/util
setenv FDIR     $LNHOME/qqf/frc                        # Directory for Output
setenv DIR      $LNHOME/qqf/out                       # Directory for Output
setenv BSFILE   $LNHOME/qqf/bs/qqf.t21l20          # Atm. BS File
setenv RSTFILE  $DIR/Restart.amat                      # Restart-Data File
setenv FRC      $FDIR/frc.t21l20.CNP.grd               # initial perturbation
setenv SFRC     $FDIR/frc.t21l20.CNP.grd               # steady forcing
setenv TRANS    $TDIR/gt2gr
setenv TEND     100 
#
#
if (! -e $DIR) mkdir -p $DIR
cd $DIR
#\rm SYSOUT
echo job started at `date` > $DIR/SYSOUT
/bin/rm -f $DIR/SYSIN
#
#      parameters
#
cat << END_OF_DATA >>! $DIR/SYSIN
 &nmrun  run='linear model'                                 &end
 &nmtime start=0,1,1,0,0,0, end=0,1,$TEND,0,0,0             &end
 &nmhdif order=4, tefold=1, tunit='DAY'                   &end
 &nmdelt delt=20, tunit='MIN', inistp=2                     &end
 &nmdamp ddragv=1,1,1,5,30,30,30,30,30,30,30,30,30,30,30,30,30,30,1,1,
         ddragd=1,1,1,5,30,30,30,30,30,30,30,30,30,30,30,30,30,30,1,1,
         ddragt=1,1,1,5,30,30,30,30,30,30,30,30,30,30,30,30,30,30,1,1,
         tunit='DAY'                                           &end
 &nminit file='$BSFILE' , DTBFR=0., DTAFTR=0., TUNIT='DAY'   &end
 &nmrstr file='$RSTFILE', tintv=1, tunit='MON',  overwt=t    &end

 &nmvdif vdifv=1.d3,1.d3,1.d3,1.d3,1.d3,1.d3,1.d3,1.d3,1.d3,1.d3,1.d3,1.d3,1.d3,1.d3,1.d3,1.d3,1.d3,1.d3,1.d3,1.d3,
         vdifd=1.d3,1.d3,1.d3,1.d3,1.d3,1.d3,1.d3,1.d3,1.d3,1.d3,1.d3,1.d3,1.d3,1.d3,1.d3,1.d3,1.d3,1.d3,1.d3,1.d3,
         vdift=1.d3,1.d3,1.d3,1.d3,1.d3,1.d3,1.d3,1.d3,1.d3,1.d3,1.d3,1.d3,1.d3,1.d3,1.d3,1.d3,1.d3,1.d3,1.d3,1.d3,                         &end
 &nmbtdif tdmpc=0.                                           &end
 &nmfrc  ffrc='$FRC',   oper=f, nfcs=1                       &end
 &nmsfrc fsfrc='$SFRC', ofrc=t, nsfcs=1, fsend=-1,1,30,0,0,0 &end

 &nmchck ocheck=f, ockall=f                                  &end
 &nmdata item='GRZ',    file=' '                             &end

 &nmhisd tintv=1, tavrg=1, tunit='DAY'                       &end
 &nmhist item='PSI',  file='psi', tintv=1, tavrg=1, tunit='DAY' &end
 &nmhist item='CHI',  file='chi', tintv=1, tavrg=1, tunit='DAY' &end
 &nmhist item='U',    file='u',   tintv=1, tavrg=1, tunit='DAY' &end
 &nmhist item='V',    file='v',   tintv=1, tavrg=1, tunit='DAY' &end
 &nmhist item='OMGF', file='w',   tintv=1, tavrg=1, tunit='DAY' &end
 &nmhist item='T',    file='t',   tintv=1, tavrg=1, tunit='DAY' &end
 &nmhist item='Z',    file='z',   tintv=1, tavrg=1, tunit='DAY' &end
 &nmhist item='PS',   file='p',   tintv=1, tavrg=1, tunit='DAY' &end
END_OF_DATA
#
#  run
#
$RUN < $DIR/SYSIN >> $DIR/SYSOUT
#
cd $TDIR
$TRANS
echo job end at `date` >> $DIR/SYSOUT
```

| 参数名 |                       含义                        |
| :----: | :-----------------------------------------------: |
| LNHOME |        home directory of the model package        |
| SYSTEM |                   architecture                    |
|  RUN   |               model executable file               |
|  FDIR  |          directory contains forcing data          |
|  DIR   | directory for first model products, Gtools format |
| BSFILE |            basic state, Gtools format             |
|  FRC   |               initial perturbation                |
|  SFRC  |                  steady forcing                   |
|  TEND  |         length of time integration in day         |

模式的运行只需要执行这个脚本即可。当然，也可以把这个脚本用bash，python，perl等等语言改写。。。