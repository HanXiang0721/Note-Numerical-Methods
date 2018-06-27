# BCC资料驱动WRF

> BCC(Beijing Climate Center)基于CCSM模式研发的全球模式，可以用于驱动WRF。但是由于两者有些差别，直接驱动是不行的，需要对BCC资料做一定的处理。

BCC资料为p-sigma混合坐标系，而WRF输入坐标系为p坐标系，实际模式运行时是Eta坐标系。因而前期思路是做好坐标转换。BCC资料用于坐标转换的公式如下：p = a*p0 + b*ps，a，b为计算系数，p0为常数，ps为地表面气压。WRF本身带有EMCWF资料坐标系转换工具，欧洲中心资料坐标系为sigma坐标系，与BCC资料比较类似，公式变形后可以使用WPS内置工具转换坐标，文件emcwf_coeffs放置计算所需的一些系数。在run时，namelist里的积分步长可能需要调整，本次积分步长120s可以正常积分，180s则发生崩溃，此外，在运行real.exe时会发生无法完成的现象，报错信息是插值阶数过高，因而采用线性插值，在namelist中设置lagrange_order=1,interp_type=1，之后即可正常运行。此外，该程序中水汽插值中关于地面水汽插值的代码被注释，可去掉。

C. calc_ecmwf_p.exe

In the course of vertically interpolating meteorological fields, the real program requires 3-d pressure and geopotential height fields on the same levels as the other atmospheric fields. The calc_ecmwf_p.exe utility may be used to create such these fields for use with ECMWF sigma-level data sets. Given a surface pressure field (or log of surface pressure field) and a list of coefficients A and B, calc_ecmwf_p.exe computes the pressure at an ECMWF sigma level k at grid point (i, j) as Pijk = Ak + Bk*Psfcij. The list of coefficients used in the pressure computation can be copied from a table appropriate to the number of sigma levels in the data set from http://www.ecmwf.int/products/data/technical/model_levels/index.html . This table should be written in plain text to a file, ecmwf_coeffs, in the current working directory; for example, with 16 sigma levels, the file emcwf_coeffs would contain something like:

0 0.000000 0.000000000
1 5000.000000 0.000000000
2 9890.519531 0.001720764
3 14166.304688 0.013197623
4 17346.066406 0.042217135
5 19121.152344 0.093761623
6 19371.250000 0.169571340
7 18164.472656 0.268015683
8 15742.183594 0.384274483
9 12488.050781 0.510830879
10 8881.824219 0.638268471
11 5437.539063 0.756384850
12 2626.257813 0.855612755
13 783.296631 0.928746223
14 0.000000 0.972985268
15 0.000000 0.992281914
16 0.000000 1.000000000

Additionally, if soil height (or soil geopotential), 3-d temperature, and 3-d specific humidity fields are available, calc_ecmwf_p.exe computes a 3-d geopotential height field, which is required to obtain an accurate vertical interpolation in the real program. Given a set of intermediate files produced by ungrib and the file ecmwf_coeffs, calc_ecmwf_p loops over all time periods in namelist.wps, and produces an additional intermediate file, PRES:YYYY-MM- DD_HH, for each time, which contains pressure and geopotential height data for each full sigma level, as well as a 3-d relative humidity field. This intermediate file should be specified to metgrid, along with the intermediate data produced by ungrib, by adding 'PRES' to the list of prefixes in the fg_name namelist variable.