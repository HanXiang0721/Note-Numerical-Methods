# WRF耦合CESM中的物理参数化方案

> WRF中有很多从CAM中移植来的参数化方案，其中典型的就是积云对流的ZhangMc方案。代码中有很多预编译选项，细述如下

为了方便CAM中得代码移植，WRF中包含了:

```text
module_cam_physconst.F
module_cam_shr_const_mod.F
module_cam_shr_kind_mod.F
module_cam_support.F
module_cam_trb_mtn_stress.F
module_cam_upper_bc.F
module_cam_wv_saturation.F等模块。
```

所以有些CAM源代码中的变量，函数等可以在这些模块中找到。

一般在代码最开头会有`#define WRF_PORT`和`#define MODAL_AERO`

之后的很多定义以及计算根据变量，函数能否在上述的模块中找到进行预编译的书写。

例如，pcols, pver, pverp这三个量可以在module_cam_support.F里找到，因此预编译选项就如下书写：

```c
#ifndef WRF_PORT
  use sped_utils,   only: masterproc
  use purged,        only: pcols, pver, pverp
#else
  use module_cam_support, only: masterproc, pcols, pver, pverp
#endif
```

整段代码最开头的`#define WRF_PORT`表明这是移植到WRF中的代码，所以后面的if就跟着这个走，上述的if预编译应该执行的是else后面的语句，因为我们在预编译时定义了WRF_PORT。

WRF_PORT这个量的意义在WRF代码中有说明：

WRF_PORT currently uses hard-wired values for the namelist input. The values could easily be setup to come from the Registry in the future. The hard-wired values are the defaults for the fv core. They should be verified by somebody knowledgable on the matter.

其实就是有些时候要读取CAM的namelist，而通过WRF_PORT可以直接把这些参数写在代码里而不是通过namelist。

由于CAM是全球模式，且主要研究目的是气候，所以很多方案在移植时需要调整。例如积云的ZhangMc方案中的对流调整时间，在CAM中是固定的。太大的对流调整时间说明对流强度较弱，较小的对流调整时间说明对流强度较强。在移植到WRF中时，对流调整时间将根据网格格距变化而变化。