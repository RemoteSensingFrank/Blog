---
title: RPC（共线条件方程）校正迭代方法分析
date: 2018-07-22 21:47:52
tags: 图像处理数学原理
categories: 图像处理
---
这次主要分析GDAL中RPC校正的实现，以及在GDAL中PRPC校正的迭代方法，另外关于迭代计算还是有一些关于迭代的疑惑：
```C++
static bool
RPCInverseTransformPoint( GDALRPCTransformInfo *psTransform,
                          double dfPixel, double dfLine, double dfUserHeight,
                          double *pdfLong, double *pdfLat )

{
    // Memo:
    // Known to work with 40 iterations with DEM on all points (int coord and
    // +0.5,+0.5 shift) of flock1.20160216_041050_0905.tif, especially on (0,0).

/* -------------------------------------------------------------------- */
/*      Compute an initial approximation based on linear                */
/*      interpolation from our reference point.                         */
/* -------------------------------------------------------------------- */
    double dfResultX =
        psTransform->adfPLToLatLongGeoTransform[0] +
        psTransform->adfPLToLatLongGeoTransform[1] * dfPixel +
        psTransform->adfPLToLatLongGeoTransform[2] * dfLine;

    double dfResultY =
        psTransform->adfPLToLatLongGeoTransform[3] +
        psTransform->adfPLToLatLongGeoTransform[4] * dfPixel +
        psTransform->adfPLToLatLongGeoTransform[5] * dfLine;

    if( psTransform->bRPCInverseVerbose )
    {
        CPLDebug("RPC", "Computing inverse transform for (pixel,line)=(%f,%f)",
                 dfPixel, dfLine);
    }
    VSILFILE* fpLog = nullptr;
    if( psTransform->pszRPCInverseLog )
    {
        fpLog =
            VSIFOpenL( CPLResetExtension(psTransform->pszRPCInverseLog, "csvt"),
                       "wb" );
        if( fpLog != nullptr )
        {
            VSIFPrintfL( fpLog, "Integer,Real,Real,Real,String,Real,Real\n" );
            VSIFCloseL( fpLog );
        }
        fpLog = VSIFOpenL( psTransform->pszRPCInverseLog, "wb" );
        if( fpLog != nullptr )
            VSIFPrintfL(
                fpLog,
                "iter,long,lat,height,WKT,error_pixel_x,error_pixel_y\n" );
    }

/* -------------------------------------------------------------------- */
/*      Now iterate, trying to find a closer LL location that will      */
/*      back transform to the indicated pixel and line.                 */
/* -------------------------------------------------------------------- */
    double dfPixelDeltaX = 0.0;
    double dfPixelDeltaY = 0.0;
    double dfLastResultX = 0.0;
    double dfLastResultY = 0.0;
    double dfLastPixelDeltaX = 0.0;
    double dfLastPixelDeltaY = 0.0;
    double dfDEMH = 0.0;
    bool bLastPixelDeltaValid = false;
    const int nMaxIterations =
        (psTransform->nMaxIterations > 0) ? psTransform->nMaxIterations :
        (psTransform->poDS != nullptr) ? 20 : 10;
    int nCountConsecutiveErrorBelow2 = 0;

    int iIter = 0;  // Used after for.
    for( ; iIter < nMaxIterations; iIter++ )
    {
        double dfBackPixel = 0.0;
        double dfBackLine = 0.0;

        // Update DEMH.
        dfDEMH = 0.0;
        double dfDEMPixel = 0.0;
        double dfDEMLine = 0.0;
        if( !GDALRPCGetHeightAtLongLat(psTransform, dfResultX, dfResultY,
                                       &dfDEMH, &dfDEMPixel, &dfDEMLine) )
        {
            if( psTransform->poDS )
            {
                CPLDebug(
                    "RPC", "DEM (pixel, line) = (%g, %g)",
                    dfDEMPixel, dfDEMLine);
            }

            // The first time, the guess might be completely out of the
            // validity of the DEM, so pickup the "reference Z" as the
            // first guess or the closest point of the DEM by snapping to it.
            if( iIter == 0 )
            {
                bool bUseRefZ = true;
                if( psTransform->poDS )
                {
                    if( dfDEMPixel >= psTransform->poDS->GetRasterXSize() )
                        dfDEMPixel = psTransform->poDS->GetRasterXSize() - 0.5;
                    else if( dfDEMPixel < 0 )
                        dfDEMPixel = 0.5;
                    if( dfDEMLine >= psTransform->poDS->GetRasterYSize() )
                        dfDEMLine = psTransform->poDS->GetRasterYSize() - 0.5;
                    else if( dfDEMPixel < 0 )
                        dfDEMPixel = 0.5;
                    if( GDALRPCGetDEMHeight( psTransform, dfDEMPixel,
                                             dfDEMLine, &dfDEMH) )
                    {
                        bUseRefZ = false;
                        CPLDebug(
                            "RPC", "Iteration %d for (pixel, line) = (%g, %g): "
                            "No elevation value at %.15g %.15g. "
                            "Using elevation %g at DEM (pixel, line) = "
                            "(%g, %g) (snapping to boundaries) instead",
                            iIter, dfPixel, dfLine,
                            dfResultX, dfResultY,
                            dfDEMH, dfDEMPixel, dfDEMLine );
                    }
                }
                if( bUseRefZ )
                {
                    dfDEMH = psTransform->dfRefZ;
                    CPLDebug(
                        "RPC", "Iteration %d for (pixel, line) = (%g, %g): "
                        "No elevation value at %.15g %.15g. "
                        "Using elevation %g of reference point instead",
                        iIter, dfPixel, dfLine,
                        dfResultX, dfResultY,
                        dfDEMH);
                }
            }
            else
            {
                CPLDebug("RPC", "Iteration %d for (pixel, line) = (%g, %g): "
                          "No elevation value at %.15g %.15g. Erroring out",
                          iIter, dfPixel, dfLine, dfResultX, dfResultY);
                if( fpLog )
                    VSIFCloseL(fpLog);
                return false;
            }
        }

        RPCTransformPoint( psTransform, dfResultX, dfResultY,
                           dfUserHeight + dfDEMH,
                           &dfBackPixel, &dfBackLine );

        dfPixelDeltaX = dfBackPixel - dfPixel;
        dfPixelDeltaY = dfBackLine - dfLine;

        if( psTransform->bRPCInverseVerbose )
        {
            CPLDebug(
                "RPC", "Iter %d: dfPixelDeltaX=%.02f, dfPixelDeltaY=%.02f, "
                "long=%f, lat=%f, height=%f",
                iIter, dfPixelDeltaX, dfPixelDeltaY,
                dfResultX, dfResultY, dfUserHeight + dfDEMH);
        }
        if( fpLog != nullptr )
        {
            VSIFPrintfL(
                fpLog, "%d,%.12f,%.12f,%f,\"POINT(%.12f %.12f)\",%f,%f\n",
                iIter, dfResultX, dfResultY, dfUserHeight + dfDEMH,
                dfResultX, dfResultY, dfPixelDeltaX, dfPixelDeltaY);
        }

        const double dfError =
            std::max(std::abs(dfPixelDeltaX), std::abs(dfPixelDeltaY));
        if( dfError < psTransform->dfPixErrThreshold )
        {
            iIter = -1;
            if( psTransform->bRPCInverseVerbose )
            {
                CPLDebug( "RPC", "Converged!" );
            }
            break;
        }
        else if( psTransform->poDS != nullptr &&
                 bLastPixelDeltaValid &&
                 dfPixelDeltaX * dfLastPixelDeltaX < 0 &&
                 dfPixelDeltaY * dfLastPixelDeltaY < 0 )
        {
            // When there is a DEM, if the error changes sign, we might
            // oscillate forever, so take a mean position as a new guess.
            if( psTransform->bRPCInverseVerbose )
            {
                CPLDebug(
                    "RPC", "Oscillation detected. "
                    "Taking mean of 2 previous results as new guess" );
            }
            dfResultX =
                ( fabs(dfPixelDeltaX) * dfLastResultX +
                  fabs(dfLastPixelDeltaX) * dfResultX ) /
                (fabs(dfPixelDeltaX) + fabs(dfLastPixelDeltaX));
            dfResultY =
                ( fabs(dfPixelDeltaY) * dfLastResultY +
                  fabs(dfLastPixelDeltaY) * dfResultY ) /
                (fabs(dfPixelDeltaY) + fabs(dfLastPixelDeltaY));
            bLastPixelDeltaValid = false;
            nCountConsecutiveErrorBelow2 = 0;
            continue;
        }

        double dfBoostFactor = 1.0;
        if( psTransform->poDS != nullptr &&
            nCountConsecutiveErrorBelow2 >= 5 && dfError < 2 )
        {
          // When there is a DEM, if we remain below a given threshold (somewhat
          // arbitrarily set to 2 pixels) for some time, apply a "boost factor"
          // for the new guessed result, in the hope we will go out of the
          // somewhat current stuck situation.
          dfBoostFactor = 10;
          if( psTransform->bRPCInverseVerbose )
          {
              CPLDebug("RPC", "Applying boost factor 10");
          }
        }

        if( dfError < 2 )
            nCountConsecutiveErrorBelow2++;
        else
            nCountConsecutiveErrorBelow2 = 0;

        const double dfNewResultX = dfResultX
            - ( dfPixelDeltaX * psTransform->adfPLToLatLongGeoTransform[1] *
                dfBoostFactor )
            - ( dfPixelDeltaY * psTransform->adfPLToLatLongGeoTransform[2] *
                dfBoostFactor );
        const double dfNewResultY = dfResultY
            - ( dfPixelDeltaX * psTransform->adfPLToLatLongGeoTransform[4] *
                dfBoostFactor )
            - ( dfPixelDeltaY * psTransform->adfPLToLatLongGeoTransform[5] *
                dfBoostFactor );

        dfLastResultX = dfResultX;
        dfLastResultY = dfResultY;
        dfResultX = dfNewResultX;
        dfResultY = dfNewResultY;
        dfLastPixelDeltaX = dfPixelDeltaX;
        dfLastPixelDeltaY = dfPixelDeltaY;
        bLastPixelDeltaValid = true;
    }
    if( fpLog != nullptr )
        VSIFCloseL( fpLog );

    if( iIter != -1 )
    {
        CPLDebug( "RPC", "Failed Iterations %d: Got: %.16g,%.16g  Offset=%g,%g",
                  iIter,
                  dfResultX, dfResultY,
                  dfPixelDeltaX, dfPixelDeltaY );
        return false;
    }

    *pdfLong = dfResultX;
    *pdfLat = dfResultY;
    return true;
}
```
上述代码中有很多判断的部分，我们忽略掉这些部分直接看干货，关于PRC校正的方法在以前的文章中已经做过比较详细的分析了，在这里我们不对校正方法进行分析，直接看迭代过程：  
首先定义了一系列的变量，包括什么迭代变量，误差限变量等，然后根据变换参数计算像素的初始坐标，计算方法为直接根据参考点进行线性插值得到的，实际上这里应该是存疑的，如果没有参考点应该怎么办，我估计这个转换参数在RPC参数中应该是保存的，所以在这里也不详细分析了，代码部分也没有具体实现，我们直接理解就是一个初始化的计算参数。迭代的计算过程为：
* 根据坐标以及DEM计算初始位置的高程，然后就是一系列的判断，包括判断DEM的坐标系与给出影像的坐标系，判断是否能够得到高程，以及一系列其他的判断；然后最后的结果就是得到DEM的高程，整个判断结束；
*  RPC反算，根据坐标和RPC参数反算出对应的像素点的坐标;
*  计算当前给出的上一次给出的像素值和反解出的像素值的误差值；
*  判断误差值是否超限了，如果误差在误差限内则说明精度满足要求直接跳出循环了，如果超过误差限则进行迭代；（p.s:在这里GDAL有一个判断我觉得做得很好，在迭代的过程中给了一个猜测参数，如果整个迭代的过程出现oscillate现象，则重新给一个猜测值，这个值的给定估计也有一点经验）
*  新的值等于原来的值加上误差项（表现为差值）

对于以上过程的前几步都是比较好理解的，但是对于这个最后一个新值的获取我有一些疑惑，如果我自己实现的话我会直接用新值带入RPC参数解算新的DEM和坐标并进行迭代，关于迭代的误差在以前的文章中也提到过，初始高程设置的不同会导致迭代次数和误差都特别大，想弄清楚两种迭代方式有什么区别，如果搞明白了我会在下一次的文章中进行详细的描述。