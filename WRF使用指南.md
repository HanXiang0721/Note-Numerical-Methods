# WRF使用指南

> WRF(Weather Research and Forecast Model)是一种使用及其广泛的中尺度数值模式，在很多领域里WRF都扮演着及其重要的角色。WRF主要包含三部分，前处理WPS，模式WRF和同化系统WRFDA。

## WRF编译安装

### 准备

预先要安装好netCDF，Jasper，libpng，zlib，并行库，编译器等等工具。

### 环境变量

```sh
ulimit  -s unlimited
ulimit  -m unlimited
ulimit  -d unlimited
ulimit  -v unlimited
ulimit  -t unlimited
ulimit  -l unlimited
set +o noclobber
export ALL_LIBS=/home/qf/soft
export PATH=${ALL_LIBS}/bin:$PATH
export LD_LIBRARY_PATH=${ALL_LIBS}/lib:$LD_LIBRARY_PATH
export INCLUDE=${ALL_LIBS}/include:$INCLUDE

export CC=icc
export FC=ifort
export SCC=icc
export CXX=icpc
export F77=ifort
export F90=ifort
export MPICC=mpicc
export MPICXX=mpicxx
export MPIF90=mpif90
export MPIF77=mpif77
export CPP='icc -E'
export CXXCPP='icpc -E'                                     
export CFLAGS='-O3 -xHost -ip -fPIC'
export CXXFLAGS='-O3 -xHost -ip -fPIC'
export FCFLAGS='-O3 -xHost -ip -fPIC'

export WRFIO_NCD_LARGE_FILE_SUPPORT=1
export WRF_DIR=/home2_hn/qf/wrfda/WRFV3
export WRFDA_DIR=/home2_hn/qf/wrfda/WRFDA
export WRFPLUS_DIR=/home2_hn/qf/wrfda/WRFPLUSV3
export WRF_HYDRO=0
export WRF_CHEM=1
export CRTM=1
export BUFR=1
export DAT_DIR=/home/qianqf/Model/wrfda_test_data
export WORK_DIR=/home/qianqf/VARWORK

export PNETCDF_PATH=${ALL_LIBS}
export LD_LIBRARY_PATH=${PNETCDF_PATH}/lib:$LD_LIBRARY_PATH
export PATH=${PNETCDF_PATH}/bin:$PATH
export INCLUDE=${PNETCDF_PATH}/include:$INCLUDE

export NETCDF=${ALL_LIBS}
export NETCDF_PATH=${NETCDF}
export NETCDF_LIB=${NETCDF}/lib
export NETCDF_INC=${NETCDF}/include
export LIB_NETCDF=${NETCDF}/lib
export INC_NETCDF=${NETCDF}/include
export LD_LIBRARY_PATH=${NETCDF}/lib:$LD_LIBRARY_PATH       
export PATH=${NETCDF}/bin:$PATH
export INCLUDE=${NETCDF}/include:$INCLUDE

export JASPERLIB=${ALL_LIBS}/lib
export JASPERINC=${ALL_LIBS}/include
```

### WRF编译安装

解压安装包，然后运行：

`./configure`

注意跳出来的文字，其中这几样比较重要：

serial: single processor

smpar: Symmetric Multi-Processing/Shared Memory Parallel (OpenMP)

dmpar: Distributed Memory Parallel (MPI)

dm+sm: Distributed Memory with Shared Memory (for example, MPI across nodes with OpenMP within a node)

然后就可以提交后台开始编译了：

`nohup ./compile -j 20 em_real >& log &`

这是编译real case的WRF。如果编译成功，应该可以在main目录下看到ndown.exe，real.exe，wrf.exe。如果编译失败，则可以先`./clean -a`(也可以不做这步，灵活处理)，找到原因后重复上述步骤。

Ideal case的编译类似，但是要选择非并行无嵌套。

### WPS编译安装

解压安装包，然后运行：

`./configure`

跳出来的信息和WRF类似。如果需要使用Grib2格式的资料，注意预先安装好Jasper。然后提交后台编译：

`nohup ./compile >& log &`

编译成功后可以在根目录下看到：geogrid.exe, ungrib.exe, metgrid.exe；同时在util目录下看到：avg_tsfc.exe, calc_ecmwf_p.exe, g1print.exe, g2print.exe, mod_levs.exe, plotfmt.exe, plotgrids.exe, rd_intermediate.exe。plotfmt.exe和plotgrids.exe编译不出来没关系，不影响使用。这一般是由于NCL用的预编译版本，WPS编译时调用出现问题导致的。如果编译失败，则可以先`./clean -a`(也可以不做这步，灵活处理)，找到原因后重复上述步骤。

### WRFDA编译安装

由于WRFDA支持四维变分，所以这里记录四维变分的安装办法。

先解压WRFPLUS，然后执行：

```sh
./configure wrfplus
./compile wrf >& compile.out
ls -ls main/*.exe
```

编译成功的话应该能够看到wrf.exe。

解压WRFDA并进入，执行：

```sh
./configure 4dvar
./compile all_wrfvar >& compile.out
ls var/build/*.exe var/obsproc/*.exe | wc -l
```

最终应输出44，表明有44个exe文件。

## WRF运行

### WPS

可以利用[WRFDomainWizard](http://www.esrl.noaa.gov/gsd/wrfportal/DomainWizard.html)选定模拟范围，垂直层数等等，修改namelist.wps，然后执行：

```sh
./geogrid.exe
ln -s ./ungrib/Variable_Tables/Vtable.GFS Vtable
./link_grib.csh /path_to_reanalysis_data
./ungrib.exe
./metgrid.exe
```

### WRF

修改namelist.input，然后把WPS生成的met_em.d*文件连接到run目录下，执行：

`mpirun -np 4 ./real.exe`

生成wrfbdy_d01，wrfinput_d*文件，然后执行：

`mpirun -np 64 ./real.exe`

### WRFDA

WRFDA的执行流程较多，笔者做循环同化不得不写脚本进行。

### Ideal Case

Ideal Case的运行方式和前面Real Case类似，但是配置不同。Ideal Case主要做的是理论研究。

## 其他

### WRF并行编译的两种方式

WRF并行编译其实有两种方式，一种是利用-j参数；

另一种则是在configure.wrf里增加J=10这样一行，同时去掉CFLAGS里-ip选项

### WRF模式中eta层的设置以及分别对应的高度

在namelist.input中，有个高度的分层。在一般的研究中，我们常以海拔高度，位势高度作为分层的标准。但是在模式中，由于每个地区的下垫面都不一样，有的平原有的高原，有的湖泊有的高山，即使在一个很小的区域，地面的海拔高度也不一样，所以按照位势高度来分层，就显得十分不合适。于是就有了eta坐标。

eta=(P-Ptop)/(Pbot-Ptop)

P代表某一层的气压，也就是你需要研究的那一层，Ptop代表模式顶层气压，Pbot代表地面气压。（WRF中好像是10hpa，不太确定）。从表达式可以看出，地面的eta值就是1，顶层就是0。这样一来，无论某个地区的下垫面的海拔高度是多少，它的eta坐标值都统一成1。当然，在模式的结果分析中，我们更想知道的是某个eta坐标对应的位势高度，比如说我们要研究离地60m高度的风速，就有必要进行二者转换。常见的转换式是:

gmp=(PH+PHB)/9.81-HGT

结果就是离地位势高度值，PH和PHB以及HGT都是模式结果中已有的变量。HGT是当地地面海拔高度。
在3.2的版本中已经有了height这个变量，可以直接对应海拔高度。经计算，发现其值和上式计算结果几乎一致，注意单位换算。
如果需要更精确的结果，则

gmp=(((PH(k)+PH(k+1)) / 2) + ((PHB(k)+(PHB(k+1)) / 2) / 9.81 – HGT

这个动力论坛上对于上面两个表达式的对比回答：

Firstly, the result of (PH+PHB)/9.8 is the height above sea level that is not what you need to check. For horizontal winds, some prognostic variables, and diagnostic variables, the center of the cell should be used. The center height above surface, absolute altitude should be calculated by running average from the top and bottom:

(((PH(k)+PH(k+1)) / 2) + ((PHB(k)+(PHB(k+1)) / 2) / 9.81 – HGT,

where PH and PHB are staggered in the vertical direction, meaning that they are at the tops and bottoms of grid cells. HGT is terrain height. This center height of grid cell is changing from model time to time based on definition of eta vertical coordinate system. You never get it fixed. So you cannot get the answer to your question by re-setting eta levels.

Secondly, for scalar properties such as temperature, tracer concentration, the modeled result spatially represents at the center of grid cell and shows a homogeneous value within entire grid box. That means the modeled value is constant from bottom to top of the grid cell. What you think about is the sub-scaled. You may use Monin-Obukov similarity theory to derive the modeled value at the height of 4.4m at which your monitor is set. For doing this you may copy the way to calculate those diagnostic variables like T2, Q2 in the model. (But U10 and V10 are different). Due to limitation of M-O theory and model as well as observation technology I am not sure you can get your results much improved even at such a small different height of 0.3 m (4.7m – 4.4m). Normally we can directly compare the modeled result and observed one obtained at surface layer. However it highly depends on what kind of parameter you are dealing with and what accuracy requirements for this measurement are. I do not know what observation you made.

### WRF模式restart的问题

WRF模式restart之后，累积性的变量会产生一些问题。如果namelist里的变量override_restart_timers = .true.则总体归结起来就是，一个wrfout文件最后的时间点减去第一个时间点就是变量累积的量，不同wrfout文件计时起点不同，不能混淆。如果为false，则与连续run没有区别。

### WRF模式用不含闰年的资料驱动

WRF模式也可以使用不含闰年的资料驱动，方法如下：
修改namelist.wrf文件中ARCH_LOCAL一行，添加-DNO_LEAP_CALENDAR，然后用./clean清除所有已经编译好的文件，重新编译即可。如果没有clean，则会出现ESMF模块不能启动的报错信息。此外有闰年与没闰年的模拟存在较大差异，需要特别注意

### WRF模式中提取海温

wrflowinput是加入海温之后的产生的。在ungrib这步，分两部分，一部分生成FILE,另外 ln -sf ungrib/Variable_Tables/Vtable.SST Vtable。在namelist.wps里：

```text
&ungrib
out_format = 'WPS',
prefix ='SST',
```

将FILE 改为SST,这时候会生成SST文件.

ungrib这步就生成了 FILE 和 SST 两种文件,在met这步

```text
&metgrid<br/>
fg_name = 'FILE','SST'
io_form_metgrid = 2,
```

### 风电场与光伏电场铺设

使用libreoffice中的excel，将地图插入到背景界面，使用网格打点，然后确定网格位置，在模式中实现。注意在铺设风电场时，由于windspec.in文件是风机的一些参数，所以可以使用Excel编辑在粘贴过去，可采用python写入excel。

### WRF中增加spectral/grid nudging和一些诊断量

WRF目前有三种逼近，一种是格点逼近，另一种是谱逼近，另一种是观测逼近。

```text
spectral nudging
&fdda
grid_fdda                           = 2,
gfdda_inname                        = "wrffdda_d<domain>",
gfdda_interval_m                    = 360,
gfdda_end_h                         = 8784,
io_form_gfdda                       = 2,
fgdt                                = 0,
if_no_pbl_nudging_uv                = 1,
if_no_pbl_nudging_t                 = 1,
if_no_pbl_nudging_q                 = 1,
if_no_pbl_nudging_ph                = 1,
if_zfac_uv                          = 1,
k_zfac_uv                           = 10,
if_zfac_t                           = 1,
k_zfac_t                            = 10,
if_zfac_q                           = 1,
k_zfac_q                            = 10,
if_zfac_ph                          = 1,
k_zfac_ph                           = 10,
guv                                 = 0.0003,
gt                                  = 0.0,
gq                                  = 0.0,
gph                                 = 0.0003,
xwavenum                            = 3,
ywavenum                            = 3,
if_ramping                          = 0,
dtramp_min                          = 60.0
/
grid nudging
&fdda
grid_fdda                           = 1
gfdda_inname                        = "wrffdda_d<domain>"
gfdda_interval_m                    = 360
gfdda_end_h                         = 888888
io_form_gfdda                       = 2
fgdt                                = 0
if_no_pbl_nudging_uv                = 1
if_no_pbl_nudging_t                 = 1
if_no_pbl_nudging_q                 = 1
if_zfac_uv                          = 1
k_zfac_uv                           = 30
if_zfac_t                           = 1
k_zfac_t                            = 30
if_zfac_q                           = 1  
k_zfac_q                            = 30
if_zfac_ph                          = 1,
k_zfac_ph                           = 30,
guv                                 = 0.0003
gt                                  = 0.0003
gq                                  = 0.0003
gph                                 = 0.0003,
if_ramping                          = 0
dtramp_min                          = 60.0
/
!增加一些诊断量
&time_control
io_form_auxinput4                   = 2
auxinput4_inname                    = 'wrflowinp_d<domain>'
auxinput4_interval                  = 360, 360, 360,
output_diagnostics                  = 1        
auxhist3_outname                    = 'wrfxtrm_d<domain>_<date>'
io_form_auxhist3                    = 2,
auxhist3_interval                   = 360,360
frames_per_auxhist3                 = 500000
auxhist23_outname                   = 'wrfpress_d<domain>_<date>'
io_form_auxhist23                   = 2,
auxhist23_interval                  = 360,360
frames_per_auxhist23                = 500000,
/
&diags
p_lev_diags                         = 1,     
num_press_levels                    = 31,     
press_levels                        = 100000,97500,95000,90000,87500,85000,77500,       
use_tot_or_hyd_p                    = 1       
/
```

### WRF模式变量输出设置

> WRF输出多少变量有两种方式：一种是参考WRF文件夹下的README.io_config，可以把Registry里的变量输出到wrfout文件中；另一种是逐层传递变量，最终在Registry里增加一些变量。

#### 修改io_config

通过修改io_config，可以增加输出一些Registry里已经声明好的变量。修改方法如下：

1.在namelist.input里修改iofields_filename这条参数，比如令其为"myfile"；

2.生成一个myfile，里面的格式如：op:streamtype:streamid:variables；

op: +(增加)，-(移除)

streamtype: i(输入资料)，h(历史输出)

streamid: 一个整数，0代表主输出，即wrfout

variables: 用逗号分隔的变量列表，变量名具体见Registry

例如：-:h:0:SEAICE，在wrfout里不输出SEAICE这个变量；+:i:5:u，在5号输出文件里输出u变量

#### 修改Registry

内部变量主要通过 Registry来控制（WRF/Registry ）。与real run相对应的是 Registry.EM。

该文件每一行分为 9列：

Type   Sym   Dims   Use   Tlev   Stag   IO   Dname   Descrip 

每个变量对应一列，比如： 

state   real   u   ikjb   dyn_em   2   X   i01rhusdf   "U"   "X WIND COMPONENT" 

这一行就对应 WRF中所用的U变量。 

控制输出的在于 IO一栏（i01rhusdf ），i:输入。r:重启 (restart)输出。如果在IO栏中有 "r"，则该变量就会输出到wrfrst\*文件中，以便于未来 restart run(hot start)时作为初始条件使用。h：输出，即输出到 history文件中。usdf：控制 nesting选项。 


如果删除 h, 则U变量将不会输出到 history文件中（wrfout* ），也不会有任何的辅助输出。

如果改为 h1, 则U 变量将不会输出到 history主文件中（wrfout* ）, 但会输出到辅助输出文件 1中。

如果改为 h01，则U 变量既会输出到 history主文件中（wrfout* ），同时也会输出到辅助输出文件 1中。

h之后可以跟 0(主输出) ，也可以跟 "1,2,3,4,5,6,7,8,9",对应1 到9号辅助输出文件。以下写法，都是合法的：h, h01, h012, h347。


在Registry中修改rainc和rainnc输出为 h02后，要在run/namelist.input 中修改添加以下变量：

auxhist2_outname = "rainfall_d<domain\>_<date\>"--辅助文件2 的命名

auxhist2_interval = 60--变量输出的间隔，以分钟为单位 

io_form_auxhist2 = 2--辅助文件 2的输出格式，2对应netcdf 

frame_per_auxhist2 = 1--每个文件中包含的输出数 

如果想要修改输出频率，可以在这里更改 auxhist2_interval注意： 

如果没有添加这些变量在namelist.input中，辅助输出文件（rainfall*）还是不能生成。

当然，在上面这些基础上，要把目标变量从WRF每个模块里传出来，一直传到动力框架，WRF的grid变量里。