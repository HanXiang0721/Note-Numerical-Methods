# CESM使用指南

> [CESM](http://www.cesm.ucar.edu/index.html)(Community Earth System Model)是一个由美国国家大气研究中心(NCAR)开发的气候系统模式，目前得到广泛的应用。本文是使用笔者一开始学习CESM时的一份记录，很多操作时间一长会忘记，所以这里就备份这份笔记。


## CESM移植安装与运行

### 预准备

首先查看目前CESM已经有的版本：

`svn list https://svn-ccsm-release.cgd.ucar.edu/model_versions`

然后下载需要的版本：

`svn co https://svn-ccsm-release.cgd.ucar.edu/model_versions/cesm1_2_1 cesm1_2_1`

其他软件，如编译器，并行库，netCDF等需要提前安装好，笔者用的是intel编译器套件Parallel Studio，同时并行库一般用的OpenMPI或者IMPI，此外需要提前下载好CESM的输入数据。

然后利用命令：

```sh
cd ccsm4/scripts
./create_newcase -l
```

列出模式支持的系统信息。

### 设置环境变量(CEMS与CAM5)

```sh
#For CESM
export CAM_ROOT=/home/qianqf/cesm1_2_1
export camcfg=/home/qianqf/cesm1_2_1/models/atm/cam/bld
export INC_NETCDF=/home/qianqf/lib4cam/include
export LIB_NETCDF=/home/qianqf/lib4cam/lib
export INC_MPI=/share/apps/openmpi-1.4.5/include
export LIB_MPI=/share/apps/openmpi-1.4.5/lib
export CSMDATA=/home2_hn/CESM/cesm-input
export MOD_NETCDF=/home/qianqf/lib4cam/include
```

### 编辑config_machines.xml文件

```text
<machine MACH="qqf">
　　　<DESC>LINUX cluster</DESC>
　　　<OS>LINUX</OS>
　　　<COMPILERS>intel</COMPILERS>
　　　<MPILIBS>openmpi</MPILIBS>
　　　<RUNDIR>/home2_hn/qianqf/$CASE/run</RUNDIR>
　　　<EXEROOT>/home2_hn/qianqf/$CASE/bld</EXEROOT>
　　　<DIN_LOC_ROOT>/home2_hn/CESM/cesm-input</DIN_LOC_ROOT>
　　　<DIN_LOC_ROOT_CLMFORC>/home2_hn/CESM/cesm-input/atm/datm7</DIN_LOC_ROOT_CLMFORC>
　　　<DOUT_S>FALSE</DOUT_S>
　　　<DOUT_S_ROOT>/home2_hn/qianqf/$CASE</DOUT_S_ROOT>
　　　<DOUT_L_MSROOT>/home2_hn/qianqf/$CASE</DOUT_L_MSROOT>
　　　<CCSM_BASELINE>UNSET</CCSM_BASELINE>
　　　<CCSM_CPRNC>UNSET</CCSM_CPRNC>
　　　<BATCHQUERY>qstat</BATCHQUERY>
　　　<BATCHSUBMIT>qsub</BATCHSUBMIT>
　　　<SUPPORTED_BY>qqf1403321992 -at- gmail.com</SUPPORTED_BY>
　　　<GMAKE_J>1</GMAKE_J>
　　　<MAX_TASKS_PER_NODE>16</MAX_TASKS_PER_NODE>
</machine>
```

### 配置env_mach_specific文件

> 由于config_machines.xml里设置了MACH="qqf"，所以env_mach_specific后缀为qqf，即env_mach_specific.qqf。但这个文件实际并不会起什么太大的作用。

```sh
#! /bin/csh -f
# -------------------------------------------------------------------------
# USERDEFINED
# Edit this file to add module load or other paths needed for the build
# and run on the system.  Can also include general env settings for machine.
# Some samples are below
# -------------------------------------------------------------------------
#source /opt/modules/default/init/csh
#if ( $COMPILER == "pgi" ) then
#  module load pgi
#endif
#module load netcdf
#limit coredumpsize unlimited
```

### 配置mkbatch文件

> 和env_mach_specific文件一样，根据config_machines.xml里设置了MACH="qqf"，所以后缀也是qqf，即mkbatch.qqf

```sh
#! /bin/csh -f
set mach = qqf
#################################################################################
if ($PHASE == set_batch) then
#################################################################################
source ./Tools/ccsm_getenv || exit -1
`set ntasks  = `${UTILROOT}/Tools/taskmaker.pl -sumonly` `
`set maxthrds = `${UTILROOT}/Tools/taskmaker.pl -maxthrds` `
nodes = $ntasks / ${MAX_TASKS_PER_NODE}
if ( $ntasks % ${MAX_TASKS_PER_NODE} > 0) then
　　@ nodes = $nodes + 1
　　@ ntasks = $nodes * ${MAX_TASKS_PER_NODE}
endif
@ taskpernode = ${MAX_TASKS_PER_NODE} / ${maxthrds}
set qname = batch
set tlimit = "00:59:00"
#--- Job name is first fifteen characters of case name ---
`set jobname = `echo ${CASE} | cut -c1-15` `
cat >! $CASEROOT/${CASE}.${mach}.run << EOF1
#PBS -l nodes=${nodes}:ppn=${taskpernode}
#!/bin/csh -f
#===============================================================================
# GENERIC_USER
# This is where the batch submission is set.  The above code computes
# the total number of tasks, nodes, and other things that can be useful
# here.  Use PBS, BSUB, or whatever the local environment supports.
#===============================================================================
##PBS -N ${jobname}
##PBS -q ${qname}
##PBS -l nodes=${nodes}:ppn=${taskpernode}
##PBS -l walltime=${tlimit}
##PBS -r n
##PBS -j oe
##PBS -S /bin/csh -V
##BSUB -l nodes=${nodes}:ppn=${taskpernode}:walltime=${tlimit}
##BSUB -q ${qname}
###BSUB -k eo
###BSUB -J $CASE
###BSUB -W ${tlimit}
#limit coredumpsize 1000000
#limit stacksize unlimited
#BSUB -J ${CASE}
#BSUB -q hpc_linux
#BSUB -n ${maxtasks}
#BSUB -o output.%J
#BSUB -e error.%J
EOF1
#################################################################################
else if ($PHASE == set_exe) then
#################################################################################
`set maxthrds = `${UTILROOT}/Tools/taskmaker.pl -maxthrds` `
`set maxtasks = `${UTILROOT}/Tools/taskmaker.pl -sumtasks`
cat >> ${CASEROOT}/${CASE}.${MACH}.run << EOF1  
# -------------------------------------------------------------------------
# Run the model
# -------------------------------------------------------------------------
sleep 25
cd \$RUNDIR
`echo "\`date\` -- CSM EXECUTION BEGINS HERE" `
#===============================================================================
# GENERIC_USER
# Launch the job here.  Some samples are commented out below
#===============================================================================
setenv OMP_NUM_THREADS ${maxthrds}
if (\$USE_MPISERIAL == "FALSE") then
　　　#
　　　# Find the correct mpirun command and comment it out
　　　# Usually it will just be mpiexec or mpirun...
　　　# Remove the echo and exit below when you've done so.
　　　#
　　　#echo "GENERIC_USER: Put the correct mpirun command in your *.run script, then remove this echo/exit"
　　　#exit 2
　　　mpiexec -n ${maxtasks} ./ccsm.exe >&! ccsm.log.\$LID
　　　#mpirun -np ${maxtasks} ./ccsm.exe >&! ccsm.log.\$LID
　　　#./ccsm.exe
else
　　　./ccsm.exe >&! ccsm.log.\$LID
endif
wait
`echo "\`date\` -- CSM EXECUTION HAS FINISHED" `
EOF1
#################################################################################
else if ($PHASE == set_larch) then
#################################################################################
#This is a place holder for a long-term archiving script
#################################################################################
else
#################################################################################
　　　echo "mkscripts.$mach"
　　　echo "  PHASE setting of $PHASE is not an accepted value"
　　　echo "  accepted values are set_batch, set_exe and set_larch"
　　　exit 1
#################################################################################
endif
#################################################################################
```

### 新建CASE

执行下面这条命令：

`./create_newcase -case test -res f19_g16 -compset X -mach qqf`

`-case`：新建的CASE的名字

`-res`：分辨率

`-compset`：需要哪些分量模式组合，如何组合等设置，具体参考手册

`-mach`：机器名，和前面的配置一致

进入名为test的case文件夹，修改env_mach_specific，env_mach_pes.xml等文件，需要注意配置用多少个CPU。
依次执行cesm_setup，check_case，check_input_data这三个脚本。
执行后缀为build的脚本进行编译，然后执行后缀为submit的脚本进行提交：`${CASE}.build，${CASE}.submit`

## CAM单独编译运行

> 前面总结的是CESM模式的编译运行，但是理论上CESM的每个分量模式都可以单独编译运行。由于笔者主要用过CAM，这里记录CAM的单独编译运行流程。

### CAM3

> CAM3不同于目前的CAM5，CAM3是谱模式，同时单独提供一个文件下载。CAM3现在用的较少，而且不能使用太新的并行库，但是CAM3编译较为简单，这里一并记录。

#### 环境变量设置

先配置好环境变量：

```sh
#For CAM3
export CAM_ROOT=/home/qianqf/cam3/cam1
export INC_NETCDF=/share/apps/local/include
export LIB_NETCDF=/share/apps/local/lib
export INC_MPI=/share/apps/openmpi-1.4.5/include
export LIB_MPI=/share/apps/openmpi-1.4.5/lib
export USER_FC=ifort
export USER_CC=icc
export CSMDATA=/home/qianqf/cam3`
```

#### 配置

把下面的这些内容写入一个configure.sh文件执行configure配置CAM3：

```sh
#!/bin/sh
$CAM_ROOT/models/atm/cam/bld/configure -cam_bld /home/qianqf/CAM3/bld -cam_exedir /home/qianqf/CAM3 -spmd -test -debug
cd ./bld
gmake
cd ..
$CAM_ROOT/models/atm/cam/bld/build-namelist -config /home/qianqf/CAM3/bld/config_cache.xml -test
```

#### 配置namelist

```text
&camexp
　absems_data            = '/home/qianqf/cam3/atm/cam/rad/abs_ems_factors_fastvx.c030508.nc'
　aeroptics              = '/home/qianqf/cam3/atm/cam/rad/AerosolOptics_c040105.nc'
　bnd_topo               = '/home/qianqf/cam3/atm/cam/topo/topo-from-cami_0000-09-01_64x128_L26_c030918.nc'
　bndtvaer               = '/home/qianqf/cam3/atm/cam/rad/AerosolMass_V_64x128_clim_c031022.nc'
　bndtvo         = '/home/qianqf/cam3/atm/cam/ozone/pcmdio3.r8.64x1_L60_clim_c970515.nc'
　bndtvs         = '/home/qianqf/sst_64X128x_201202modify.nc'
　caseid         = 'camrun2013avg'
　iyear_ad               = 1950
　ncdata         = '/home/qianqf/cam3/atm/cam/inic/gaus/cami_0000-09-01_64x128_L26_c030918.nc'
　start_ymd      = 20000101
　nelapse                = -3650
　nsrest         = 0
　scon           = 1.367E6
　co2vmr         = 3.550e-4
/
&clmexp
　finidat                = '/home/qianqf/cam3/lnd/clm2/inidata_2.1/cam/clmi_0000-09-01_64x128_T42_USGS_c030609.nc'
　fpftcon                = '/home/qianqf/cam3/lnd/clm2/pftdata/pft-physiology'
　fsurdat                = '/home/qianqf/cam3/lnd/clm2/srfdata/cam/clms_64x128_USGS_c030605.nc'
/
```

#### 运行

`./cam < namelist`

### CAM5
> CAM5使用的一般流程如下：运行config.sh-->进入bld文件夹下，运行gmake -j2 或者 gmake-->运行namelist.sh-->修改各项参数-->运行CAM5。CAM5的环境变量可以与CESM设置在一起，因为CAM5和CAM3不同，不再单独提供一个包给用户使用。

#### config.sh配置

```sh
#!/bin/sh
#$camcfg/configure --help
#$camcfg/configure -dyn fv -hgrid 10x15 -nosmp -nospmd -test -ice sice -ocn socn -lnd slnd -rof srof -cam_bld /home/qianqf/camtest/fuck -cc icc -debug -fc ifort -fc_type intel -target_os linux
#$camcfg/configure -dyn fv -hgrid 4x5 -nosmp -spmd -ntasks 16 -test -ice cice -ocn docn -lnd clm -rof rtm -cam_bld /home3_hn/qianqf/CAM/bld -cc mpicc -debug -fc mpif90 -fc_type intel -target_os linux
$camcfg/configure -dyn fv -hgrid 0.9x1.25 -carma none -nosmp -spmd -ntasks 160 -test -cam_bld /home/qianqf/CAM5/try/bld -cc mpicc -debug -fc mpif90 -fc_type intel -target_os linux`
```

#### namelist.sh设置namelist

```sh
#!/bin/sh
$camcfg/build-namelist -test -config /home/wjh/CAM5/try/bld/config_cache.xml -ntasks 160
```

#### 运行

`./cam < namelist`

### 修改海冰与海温强迫场和制作初始场
海温和海冰的数据都放在不同的文件里，ice_in文件里面的stream_domfilename和stream_fldfilename两个设置分别指向强迫场的数据。文件docn.stream.txt中filepath和filename两个参数也需要修改成对应强迫场的文件位置。
初始场的制作，需要先在atm_in中ncdata参数的下一行增加inithist='6-HOURLY'，和inithist_all=True，然后运行一段时间即可。