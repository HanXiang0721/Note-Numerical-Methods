# 环境变量配置

```shell
# Linear Baroclinic Model
export LNHOME=/var/opt/ln_solver
```

# 制作Residual Forcing文件R

背景场制作方式与LBM相同。



打开`/var/opt/ln_solver/Lmake.inc `，修改下面这几行

```shell
#  include file for Makefile for linear solver
#
# set environment LNHOME	in your .cshrc
#
###### Architecture          ################
ARC  = N000
#ARC  = sun
#ARC  = alpha
#ARC  = sgi
#ARC  = sr8000

###### Model type            ################

### time-advance linear model (incl. storm track model)
#PROJECT          = tintgr

### standard, making linear matrix (incl. stationary wave model)
#PROJECT         = mkamat

### accerelated iterative solver (AIM)
#PROJECT         = aim

### nonlinear, dynamical core
PROJECT         = dcore

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
#MODELOPT = -DOPT_CLASSIC
######## moist model
#MODELOPT = 
#MODELOPT = -DOPT_POSDEF
#MODELOPT = -DOPT_OUTPOSDEF

### lbm/swm
######## dry model
#MODELOPT = -DOPT_MKMAT -DOPT_CLASSIC
#MODELOPT = -DOPT_MKMAT -DOPT_OWALL -DOPT_CLASSIC
#MODELOPT = -DOPT_WVFRC -DOPT_ORHS -DOPT_CLASSIC
######## moist model
#MODELOPT = -DOPT_MKMAT 
#MODELOPT = -DOPT_MKMAT -DOPT_OWALL
#MODELOPT = -DOPT_WVFRC -DOPT_ORHS
#
# GRDDMP: linear damping in grid space not in wave space
#MODELOPT = -DOPT_MKMAT -DOPT_OWALL -DOPT_GRDDMP
#
# DYNZM: use zonal mean BS for L but not for Lc
#MODELOPT = -DOPT_MKMAT -DOPT_OWALL -DOPT_DYNZM
#MODELOPT = -DOPT_WVFRC -DOPT_ORHS
# as of 04/04/08
#MODELOPT = -DOPT_WVFRC -DOPT_ORHS -DOPT_OWALL -DOPT_ADDCPL
#
######## dry AIM
#MODELOPT = -DOPT_CLASSIC

### mLBM-CZ
# diag: binary 1
##MODELOPT = -DOPT_MKMAT -DOPT_DYNZM
# diag: binary 2
#MODELOPT = -DOPT_WVFRC -DOPT_ORHS -DOPT_ADDCPL
# diag: binary 3
#MODELOPT = -DOPT_WVFRC -DOPT_ORHS -DOPT_NOMOIST -DOPT_DYNSW

### nonlinear dynamical core
#MODELOPT = 
MODELOPT = -DOPT_RWRIT
#MODELOPT = -DOPT_PBL
#MODELOPT = -DOPT_NOVDF

### barotropic model
#MODELOPT = -DOPT_MKMAT
#MODELOPT = -DOPT_MKMAT -DOPT_OWALL
#MODELOPT = -DOPT_WVFRC 

### mLBM-CZ
#MODELOPT = -DOPT_ORGCZ
#MODELOPT = 

### orographic forcing
#MODELOPT = -DOPT_CLASSIC
```

然后：

```shell
cd $LNHOME/model/src
make clean
make clean.special
make lbm
```

这样就可以开始计算R了，计算R使用的脚本如下：

```shell
#!/bin/csh -f
#
#      sample script for nonlinear model run (dry model)
#
# NQS command for mail
#@$-q short
#
setenv LNHOME   /home/qqf/Documents/Non-Linear-Dynamical-Model    # ROOT of model
setenv LBMDIR   $LNHOME/model                          # ROOT of LBM 
setenv SYSTEM   N000                                   # execute system
setenv RUN      $LBMDIR/bin/$SYSTEM/lbm2.t42ml20cdcore # Excutable file
setenv FDIR     $LNHOME/qqf/frc                       # Directory for Output
setenv DIR      $LNHOME/qqf/Gtools                       # Directory for Output
#setenv INITFILE $LNHOME/bs/gt3/ncepsum.t21l20          # Atm. BS File
setenv INITFILE $LNHOME/qqf/bs/qqf.t42l20        # Atm. initial
setenv RSTFILE  $DIR/Restart.amat                      # Restart-Data File
setenv DATZ     $LNHOME/bs/gt3/grz.t42                 # topography
setenv RFRC     $FDIR/frc-r.nonlin.grd                 # residual forcing
setenv SFRC                # steady forcing
setenv TEND     31
#
#
if (! -e $DIR) mkdir -p $DIR
cd $DIR
#\rm -f SYSOUT
echo job started at `date` > $DIR/SYSOUT
/bin/rm -f $DIR/SYSIN
#
#      parameters
#
cat << END_OF_DATA >>! $DIR/SYSIN

 &nmrun  run='nonlinear model'                              &end
 &nmtime start=0,1,1,0,0,0, end=0,1,$TEND,0,0,0             &end
 &nmdelt delt=40, tunit='MIN', inistp=2                     &end
 &nmhdif order=4, tefold=24, tunit='HOUR'                   &end
 &nminit file='$INITFILE' , DTBFR=0., DTAFTR=0., TUNIT='DAY' &end
 &nmrstr file='$RSTFILE', tintv=1, tunit='MON',  overwt=t   &end

 &nmanm  oanm=f                                              &end
 &nmrfrc rsfrc='$RFRC'                                       &end
 &nmsfrc fsfrc='$SFRC'                                       &end

 &nmchck ocheck=f, ockall=f                                  &end
 &nmdata item='GRZ',    file='$DATZ'                         &end

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
( $RUN < $DIR/SYSIN >> $DIR/SYSOUT ) >& $DIR/ERROUT
echo job end at `date` >> $DIR/SYSOUT

```

运行完成后可以得到R，R的路径由环境变量RFRC决定

计算R时候，SFRC环境变量设置为空值

# 运行试验

打开`/var/opt/ln_solver/Lmake.inc `，修改下面这几行

```shell
#  include file for Makefile for linear solver
#
# set environment LNHOME	in your .cshrc
#
###### Architecture          ################
ARC  = N000
#ARC  = sun
#ARC  = alpha
#ARC  = sgi
#ARC  = sr8000

###### Model type            ################

### time-advance linear model (incl. storm track model)
#PROJECT          = tintgr

### standard, making linear matrix (incl. stationary wave model)
#PROJECT         = mkamat

### accerelated iterative solver (AIM)
#PROJECT         = aim

### nonlinear, dynamical core
PROJECT         = dcore

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

### nonlinear dynamical core
MODELOPT = 
#MODELOPT = -DOPT_RWRIT
#MODELOPT = -DOPT_PBL
#MODELOPT = -DOPT_NOVDF

```

主要是需要重新编译一次，和计算R时候的区别是编译选项变了

control试验就直接运行

然后，敏感性试验运行脚本需要提前做好steady forcing场，做法和LBM相同

```shell
#!/bin/csh -f
#
#      sample script for nonlinear model run (dry model)
#
# NQS command for mail
#@$-q short
#
setenv LNHOME   /home/qqf/Documents/Non-Linear-Dynamical-Model    # ROOT of model
setenv LBMDIR   $LNHOME/model                          # ROOT of LBM 
setenv SYSTEM   N000                                   # execute system
setenv RUN      $LBMDIR/bin/$SYSTEM/lbm2.t42ml20cdcore # Excutable file
setenv FDIR     $LNHOME/qqf/frc                       # Directory for Output
setenv DIR      $LNHOME/qqf/Gtools                       # Directory for Output
#setenv INITFILE $LNHOME/bs/gt3/ncepsum.t21l20          # Atm. BS File
setenv INITFILE $LNHOME/qqf/bs/qqf.t42l20        # Atm. initial
setenv RSTFILE  $DIR/Restart.amat                      # Restart-Data File
setenv DATZ     $LNHOME/bs/gt3/grz.t42                 # topography
setenv RFRC     $FDIR/frc-r.nonlin.grd                 # residual forcing
setenv SFRC     $FDIR/frc.t42l20.CNP.grd           # steady forcing
setenv TEND     31
#
#
if (! -e $DIR) mkdir -p $DIR
cd $DIR
#\rm -f SYSOUT
echo job started at `date` > $DIR/SYSOUT
/bin/rm -f $DIR/SYSIN
#
#      parameters
#
cat << END_OF_DATA >>! $DIR/SYSIN

 &nmrun  run='nonlinear model'                              &end
 &nmtime start=0,1,1,0,0,0, end=0,1,$TEND,0,0,0             &end
 &nmdelt delt=40, tunit='MIN', inistp=2                     &end
 &nmhdif order=4, tefold=24, tunit='HOUR'                   &end
 &nminit file='$INITFILE' , DTBFR=0., DTAFTR=0., TUNIT='DAY' &end
 &nmrstr file='$RSTFILE', tintv=1, tunit='MON',  overwt=t   &end

 &nmanm  oanm=f                                              &end
 &nmrfrc rsfrc='$RFRC'                                       &end
 &nmsfrc fsfrc='$SFRC'                                       &end

 &nmchck ocheck=f, ockall=f                                  &end
 &nmdata item='GRZ',    file='$DATZ'                         &end

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
( $RUN < $DIR/SYSIN >> $DIR/SYSOUT ) >& $DIR/ERROUT
echo job end at `date` >> $DIR/SYSOUT
```

这次运行需要配置好SFRC环境变量，指向之前做好的forcing场

然后运行就ok了

最后的结果用EXP-control即可