
<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

- [环境搭建](#环境搭建)
  - [下载设备驱动程序](#下载设备驱动程序)
  - [VS 环境配置](#vs-环境配置)
- [示例](#示例)
  - [Demo1](#demo1)
  - [注意点](#注意点)
    - [Haptic Frames](#haptic-frames)
    - [同步调用与异步调用](#同步调用与异步调用)
    - [回调函数返回值](#回调函数返回值)

<!-- /code_chunk_output -->

# 环境搭建

## 下载设备驱动程序

https://support.3dsystems.com/s/article/Downloads?language=en_US 从这里可以下载**设备驱动**和 **OpenHaptics**。

## VS 环境配置



# 示例

## Demo1

实时获取操作手柄的位置信息，并在一定范围内模拟引力。

```javascript
#include <stdio.h>
#include <string.h>
#include <conio.h>

#include <HD/hd.h>
#include <HDU/hduError.h>
#include <HDU/hduVector.h>

HDCallbackCode HDCALLBACK gravityWellCallback(void *data);

int main(int argc, char* argv[])
{    
    HDErrorInfo error;
    HDSchedulerHandle hGravityWell;

    HHD hHD = hdInitDevice(HD_DEFAULT_DEVICE);  // 设备初始化
    if (HD_DEVICE_ERROR(error = hdGetError())) 
    {
        hduPrintError(stderr, &error, "Failed to initialize haptic device");
        fprintf(stderr, "\nPress any key to quit.\n");
        return -1;
    }
    printf("Hello Haptic Device!\n");
    printf("Found device model: %s.\n\n", hdGetString(HD_DEVICE_MODEL_TYPE));

    // 异步调用
    hGravityWell = hdScheduleAsynchronous(gravityWellCallback, 0, HD_MAX_SCHEDULER_PRIORITY);

    hdEnable(HD_FORCE_OUTPUT);  // 启动力
    hdStartScheduler();  // 启动调度程序？

    /* Check for errors and abort if so. */
    if (HD_DEVICE_ERROR(error = hdGetError()))
    {
        hduPrintError(stderr, &error, "Failed to start scheduler");
        fprintf(stderr, "\nPress any key to quit.\n");
        return -1;
    }

    // 按任意键退出程序
    printf("Press any key to quit.\n\n");
    while (!kbhit())
    {
        if (!hdWaitForCompletion(hGravityWell，HD_WAIT_CHECK_STATUS))
        {
            fprintf(stderr, "Press any key to quit.\n");     
            break;
        }
    }

    hdStopScheduler();  // 一般这个是清理的第一步
    hdUnschedule(hGravityWell);  // 利用回调函数的句柄，进行清理
    hdDisableDevice(hHD);  // Disables a device.

    return 0;
}

HDCallbackCode HDCALLBACK gravityWellCallback(void *data)
{
    const HDdouble kStiffness = 0.075; // N/mm
    const HDdouble kGravityWellInfluence = 40; // mm

    static const hduVector3Dd wellPos = {0,0,0};  // 引力中心

    HDErrorInfo error;
    hduVector3Dd position;
    hduVector3Dd force;
    hduVector3Dd positionTwell;

    HHD hHD = hdGetCurrentDevice();  // 得到当前设备的句柄

    hdBeginFrame(hHD);  // frame 开始，保证数据访问一致性

    hdGetDoublev(HD_CURRENT_POSITION, position);  // 获取当前设备的位置
    
    memset(force, 0, sizeof(hduVector3Dd));
    
    // positionTwell = wellPos - position
    hduVecSubtract(positionTwell, wellPos, position);
    
    if (hduVecMagnitude(positionTwell) < kGravityWellInfluence)
    {
        // force = kStiffness * positionTwell
        hduVecScale(force, positionTwell, kStiffness);
    }

    hdSetDoublev(HD_CURRENT_FORCE, force);  // 发送力信息给设备
    
    hdEndFrame(hHD);  // frame 结束

    if (HD_DEVICE_ERROR(error = hdGetError()))
    {
        hduPrintError(stderr, &error, 
                      "Error detected while rendering gravity well\n");
        if (hduIsSchedulerError(&error))
        {
            return HD_CALLBACK_DONE;
        }
    }

    /* Signify that the callback should continue running, i.e. that
       it will be called again the next scheduler tick. */
    return HD_CALLBACK_CONTINUE;
}
```

## 注意点

### Haptic Frames

为了保证数据访问的一致性，OpenHaptics 提供了一种 frame 框架结构，使用 hdBeginFrame 和 hdEndFrame 作为访问的开始和结束。

在同一 frame 中多次调用 hdGet 类函数获取信息会得到相同的结果；多次调用 hdSet 类函数设置同一状态，则后一次的调用会替代前一次的结果。

### 同步调用与异步调用

**同步调用：** 调用完成后返回，应用程序线程等待同步调用完成后继续。例如用于获取设备的当前状态。

**异步调用：** 立即返回，循环处理任务，得到连续的值。例如用于连续化表示受力效果。

### 回调函数返回值

**HD_CALLBACK_DONE：** 回调函数只执行一次。
**HD_CALLBACK_CONTINUE：** 回调函数循环执行。
　