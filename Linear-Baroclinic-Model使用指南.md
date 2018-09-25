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
|   hamp   |                      amplitude in 1/day                      |
|   xdil   |              zonal extent from center longitude              |
|   ydil   |            meridional extent from center latitude            |
|   xcnt   |                   center longitude, 0-360                    |
|   ycnt   |                   center latitude, -90~90                    |
|   kvpr   | vertical profile of forcing: 1: sinusoidal, 2: gamma, 3: uniform |
|   vamp   |                      amplitude in 1/day                      |
|   vdil   |             dilation parameter, only for kvpr=2              |
|   vcnt   |                    center level in sigma                     |
|  owall   |                      full matrix or PWM                      |
| oclassic |                         moist model?                         |

文件中的nmhpr和nmvpr两段分别定义强迫的水平和垂直结构

注意：hamp和vamp如果同号（都设置为正数或者负数），则最终的结果相当于设置热强迫；如果为异号，则最终结果相当于设置冷强迫



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

对于T42的模拟，脚本会稍微有些不同

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
setenv LNHOME   /home/qqf/Documents/Linear-Baroclinic-Model   # ROOT of model
setenv LBMDIR   $LNHOME/model                          # ROOT of LBM 
setenv SYSTEM   N000                                 # execute system
setenv RUN      $LBMDIR/bin/$SYSTEM/lbm2.t42ml20ctintgr # Excutable file
setenv TDIR     $LNHOME/solver/util
setenv FDIR     $LNHOME/qqf/frc                        # Directory for Output
setenv DIR      $LNHOME/qqf/Gtools                       # Directory for Output
setenv BSFILE   $LNHOME/qqf/bs/qqf.t42l20          # Atm. BS File
setenv RSTFILE  $DIR/Restart.amat                      # Restart-Data File
setenv FRC      $FDIR/frc.t42l20.CNP.grd               # initial perturbation
setenv SFRC     $FDIR/frc.t42l20.CNP.grd               # steady forcing
setenv TRANS    $TDIR/gt2gr
setenv TEND     25
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
 &nmhdif order=4, tefold=0.0833, tunit='DAY'                &end
 &nmdelt delt=20, tunit='MIN', inistp=2                     &end
 &nmdamp ddragv=0.5,0.5,0.5,5,30,30,30,30,30,30,30,30,30,30,30,30,30,30,1,0.5,
         ddragd=0.5,0.5,0.5,5,30,30,30,30,30,30,30,30,30,30,30,30,30,30,1,0.5,
         ddragt=0.5,0.5,0.5,5,30,30,30,30,30,30,30,30,30,30,30,30,30,30,1,0.5,
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



PS：LBM是线性模式，所以刚开始积分的时候得到的是大气的第一响应，后面积分中，非线性性会起作用，导致结果失去意义。一般情况下，线性模式只看前20天左右的积分结果。



# 自制背景场

实际使用过程中，我们往往会需要自制背景场。这里给出的程序是利用NCEP-DOE Reanalysis2的数据制作。需要变量有8个。

在使用时，编译命令为：

```shell
gfortran -I/usr/include -L/usr/lib/x86_64-linux-gnu -l netcdff  create_lbm_bs.f90 
```

```fortran
program main
implicit none

! this is the ncep grid settings
integer :: nx=144, ny=73, nt=12, nz=17,nz1=8,nz2=12

! this is the T21 grid settings
integer :: nxo=64, nyo=32

integer :: k,l

character*100 :: filepath
character*20 :: field
real,allocatable :: HGT(:,:,:,:),AIR(:,:,:,:),UWND(:,:,:,:),VWND(:,:,:,:), &
        RHUM(:,:,:,:),OMEGA(:,:,:,:),Q(:,:,:,:),PRES(:,:,:,:)
real,allocatable :: HGTO(:,:,:,:),AIRO(:,:,:,:),UWNDO(:,:,:,:),VWNDO(:,:,:,:), &
        RHUMO(:,:,:,:),OMEGAO(:,:,:,:),QO(:,:,:,:),PRESO(:,:,:,:)
real,allocatable :: level(:),lat(:),lon(:)
real,allocatable :: lat_out(:), lon_out(:), level1(:), level2(:)


allocate(HGT(nx,ny,nz,nt))
allocate(AIR(nx,ny,nz,nt))
allocate(UWND(nx,ny,nz,nt))
allocate(VWND(nx,ny,nz,nt))

allocate(Q(nx,ny,nz1,nt))
allocate(RHUM(nx,ny,nz1,nt))

allocate(OMEGA(nx,ny,nz2,nt))

allocate(PRES(nx,ny,1,nt))

allocate(HGTO(nxo,nyo,nz,nt))
allocate(AIRO(nxo,nyo,nz,nt))
allocate(UWNDO(nxo,nyo,nz,nt))
allocate(VWNDO(nxo,nyo,nz,nt))

allocate(RHUMO(nxo,nyo,nz1,nt))
allocate(QO(nxo,nyo,nz1,nt))

allocate(OMEGAO(nxo,nyo,nz2,nt))

allocate(PRESO(nxo,nyo,1,nt))

allocate(level(nz))
allocate(lat(ny))
allocate(lon(nx))

allocate(lat_out(nyo))
allocate(lon_out(nxo))

allocate(level1(nz1))
allocate(level2(nz2))

filepath="./ncep.nc"

field="hgt"
call readfield(filepath,field,HGT,nx,ny,nz,nt)

field="air"
call readfield(filepath,field,AIR,nx,ny,nz,nt)

field="rhum"
call readfield(filepath,field,RHUM,nx,ny,nz1,nt)

field="omega"
call readfield(filepath,field,OMEGA,nx,ny,nz2,nt)

field="uwnd"
call readfield(filepath,field,UWND,nx,ny,nz,nt)

field="vwnd"
call readfield(filepath,field,VWND,nx,ny,nz,nt)

field="q"
call readfield(filepath,field,Q,nx,ny,nz1,nt)

field="pres"
call readfield(filepath,field,PRES,nx,ny,1,nt)

field="level"
call readoned(filepath,field,level,nz)

field="lat"
call readoned(filepath,field,lat,ny)

field="lon"
call readoned(filepath,field,lon,nx)

field="level1"
call readoned(filepath,field,level1,nz1)

field="level2"
call readoned(filepath,field,level2,nz2)

lat_out = & 
(/-85.761,-80.269,-74.745,-69.213,-63.679,-58.143, & 
-52.607,-47.070,-41.532,-35.995,-30.458,-24.920,  &
-19.382,-13.844,-8.3067,-2.7689,2.7689,8.3067, & 
13.844,19.382,24.920,30.458,35.995,41.532,47.070, & 
52.607,58.143,63.679,69.213,74.745,80.269,85.761/)

lon_out = &
(/0.0, 5.625, 11.25, 16.875, 22.5, 28.125, 33.75, & 
39.375, 45.0, 50.625, 56.25, 61.875, 67.5, 73.125, &
78.75, 84.375,90.0, 95.625, 101.25, 106.875, 112.5, &
118.125, 123.75, 129.375, 135.0, 140.625, 146.25, &
151.875, 157.5, 163.125, 168.75, 174.375,180.0, &
185.625, 191.25, 196.875, 202.5, 208.125, 213.75, &
219.375, 225.0, 230.625, 236.25, 241.875, 247.5, &
253.125, 258.75, 264.375,270.0, 275.625, 281.25, &
286.875, 292.5, 298.125, 303.75, 309.375, 315.0, &
320.625, 326.25, 331.875, 337.5, 343.125, 348.75, 354.375/)

open(unit=1,file='ncep.clim.y79-14.t21.grd',form='unformatted',convert='big_endian')
open(unit=2,file='ncep.clim.y79-14.ps.t21.grd',form='unformatted',convert='big_endian')


do k=1,nt
        call INTP(nx,ny,nz,nxo,nyo,lon,lat,lon_out,lat_out,-9.9e8,HGT(:,:,:,k),HGTO(:,:,:,k))
        call INTP(nx,ny,nz,nxo,nyo,lon,lat,lon_out,lat_out,-9.9e8,AIR(:,:,:,k),AIRO(:,:,:,k))
        call INTP(nx,ny,nz1,nxo,nyo,lon,lat,lon_out,lat_out,-9.9e8,RHUM(:,:,:,k),RHUMO(:,:,:,k))
        call INTP(nx,ny,nz,nxo,nyo,lon,lat,lon_out,lat_out,-9.9e8,UWND(:,:,:,k),UWNDO(:,:,:,k))
        call INTP(nx,ny,nz,nxo,nyo,lon,lat,lon_out,lat_out,-9.9e8,VWND(:,:,:,k),VWNDO(:,:,:,k))
        call INTP(nx,ny,nz1,nxo,nyo,lon,lat,lon_out,lat_out,-9.9e8,Q(:,:,:,k),QO(:,:,:,k))
        call INTP(nx,ny,nz2,nxo,nyo,lon,lat,lon_out,lat_out,-9.9e8,OMEGA(:,:,:,k),OMEGAO(:,:,:,k))
        call INTP(nx,ny,1,nxo,nyo,lon,lat,lon_out,lat_out,-9.9e8,PRES(:,:,:,k),PRESO(:,:,:,k))
end do

PRESO = PRESO / 100.0

do k=1,nt
   do l=1,nz
      write(1) HGTO(:,:,l,k)
   end do
   do l=1,nz1
      write(1) RHUMO(:,:,l,k)
   end do
   do l=1,nz1
      write(1) QO(:,:,l,k)
   end do
   do l=1,nz
      write(1) AIRO(:,:,l,k)
   end do
   do l=1,nz
      write(1) UWNDO(:,:,l,k)
   end do
   do l=1,nz
      write(1) VWNDO(:,:,l,k)
   end do
   do l=1,nz2
      write(1) OMEGAO(:,:,l,k)
   end do
   write(2) PRESO(:,:,1,k)
end do
end program main


subroutine readoned(filepath,field,VAR,nd)
use netcdf
implicit none
integer :: nd
character*20 :: field
real :: VAR(nd)
integer::ncid,ncfile,dimidvar
integer::varidvar
character*100 :: filepath
ncfile=NF90_OPEN(filepath,nf90_nowrite,ncid)
ncfile=NF90_INQ_VARID(ncid,field,varidvar)
ncfile=NF90_GET_VAR(ncid,varidvar,VAR)
ncfile=NF90_CLOSE(ncid)
end subroutine readoned


subroutine readfield(filepath,field,VAR,nx,ny,nz,nt)
use netcdf
implicit none
integer :: nx,ny,nz,nt
character*20 :: field
real :: VAR(nx,ny,nz,nt)
integer::ncid,ncfile,dimidvar
integer::varidvar
character*100 :: filepath
ncfile=NF90_OPEN(filepath,nf90_nowrite,ncid)
ncfile=NF90_INQ_VARID(ncid,field,varidvar)
ncfile=NF90_GET_VAR(ncid,varidvar,VAR)
ncfile=NF90_CLOSE(ncid)
end subroutine readfield

!==============below is from intp.f of LBM, in ln_solver/solver/custom==============


!* SUPPLEMENT ROUTINE IN LBM2.0 on 2001/10/22
!*=============================================================
      SUBROUTINE INTP(IMAX,JMAX,KMAX,IDIM,JDIM,ALONI,COLRAI,ALONO,COLRAO,DMS,DIN,DOUT)
!*
!*     spatial interporation
!*
!C     USAGE:
!C        IMAX: x-dimension for input data
!C        JMAX: y-dimension for input data
!C        KMAX: z-dimension for input data
!C        IDIM: x-dimension for output data
!C        JDIM: y-dimension for output data
!C        ALONI : array of longitude (0 - 360  deg.) for input
!C        COLRAI: array of latitude  (-90 - 90 deg.) for input
!C        ALONO : array of longitude (0 - 360  deg.) for output
!C        COLRAO: array of latitude  (-90 - 90 deg.) for output
!C        DMS   : value for missing grid
!C        DIN   : input array, DIN=DIN(IMAX,JMAX,KMAX)
!C        DOUT  : output array, DOUT=DOUT(IDIM,JDIM,KMAX)
!C
!*
      INTEGER   IMAX, JMAX, KMAX
      INTEGER   IDIM, JDIM
      REAL*4    DMS
      REAL*4    DIN(IMAX,JMAX,KMAX),DOUT(IDIM,JDIM,KMAX)
      REAL*4    ALONI(IMAX),ALONO(IDIM)
      REAL*4    COLRAO(JDIM),COLRAI(JMAX)
!*
      INTEGER   I, J, K
!*
      CALL LINTPMS(IMAX,JMAX,KMAX,IDIM,JDIM,ALONI,COLRAI,ALONO,COLRAO,DMS,DIN,DOUT)
!*
      DO K = 1, KMAX
         DO J = 1, JDIM
            IF( COLRAO( J ) .LT. COLRAI( 1    ) .OR. COLRAO( J ) .GT. COLRAI( JMAX )      ) THEN
               DO I = 1, IDIM
                  DOUT( I,J,K ) = DMS
               ENDDO    
            ENDIF
         ENDDO
      ENDDO
!*
      RETURN
      END SUBROUTINE INTP
!*=============================================================
      SUBROUTINE LINTPMS(IMAX,JMAX,KMAX,IDIM,JDIM,ALONI,COLRAI,ALONO,COLRAO,DMS,DIN,DOUT)
!*
      INTEGER   MAXDM
      PARAMETER ( MAXDM = 1000 )
!*
      INTEGER   IMAX, JMAX, KMAX
      INTEGER   IDIM, JDIM
      REAL*4    DMS
      REAL*4    DIN(IMAX,JMAX,KMAX),DOUT(IDIM,JDIM,KMAX)
      REAL*4    DX(MAXDM),DY(MAXDM),LX(MAXDM),LY(MAXDM),LXP(MAXDM)
      REAL*4    ALONI(IMAX),ALONO(IDIM)
      REAL*4    COLRAO(JDIM),COLRAI(JMAX),WEI(4)

      REAL*4    DLON, FLON, FX, FY, WSUM
      INTEGER   ID, JD, IP, JP, IDP, K
      LOGICAL   FIRST/.TRUE./
      DATA      LY/MAXDM*0/                 ! 1/10/95
!*
      IF(FIRST) THEN
         FIRST=.FALSE.
         DLON=ALONI(2)-ALONI(1)
         DO 100 JP=1,JDIM
            DO 120 JD=2,JMAX
               IF(COLRAI(JD).GE.COLRAO(JP)) THEN
                  LY(JP)=JD
                  DY(JP)=(COLRAO(JP)-COLRAI(JD-1))/(COLRAI(JD)-COLRAI(JD-1))
                  GO TO 100
               ENDIF
  120       CONTINUE
  100    CONTINUE
         DO 140 IP=1,IDIM
            FLON=ALONO(IP)
            IF(ALONO(IP) .GT. 360.) FLON=ALONO(IP)-360.
            DO 141 ID=1,IMAX-1
               IF(ALONI(ID).LE.FLON .AND. ALONI(ID+1).GT.FLON) GO TO 142
  141       CONTINUE
            ID=IMAX
  142       LX(IP)=ID
            IF(LX(IP) .GT. IMAX) LX(IP)=LX(IP)-IMAX
            IDP=MOD(ID,IMAX)+1
            LXP(IP)=IDP
            DX(IP)=(FLON-ALONI(ID))/DLON
            IF(DX(IP) .LT. 0.0) DX(IP)=DX(IP)+360./DLON 
  140    CONTINUE
      ENDIF
!*
      DO 200  K=1,KMAX
         DO 220 JP=1,JDIM
         JD=LY(JP)
         FY=DY(JP)
         DO 220 IP=1,IDIM
         ID=LX(IP)
         IDP=LXP(IP)
         FX=DX(IP)
         WEI(1)=FX*FY
         WEI(2)=(1.0-FX)*FY
         WEI(3)=FX*(1.0-FY)
         WEI(4)=(1.0-FX)*(1.0-FY)
         IF(DIN(IDP ,JD  ,K) .EQ. DMS) WEI(1)=0.0
         IF(DIN(ID  ,JD  ,K) .EQ. DMS) WEI(2)=0.0
         IF(DIN(IDP ,JD-1,K) .EQ. DMS) WEI(3)=0.0
         IF(DIN(ID  ,JD-1,K) .EQ. DMS) WEI(4)=0.0
         WSUM=WEI(1)+WEI(2)+WEI(3)+WEI(4)
         IF(WSUM .LE. 0.0 .OR. JD .LE. 1) THEN
            DOUT(IP,JP,K)=DMS
         ELSE
            DOUT(IP,JP,K)=WEI(1)*DIN(IDP ,JD  ,K)+WEI(2)*DIN(ID  ,JD  ,K)+WEI(3)*DIN(IDP ,JD-1,K)+WEI(4)*DIN(ID  ,JD-1,K)
            DOUT(IP,JP,K)=DOUT(IP,JP,K)/WSUM
         ENDIF
  220    CONTINUE
  200 CONTINUE
!*
      RETURN
      END SUBROUTINE LINTPMS
```

对于T42的模拟也很类似，仅仅修改下经纬度即可

```fortran
program main
implicit none

! this is the ncep grid settings
integer :: nx=144, ny=73, nt=12, nz=17,nz1=8,nz2=12

! this is the T21 grid settings
integer :: nxo=128, nyo=64

integer :: k,l

character*100 :: filepath
character*20 :: field
real,allocatable :: HGT(:,:,:,:),AIR(:,:,:,:),UWND(:,:,:,:),VWND(:,:,:,:), &
        RHUM(:,:,:,:),OMEGA(:,:,:,:),Q(:,:,:,:),PRES(:,:,:,:)
real,allocatable :: HGTO(:,:,:,:),AIRO(:,:,:,:),UWNDO(:,:,:,:),VWNDO(:,:,:,:), &
        RHUMO(:,:,:,:),OMEGAO(:,:,:,:),QO(:,:,:,:),PRESO(:,:,:,:)
real,allocatable :: level(:),lat(:),lon(:)
real,allocatable :: lat_out(:), lon_out(:), level1(:), level2(:)


allocate(HGT(nx,ny,nz,nt))
allocate(AIR(nx,ny,nz,nt))
allocate(UWND(nx,ny,nz,nt))
allocate(VWND(nx,ny,nz,nt))

allocate(Q(nx,ny,nz1,nt))
allocate(RHUM(nx,ny,nz1,nt))

allocate(OMEGA(nx,ny,nz2,nt))

allocate(PRES(nx,ny,1,nt))

allocate(HGTO(nxo,nyo,nz,nt))
allocate(AIRO(nxo,nyo,nz,nt))
allocate(UWNDO(nxo,nyo,nz,nt))
allocate(VWNDO(nxo,nyo,nz,nt))

allocate(RHUMO(nxo,nyo,nz1,nt))
allocate(QO(nxo,nyo,nz1,nt))

allocate(OMEGAO(nxo,nyo,nz2,nt))

allocate(PRESO(nxo,nyo,1,nt))

allocate(level(nz))
allocate(lat(ny))
allocate(lon(nx))

allocate(lat_out(nyo))
allocate(lon_out(nxo))

allocate(level1(nz1))
allocate(level2(nz2))

filepath="./ncep.nc"

field="hgt"
call readfield(filepath,field,HGT,nx,ny,nz,nt)

field="air"
call readfield(filepath,field,AIR,nx,ny,nz,nt)

field="rhum"
call readfield(filepath,field,RHUM,nx,ny,nz1,nt)

field="omega"
call readfield(filepath,field,OMEGA,nx,ny,nz2,nt)

field="uwnd"
call readfield(filepath,field,UWND,nx,ny,nz,nt)

field="vwnd"
call readfield(filepath,field,VWND,nx,ny,nz,nt)

field="q"
call readfield(filepath,field,Q,nx,ny,nz1,nt)

field="pres"
call readfield(filepath,field,PRES,nx,ny,1,nt)

field="level"
call readoned(filepath,field,level,nz)

field="lat"
call readoned(filepath,field,lat,ny)

field="lon"
call readoned(filepath,field,lon,nx)

field="level1"
call readoned(filepath,field,level1,nz1)

field="level2"
call readoned(filepath,field,level2,nz2)

lat_out = & 
(/-87.8638, -85.0965, -82.3129, -79.5256, -76.7369, -73.9475, -71.1577, &
-68.3678, -65.5776, -62.7873, -59.997, -57.2066, -54.4162, -51.6257, &
-48.8352, -46.0447, -43.2542, -40.4636, -37.6731, -34.8825, -32.0919, &
-29.3014, -26.5108, -23.7202, -20.9296, -18.139, -15.3484, -12.5578, &
-9.76715, -6.97653, -4.18592, -1.39531, 1.39531, 4.18592, 6.97653, &
9.76715, 12.5578, 15.3484, 18.139, 20.9296, 23.7202, 26.5108, 29.3014, &
32.0919, 34.8825, 37.6731, 40.4636, 43.2542, 46.0447, 48.8352, 51.6257, &
54.4162, 57.2066, 59.997, 62.7873, 65.5776, 68.3678, 71.1577, 73.9475, &
76.7369, 79.5256, 82.3129, 85.0965, 87.8638/)

lon_out = &
(/0.0, 2.8125, 5.625, 8.4375, 11.25, 14.0625, 16.875, 19.6875, 22.5, &
25.3125, 28.125, 30.9375, 33.75, 36.5625, 39.375, 42.1875, 45.0, 47.8125, &
50.625, 53.4375, 56.25, 59.0625, 61.875, 64.6875, 67.5, 70.3125, 73.125, &
75.9375, 78.75, 81.5625, 84.375, 87.1875, 90.0, 92.8125, 95.625, 98.4375, &
101.25, 104.0625, 106.875, 109.6875, 112.5, 115.3125, 118.125, 120.9375, &
123.75, 126.5625, 129.375, 132.1875, 135.0, 137.8125, 140.625, 143.4375, &
146.25, 149.0625, 151.875, 154.6875, 157.5, 160.3125, 163.125, 165.9375, & 
168.75, 171.5625, 174.375, 177.1875, 180.0, 182.8125, 185.625, 188.4375, &
191.25, 194.0625, 196.875, 199.6875, 202.5, 205.3125, 208.125, 210.9375, &
213.75, 216.5625, 219.375, 222.1875, 225.0, 227.8125, 230.625, 233.4375, &
236.25, 239.0625, 241.875, 244.6875, 247.5, 250.3125, 253.125, 255.9375, &
258.75, 261.5625, 264.375, 267.1875, 270.0, 272.8125, 275.625, 278.4375, &
281.25, 284.0625, 286.875, 289.6875, 292.5, 295.3125, 298.125, 300.9375, &
303.75, 306.5625, 309.375, 312.1875, 315.0, 317.8125, 320.625, 323.4375, &
326.25, 329.0625, 331.875, 334.6875, 337.5, 340.3125, 343.125, 345.9375, &
348.75, 351.5625, 354.375, 357.1875/)

open(unit=1,file='ncep.clim.y79-14.t42.grd',form='unformatted',convert='big_endian')
open(unit=2,file='ncep.clim.y79-14.ps.t42.grd',form='unformatted',convert='big_endian')


do k=1,nt
        call INTP(nx,ny,nz,nxo,nyo,lon,lat,lon_out,lat_out,-9.9e8,HGT(:,:,:,k),HGTO(:,:,:,k))
        call INTP(nx,ny,nz,nxo,nyo,lon,lat,lon_out,lat_out,-9.9e8,AIR(:,:,:,k),AIRO(:,:,:,k))
        call INTP(nx,ny,nz1,nxo,nyo,lon,lat,lon_out,lat_out,-9.9e8,RHUM(:,:,:,k),RHUMO(:,:,:,k))
        call INTP(nx,ny,nz,nxo,nyo,lon,lat,lon_out,lat_out,-9.9e8,UWND(:,:,:,k),UWNDO(:,:,:,k))
        call INTP(nx,ny,nz,nxo,nyo,lon,lat,lon_out,lat_out,-9.9e8,VWND(:,:,:,k),VWNDO(:,:,:,k))
        call INTP(nx,ny,nz1,nxo,nyo,lon,lat,lon_out,lat_out,-9.9e8,Q(:,:,:,k),QO(:,:,:,k))
        call INTP(nx,ny,nz2,nxo,nyo,lon,lat,lon_out,lat_out,-9.9e8,OMEGA(:,:,:,k),OMEGAO(:,:,:,k))
        call INTP(nx,ny,1,nxo,nyo,lon,lat,lon_out,lat_out,-9.9e8,PRES(:,:,:,k),PRESO(:,:,:,k))
end do

PRESO = PRESO / 100.0

do k=1,nt
   do l=1,nz
      write(1) HGTO(:,:,l,k)
   end do
   do l=1,nz1
      write(1) RHUMO(:,:,l,k)
   end do
   do l=1,nz1
      write(1) QO(:,:,l,k)
   end do
   do l=1,nz
      write(1) AIRO(:,:,l,k)
   end do
   do l=1,nz
      write(1) UWNDO(:,:,l,k)
   end do
   do l=1,nz
      write(1) VWNDO(:,:,l,k)
   end do
   do l=1,nz2
      write(1) OMEGAO(:,:,l,k)
   end do
   write(2) PRESO(:,:,1,k)
end do
end program main


subroutine readoned(filepath,field,VAR,nd)
use netcdf
implicit none
integer :: nd
character*20 :: field
real :: VAR(nd)
integer::ncid,ncfile,dimidvar
integer::varidvar
character*100 :: filepath
ncfile=NF90_OPEN(filepath,nf90_nowrite,ncid)
ncfile=NF90_INQ_VARID(ncid,field,varidvar)
ncfile=NF90_GET_VAR(ncid,varidvar,VAR)
ncfile=NF90_CLOSE(ncid)
end subroutine readoned


subroutine readfield(filepath,field,VAR,nx,ny,nz,nt)
use netcdf
implicit none
integer :: nx,ny,nz,nt
character*20 :: field
real :: VAR(nx,ny,nz,nt)
integer::ncid,ncfile,dimidvar
integer::varidvar
character*100 :: filepath
ncfile=NF90_OPEN(filepath,nf90_nowrite,ncid)
ncfile=NF90_INQ_VARID(ncid,field,varidvar)
ncfile=NF90_GET_VAR(ncid,varidvar,VAR)
ncfile=NF90_CLOSE(ncid)
end subroutine readfield

!==============below is from intp.f of LBM, in ln_solver/solver/custom==============


!* SUPPLEMENT ROUTINE IN LBM2.0 on 2001/10/22
!*=============================================================
      SUBROUTINE INTP(IMAX,JMAX,KMAX,IDIM,JDIM,ALONI,COLRAI,ALONO,COLRAO,DMS,DIN,DOUT)
!*
!*     spatial interporation
!*
!C     USAGE:
!C        IMAX: x-dimension for input data
!C        JMAX: y-dimension for input data
!C        KMAX: z-dimension for input data
!C        IDIM: x-dimension for output data
!C        JDIM: y-dimension for output data
!C        ALONI : array of longitude (0 - 360  deg.) for input
!C        COLRAI: array of latitude  (-90 - 90 deg.) for input
!C        ALONO : array of longitude (0 - 360  deg.) for output
!C        COLRAO: array of latitude  (-90 - 90 deg.) for output
!C        DMS   : value for missing grid
!C        DIN   : input array, DIN=DIN(IMAX,JMAX,KMAX)
!C        DOUT  : output array, DOUT=DOUT(IDIM,JDIM,KMAX)
!C
!*
      INTEGER   IMAX, JMAX, KMAX
      INTEGER   IDIM, JDIM
      REAL*4    DMS
      REAL*4    DIN(IMAX,JMAX,KMAX),DOUT(IDIM,JDIM,KMAX)
      REAL*4    ALONI(IMAX),ALONO(IDIM)
      REAL*4    COLRAO(JDIM),COLRAI(JMAX)
!*
      INTEGER   I, J, K
!*
      CALL LINTPMS(IMAX,JMAX,KMAX,IDIM,JDIM,ALONI,COLRAI,ALONO,COLRAO,DMS,DIN,DOUT)
!*
      DO K = 1, KMAX
         DO J = 1, JDIM
            IF( COLRAO( J ) .LT. COLRAI( 1    ) .OR. COLRAO( J ) .GT. COLRAI( JMAX )      ) THEN
               DO I = 1, IDIM
                  DOUT( I,J,K ) = DMS
               ENDDO    
            ENDIF
         ENDDO
      ENDDO
!*
      RETURN
      END SUBROUTINE INTP
!*=============================================================
      SUBROUTINE LINTPMS(IMAX,JMAX,KMAX,IDIM,JDIM,ALONI,COLRAI,ALONO,COLRAO,DMS,DIN,DOUT)
!*
      INTEGER   MAXDM
      PARAMETER ( MAXDM = 1000 )
!*
      INTEGER   IMAX, JMAX, KMAX
      INTEGER   IDIM, JDIM
      REAL*4    DMS
      REAL*4    DIN(IMAX,JMAX,KMAX),DOUT(IDIM,JDIM,KMAX)
      REAL*4    DX(MAXDM),DY(MAXDM),LX(MAXDM),LY(MAXDM),LXP(MAXDM)
      REAL*4    ALONI(IMAX),ALONO(IDIM)
      REAL*4    COLRAO(JDIM),COLRAI(JMAX),WEI(4)

      REAL*4    DLON, FLON, FX, FY, WSUM
      INTEGER   ID, JD, IP, JP, IDP, K
      LOGICAL   FIRST/.TRUE./
      DATA      LY/MAXDM*0/                 ! 1/10/95
!*
      IF(FIRST) THEN
         FIRST=.FALSE.
         DLON=ALONI(2)-ALONI(1)
         DO 100 JP=1,JDIM
            DO 120 JD=2,JMAX
               IF(COLRAI(JD).GE.COLRAO(JP)) THEN
                  LY(JP)=JD
                  DY(JP)=(COLRAO(JP)-COLRAI(JD-1))/(COLRAI(JD)-COLRAI(JD-1))
                  GO TO 100
               ENDIF
  120       CONTINUE
  100    CONTINUE
         DO 140 IP=1,IDIM
            FLON=ALONO(IP)
            IF(ALONO(IP) .GT. 360.) FLON=ALONO(IP)-360.
            DO 141 ID=1,IMAX-1
               IF(ALONI(ID).LE.FLON .AND. ALONI(ID+1).GT.FLON) GO TO 142
  141       CONTINUE
            ID=IMAX
  142       LX(IP)=ID
            IF(LX(IP) .GT. IMAX) LX(IP)=LX(IP)-IMAX
            IDP=MOD(ID,IMAX)+1
            LXP(IP)=IDP
            DX(IP)=(FLON-ALONI(ID))/DLON
            IF(DX(IP) .LT. 0.0) DX(IP)=DX(IP)+360./DLON 
  140    CONTINUE
      ENDIF
!*
      DO 200  K=1,KMAX
         DO 220 JP=1,JDIM
         JD=LY(JP)
         FY=DY(JP)
         DO 220 IP=1,IDIM
         ID=LX(IP)
         IDP=LXP(IP)
         FX=DX(IP)
         WEI(1)=FX*FY
         WEI(2)=(1.0-FX)*FY
         WEI(3)=FX*(1.0-FY)
         WEI(4)=(1.0-FX)*(1.0-FY)
         IF(DIN(IDP ,JD  ,K) .EQ. DMS) WEI(1)=0.0
         IF(DIN(ID  ,JD  ,K) .EQ. DMS) WEI(2)=0.0
         IF(DIN(IDP ,JD-1,K) .EQ. DMS) WEI(3)=0.0
         IF(DIN(ID  ,JD-1,K) .EQ. DMS) WEI(4)=0.0
         WSUM=WEI(1)+WEI(2)+WEI(3)+WEI(4)
         IF(WSUM .LE. 0.0 .OR. JD .LE. 1) THEN
            DOUT(IP,JP,K)=DMS
         ELSE
            DOUT(IP,JP,K)=WEI(1)*DIN(IDP ,JD  ,K)+WEI(2)*DIN(ID  ,JD  ,K)+WEI(3)*DIN(IDP ,JD-1,K)+WEI(4)*DIN(ID  ,JD-1,K)
            DOUT(IP,JP,K)=DOUT(IP,JP,K)/WSUM
         ENDIF
  220    CONTINUE
  200 CONTINUE
!*
      RETURN
      END SUBROUTINE LINTPMS
```

8个变量分别写出到两个文件中，一个文件中仅含地表气压，另一个文件包含其余变量。为了出图，可以配上ctl文件，然后用cdo转换为nc格式文件：

```shell
cdo -f nc import_binary XXX.ctl XXX.nc
```

```
* NCEP climatology interporated into T21 Gaussian grid
*
dset ^ncep.clim.y79-14.t21.grd
OPTIONS SEQUENTIAL
options big_endian
undef 9.999E+20
title NCEP cdas climatology (T21)
XDEF 64 LINEAR 0. 5.625
YDEF 32 LEVELS -85.761 -80.269 -74.745 -69.213 -63.679 -58.143 -52.607 
-47.070 -41.532 -35.995 -30.458 -24.920 -19.382 -13.844 -8.3067 -2.7689 
2.7689 8.3067 13.844 19.382 24.920 30.458 35.995 41.532 47.070 52.607 
58.143 63.679 69.213 74.745 80.269 85.761
tdef      12 linear 00Z01jan79 1mo
zdef 17 levels
1000 925 850 700 600 500 400 300 250 200 150 100 70 50 30 20 10
vars      7
z    17 99 Geopotential height [gpm]
rh    8 99 Relative humidity [%]
q     8 99 Specific humidity [kg/kg]
t    17 99 Temperature [K]
u    17 99 zonal wind [m/s]
v    17 99 meridional wind [m/s]
omg  12 99 Pressure vertical velocity [Pa/s]
endvars
```

```
* NCEP climatology interporated into T21 Gaussian grid
*
DSET ^ncep.clim.y79-14.ps.t21.grd
OPTIONS SEQUENTIAL
options big_endian
TITLE NCEP cdas climateology (T21)
UNDEF -999.
XDEF 64 LINEAR 0 5.625
YDEF 32 LEVELS -85.761 -80.269 -74.745 -69.213 -63.679 -58.143 -52.607 
-47.070 -41.532 -35.995 -30.458 -24.920 -19.382 -13.844 -8.3067 -2.7689  
2.7689 8.3067 13.844 19.382 24.920 30.458 35.995 41.532 47.070 52.607
58.143 63.679 69.213 74.745 80.269 85.761
ZDEF 1 LEVELS 1000
TDEF 12 LINEAR 00Z01jan79 1mo
VARS 1
ps   0 10 surface pressure
ENDVARS
```

**注意**：文件中一律使用big endian的方式写入

对于我使用的NCEP-DOE Reanalysis2的数据，可以先把数据合并为一个文件，然后再利用上面这段fortran程序处理。这里给出ncl的预处理程序

```
load "$NCARG_ROOT/lib/ncarg/nclscripts/contrib/calendar_decode2.ncl"

START_YEAR = 1979
END_YEAR = 2014

f1 = addfile("./nc/air.mon.mean.nc","r")
f2 = addfile("./nc/hgt.mon.mean.nc","r")  
f3 = addfile("./nc/omega.mon.mean.nc","r")
f4 = addfile("./nc/rhum.mon.mean.nc","r")  
f5 = addfile("./nc/uwnd.mon.mean.nc","r")
f6 = addfile("./nc/vwnd.mon.mean.nc","r")
f7 = addfile("./nc/pres.sfc.mon.mean.nc","r")

time = calendar_decode2(f1->time,0)
year_idx=ind(time(:,0).ge.START_YEAR.and.time(:,0).le.END_YEAR)
lat = f1->lat
lon = f1->lon
level = f1->level

air = short2flt(f1->air)
hgt = short2flt(f2->hgt)
omega = short2flt(f3->omega)
rhum = short2flt(f4->rhum)
uwnd = short2flt(f5->uwnd)
vwnd = short2flt(f6->vwnd)
pres = short2flt(f7->pres)
q    = mixhum_ptrh(conform(air,level,1),air,rhum,2) 
copy_VarCoords(air,q)

air_clim = clmMonTLLL(air(year_idx,:,:,:))
hgt_clim = clmMonTLLL(hgt(year_idx,:,:,:))
omega_clim = clmMonTLLL(omega(year_idx,:,:,:))
rhum_clim = clmMonTLLL(rhum(year_idx,:,:,:))
uwnd_clim = clmMonTLLL(uwnd(year_idx,:,:,:))
vwnd_clim = clmMonTLLL(vwnd(year_idx,:,:,:))
pres_clim = clmMonTLL(pres(year_idx,:,:))
q_clim = clmMonTLLL(q(year_idx,:,:,:))

level1 = (/1000., 925., 850., 700., 600., 500., 400., 300./)
level2 = (/1000., 925., 850., 700., 600., 500., 400., 300., 250., 200., 150., 100./)

air_out = new((/12,17,73,144/),"float")
hgt_out = new((/12,17,73,144/),"float")
uwnd_out = new((/12,17,73,144/),"float")
vwnd_out = new((/12,17,73,144/),"float")

omega_out = new((/12,12,73,144/),"float")

rhum_out = new((/12,8,73,144/),"float")
q_out = new((/12,8,73,144/),"float")

pres_out = new((/12,1,73,144/),"float")

air_out = air_clim(month|:,level|:,lat|::-1,lon|:)
hgt_out = hgt_clim(month|:,level|:,lat|::-1,lon|:)
uwnd_out = uwnd_clim(month|:,level|:,lat|::-1,lon|:)
vwnd_out = vwnd_clim(month|:,level|:,lat|::-1,lon|:)

omega_out = omega_clim(month|:,level|0:11,lat|::-1,lon|:)
omega_out!1="level2"
omega_out&level2=level2

rhum_out = rhum_clim(month|:,level|0:7,lat|::-1,lon|:)
rhum_out!1="level1"
rhum_out&level1=level1

q_out = q_clim(month|:,level|0:7,lat|::-1,lon|:)
q_out!1="level1"
q_out&level1=level1

pres_out(:,0,:,:) = pres_clim(month|:,lat|::-1,lon|:)
;printVarSummary(omega_out)
;air_out = air_clim(lon|:,lat|::-1,level|:,month|:)
;hgt_out = hgt_clim(lon|:,lat|::-1,level|:,month|:)
;omega_out = omega_clim(lon|:,lat|::-1,level|:,month|:)
;rhum_out = rhum_clim(lon|:,lat|::-1,level|:,month|:)
;uwnd_out = uwnd_clim(lon|:,lat|::-1,level|:,month|:)
;vwnd_out = vwnd_clim(lon|:,lat|::-1,level|:,month|:)
;q_out = q_clim(lon|:,lat|::-1,level|:,month|:)


outfile = addfile("ncep.nc","c")
outfile->level = level
outfile->level1 = level1
outfile->level2 = level2
outfile->lon = lon
outfile->lat = lat(::-1)
outfile->air = air_out
outfile->hgt = hgt_out
outfile->omega = omega_out
outfile->rhum = rhum_out
outfile->uwnd = uwnd_out
outfile->vwnd = vwnd_out
outfile->q = q_out
outfile->pres = pres_out
```

# 强迫场中设置两个强迫源

默认的mkfrcng只能设置一个强迫源，通过修改mkfrcng的源文件，我们可以实现在一个强迫场中设置两个强迫源。我的做法是直接新建mymkfrcng.f文件，然后内容如下：

```fortran
      PROGRAM MKFRCNG
*
*     derived from out_agcm/donly/mktfrc.f on 1998/11/08
*     This routine makes simplified forcing function
*     for the 3-D linear response.
*
*     Alternative selection of the forcing shape is:
*      Horizontal: Elliptic function/zonally uniform function 
*      Vertical  : Sinusoidal function/Gamma function/uniform
*
      include 'dim.f'
*
*
      INTEGER      MAXN
      PARAMETER (  MAXN = 2*NMAX*(NVAR*KMAX+1) )
*
      REAL*8       GFRCT( IDIM*JMAX, KMAX)  
      REAL*8       WFRCT( NMDIM, KMAX)  
      REAL*8       WFRCF( NMDIM, KMAX)  
      REAL*8       WXVOR( MAXN, KMAX, 0:NTR)  
      REAL*8       WXDIV( MAXN, KMAX, 0:NTR)  
      REAL*8       WXTMP( MAXN, KMAX, 0:NTR)  
      REAL*8       WXPS ( MAXN, 0:NTR )
      REAL*8       WXSPH( MAXN, KMAX, 0:NTR)  
*
      REAL*8       ALON( IDIM, JMAX )
      REAL*8       ALAT( IDIM, JMAX )
      REAL*8       SIG( KMAX )
      REAL*8       DLON( IDIM, JMAX ) !! unused
      REAL*8       SIGM( KMAX+1 )     !! unused
      REAL*8       DSIG( KMAX )       !! unused
      REAL*8       DSIGM( KMAX+1 )    !! unused
      CHARACTER    HALON *(16)        !! unused
      CHARACTER    HSIG *(16)         !! unused
      CHARACTER    HSIGM *(16)        !! unused
*
*     [work]
      REAL*8       AZ( KMAX )
      REAL*8       AXY( IDIM, JMAX)
      REAL*8       DUM( IMAX, JMAX)
      REAL*8       DUMX( IMAX, JMAX)
      REAL*8       PI
      REAL*8       V1, V2, V
      INTEGER      NMO   ( 2, 0:MMAX, 0:LMAX ) !! order of spect. suffix
      INTEGER      IFM, IFG
      INTEGER      I, J, K, L, M, IJ, KV, IW, JW( 0:NTR ), LEND
      INTEGER      NOUT
      REAL*8       S2D
*
*     [intrinsic]
      INTRINSIC    DSIN, DSQRT, DBLE, SNGL
*
      CHARACTER    CHPR( MAXH )*15
      CHARACTER    CVPR( MAXV )*15
*
      LOGICAL      OVOR         !! set vorticity forcing ? 
      LOGICAL      ODIV         !! set divergence forcing ? 
      LOGICAL      OTMP         !! set tmperature forcing ? 
      LOGICAL      OPS          !! set sfc. pressure forcing ? 
      LOGICAL      OSPH         !! set humidity forcing ? 
      LOGICAL      OWALL        !! write all the wavenumber
      LOGICAL      OCLASSIC     !! classic dry model
      CHARACTER*90 CFM          !! output file name (matrix)
      CHARACTER*90 CFG          !! output file name (grads)
      INTEGER      KHPR         !! type of horizontal profile
      INTEGER      KVPR         !! type of vertical profile
      REAL*8       HAMP         !! horizontal amplitude
      REAL*8       VAMP         !! vertical amplitude
      REAL*8       XDIL         !! x-dilation for Elliptic-shape
      REAL*8       YDIL         !! y-dilation for Elliptic-shape
      REAL*8       VDIL         !! dilation for Gamma-shape
      REAL*8       XCNT         !! x-center for Elliptic-shape
      REAL*8       YCNT         !! y-center for Elliptic-shape
      REAL*8       VCNT         !! vertical center for Gamma-shape
      REAL*8       FACT( 5 )    !! factor (unused)


      ! following is used to set another force
      REAL*8       V1QQF, V2QQF, VQQF
      REAL*8       HAMP2         !! horizontal amplitude
      REAL*8       XDIL2         !! x-dilation for Elliptic-shape
      REAL*8       YDIL2         !! y-dilation for Elliptic-shape
      REAL*8       XCNT2         !! x-center for Elliptic-shape
      REAL*8       YCNT2         !! y-center for Elliptic-shape


      NAMELIST    /NMFIN/   CFM, CFG, FACT
      NAMELIST    /NMVAR/   OVOR, ODIV, OTMP, OPS, OSPH
      NAMELIST    /NMHPR/   KHPR, HAMP, XDIL, YDIL, XCNT, YCNT
      NAMELIST    /NMVPR/   KVPR, VAMP, VDIL, VCNT
      NAMELIST    /NMALL/   OWALL
      NAMELIST    /NMCLS/   OCLASSIC
*
      DATA         NOUT / 6 /
      DATA         S2D / 86400.D0 /
      DATA         CHPR
     $           /'Elliptic       ','Zonally uniform'/
      DATA         CVPR
     $           /'Sinusoidal     ','Gamma          ',
     $            'Uniform        '/
*                  123456789012345   123456789012345   
      DATA    OVOR, ODIV, OTMP, OPS, OSPH
     &       / .FALSE., .FALSE., .FALSE., .FALSE., .FALSE. /
*
*     open the NAMELIST file
*
      REWIND( 1 )
      OPEN ( 0, FILE='IEEE_ERROR')
      OPEN ( 1, FILE = 'SETPAR', STATUS = 'OLD')
*
      READ( 1, NMFIN) 
      REWIND( 1 )
      READ( 1, NMVAR) 
      REWIND( 1 )
      READ( 1, NMHPR) 
      REWIND( 1 )
      READ( 1, NMVPR) 
      REWIND( 1 )
      READ( 1, NMALL) 
      REWIND( 1 )
      READ( 1, NMCLS) 
*
*
      WRITE( NOUT, * ) '### MAKE FORCING MATRIX ###'
*
      CALL SETCONS
      CALL SPSTUP               !! spherical harmonic functions
      CALL SETNMO2
     O     ( NMO   ,
     D     MMAX  , LMAX  , NMAX  , MINT    )

      WRITE( NOUT, * )
      WRITE( NOUT, * ) '................'
      WRITE( NOUT, * ) 'Selected shape:'
      WRITE( NOUT, * ) '  Horizontal:',CHPR( KHPR )
      WRITE( NOUT, * ) '  Vertical  :',CVPR( KVPR )
      WRITE( NOUT, * ) '................'
      WRITE( NOUT, * )
*
*     open files
*
      IFM = 10
      IFG = 20
      OPEN ( IFM, FILE = CFM, FORM = 'UNFORMATTED', STATUS = 'UNKNOWN' ) 
      OPEN ( IFG, FILE = CFG, FORM = 'UNFORMATTED', STATUS = 'UNKNOWN' )
      WRITE( NOUT, * ) 'Matrix file         :', CFM
      WRITE( NOUT, * ) 'Matrix file (GrADS) :', CFG
      WRITE( NOUT, * ) '................'
      WRITE( NOUT, * ) 
*
      PI = ATAN( 1.0D0 ) * 4.0D0
      CALL SETLON
     O         ( ALON  , DLON , HALON  )
      CALL SETLAT
     O         ( ALAT  , DLON , HALON  )
      CALL SETSIG
     O         ( SIG   , DSIG , HSIG  , 
     O           SIGM  , DSIGM, HSIGM  )
*
      DO 1 J = 1, JMAX
         DO 1 I = 1, IDIM
            ALON( I, J) = ALON( I, J) * 360.D0 / ( 2.D0 * PI )
            ALAT( I, J) = ALAT( I, J) * 360.D0 / ( 2.D0 * PI )
 1    CONTINUE
*
      CALL SETZ( GFRCT, IDIM*JMAX*KMAX )
      CALL SETZ( DUM, IMAX*JMAX )
      CALL SETZ( WFRCT, NMDIM*KMAX )
      CALL SETZ( WFRCF, NMDIM*KMAX )
*
*     vertical profile
*
      IF( KMAX .GT. 1 ) THEN
         DO 10 K = 1, KMAX
            IF( KVPR .EQ. 1 ) AZ( K ) = VAMP * DSIN( PI * SIG( K ) )
            IF( KVPR .EQ. 2 ) AZ( K ) = VAMP * 
     $           DEXP( - VDIL * ( SIG( K ) - VCNT )**2 )
            IF( KVPR .EQ. 3 ) AZ( K ) = VAMP 
 10      CONTINUE
         WRITE( NOUT, * ) 'Set vertical shape'
         WRITE( NOUT, * ) '................'
         WRITE( NOUT, * ) 
      ELSE
         AZ( K ) = VAMP
      ENDIF
*
*     horizontal profile
*
      IJ = 0
      DO 20 J = 1, JMAX
         DO 30 I = 1, IDIM
            IJ = IJ + 1
*
            IF( KHPR .EQ. 1 ) THEN !! Elliptic
               IF( XDIL .LE. 0.D0 .OR. YDIL .LE. 0.D0 ) THEN
                  WRITE( NOUT, *) 
     $                 '### Parameter XDIL/YDIL not correct ###' 
                  STOP
               ENDIF
*
               V1 = ( ALON( I, J ) - XCNT )**2 / XDIL**2
               IF( ALON(I,J).LT.180. .AND. XCNT+XDIL.GE.360. )
     $              V1 = ( 360. + ALON( I, J ) - XCNT )**2 / XDIL**2
               IF( ALON(I,J).GT.180. .AND. XCNT-XDIL.LE.0. )
     $              V1 = ( ALON( I, J ) - 360. - XCNT )**2 / XDIL**2
               V2 = ( ALAT( I, J ) - YCNT )**2 / YDIL**2
               V = V1 + V2
               !IF( V .GT. 1. ) THEN
               !   AXY( I, J) = 0.D0
               !ELSE
               !   V = DSQRT( V ) 
               !   AXY( I, J) = HAMP * ( 1.D0 - V )
               !ENDIF

               ! used to set another force
               !      REAL*8       HAMP2         !! horizontal amplitude
               !      REAL*8       XDIL2         !! x-dilation for Elliptic-shape
               !      REAL*8       YDIL2         !! y-dilation for Elliptic-shape
               !      REAL*8       XCNT2         !! x-center for Elliptic-shape
               !      REAL*8       YCNT2         !! y-center for Elliptic-shape
               HAMP2 = 2.
               XDIL2 = 25.
               YDIL2 = 11.
               XCNT2 = 75.
               YCNT2 = 0.
               V1QQF = ( ALON( I, J ) - XCNT2 )**2 / XDIL2**2
               IF( ALON(I,J).LT.180. .AND. XCNT2+XDIL2.GE.360. )
     $             V1QQF = ( 360. + ALON( I, J ) - XCNT2 )**2 / XDIL2**2
               IF( ALON(I,J).GT.180. .AND. XCNT2-XDIL2.LE.0. )
     $             V1QQF = ( ALON( I, J ) - 360. - XCNT2 )**2 / XDIL2**2
               V2QQF = ( ALAT( I, J ) - YCNT2 )**2 / YDIL2**2
               VQQF = V1QQF + V2QQF

               IF( VQQF .GT. 1. .AND. V .GT. 1. ) THEN
                  AXY( I, J) = 0.D0
               ELSE
                  IF(VQQF .LE. 1.) THEN
                      VQQF = DSQRT( VQQF ) 
                      AXY( I, J) = HAMP2 * ( 1.D0 - VQQF )
                  ENDIF
                  IF(V .LE. 1.) THEN
                      V = DSQRT( V )
                      AXY( I, J) = HAMP * ( 1.D0 - V )
                  ENDIF
               ENDIF

            ENDIF
               
            IF( KHPR .EQ. 2 ) THEN !! zonal uniform
*
               IF( ALAT( I, J) .GE. YCNT - YDIL .AND.
     $             ALAT( I, J) .LE. YCNT + YDIL ) THEN
                  V = HAMP
               ELSE
                  V = 0.D0
               ENDIF

               AXY( I, J) = V
            ENDIF

            KV = KMAX
            IF( OPS  ) KV = 1 
            DO 40 K = 1, KV
               GFRCT( IJ, K) = AXY( I, J) * AZ( K ) / S2D
 40         CONTINUE
*
 30      CONTINUE
 20   CONTINUE
      WRITE( NOUT, * ) 'Set horizontal shape'
      WRITE( NOUT, * ) '................'
      WRITE( NOUT, * ) 
*
*     write GrADS file
*
      DO 100 K = 1, KMAX
         IF( OVOR ) THEN
            DO 110 J = 1, JMAX
               DO 110 I = 1, IMAX
                  IJ = (J-1)*IDIM + I
                  DUMX( I, J) = GFRCT( IJ, K)
 110        CONTINUE
            WRITE( IFG ) ((SNGL(DUMX(I,J)),I=1,IMAX),J=1,JMAX)
         ELSE
            WRITE( IFG ) ((SNGL(DUM(I,J)),I=1,IMAX),J=1,JMAX)
         ENDIF
 100  CONTINUE
*
      DO 120 K = 1, KMAX
         IF( ODIV ) THEN
            DO 130 J = 1, JMAX
               DO 130 I = 1, IMAX
                  IJ = (J-1)*IDIM + I
                  DUMX( I, J) = GFRCT( IJ, K)
 130        CONTINUE
            WRITE( IFG ) ((SNGL(DUMX(I,J)),I=1,IMAX),J=1,JMAX)
         ELSE
            WRITE( IFG ) ((SNGL(DUM(I,J)),I=1,IMAX),J=1,JMAX)
         ENDIF
 120  CONTINUE
*
      DO 140 K = 1, KMAX
         IF( OTMP ) THEN
            DO 150 J = 1, JMAX
               DO 150 I = 1, IMAX
                  IJ = (J-1)*IDIM + I
                  DUMX( I, J) = GFRCT( IJ, K)
 150        CONTINUE
            WRITE( IFG ) ((SNGL(DUMX(I,J)),I=1,IMAX),J=1,JMAX)
         ELSE
            WRITE( IFG ) ((SNGL(DUM(I,J)),I=1,IMAX),J=1,JMAX)
         ENDIF
 140  CONTINUE
*
      IF( OPS ) THEN
         DO 160 J = 1, JMAX
            DO 160 I = 1, IMAX
               IJ = (J-1)*IDIM + I
               DUMX( I, J) = GFRCT( IJ, 1)
 160     CONTINUE
         WRITE( IFG ) ((SNGL(DUMX(I,J)),I=1,IMAX),J=1,JMAX)
      ELSE
         WRITE( IFG ) ((SNGL(DUM(I,J)),I=1,IMAX),J=1,JMAX)
      ENDIF
*
      IF( .NOT. OCLASSIC ) THEN
         DO 170 K = 1, KMAX
            IF( OSPH ) THEN
               DO 180 J = 1, JMAX
                  DO 180 I = 1, IMAX
                     IJ = (J-1)*IDIM + I
                     DUMX( I, J) = GFRCT( IJ, K)
 180           CONTINUE
               WRITE( IFG ) ((SNGL(DUMX(I,J)),I=1,IMAX),J=1,JMAX)
            ELSE
               WRITE( IFG ) ((SNGL(DUM(I,J)),I=1,IMAX),J=1,JMAX)
            ENDIF
 170     CONTINUE
      ENDIF
*
      CLOSE( IFG )
      WRITE( NOUT, * ) 'Written to GrADS file'
      WRITE( NOUT, * ) '................'
      WRITE( NOUT, * ) 
*
*     grid to wave (forcing matrix)
*
      IF( OPS ) THEN
         CALL G2W
     O        ( WFRCT   ,
     I          GFRCT   , '    ', 'POSO', 1    )
      ELSE
         CALL G2W
     O        ( WFRCT   ,
     I          GFRCT   , '    ', 'POSO', KMAX )
      ENDIF
*
*     write down
*
      IW = 0
      DO 200 M = 0, NTR
         LEND = MIN( LMAX, NMAX-M)
         DO 220 K = 1, KMAX
            IW = 0
            DO 210 L = 0, LEND
               IF( M .EQ. 0 .AND. L .EQ. 0 ) GOTO 210
               I = NMO( 1, M, L)
               J = NMO( 2, M, L)
               IW = IW + 1
               WXVOR(IW,K,M)            = WFRCF(I,K)
               WXDIV(IW,K,M)            = WFRCF(I,K)
               WXTMP(IW,K,M)            = WFRCF(I,K)
               WXPS (IW,M  )            = WFRCF(I,1)
               WXSPH(IW,K,M)            = WFRCF(I,K)
               IF( OVOR ) WXVOR(IW,K,M) = WFRCT(I,K)
               IF( ODIV ) WXDIV(IW,K,M) = WFRCT(I,K)
               IF( OTMP ) WXTMP(IW,K,M) = WFRCT(I,K)
               IF(  OPS ) WXPS (IW,M  ) = WFRCT(I,1)
               IF( OSPH ) WXSPH(IW,K,M) = WFRCT(I,K)
               IF( M .EQ. 0 ) GOTO 210
               IW = IW + 1
               WXVOR(IW,K,M)            = WFRCF(J,K)
               WXDIV(IW,K,M)            = WFRCF(J,K)
               WXTMP(IW,K,M)            = WFRCF(J,K)
               WXPS (IW,M  )            = WFRCF(J,1)
               WXSPH(IW,K,M)            = WFRCF(J,K)
               IF( OVOR ) WXVOR(IW,K,M) = WFRCT(J,K)
               IF( ODIV ) WXDIV(IW,K,M) = WFRCT(J,K)
               IF( OTMP ) WXTMP(IW,K,M) = WFRCT(J,K)
               IF(  OPS ) WXPS (IW,M  ) = WFRCT(J,1)
               IF( OSPH ) WXSPH(IW,K,M) = WFRCT(J,K)
  210       CONTINUE
  220    CONTINUE
         IF( .NOT. OWALL ) THEN
            IF( OCLASSIC ) THEN
               WRITE( IFM ) 
     $              ((WXVOR(I,K,M),I=1,IW),K=1,KMAX),
     $              ((WXDIV(I,K,M),I=1,IW),K=1,KMAX),
     $              ((WXTMP(I,K,M),I=1,IW),K=1,KMAX),
     $              ( WXPS (I,M)  ,I=1,IW          )
            ELSE
               WRITE( IFM ) 
     $              ((WXVOR(I,K,M),I=1,IW),K=1,KMAX),
     $              ((WXDIV(I,K,M),I=1,IW),K=1,KMAX),
     $              ((WXTMP(I,K,M),I=1,IW),K=1,KMAX),
     $              ( WXPS (I,M)  ,I=1,IW          ),
     $              ((WXSPH(I,K,M),I=1,IW),K=1,KMAX)
            ENDIF
         ELSE
            JW( M ) = IW
         ENDIF
  200 CONTINUE
      IF( OWALL ) THEN
         IF( OCLASSIC ) THEN
            WRITE( IFM ) 
     $           (((WXVOR(I,K,M),I=1,JW(M)),K=1,KMAX),
     $            ((WXDIV(I,K,M),I=1,JW(M)),K=1,KMAX),
     $            ((WXTMP(I,K,M),I=1,JW(M)),K=1,KMAX),
     $            ( WXPS (I,M)  ,I=1,JW(M)          ),M=0,NTR)
         ELSE
            WRITE( IFM ) 
     $           (((WXVOR(I,K,M),I=1,JW(M)),K=1,KMAX),
     $            ((WXDIV(I,K,M),I=1,JW(M)),K=1,KMAX),
     $            ((WXTMP(I,K,M),I=1,JW(M)),K=1,KMAX),
     $            ( WXPS (I,M)  ,I=1,JW(M)          ),
     $            ((WXSPH(I,K,M),I=1,JW(M)),K=1,KMAX),M=0,NTR)
         ENDIF
      ENDIF
      CLOSE( IFM )
      IF( OWALL ) THEN
         WRITE( NOUT, * ) 'Written to matrix file (all)'
      ELSE
         WRITE( NOUT, * ) 'Written to matrix file (pwm)'
      ENDIF
      WRITE( NOUT, * ) '................'
      WRITE( NOUT, * ) 
*
      WRITE( NOUT, * )
      WRITE( NOUT, * ) '### END OF EXECUTION ###'
      WRITE( NOUT, * )
*
*
  999 STOP
      END
*######################
      SUBROUTINE SETZ( A, IA )
*
*
      INTEGER IA
      REAL*8  A( IA )

      REAL*8  ZERO
      DATA    ZERO / 0.D0 /
      INTEGER I
*
      DO 10 I = 1, IA
         A( I ) = ZERO
 10   CONTINUE

      RETURN
      END
*##########################
      SUBROUTINE SETNMO2    !! order of matrix
     O         ( NMO   ,
     D           MMAX  , LMAX  , NMAX  , MINT    )
*
*   [PARAM] 
      INTEGER    MMAX
      INTEGER    LMAX
      INTEGER    NMAX
      INTEGER    MINT
*
*   [OUTPUT]
      INTEGER    NMO   ( 2, 0:MMAX, 0:LMAX ) !! order of spect. suffix
*
*   [INTERNAL WORK] 
      INTEGER    L, M, MEND, NMH
*
      NMH  = 0
      DO 2200 L = 0, LMAX
         MEND = MIN( MMAX, NMAX-L )
         DO 2100 M = 0, MEND, MINT
            NMH = NMH + 1
            IF ( MMAX .EQ. 0 ) THEN
               NMO ( 1, M, L ) = NMH
               NMO ( 2, M, L ) = NMH
            ELSE
               NMO ( 1, M, L ) = 2* NMH - 1
               NMO ( 2, M, L ) = 2* NMH
            ENDIF
 2100    CONTINUE
 2200 CONTINUE
*
      RETURN
      END
```
上面这部分code是设置两个强迫，但是两个强迫的垂直结构是一致的。下面这段代码是两个强迫分别有不同的垂直结构

```fortran
      PROGRAM MKFRCNG
*
*     derived from out_agcm/donly/mktfrc.f on 1998/11/08
*     This routine makes simplified forcing function
*     for the 3-D linear response.
*
*     Alternative selection of the forcing shape is:
*      Horizontal: Elliptic function/zonally uniform function 
*      Vertical  : Sinusoidal function/Gamma function/uniform
*
      include 'dim.f'
*
*
      INTEGER      MAXN
      PARAMETER (  MAXN = 2*NMAX*(NVAR*KMAX+1) )
*
      REAL*8       GFRCT( IDIM*JMAX, KMAX)  
      REAL*8       WFRCT( NMDIM, KMAX)  
      REAL*8       WFRCF( NMDIM, KMAX)  
      REAL*8       WXVOR( MAXN, KMAX, 0:NTR)  
      REAL*8       WXDIV( MAXN, KMAX, 0:NTR)  
      REAL*8       WXTMP( MAXN, KMAX, 0:NTR)  
      REAL*8       WXPS ( MAXN, 0:NTR )
      REAL*8       WXSPH( MAXN, KMAX, 0:NTR)  
*
      REAL*8       ALON( IDIM, JMAX )
      REAL*8       ALAT( IDIM, JMAX )
      REAL*8       SIG( KMAX )
      REAL*8       DLON( IDIM, JMAX ) !! unused
      REAL*8       SIGM( KMAX+1 )     !! unused
      REAL*8       DSIG( KMAX )       !! unused
      REAL*8       DSIGM( KMAX+1 )    !! unused
      CHARACTER    HALON *(16)        !! unused
      CHARACTER    HSIG *(16)         !! unused
      CHARACTER    HSIGM *(16)        !! unused
*
*     [work]
      REAL*8       AZ( KMAX )
      REAL*8       AXY( IDIM, JMAX)
      REAL*8       DUM( IMAX, JMAX)
      REAL*8       DUMX( IMAX, JMAX)
      REAL*8       PI
      REAL*8       V1, V2, V
      INTEGER      NMO   ( 2, 0:MMAX, 0:LMAX ) !! order of spect. suffix
      INTEGER      IFM, IFG
      INTEGER      I, J, K, L, M, IJ, KV, IW, JW( 0:NTR ), LEND
      INTEGER      NOUT
      REAL*8       S2D
*
*     [intrinsic]
      INTRINSIC    DSIN, DSQRT, DBLE, SNGL
*
      CHARACTER    CHPR( MAXH )*15
      CHARACTER    CVPR( MAXV )*15
*
      LOGICAL      OVOR         !! set vorticity forcing ? 
      LOGICAL      ODIV         !! set divergence forcing ? 
      LOGICAL      OTMP         !! set tmperature forcing ? 
      LOGICAL      OPS          !! set sfc. pressure forcing ? 
      LOGICAL      OSPH         !! set humidity forcing ? 
      LOGICAL      OWALL        !! write all the wavenumber
      LOGICAL      OCLASSIC     !! classic dry model
      CHARACTER*90 CFM          !! output file name (matrix)
      CHARACTER*90 CFG          !! output file name (grads)
      INTEGER      KHPR         !! type of horizontal profile
      INTEGER      KVPR         !! type of vertical profile
      REAL*8       HAMP         !! horizontal amplitude
      REAL*8       VAMP         !! vertical amplitude
      REAL*8       XDIL         !! x-dilation for Elliptic-shape
      REAL*8       YDIL         !! y-dilation for Elliptic-shape
      REAL*8       VDIL         !! dilation for Gamma-shape
      REAL*8       XCNT         !! x-center for Elliptic-shape
      REAL*8       YCNT         !! y-center for Elliptic-shape
      REAL*8       VCNT         !! vertical center for Gamma-shape
      REAL*8       FACT( 5 )    !! factor (unused)


      ! following is used to set another force
      REAL*8       V1QQF, V2QQF, VQQF
      REAL*8       HAMP2         !! horizontal amplitude
      REAL*8       XDIL2         !! x-dilation for Elliptic-shape
      REAL*8       YDIL2         !! y-dilation for Elliptic-shape
      REAL*8       XCNT2         !! x-center for Elliptic-shape
      REAL*8       YCNT2         !! y-center for Elliptic-shape
      REAL*8       VAMP2         !! vertical amplitude
      REAL*8       VDIL2         !! dilation for Gamma-shape
      REAL*8       VCNT2         !! vertical center for Gamma-shape
      REAL*8       AZ2( KMAX )


      NAMELIST    /NMFIN/   CFM, CFG, FACT
      NAMELIST    /NMVAR/   OVOR, ODIV, OTMP, OPS, OSPH
      NAMELIST    /NMHPR/   KHPR, HAMP, XDIL, YDIL, XCNT, YCNT
      NAMELIST    /NMVPR/   KVPR, VAMP, VDIL, VCNT
      NAMELIST    /NMALL/   OWALL
      NAMELIST    /NMCLS/   OCLASSIC
*
      DATA         NOUT / 6 /
      DATA         S2D / 86400.D0 /
      DATA         CHPR
     $           /'Elliptic       ','Zonally uniform'/
      DATA         CVPR
     $           /'Sinusoidal     ','Gamma          ',
     $            'Uniform        '/
*                  123456789012345   123456789012345   
      DATA    OVOR, ODIV, OTMP, OPS, OSPH
     &       / .FALSE., .FALSE., .FALSE., .FALSE., .FALSE. /
*
*     open the NAMELIST file
*
      REWIND( 1 )
      OPEN ( 0, FILE='IEEE_ERROR')
      OPEN ( 1, FILE = 'SETPAR', STATUS = 'OLD')
*
      READ( 1, NMFIN) 
      REWIND( 1 )
      READ( 1, NMVAR) 
      REWIND( 1 )
      READ( 1, NMHPR) 
      REWIND( 1 )
      READ( 1, NMVPR) 
      REWIND( 1 )
      READ( 1, NMALL) 
      REWIND( 1 )
      READ( 1, NMCLS) 
*
*
      WRITE( NOUT, * ) '### MAKE FORCING MATRIX ###'
*
      CALL SETCONS
      CALL SPSTUP               !! spherical harmonic functions
      CALL SETNMO2
     O     ( NMO   ,
     D     MMAX  , LMAX  , NMAX  , MINT    )

      WRITE( NOUT, * )
      WRITE( NOUT, * ) '................'
      WRITE( NOUT, * ) 'Selected shape:'
      WRITE( NOUT, * ) '  Horizontal:',CHPR( KHPR )
      WRITE( NOUT, * ) '  Vertical  :',CVPR( KVPR )
      WRITE( NOUT, * ) '................'
      WRITE( NOUT, * )
*
*     open files
*
      IFM = 10
      IFG = 20
      OPEN ( IFM, FILE = CFM, FORM = 'UNFORMATTED', STATUS = 'UNKNOWN' ) 
      OPEN ( IFG, FILE = CFG, FORM = 'UNFORMATTED', STATUS = 'UNKNOWN' )
      WRITE( NOUT, * ) 'Matrix file         :', CFM
      WRITE( NOUT, * ) 'Matrix file (GrADS) :', CFG
      WRITE( NOUT, * ) '................'
      WRITE( NOUT, * ) 
*
      PI = ATAN( 1.0D0 ) * 4.0D0
      CALL SETLON
     O         ( ALON  , DLON , HALON  )
      CALL SETLAT
     O         ( ALAT  , DLON , HALON  )
      CALL SETSIG
     O         ( SIG   , DSIG , HSIG  , 
     O           SIGM  , DSIGM, HSIGM  )
*
      DO 1 J = 1, JMAX
         DO 1 I = 1, IDIM
            ALON( I, J) = ALON( I, J) * 360.D0 / ( 2.D0 * PI )
            ALAT( I, J) = ALAT( I, J) * 360.D0 / ( 2.D0 * PI )
 1    CONTINUE
*
      CALL SETZ( GFRCT, IDIM*JMAX*KMAX )
      CALL SETZ( DUM, IMAX*JMAX )
      CALL SETZ( WFRCT, NMDIM*KMAX )
      CALL SETZ( WFRCF, NMDIM*KMAX )
*
*     vertical profile
*
      VAMP2 = 4.
      VDIL2 = 20.
      VCNT2 = 0.45
      IF( KMAX .GT. 1 ) THEN
         DO 10 K = 1, KMAX
            IF( KVPR .EQ. 1 ) AZ( K ) = VAMP * DSIN( PI * SIG( K ) )
            IF( KVPR .EQ. 2 ) AZ( K ) = VAMP * 
     $           DEXP( - VDIL * ( SIG( K ) - VCNT )**2 )
            IF( KVPR .EQ. 2 ) AZ2( K ) = VAMP2 * 
     $           DEXP( - VDIL2 * ( SIG( K ) - VCNT2 )**2 )
            IF( KVPR .EQ. 3 ) AZ( K ) = VAMP 
 10      CONTINUE
         WRITE( NOUT, * ) 'Set vertical shape'
         WRITE( NOUT, * ) '................'
         WRITE( NOUT, * ) 
      ELSE
         AZ( K ) = VAMP
      ENDIF
*
*     horizontal profile
*
      IJ = 0
      DO 20 J = 1, JMAX
         DO 30 I = 1, IDIM
            IJ = IJ + 1
*
            IF( KHPR .EQ. 1 ) THEN !! Elliptic
               IF( XDIL .LE. 0.D0 .OR. YDIL .LE. 0.D0 ) THEN
                  WRITE( NOUT, *) 
     $                 '### Parameter XDIL/YDIL not correct ###' 
                  STOP
               ENDIF
*
               V1 = ( ALON( I, J ) - XCNT )**2 / XDIL**2
               IF( ALON(I,J).LT.180. .AND. XCNT+XDIL.GE.360. )
     $              V1 = ( 360. + ALON( I, J ) - XCNT )**2 / XDIL**2
               IF( ALON(I,J).GT.180. .AND. XCNT-XDIL.LE.0. )
     $              V1 = ( ALON( I, J ) - 360. - XCNT )**2 / XDIL**2
               V2 = ( ALAT( I, J ) - YCNT )**2 / YDIL**2
               V = V1 + V2
               !IF( V .GT. 1. ) THEN
               !   AXY( I, J) = 0.D0
               !ELSE
               !   V = DSQRT( V ) 
               !   AXY( I, J) = HAMP * ( 1.D0 - V )
               !ENDIF

               ! used to set another force
               !      REAL*8       HAMP2         !! horizontal amplitude
               !      REAL*8       XDIL2         !! x-dilation for Elliptic-shape
               !      REAL*8       YDIL2         !! y-dilation for Elliptic-shape
               !      REAL*8       XCNT2         !! x-center for Elliptic-shape
               !      REAL*8       YCNT2         !! y-center for Elliptic-shape
               HAMP2 = 1.
               XDIL2 = 40.
               YDIL2 = 12.
               XCNT2 = 210.
               YCNT2 = 0.
               V1QQF = ( ALON( I, J ) - XCNT2 )**2 / XDIL2**2
               IF( ALON(I,J).LT.180. .AND. XCNT2+XDIL2.GE.360. )
     $             V1QQF = ( 360. + ALON( I, J ) - XCNT2 )**2 / XDIL2**2
               IF( ALON(I,J).GT.180. .AND. XCNT2-XDIL2.LE.0. )
     $             V1QQF = ( ALON( I, J ) - 360. - XCNT2 )**2 / XDIL2**2
               V2QQF = ( ALAT( I, J ) - YCNT2 )**2 / YDIL2**2
               VQQF = V1QQF + V2QQF

               IF( VQQF .GT. 1. .AND. V .GT. 1. ) THEN
                  AXY( I, J) = 0.D0
               ELSE
                  IF(VQQF .LE. 1.) THEN
                      VQQF = DSQRT( VQQF ) 
                      AXY( I, J) = HAMP2 * ( 1.D0 - VQQF )
                      KV = KMAX
                      IF( OPS  ) KV = 1 
                      DO 40 K = 1, KV
                          GFRCT( IJ, K) = AXY( I, J) * AZ2( K ) / S2D
 40                   CONTINUE
                  ENDIF
                  IF(V .LE. 1.) THEN
                      V = DSQRT( V )
                      AXY( I, J) = HAMP * ( 1.D0 - V )
                      KV = KMAX
                      IF( OPS  ) KV = 1 
                      DO 9999 K = 1, KV
                          GFRCT( IJ, K) = AXY( I, J) * AZ( K ) / S2D
 9999                 CONTINUE
                  ENDIF
               ENDIF

            ENDIF
               
            IF( KHPR .EQ. 2 ) THEN !! zonal uniform
*
               IF( ALAT( I, J) .GE. YCNT - YDIL .AND.
     $             ALAT( I, J) .LE. YCNT + YDIL ) THEN
                  V = HAMP
               ELSE
                  V = 0.D0
               ENDIF

               AXY( I, J) = V
            ENDIF

*            KV = KMAX
*            IF( OPS  ) KV = 1 
*            DO 40 K = 1, KV
*               GFRCT( IJ, K) = AXY( I, J) * AZ( K ) / S2D
* 40         CONTINUE
*
 30      CONTINUE
 20   CONTINUE
      WRITE( NOUT, * ) 'Set horizontal shape'
      WRITE( NOUT, * ) '................'
      WRITE( NOUT, * ) 
*
*     write GrADS file
*
      DO 100 K = 1, KMAX
         IF( OVOR ) THEN
            DO 110 J = 1, JMAX
               DO 110 I = 1, IMAX
                  IJ = (J-1)*IDIM + I
                  DUMX( I, J) = GFRCT( IJ, K)
 110        CONTINUE
            WRITE( IFG ) ((SNGL(DUMX(I,J)),I=1,IMAX),J=1,JMAX)
         ELSE
            WRITE( IFG ) ((SNGL(DUM(I,J)),I=1,IMAX),J=1,JMAX)
         ENDIF
 100  CONTINUE
*
      DO 120 K = 1, KMAX
         IF( ODIV ) THEN
            DO 130 J = 1, JMAX
               DO 130 I = 1, IMAX
                  IJ = (J-1)*IDIM + I
                  DUMX( I, J) = GFRCT( IJ, K)
 130        CONTINUE
            WRITE( IFG ) ((SNGL(DUMX(I,J)),I=1,IMAX),J=1,JMAX)
         ELSE
            WRITE( IFG ) ((SNGL(DUM(I,J)),I=1,IMAX),J=1,JMAX)
         ENDIF
 120  CONTINUE
*
      DO 140 K = 1, KMAX
         IF( OTMP ) THEN
            DO 150 J = 1, JMAX
               DO 150 I = 1, IMAX
                  IJ = (J-1)*IDIM + I
                  DUMX( I, J) = GFRCT( IJ, K)
 150        CONTINUE
            WRITE( IFG ) ((SNGL(DUMX(I,J)),I=1,IMAX),J=1,JMAX)
         ELSE
            WRITE( IFG ) ((SNGL(DUM(I,J)),I=1,IMAX),J=1,JMAX)
         ENDIF
 140  CONTINUE
*
      IF( OPS ) THEN
         DO 160 J = 1, JMAX
            DO 160 I = 1, IMAX
               IJ = (J-1)*IDIM + I
               DUMX( I, J) = GFRCT( IJ, 1)
 160     CONTINUE
         WRITE( IFG ) ((SNGL(DUMX(I,J)),I=1,IMAX),J=1,JMAX)
      ELSE
         WRITE( IFG ) ((SNGL(DUM(I,J)),I=1,IMAX),J=1,JMAX)
      ENDIF
*
      IF( .NOT. OCLASSIC ) THEN
         DO 170 K = 1, KMAX
            IF( OSPH ) THEN
               DO 180 J = 1, JMAX
                  DO 180 I = 1, IMAX
                     IJ = (J-1)*IDIM + I
                     DUMX( I, J) = GFRCT( IJ, K)
 180           CONTINUE
               WRITE( IFG ) ((SNGL(DUMX(I,J)),I=1,IMAX),J=1,JMAX)
            ELSE
               WRITE( IFG ) ((SNGL(DUM(I,J)),I=1,IMAX),J=1,JMAX)
            ENDIF
 170     CONTINUE
      ENDIF
*
      CLOSE( IFG )
      WRITE( NOUT, * ) 'Written to GrADS file'
      WRITE( NOUT, * ) '................'
      WRITE( NOUT, * ) 
*
*     grid to wave (forcing matrix)
*
      IF( OPS ) THEN
         CALL G2W
     O        ( WFRCT   ,
     I          GFRCT   , '    ', 'POSO', 1    )
      ELSE
         CALL G2W
     O        ( WFRCT   ,
     I          GFRCT   , '    ', 'POSO', KMAX )
      ENDIF
*
*     write down
*
      IW = 0
      DO 200 M = 0, NTR
         LEND = MIN( LMAX, NMAX-M)
         DO 220 K = 1, KMAX
            IW = 0
            DO 210 L = 0, LEND
               IF( M .EQ. 0 .AND. L .EQ. 0 ) GOTO 210
               I = NMO( 1, M, L)
               J = NMO( 2, M, L)
               IW = IW + 1
               WXVOR(IW,K,M)            = WFRCF(I,K)
               WXDIV(IW,K,M)            = WFRCF(I,K)
               WXTMP(IW,K,M)            = WFRCF(I,K)
               WXPS (IW,M  )            = WFRCF(I,1)
               WXSPH(IW,K,M)            = WFRCF(I,K)
               IF( OVOR ) WXVOR(IW,K,M) = WFRCT(I,K)
               IF( ODIV ) WXDIV(IW,K,M) = WFRCT(I,K)
               IF( OTMP ) WXTMP(IW,K,M) = WFRCT(I,K)
               IF(  OPS ) WXPS (IW,M  ) = WFRCT(I,1)
               IF( OSPH ) WXSPH(IW,K,M) = WFRCT(I,K)
               IF( M .EQ. 0 ) GOTO 210
               IW = IW + 1
               WXVOR(IW,K,M)            = WFRCF(J,K)
               WXDIV(IW,K,M)            = WFRCF(J,K)
               WXTMP(IW,K,M)            = WFRCF(J,K)
               WXPS (IW,M  )            = WFRCF(J,1)
               WXSPH(IW,K,M)            = WFRCF(J,K)
               IF( OVOR ) WXVOR(IW,K,M) = WFRCT(J,K)
               IF( ODIV ) WXDIV(IW,K,M) = WFRCT(J,K)
               IF( OTMP ) WXTMP(IW,K,M) = WFRCT(J,K)
               IF(  OPS ) WXPS (IW,M  ) = WFRCT(J,1)
               IF( OSPH ) WXSPH(IW,K,M) = WFRCT(J,K)
  210       CONTINUE
  220    CONTINUE
         IF( .NOT. OWALL ) THEN
            IF( OCLASSIC ) THEN
               WRITE( IFM ) 
     $              ((WXVOR(I,K,M),I=1,IW),K=1,KMAX),
     $              ((WXDIV(I,K,M),I=1,IW),K=1,KMAX),
     $              ((WXTMP(I,K,M),I=1,IW),K=1,KMAX),
     $              ( WXPS (I,M)  ,I=1,IW          )
            ELSE
               WRITE( IFM ) 
     $              ((WXVOR(I,K,M),I=1,IW),K=1,KMAX),
     $              ((WXDIV(I,K,M),I=1,IW),K=1,KMAX),
     $              ((WXTMP(I,K,M),I=1,IW),K=1,KMAX),
     $              ( WXPS (I,M)  ,I=1,IW          ),
     $              ((WXSPH(I,K,M),I=1,IW),K=1,KMAX)
            ENDIF
         ELSE
            JW( M ) = IW
         ENDIF
  200 CONTINUE
      IF( OWALL ) THEN
         IF( OCLASSIC ) THEN
            WRITE( IFM ) 
     $           (((WXVOR(I,K,M),I=1,JW(M)),K=1,KMAX),
     $            ((WXDIV(I,K,M),I=1,JW(M)),K=1,KMAX),
     $            ((WXTMP(I,K,M),I=1,JW(M)),K=1,KMAX),
     $            ( WXPS (I,M)  ,I=1,JW(M)          ),M=0,NTR)
         ELSE
            WRITE( IFM ) 
     $           (((WXVOR(I,K,M),I=1,JW(M)),K=1,KMAX),
     $            ((WXDIV(I,K,M),I=1,JW(M)),K=1,KMAX),
     $            ((WXTMP(I,K,M),I=1,JW(M)),K=1,KMAX),
     $            ( WXPS (I,M)  ,I=1,JW(M)          ),
     $            ((WXSPH(I,K,M),I=1,JW(M)),K=1,KMAX),M=0,NTR)
         ENDIF
      ENDIF
      CLOSE( IFM )
      IF( OWALL ) THEN
         WRITE( NOUT, * ) 'Written to matrix file (all)'
      ELSE
         WRITE( NOUT, * ) 'Written to matrix file (pwm)'
      ENDIF
      WRITE( NOUT, * ) '................'
      WRITE( NOUT, * ) 
*
      WRITE( NOUT, * )
      WRITE( NOUT, * ) '### END OF EXECUTION ###'
      WRITE( NOUT, * )
*
*
  999 STOP
      END
*######################
      SUBROUTINE SETZ( A, IA )
*
*
      INTEGER IA
      REAL*8  A( IA )

      REAL*8  ZERO
      DATA    ZERO / 0.D0 /
      INTEGER I
*
      DO 10 I = 1, IA
         A( I ) = ZERO
 10   CONTINUE

      RETURN
      END
*##########################
      SUBROUTINE SETNMO2    !! order of matrix
     O         ( NMO   ,
     D           MMAX  , LMAX  , NMAX  , MINT    )
*
*   [PARAM] 
      INTEGER    MMAX
      INTEGER    LMAX
      INTEGER    NMAX
      INTEGER    MINT
*
*   [OUTPUT]
      INTEGER    NMO   ( 2, 0:MMAX, 0:LMAX ) !! order of spect. suffix
*
*   [INTERNAL WORK] 
      INTEGER    L, M, MEND, NMH
*
      NMH  = 0
      DO 2200 L = 0, LMAX
         MEND = MIN( MMAX, NMAX-L )
         DO 2100 M = 0, MEND, MINT
            NMH = NMH + 1
            IF ( MMAX .EQ. 0 ) THEN
               NMO ( 1, M, L ) = NMH
               NMO ( 2, M, L ) = NMH
            ELSE
               NMO ( 1, M, L ) = 2* NMH - 1
               NMO ( 2, M, L ) = 2* NMH
            ENDIF
 2100    CONTINUE
 2200 CONTINUE
*
      RETURN
      END
```

下面这段代码，是如何同时设置不同的强迫，例如，同时设置涡度和温度强迫，两种强迫具有不同的结构

```fortran
      PROGRAM MKFRCNG
*
*     derived from out_agcm/donly/mktfrc.f on 1998/11/08
*     This routine makes simplified forcing function
*     for the 3-D linear response.
*
*     Alternative selection of the forcing shape is:
*      Horizontal: Elliptic function/zonally uniform function 
*      Vertical  : Sinusoidal function/Gamma function/uniform
*
      include 'dim.f'
*
*
      INTEGER      MAXN
      PARAMETER (  MAXN = 2*NMAX*(NVAR*KMAX+1) )
*
      REAL*8       GFRCT( IDIM*JMAX, KMAX)  
      REAL*8       WFRCT( NMDIM, KMAX)  
      REAL*8       WFRCF( NMDIM, KMAX)  
      REAL*8       WXVOR( MAXN, KMAX, 0:NTR)  
      REAL*8       WXDIV( MAXN, KMAX, 0:NTR)  
      REAL*8       WXTMP( MAXN, KMAX, 0:NTR)  
      REAL*8       WXPS ( MAXN, 0:NTR )
      REAL*8       WXSPH( MAXN, KMAX, 0:NTR)  
*
      REAL*8       ALON( IDIM, JMAX )
      REAL*8       ALAT( IDIM, JMAX )
      REAL*8       SIG( KMAX )
      REAL*8       DLON( IDIM, JMAX ) !! unused
      REAL*8       SIGM( KMAX+1 )     !! unused
      REAL*8       DSIG( KMAX )       !! unused
      REAL*8       DSIGM( KMAX+1 )    !! unused
      CHARACTER    HALON *(16)        !! unused
      CHARACTER    HSIG *(16)         !! unused
      CHARACTER    HSIGM *(16)        !! unused
*
*     [work]
      REAL*8       AZ( KMAX )
      REAL*8       AXY( IDIM, JMAX)
      REAL*8       DUM( IMAX, JMAX)
      REAL*8       DUMX( IMAX, JMAX)
      REAL*8       PI
      REAL*8       V1, V2, V
      INTEGER      NMO   ( 2, 0:MMAX, 0:LMAX ) !! order of spect. suffix
      INTEGER      IFM, IFG
      INTEGER      I, J, K, L, M, IJ, KV, IW, JW( 0:NTR ), LEND
      INTEGER      NOUT
      REAL*8       S2D
*
*     [intrinsic]
      INTRINSIC    DSIN, DSQRT, DBLE, SNGL
*
      CHARACTER    CHPR( MAXH )*15
      CHARACTER    CVPR( MAXV )*15
*
      LOGICAL      OVOR         !! set vorticity forcing ? 
      LOGICAL      ODIV         !! set divergence forcing ? 
      LOGICAL      OTMP         !! set tmperature forcing ? 
      LOGICAL      OPS          !! set sfc. pressure forcing ? 
      LOGICAL      OSPH         !! set humidity forcing ? 
      LOGICAL      OWALL        !! write all the wavenumber
      LOGICAL      OCLASSIC     !! classic dry model
      CHARACTER*90 CFM          !! output file name (matrix)
      CHARACTER*90 CFG          !! output file name (grads)
      INTEGER      KHPR         !! type of horizontal profile
      INTEGER      KVPR         !! type of vertical profile
      REAL*8       HAMP         !! horizontal amplitude
      REAL*8       VAMP         !! vertical amplitude
      REAL*8       XDIL         !! x-dilation for Elliptic-shape
      REAL*8       YDIL         !! y-dilation for Elliptic-shape
      REAL*8       VDIL         !! dilation for Gamma-shape
      REAL*8       XCNT         !! x-center for Elliptic-shape
      REAL*8       YCNT         !! y-center for Elliptic-shape
      REAL*8       VCNT         !! vertical center for Gamma-shape
      REAL*8       FACT( 5 )    !! factor (unused)


      ! following is used to set another force
      REAL*8       HAMP2         !! horizontal amplitude
      REAL*8       XDIL2         !! x-dilation for Elliptic-shape
      REAL*8       YDIL2         !! y-dilation for Elliptic-shape
      REAL*8       XCNT2         !! x-center for Elliptic-shape
      REAL*8       YCNT2         !! y-center for Elliptic-shape
      REAL*8       VAMP2         !! vertical amplitude
      REAL*8       VDIL2         !! dilation for Gamma-shape
      REAL*8       VCNT2         !! vertical center for Gamma-shape
      REAL*8       KVPR2
      REAL*8       KHPR2
      REAL*8       AZ2( KMAX )
      REAL*8       GFRCT2( IDIM*JMAX, KMAX)  



      NAMELIST    /NMFIN/   CFM, CFG, FACT
      NAMELIST    /NMVAR/   OVOR, ODIV, OTMP, OPS, OSPH
      NAMELIST    /NMHPR/   KHPR, HAMP, XDIL, YDIL, XCNT, YCNT
      NAMELIST    /NMVPR/   KVPR, VAMP, VDIL, VCNT
      NAMELIST    /NMALL/   OWALL
      NAMELIST    /NMCLS/   OCLASSIC
*
      DATA         NOUT / 6 /
      DATA         S2D / 86400.D0 /
      DATA         CHPR
     $           /'Elliptic       ','Zonally uniform'/
      DATA         CVPR
     $           /'Sinusoidal     ','Gamma          ',
     $            'Uniform        '/
*                  123456789012345   123456789012345   
      DATA    OVOR, ODIV, OTMP, OPS, OSPH
     &       / .FALSE., .FALSE., .FALSE., .FALSE., .FALSE. /
*
*     open the NAMELIST file
*
      REWIND( 1 )
      OPEN ( 0, FILE='IEEE_ERROR')
      OPEN ( 1, FILE = 'SETPAR', STATUS = 'OLD')
*
      READ( 1, NMFIN) 
      REWIND( 1 )
      READ( 1, NMVAR) 
      REWIND( 1 )
      READ( 1, NMHPR) 
      REWIND( 1 )
      READ( 1, NMVPR) 
      REWIND( 1 )
      READ( 1, NMALL) 
      REWIND( 1 )
      READ( 1, NMCLS) 
*
*
      WRITE( NOUT, * ) '### MAKE FORCING MATRIX ###'
*
      CALL SETCONS
      CALL SPSTUP               !! spherical harmonic functions
      CALL SETNMO2
     O     ( NMO   ,
     D     MMAX  , LMAX  , NMAX  , MINT    )

      WRITE( NOUT, * )
      WRITE( NOUT, * ) '................'
      WRITE( NOUT, * ) 'Selected shape:'
      WRITE( NOUT, * ) '  Horizontal:',CHPR( KHPR )
      WRITE( NOUT, * ) '  Vertical  :',CVPR( KVPR )
      WRITE( NOUT, * ) '................'
      WRITE( NOUT, * )
*
*     open files
*
      IFM = 10
      IFG = 20
      OPEN ( IFM, FILE = CFM, FORM = 'UNFORMATTED', STATUS = 'UNKNOWN' ) 
      OPEN ( IFG, FILE = CFG, FORM = 'UNFORMATTED', STATUS = 'UNKNOWN' )
      WRITE( NOUT, * ) 'Matrix file         :', CFM
      WRITE( NOUT, * ) 'Matrix file (GrADS) :', CFG
      WRITE( NOUT, * ) '................'
      WRITE( NOUT, * ) 
*
      PI = ATAN( 1.0D0 ) * 4.0D0
      CALL SETLON
     O         ( ALON  , DLON , HALON  )
      CALL SETLAT
     O         ( ALAT  , DLON , HALON  )
      CALL SETSIG
     O         ( SIG   , DSIG , HSIG  , 
     O           SIGM  , DSIGM, HSIGM  )
*
      DO 1 J = 1, JMAX
         DO 1 I = 1, IDIM
            ALON( I, J) = ALON( I, J) * 360.D0 / ( 2.D0 * PI )
            ALAT( I, J) = ALAT( I, J) * 360.D0 / ( 2.D0 * PI )
 1    CONTINUE
*
      CALL SETZ( GFRCT, IDIM*JMAX*KMAX )
      CALL SETZ( DUM, IMAX*JMAX )
      CALL SETZ( WFRCT, NMDIM*KMAX )
      CALL SETZ( WFRCF, NMDIM*KMAX )
*
*     vertical profile
*
      VAMP2 = 3.
      VDIL2 = 20.
      VCNT2 = 0.5
      KVPR2 = 2
      IF( KMAX .GT. 1 ) THEN
         DO 10 K = 1, KMAX
            IF( KVPR .EQ. 1 ) AZ( K ) = VAMP * DSIN( PI * SIG( K ) )

            IF( KVPR2 .EQ. 1 ) AZ2( K ) = VAMP2 * DSIN( PI * SIG( K ) )

            IF( KVPR .EQ. 2 ) AZ( K ) = VAMP * 
     $           DEXP( - VDIL * ( SIG( K ) - VCNT )**2 )

            IF( KVPR2 .EQ. 2 ) AZ2( K ) = VAMP2 * 
     $           DEXP( - VDIL2 * ( SIG( K ) - VCNT2 )**2 )

            IF( KVPR .EQ. 3 ) AZ( K ) = VAMP 

            IF( KVPR2 .EQ. 3 ) AZ2( K ) = VAMP2 
 10      CONTINUE
         WRITE( NOUT, * ) 'Set vertical shape'
         WRITE( NOUT, * ) '................'
         WRITE( NOUT, * ) 
      ELSE
         AZ( K ) = VAMP
      ENDIF




*
*     horizontal profile
*
      IJ = 0
      DO 20 J = 1, JMAX
         DO 30 I = 1, IDIM
            IJ = IJ + 1
*
            IF( KHPR .EQ. 1 ) THEN !! Elliptic
               IF( XDIL .LE. 0.D0 .OR. YDIL .LE. 0.D0 ) THEN
                  WRITE( NOUT, *) 
     $                 '### Parameter XDIL/YDIL not correct ###' 
                  STOP
               ENDIF
*
               V1 = ( ALON( I, J ) - XCNT )**2 / XDIL**2
               IF( ALON(I,J).LT.180. .AND. XCNT+XDIL.GE.360. )
     $              V1 = ( 360. + ALON( I, J ) - XCNT )**2 / XDIL**2
               IF( ALON(I,J).GT.180. .AND. XCNT-XDIL.LE.0. )
     $              V1 = ( ALON( I, J ) - 360. - XCNT )**2 / XDIL**2
               V2 = ( ALAT( I, J ) - YCNT )**2 / YDIL**2
               V = V1 + V2
               IF( V .GT. 1. ) THEN
                  AXY( I, J) = 0.D0
               ELSE
                  V = DSQRT( V ) 
                  AXY( I, J) = HAMP * ( 1.D0 - V )
               ENDIF
            ENDIF
               
            IF( KHPR .EQ. 2 ) THEN !! zonal uniform
*
               IF( ALAT( I, J) .GE. YCNT - YDIL .AND.
     $             ALAT( I, J) .LE. YCNT + YDIL ) THEN
                  V = HAMP
               ELSE
                  V = 0.D0
               ENDIF

               AXY( I, J) = V
            ENDIF

            KV = KMAX
            IF( OPS  ) KV = 1 
            DO 40 K = 1, KV
               GFRCT( IJ, K) = AXY( I, J) * AZ( K ) / S2D
 40         CONTINUE
*
 30      CONTINUE
 20   CONTINUE
      WRITE( NOUT, * ) 'Set horizontal shape'
      WRITE( NOUT, * ) '................'
      WRITE( NOUT, * ) 


               HAMP2 = -1.
               XDIL2 = 5.
               YDIL2 = 5.
               XCNT2 = 285.
               YCNT2 = 30.
               KHPR2 = 1
*
*     horizontal profile
*
      IJ = 0
      DO 201 J = 1, JMAX
         DO 301 I = 1, IDIM
            IJ = IJ + 1
*
            IF( KHPR2 .EQ. 1 ) THEN !! Elliptic
               IF( XDIL2 .LE. 0.D0 .OR. YDIL2 .LE. 0.D0 ) THEN
                  WRITE( NOUT, *) 
     $                 '### Parameter XDIL/YDIL not correct ###' 
                  STOP
               ENDIF
*
               V1 = ( ALON( I, J ) - XCNT2 )**2 / XDIL2**2
               IF( ALON(I,J).LT.180. .AND. XCNT2+XDIL2.GE.360. )
     $              V1 = ( 360. + ALON( I, J ) - XCNT2 )**2 / XDIL2**2
               IF( ALON(I,J).GT.180. .AND. XCNT2-XDIL2.LE.0. )
     $              V1 = ( ALON( I, J ) - 360. - XCNT2 )**2 / XDIL2**2
               V2 = ( ALAT( I, J ) - YCNT2 )**2 / YDIL2**2
               V = V1 + V2
               IF( V .GT. 1. ) THEN
                  AXY( I, J) = 0.D0
               ELSE
                  V = DSQRT( V ) 
                  AXY( I, J) = HAMP2 * ( 1.D0 - V )
               ENDIF
            ENDIF
               
            IF( KHPR2 .EQ. 2 ) THEN !! zonal uniform
*
               IF( ALAT( I, J) .GE. YCNT2 - YDIL2 .AND.
     $             ALAT( I, J) .LE. YCNT2 + YDIL2 ) THEN
                  V = HAMP2
               ELSE
                  V = 0.D0
               ENDIF

               AXY( I, J) = V
            ENDIF

            KV = KMAX
            IF( OPS  ) KV = 1 
            DO 401 K = 1, KV
               GFRCT2( IJ, K) = AXY( I, J) * AZ2( K ) / S2D
 401         CONTINUE
*
 301      CONTINUE
 201   CONTINUE
      WRITE( NOUT, * ) 'Set horizontal shape'
      WRITE( NOUT, * ) '................'
      WRITE( NOUT, * ) 








*
*     write GrADS file
*
      DO 100 K = 1, KMAX
         IF( OVOR ) THEN
            DO 110 J = 1, JMAX
               DO 110 I = 1, IMAX
                  IJ = (J-1)*IDIM + I
                  DUMX( I, J) = GFRCT( IJ, K)
 110        CONTINUE
            WRITE( IFG ) ((SNGL(DUMX(I,J)),I=1,IMAX),J=1,JMAX)
         ELSE
            WRITE( IFG ) ((SNGL(DUM(I,J)),I=1,IMAX),J=1,JMAX)
         ENDIF
 100  CONTINUE
*
      DO 120 K = 1, KMAX
         IF( ODIV ) THEN
            DO 130 J = 1, JMAX
               DO 130 I = 1, IMAX
                  IJ = (J-1)*IDIM + I
                  DUMX( I, J) = GFRCT( IJ, K)
 130        CONTINUE
            WRITE( IFG ) ((SNGL(DUMX(I,J)),I=1,IMAX),J=1,JMAX)
         ELSE
            WRITE( IFG ) ((SNGL(DUM(I,J)),I=1,IMAX),J=1,JMAX)
         ENDIF
 120  CONTINUE
*
      DO 140 K = 1, KMAX
         IF( OTMP ) THEN
            DO 150 J = 1, JMAX
               DO 150 I = 1, IMAX
                  IJ = (J-1)*IDIM + I
                  DUMX( I, J) = GFRCT2( IJ, K)
 150        CONTINUE
            WRITE( IFG ) ((SNGL(DUMX(I,J)),I=1,IMAX),J=1,JMAX)
         ELSE
            WRITE( IFG ) ((SNGL(DUM(I,J)),I=1,IMAX),J=1,JMAX)
         ENDIF
 140  CONTINUE
*
      IF( OPS ) THEN
         DO 160 J = 1, JMAX
            DO 160 I = 1, IMAX
               IJ = (J-1)*IDIM + I
               DUMX( I, J) = GFRCT( IJ, 1)
 160     CONTINUE
         WRITE( IFG ) ((SNGL(DUMX(I,J)),I=1,IMAX),J=1,JMAX)
      ELSE
         WRITE( IFG ) ((SNGL(DUM(I,J)),I=1,IMAX),J=1,JMAX)
      ENDIF
*
      IF( .NOT. OCLASSIC ) THEN
         DO 170 K = 1, KMAX
            IF( OSPH ) THEN
               DO 180 J = 1, JMAX
                  DO 180 I = 1, IMAX
                     IJ = (J-1)*IDIM + I
                     DUMX( I, J) = GFRCT( IJ, K)
 180           CONTINUE
               WRITE( IFG ) ((SNGL(DUMX(I,J)),I=1,IMAX),J=1,JMAX)
            ELSE
               WRITE( IFG ) ((SNGL(DUM(I,J)),I=1,IMAX),J=1,JMAX)
            ENDIF
 170     CONTINUE
      ENDIF
*
      CLOSE( IFG )
      WRITE( NOUT, * ) 'Written to GrADS file'
      WRITE( NOUT, * ) '................'
      WRITE( NOUT, * ) 
*
*     grid to wave (forcing matrix)
*
      IF( OPS ) THEN
         CALL G2W
     O        ( WFRCT   ,
     I          GFRCT   , '    ', 'POSO', 1    )
      ELSE
         CALL G2W
     O        ( WFRCT   ,
     I          GFRCT   , '    ', 'POSO', KMAX )
      ENDIF
*
*     write down
*
      IW = 0
      DO 200 M = 0, NTR
         LEND = MIN( LMAX, NMAX-M)
         DO 220 K = 1, KMAX
            IW = 0
            DO 210 L = 0, LEND
               IF( M .EQ. 0 .AND. L .EQ. 0 ) GOTO 210
               I = NMO( 1, M, L)
               J = NMO( 2, M, L)
               IW = IW + 1
               WXVOR(IW,K,M)            = WFRCF(I,K)
               WXDIV(IW,K,M)            = WFRCF(I,K)
               WXTMP(IW,K,M)            = WFRCF(I,K)
               WXPS (IW,M  )            = WFRCF(I,1)
               WXSPH(IW,K,M)            = WFRCF(I,K)
               IF( OVOR ) WXVOR(IW,K,M) = WFRCT(I,K)
               IF( ODIV ) WXDIV(IW,K,M) = WFRCT(I,K)
               IF( OTMP ) WXTMP(IW,K,M) = WFRCT(I,K)
               IF(  OPS ) WXPS (IW,M  ) = WFRCT(I,1)
               IF( OSPH ) WXSPH(IW,K,M) = WFRCT(I,K)
               IF( M .EQ. 0 ) GOTO 210
               IW = IW + 1
               WXVOR(IW,K,M)            = WFRCF(J,K)
               WXDIV(IW,K,M)            = WFRCF(J,K)
               WXTMP(IW,K,M)            = WFRCF(J,K)
               WXPS (IW,M  )            = WFRCF(J,1)
               WXSPH(IW,K,M)            = WFRCF(J,K)
               IF( OVOR ) WXVOR(IW,K,M) = WFRCT(J,K)
               IF( ODIV ) WXDIV(IW,K,M) = WFRCT(J,K)
               IF( OTMP ) WXTMP(IW,K,M) = WFRCT(J,K)
               IF(  OPS ) WXPS (IW,M  ) = WFRCT(J,1)
               IF( OSPH ) WXSPH(IW,K,M) = WFRCT(J,K)
  210       CONTINUE
  220    CONTINUE
         IF( .NOT. OWALL ) THEN
            IF( OCLASSIC ) THEN
               WRITE( IFM ) 
     $              ((WXVOR(I,K,M),I=1,IW),K=1,KMAX),
     $              ((WXDIV(I,K,M),I=1,IW),K=1,KMAX),
     $              ((WXTMP(I,K,M),I=1,IW),K=1,KMAX),
     $              ( WXPS (I,M)  ,I=1,IW          )
            ELSE
               WRITE( IFM ) 
     $              ((WXVOR(I,K,M),I=1,IW),K=1,KMAX),
     $              ((WXDIV(I,K,M),I=1,IW),K=1,KMAX),
     $              ((WXTMP(I,K,M),I=1,IW),K=1,KMAX),
     $              ( WXPS (I,M)  ,I=1,IW          ),
     $              ((WXSPH(I,K,M),I=1,IW),K=1,KMAX)
            ENDIF
         ELSE
            JW( M ) = IW
         ENDIF
  200 CONTINUE
      IF( OWALL ) THEN
         IF( OCLASSIC ) THEN
            WRITE( IFM ) 
     $           (((WXVOR(I,K,M),I=1,JW(M)),K=1,KMAX),
     $            ((WXDIV(I,K,M),I=1,JW(M)),K=1,KMAX),
     $            ((WXTMP(I,K,M),I=1,JW(M)),K=1,KMAX),
     $            ( WXPS (I,M)  ,I=1,JW(M)          ),M=0,NTR)
         ELSE
            WRITE( IFM ) 
     $           (((WXVOR(I,K,M),I=1,JW(M)),K=1,KMAX),
     $            ((WXDIV(I,K,M),I=1,JW(M)),K=1,KMAX),
     $            ((WXTMP(I,K,M),I=1,JW(M)),K=1,KMAX),
     $            ( WXPS (I,M)  ,I=1,JW(M)          ),
     $            ((WXSPH(I,K,M),I=1,JW(M)),K=1,KMAX),M=0,NTR)
         ENDIF
      ENDIF
      CLOSE( IFM )
      IF( OWALL ) THEN
         WRITE( NOUT, * ) 'Written to matrix file (all)'
      ELSE
         WRITE( NOUT, * ) 'Written to matrix file (pwm)'
      ENDIF
      WRITE( NOUT, * ) '................'
      WRITE( NOUT, * ) 
*
      WRITE( NOUT, * )
      WRITE( NOUT, * ) '### END OF EXECUTION ###'
      WRITE( NOUT, * )
*
*
  999 STOP
      END
*######################
      SUBROUTINE SETZ( A, IA )
*
*
      INTEGER IA
      REAL*8  A( IA )

      REAL*8  ZERO
      DATA    ZERO / 0.D0 /
      INTEGER I
*
      DO 10 I = 1, IA
         A( I ) = ZERO
 10   CONTINUE

      RETURN
      END
*##########################
      SUBROUTINE SETNMO2    !! order of matrix
     O         ( NMO   ,
     D           MMAX  , LMAX  , NMAX  , MINT    )
*
*   [PARAM] 
      INTEGER    MMAX
      INTEGER    LMAX
      INTEGER    NMAX
      INTEGER    MINT
*
*   [OUTPUT]
      INTEGER    NMO   ( 2, 0:MMAX, 0:LMAX ) !! order of spect. suffix
*
*   [INTERNAL WORK] 
      INTEGER    L, M, MEND, NMH
*
      NMH  = 0
      DO 2200 L = 0, LMAX
         MEND = MIN( MMAX, NMAX-L )
         DO 2100 M = 0, MEND, MINT
            NMH = NMH + 1
            IF ( MMAX .EQ. 0 ) THEN
               NMO ( 1, M, L ) = NMH
               NMO ( 2, M, L ) = NMH
            ELSE
               NMO ( 1, M, L ) = 2* NMH - 1
               NMO ( 2, M, L ) = 2* NMH
            ENDIF
 2100    CONTINUE
 2200 CONTINUE
*
      RETURN
      END
```

然后修改下Makefile，把mymkfrcng加入编译的队伍就行了

```
include ../../Lmake.inc

include ../include/make.inc.$(ARC)

SDEC = dim.f

CDEC = hdim.f

BSGRD = bsgrd.o

REDST = redist.o

MKFRC = mkfrcng.o

MYMKFRC = mymkfrcng.o

MKFRCS = mkfrcsst.o

MKFRCB = mkfrcbr.o

FVEC  = fvec.o

GTGR  = gt2gr.o

NCPBS = ncepsbs.o

NCPBBS = ncep1vbs.o

ECMBS = ecmsbs.o

all: dec hdec bsgrd redist mkfrcng mkfrcsst fvec gt2gr ncepsbs ncep1vbs ecmsbs mymkfrcng

bs: dec hdec bsgrd ncepsbs ncep1vbs ecmsbs

br: dec hdec mkfrcbr gt2gr ncep1vbs

dec: $(SDEC) ; \
     $(CP) ../include/dim_$(HRES)$(VRES)$(ZWTRN).f $(SDEC) ; \
     sed -e 's/BYTE/'$(MBYT)'/g' $(SDEC) > dimtmp.f ; \
     $(MV) dimtmp.f $(SDEC) 

hdec: $(CDEC) ; \
     $(CP) ../include/hdim_$(HRES).f $(CDEC) ; \
     sed -e 's/BYTE/'$(MBYT)'/g' $(CDEC) > hdimtmp.f ; \
     $(MV) hdimtmp.f $(CDEC) 

bsgrd: $(BSGRD) ; \
     $(LOADER) $(LOADOPTS) -o $@ \
     $(BSGRD) $(LAPACKLIB) $(BLASLIB) $(LDLIBS)

redist: $(REDST) ; \
     $(LOADER) $(LOADOPTS) -o $@ \
     $(REDST) $(LAPACKLIB) $(BLASLIB) $(LDLIBS)

mkfrcng: $(MKFRC) ; \
     $(LOADER) $(LOADOPTS) -o $@ \
     $(MKFRC) $(LAPACKLIB) $(BLASLIB) $(LDLIBS)

mymkfrcng: $(MYMKFRC) ; \
     $(LOADER) $(LOADOPTS) -o $@ \
     $(MYMKFRC) $(LAPACKLIB) $(BLASLIB) $(LDLIBS)

mkfrcsst: $(MKFRCS) ; \
     $(LOADER) $(LOADOPTS) -o $@ \
     $(MKFRCS) $(LAPACKLIB) $(BLASLIB) $(LDLIBS)

mkfrcbr: $(MKFRCB) ; \
     $(LOADER) $(LOADOPTS) -o $@ \
     $(MKFRCB) $(LAPACKLIB) $(BLASLIB) $(LDLIBS)

fvec:	 $(FVEC) ; \
     $(LOADER) $(LOADOPTS) -o $@ \
     $(FVEC) $(LAPACKLIB) $(BLASLIB) $(LDLIBS)

gt2gr:	 $(GTGR) ; \
     $(LOADER) $(LOADOPTS) -o $@ \
     $(GTGR) $(LAPACKLIB) $(BLASLIB) $(LDLIBS)

ncepsbs: $(NCPBS) ; \
     $(LOADER) $(LOADOPTS) -o $@ \
     $(NCPBS) $(LAPACKLIB) $(BLASLIB) $(LDLIBS)

ncep1vbs: $(NCPBBS) ; \
     $(LOADER) $(LOADOPTS) -o $@ \
     $(NCPBBS) $(LAPACKLIB) $(BLASLIB) $(LDLIBS)

ecmsbs: $(ECMBS) ; \
     $(LOADER) $(LOADOPTS) -o $@ \
     $(ECMBS) $(LAPACKLIB) $(BLASLIB) $(LDLIBS)

FRC:
	@FRC=$(FRC)

clean:
	rm -f *.o *~ dec hdec bsgrd redist mkfrcng mkfrcsst fvec gt2gr ncepsbs ncep1vbs ecmsbs mymkfrcng

.f.o : ; $(FORTRAN) $(OPTS) -c $<
```



# 备忘

最好使用intel编译器。利用gfortran其实也可以编译。但是这里会产生一个问题，就是在编译`$LNHOME/model/src/sysdep/yN000.F`时，会报“idate”，“itime”之类的函数没找到。这些函数其实是fortran的内置函数，但是在yN000.F里被声明为EXTERNAL，只需要把源程序中的EXTERNAL改为INTRINSIC就能够编译通过

intel编译器的big endian编译选项为-convert big_endian，而GNU的为-convert=big-endian，编译时注意别写错了
