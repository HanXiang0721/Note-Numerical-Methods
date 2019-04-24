# 环境变量配置

```shell
# Linear Baroclinic Model
export LNHOME=/var/opt/ln_solver
```

# 制作Residual Forcing文件

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

