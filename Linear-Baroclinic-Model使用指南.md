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

