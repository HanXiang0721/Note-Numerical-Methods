# SPEEDY模式使用指南

SPEEDY是意大利理论物理中心研发的模式，这个模式是个T30L5的全耦合模式。最大的优势在于移植方便，模拟速度快

SPEEDY和CESM一样，是按照CASE进行编译运行

下面是一个运行海温强迫的脚本

```shell
#!/bin/sh



# model setups

# nmonths: run months

# icland: coupling flag for land surface temp, 0 is no, 1 is land model

# icsea: coupleing flag for sea surface temp, 0 is prescribe

# icice: coupleing flag for sea ice temp, 0 is no, 1 is ice model

# isstan: SST anomaly flag, 0 is no, only clim SST, 1 is oberved anomaly

nmonths=1200

ICLAND=1

ICSEA=0

ICICE=1

ISSTAN=1



# $res = resolution (eg t21, t30)

# $expno = experiment no. (eg 111)

# $exprsno = experiment no. for restart file ( 0 = no restart ) 



res='t30'

expno='000'

exprsno=0



# SST anomaly file

sstfile="..\/myfrc\/P2\/P2.grd"

sed -i "16s/^.*.$/cp $sstfile fort.30/g" ../ver41.5.input/inpfiles.s



# modify namelist



sed -i "4s/^.*.$/      NMONTS = $nmonths/g" ../ver41.5.input/cls_instep.h 

sed -i "22s/^.*.$/      ICLAND = $ICLAND/g" ../ver41.5.input/cls_instep.h 

sed -i "23s/^.*.$/      ICSEA  = $ICSEA/g" ../ver41.5.input/cls_instep.h 

sed -i "24s/^.*.$/      ICICE  = $ICICE/g" ../ver41.5.input/cls_instep.h 

sed -i "25s/^.*.$/      ISSTAN = $ISSTAN/g" ../ver41.5.input/cls_instep.h 





# Define directory names

# set -x



UT=..	

SA=$UT/source

CA=$UT/tmp

mkdir $UT/output/exp_$expno

CB=$UT/output/exp_$expno

CC=$UT/ver41.5.input

CD=$UT/output/exp_$exprsno	



mkdir $UT/input/exp_$expno



echo "model version   :   41"  > $UT/input/exp_$expno/run_setup

echo "hor. resolution : " $res  >> $UT/input/exp_$expno/run_setup

echo "experiment no.  : " $expno  >> $UT/input/exp_$expno/run_setup

echo "restart exp. no.: " $exprsno  >> $UT/input/exp_$expno/run_setup

	

# Copy files from basic version directory



echo "copying from $SA/source to $CA"

rm -f $CA/*



cp $SA/makefile $CA/

cp $SA/*.f      $CA/

cp $SA/*.h      $CA/

cp $SA/*.s      $CA/



cp $CA/par_horres_$res.h   $CA/atparam.h

cp $CA/par_verres.h      $CA/atparam1.h 



# Copy parameter and namelist files from user's .input directory



echo "ver41.input new files ..."

ls $UT/ver41.5.input



echo "copying parameter and namelist files from $UT/ver41.input "

cp $UT/ver41.5.input/cls_*.h     $CA/

cp $UT/ver41.5.input/inpfiles.s  $CA/

cp $UT/ver41.5.input/cls_*.h     $UT/input/exp_$expno

cp $UT/ver41.5.input/inpfiles.s  $UT/input/exp_$expno



# Copy modified model files from user's update directory



echo "update new files ..."

ls $UT/update



echo "copying modified model files from $UT/update"

cp $UT/update/*.f   $CA/

cp $UT/update/*.h   $CA/

cp $UT/update/make* $CA/	

cp $UT/update/*.f   $UT/input/exp_$expno

cp $UT/update/*.h   $UT/input/exp_$expno

cp $UT/update/make* $UT/input/exp_$expno

# Set input files

cd $CA

# Set experiment no. and restart file (if needed)
echo $exprsno >  fort.2
echo $expno >> fort.2

if [ $exprsno != 0 ] ; then

  echo "link restart file atgcm$exprsno.rst to fort.3"

  ln -s $CD/atgcm$exprsno.rst fort.3

fi 
# Link input files
echo 'link input files to fortran units'
sh inpfiles.s $res
ls -l fort.*
echo ' compiling at_gcm - calling make'
make imp.exe  
#
# create and execute a batch job to run the model
#
cat > run.job << EOF1
set -x
cd $CA
pwd
echo 'the executable file...'
ls -l imp.exe
time ./imp.exe > out.lis
mv out.lis $CB/atgcm$expno.lis
mv fort.10 $CB/atgcm$expno.rst
mv at*$expno.ctl   $CB
mv at*$expno_*.grd $CB
mv day*$expno.ctl   $CB
mv day*$expno_*.grd $CB
cd $CB
chmod 644 at*$expno.* 
EOF1

nohup sh run.job &
```

而海温强迫场文件的制作非常简单，具体脚本如下：

```ncl
ff = addfile("anomaly.nc","r")



latin = ff->latitude(::-1)

lonin = lonFlip(ff->longitude)

sst = lonFlip(ff->anomaly(::-1,:))

lonin(180:) = lonin(180:)+360.0



ntout = 1932 ;1759



latout = (/-87.16,  -83.47,  -79.78,  -76.07,  -72.36,  -68.65,  -64.94, -61.23,  -57.52,  -53.81,  -50.10,  -46.39,  -42.68,  -38.97,  -35.26,  -31.54,  -27.83,  -24.12,  -20.41,  -16.70,  -12.99,   -9.28,   -5.57,   -1.86, 1.86,5.57,    9.28,   12.99,   16.70,   20.41,   24.12,   27.83,   31.54,   35.26, 38.97,   42.68,   46.39,   50.10,   53.81,   57.52,   61.23,   64.94,   68.65, 72.36,   76.07,   79.78,   83.47,   87.16/)



lonout = fspan(0.0,356.25,96)



nlat = dimsizes(latout)

nlon = dimsizes(lonout)



sstout = linint2_Wrap(lonin,latin,sst,True,lonout,latout,1)

delete(sstout@_FillValue)

delete(sstout@missing_value)

printVarSummary(sstout)



do jj = 0, nlat-1

do kk = 0, nlon-1

if(sstout(jj,kk).lt.-10.0) then

sstout(jj,kk) = 0

end if

end do

end do





anom1 = sstout

anom2 = sstout



idxLat1 = ind(latout.le.40.0.or.latout.ge.50.0)

idxLon1 = ind(lonout.le.200.0.or.lonout.ge.215.0)



idxLat2 = ind(latout.le.30.0.or.latout.ge.70.0)

idxLon2 = ind(lonout.le.280.0.or.lonout.ge.350.0)



anom1(idxLat1,:)=0

anom1(:,idxLon1)=0

anom2(idxLat2,:)=0

anom2(:,idxLon2)=0





pacific = anom1

atlantic = anom2

both = anom1+anom2





ofile1 = "pacific.grd"

ofile2 = "atlantic.grd"

ofile3 = "both.grd"



setfileoption("bin","WriteByteOrder","BigEndian")



do ii = 0, ntout-1

ssto = pacific(::-1,:)

fbinrecwrite (ofile1, -1, ssto)

end do

delete(ssto)



do ii = 0, ntout-1

ssto = atlantic(::-1,:)

fbinrecwrite (ofile2, -1, ssto)

end do

delete(ssto)



do ii = 0, ntout-1

ssto = both(::-1,:)

fbinrecwrite (ofile3, -1, ssto)

end do

delete(ssto)
```

只要写成GRADS可读的文件就行了。

生成后可以用cdo的命令`cdo -f nc import_binary force.ctl force.nc`转换成nc格式后打开看看

SPEEDY模式运行完成后会直接生成对应的ctl文件，可以用上面这条命令转换为nc格式后再进行分析，非常方便

