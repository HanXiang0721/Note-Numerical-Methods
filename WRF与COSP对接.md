# WRF与COSP对接

> The CFMIP Observation Simulator Package (COSP)是一种云模拟器，可以将模式中模拟的云进行处理，从而实现和卫星观测直接比较。云模拟器存在的价值主要在于，一些云的特征，例如云量，是不能直接与卫星观测比较的。WRF不提供面向COSP的接口，而CAM已经与COSP实现了完美对接。

MODIS卫星：需要输出光学厚度，查看Registry中的TAUCLD几个参数，只能使用CAM辐射方案

CloudSat等其他卫星：需要输出云滴有效半径，具体视云微方案决定，但至少是Double Moment方案

在WRF的云微方案中，云滴有效半径是不直接输出的，Thompson方案可以通过修改io输出其中几个，但Morrison和WDM6只能修改原代码，通过接口一步步传出来。

有些变量没有，需要通过WRF的输出量自行诊断出来。