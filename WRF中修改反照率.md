# WRF中修改反照率

wrf本身的 albedo是根据 Vtable定的，所以是不停变化的。首先要将albedo改为月的 albedo，方法是在 namelist.input 中的 &physics 中加入 usemonalb = .true.命令，然后就是利用ncl修改即可。在模式中单纯修改反照率，如果面积过大，就会造成地面平衡破坏太严重，模式易于崩溃。

由于模式会在计算过程中产生降雪，导致反射率发生变化。因此，我们需要将雪从模式中除去。方法就是每过一段时间重启一次，然后对wrfrst文件中的雪进行修改。

修改反射率与修改雪的范例代码如下所示

```text
f1	= addfile("wrfinput_d01","w") 
f2	= addfile("wrflowinp_d01","w")
fs		= systemfunc("ls -t wrfrst*")
f3		= addfile(fs(0),"w")

albedo1	= f1->ALBBCK
albedo2	= f2->ALBBCK
albedo3	= f3->ALBBCK
albedo3a	= f3->ALBEDO
albedo3b	= f3->SNOALB
snow1	= f3->ACSNOW
snow2	= f3->SNOW
snow3	= f3->SNOWH
snow4	= f3->SNOWC

albedo1(:,100:101,62:63)	=	0.305
albedo2(:,100:101,62:63)	=	0.305

f1->ALBBCK	= albedo1
f2->ALBBCK	= albedo2

albedo3(:,100:101,62:63)	=	0.305
albedo3a(:,100:101,62:63)	=	0.305
albedo3b(:,100:101,62:63)	=	0.305

snow1(:,100:101,62:63)	=	0
snow2(:,100:101,62:63)	=	0
snow3(:,100:101,62:63)	=	0
snow4(:,100:101,62:63)	=	0

f3->ALBBCK	= albedo3
f3->ALBDO	= albedo3a
f3->SNOALB	= albedo3b
f3->ACSNOW	= snow1
f3->SNOW	= snow2
f3->SNOWH	= snow3
f3->SNOWC	= snow4
```