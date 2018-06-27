# CLM中SurfaceAlbedo的修改方法

cesm模式中，`cesm1_2_1/models/lnd/clm/src/clm4_5/biogeophys`目录下的SurfaceAlbedoMod.F90是修改的主要文件。

SurfaceAlbedoMod.F90文件中，有SurfaceAlbedo和TwoStream两个函数,其中SurfaceAlbedo的接口如下：

```fortran
  subroutine SurfaceAlbedo(lbg, ubg, lbc, ubc, lbp, ubp, &
                           num_nourbanc, filter_nourbanc, &
                           num_nourbanp, filter_nourbanp, &
                           nextsw_cday, declinp1)
```

这个仅仅是接口而已。CESM中有些量并不是通过接口传入，而是通过指针直接访问内存，所以需要具体看其中的变量声明。在所有的变量中，有三种类型需要注意：

gridcell level，column level，pft level （plant function type）

```fortran
integer , intent(in) :: lbg, ubg                  ! gridcell bounds
```

lbg，ubg这两个变量值得注意，他们在下面这段程序中使用到：

```fortran
    do g = lbg, ubg
       coszen_gcell(g) = shr_orb_cosz (nextsw_cday, lat(g), lon(g), declinp1)
    end do
```

这段程序意义先不管，但是它表明了lat，lon两个数组的下标范围以及gridcell level的范围

然后我们来看我们的目标数组：

```fortran
    real(r8), pointer :: albgrd(:,:)        ! ground albedo (direct)
    real(r8), pointer :: albgri(:,:)        ! ground albedo (diffuse)
    real(r8), pointer :: albd(:,:)          ! surface albedo (direct)
    real(r8), pointer :: albi(:,:)          ! surface albedo (diffuse)
```

反射率的这几个数组的实际分配是在

```fortran
    ! Assign local pointers to derived subtypes components (column-level)
    cgridcell      =>col%gridcell
    h2osno        => cws%h2osno
    albgrd        => cps%albgrd
    albgri        => cps%albgri

    ! Assign local pointers to derived subtypes components (pft-level)
    pactive   => pft%active
    plandunit =>pft%landunit
    pgridcell =>pft%gridcell
    pcolumn   =>pft%column
    albd      => pps%albd
    albi      => pps%albi
    fabd      => pps%fabd
```

albgrd和albgri两个量是column level，而albd是pft level

更进一步，看到整个程序的末尾：

```fortran
   ! Determine values for non-vegetated pfts where coszen > 0
    do ib = 1,numrad
       do fp = 1,num_novegsol
          p = filter_novegsol(fp)
          c = pcolumn(p)
          fabd(p,ib) = 0._r8
          fabd_sun(p,ib) = 0._r8
          fabd_sha(p,ib) = 0._r8
          fabi(p,ib) = 0._r8
          fabi_sun(p,ib) = 0._r8
          fabi_sha(p,ib) = 0._r8
          ftdd(p,ib) = 1._r8
          ftid(p,ib) = 0._r8
          ftii(p,ib) = 1._r8
          albd(p,ib) = albgrd(c,ib)
          albi(p,ib) = albgri(c,ib)
!         if (ib == 1) then
!            do iv = 1, nrad(p)
!               fabd_sun_z(p,iv) = 0._r8
!               fabd_sha_z(p,iv) = 0._r8
!               fabi_sun_z(p,iv) = 0._r8
!               fabi_sha_z(p,iv) = 0._r8
!               fsun_z(p,iv) = 0._r8
!            end do
!         end if
       end do
    end do
  end subroutine SurfaceAlbedo
```

我们可以看到，对于非植被覆盖的pft，albd的值是来自albgrd。这两个数组之间的对应关系，通过一个很关键的函数：pcolumn

从程序中可以找到下面这段内容：

```fortran
    logical , pointer :: pactive(:)   ! true=>do computations on this pft (see reweightMod for details)
    integer , pointer :: pgridcell(:) ! gridcell of corresponding pft
    integer , pointer :: plandunit(:) ! index into landunit level quantities
    integer , pointer :: itypelun(:)  ! landunit type
    integer , pointer :: pcolumn(:)   ! column of corresponding pft
    integer , pointer :: cgridcell(:) ! gridcell of corresponding column
```

pcolumn起到的作用是pft的量的下标转换为column的量的下标
cgridcell起到的作用是column的量的下标转为gridcell的量的下标
pgridcell起到的作用则是gridcell的量转为pft量的下标
这三个函数形成了一个闭环

然后我们来看albgrd的计算程序：

```fortran
    ! ground albedos and snow-fraction weighting of snow absorption factors
    do ib = 1, nband
       do fc = 1,num_nourbanc
          c = filter_nourbanc(fc)
          if (coszen(c) > 0._r8) then
             ! ground albedo was originally computed in SoilAlbedo, but is now computed here
             ! because the order of SoilAlbedo and SNICAR_RT was switched for SNICAR.
             albgrd(c,ib) = albsod(c,ib)*(1._r8-frac_sno(c)) + albsnd(c,ib)*frac_sno(c)
             albgri(c,ib) = albsoi(c,ib)*(1._r8-frac_sno(c)) + albsni(c,ib)*frac_sno(c)
```

因此，可以在这里对albgrd的计算进行调整。

另一个方面是，如果我们需要对特定时间范围内的反照率进行修改，那么，需要获取时间。在clm中，有获取当前计算步的时间的函数，这个函数叫get_curr_date,在cesm1_2_1/models/lnd/clm/src/util_share/clm_time_manager.F90文件中。

因此，在模式中修改反照率的程序可以如下：

```fortran
             g=cgridcell(c)
             if((lat(g).ge.(32.0*3.14/180.0) .and. lat(g).le.(37.0*3.14/180.0).and.&
              lon(g).ge.(78.0*3.14/180.0) .and. lon(g).le.(92.0*3.14/180.0)).and.(&
               mon.eq.11 .or. mon.eq.12 .or. mon.eq.1 .or. mon.eq.2 .or. mon.eq.3))  then
!                   albgrd(c,ib) = 0.9
!                   albgri(c,ib) = 0.9
                   albgrd(c,ib) = albsod(c,ib)*(1._r8-frac_sno(c)) + albsnd(c,ib)*frac_sno(c)
                   albgri(c,ib) = albsoi(c,ib)*(1._r8-frac_sno(c)) + albsni(c,ib)*frac_sno(c)
                   albgrd(c,ib) =
min(albgrd(c,ib)+albgrd(c,ib)*0.8, 0.9)   ! Increase 80%
                   albgri(c,ib) = min(albgri(c,ib)+albgri(c,ib)*0.8, 0.9)
             else
                   albgrd(c,ib) = albsod(c,ib)*(1._r8-frac_sno(c)) + albsnd(c,ib)*frac_sno(c)
                   albgri(c,ib) = albsoi(c,ib)*(1._r8-frac_sno(c)) + albsni(c,ib)*frac_sno(c)
             end if
```

完整的程序请看下面。其中QQF标注的是修改的部分

[SurfaceAlbedoMod.F90](SurfaceAlbedoMod.F90)
