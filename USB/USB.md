# 目录  

- [目录](#目录)
  - [0. 前言](#0-前言)
  - [1. 官方xusbps\_intr\_example代码分析](#1-官方xusbps_intr_example代码分析)
    - [1.1 找到中断在何处设置的](#11-找到中断在何处设置的)
      - [1.1.1 UsbSetupIntrSystem](#111-usbsetupintrsystem)
    - [1.2 理清中断的关系](#12-理清中断的关系)
    - [1.3 请根据源码和注释理解 UsbIntrExample](#13-请根据源码和注释理解-usbintrexample)
  - [2 总结](#2-总结)

## 0. 前言  

通过前面的中断控制器学习, 了解到 ZYNQ 为不同的中断设置了各自的 ID, 因此在 USB 中当然也有这一步, 不过要区分一点, 中断控制器提到的 XScuGic_Connect 只为每个器件设置中断, 如 DMA 的中断, USB 的中断. 不同器件内还有许多不同的中断, 比如 USB 收到数据和发送数据完毕的中断, 这些中断是不使用 XScuGic_Connect 连接的, 而是使用 XScuGic_Connect 为器件设置的handler进一步进行分发.  

- 中断控制器->器件的总中断handler(根据器件不同有一个或者多个,这里会选择其中一个)->不同事件的handler  

接下来根据**官方的usb代码**和[网友的代码](https://github.com/lapauwThomas/Zynq_winUSB)对SDK下如何操作USB进行讲解  

## 1. 官方xusbps_intr_example代码分析  

以下代码分析并非按照顺序讲解, 而是按照中断控制器的[1.2 正点原子在DMA的例子](../中断控制器/中断控制器.md/#12-正点原子在dma的例子)所划分的区域进行分析, 不过在最后会按照代码顺序再理一次.  

在main函数中很明显只有UsbIntrExample函数其主要作用, 此例子的代码都在此函数中, 请跳转到此函数定义所在的位置.  

### 1.1 找到中断在何处设置的  

打开 UsbIntrExample 函数的定义发现是一些在中断控制器中没见过的内容, 但是有一些类似的函数, 比如 XUsbPs_LookupConfig 和 XUsbPs_CfgInitialize, 这些都是用来操作USB的东西(即器件独有的函数和参数), 我们先找到和中断控制器有关的内容. 在xusbps_intr_example.c的198行的 **UsbSetupIntrSystem** 便是与中断控制器有关的内容.  

```c
Status = UsbSetupIntrSyste(IntcInstancePtr,
                UsbInstancePtr,
                UsbIntrId);
```

#### 1.1.1 UsbSetupIntrSystem  

在此函数定义中便可以看见一系列和 XScuGic 有关的函数, 即中断控制器所讲述的内容, 根据[1.2 正点原子在DMA的例子](../中断控制器/中断控制器.md/#12-正点原子在dma的例子), 可以发现和USB这个器件有关的函数(和其他器件相比仅是参数不同),在544-553行:  

```c
//->->根据使用的设备不同而有所不同(函数的参数不同)
Status = XScuGic_Connect(IntcInstancePtr,UsbIntrId,
            (Xil_ExceptionHandler)XUsbPs_IntrHandler,
            (void *)UsbInstancePtr);
if (Status != XST_SUCCESS) {
    return Status;
}
/*
 * Enable the interrupt for the device.
 */
XScuGic_Enable(IntcInstancePtr, UsbIntrId);
```  

可以看见544行为 UsbIntrId 设置了中断处理函数**XUsbPs_IntrHandler**, 即PS发现USB触发中断后, 第一个调用的USB相关的函数就是 **XUsbPs_IntrHandler** , 此函数会进一步分发和USB相关的更为细致的handler.  

在XUsbPs_IntrHandler(这个函数一般无需更改)的定义处, 有以下几点需要注意:  

- 此函数一般不需要更改  
- 在182行之前和USB端点无关的内容之前都无需太过关注, 不过您也可以阅读源码获得更多信息, 在182行后可以看见XUsbPs_IntrHandler如何进一步分发handler(分发到XUsbPs_IntrHandleRX和XUsbPs_IntrHandleTX,这两个函数又会再一次分发), 后续讲解和USB本身有关的内容的时候会再次关注此处.
- USB物理器件触发PS中断 -> XUsbPs_IntrHandler -> XUsbPs_IntrHandleRX/TX -> 和端点有关的handler  

### 1.2 理清中断的关系  

[1.1 找到中断在何处设置的](#11-找到中断在何处设置的)所讲述的是从USB触发中断到PS跳转到USB总中断处理handler的路径, 仔细观察就会发现USB的一条中断线如何区分那么多种USB状态呢, 为何不使用一堆中断线然后Connect这些中断到不同的handler. 在硬件上这点肯定是可以实现的, 但是没有必要, PS在收到中断后, USB总中断处理可以通过读取USB的寄存器获得更详细的信息, 使用这些信息再将中断进一步分发, 这部分内容就在**xusbps_intr_example.c**的**277**和**288**行.  

```c
Status = XUsbPs_EpSetHandle(UsbInstancePtr, 0,
            XUSBPS_EP_DIRECTION_OUT,
            XUsbPs_Ep0EventHandler, UsbInstancePtr);
/* Set the handler for handling endpoint 1events.
 *
 * Note that for this example we do not need to register a handler for
 * TX complete events as we only send data using static data buffers
 * that do not need to be free()d or returned to the OS after they have
 * been sent.
 */
Status = XUsbPs_EpSetHandle(UsbInstancePtr, 1,
            XUSBPS_EP_DIRECTION_OUT,
            XUsbPs_Ep1EventHandler, UsbInstancePtr);
```  
这里使用了和 XScuGic_Connect 功能类似的函数 XUsbPs_EpSetHandle, 不过要注意区分一下, XScuGic_Connect可以理解为将USB中断ID和USB中断handler直接和硬件的中断控制器相关联, 而 XUsbPs_EpSetHandle 则是在USB中断 handler 中注册一个中断, 在USB中断 handler 会进行搜索的某个结构体中加入相应的信息.  

那么我们看看, USB中断 handler 会搜索哪个结构体呢, 找到 **XUsbPs_IntrHandler** 的定义处的182和188行, 找到 **XUsbPs_IntrHandleRX** 或者 **XUsbPs_IntrHandleTX** 的定义处, 这里以 **XUsbPs_IntrHandleRX** 为例, 看341行:  

```c
if (Ep->HandlerFunc) {
    Ep->HandlerFunc(Ep->HandlerRef, Index,
            XUSBPS_EP_EVENT_DATA_RX, NULL);
```  

可以看见在这调用了 **Ep->HandlerFunc** ,而334行有 **Ep = &InstancePtr->DeviceConfig.Ep[Index].Out**, 可以看出上述的结构体应该就是 **XUsbPs**, 或者说是 **XUsbPs** 中的 **DeviceConfig.Ep**, 现在请找到 **XUsbPs_EpSetHandler** 的定义处验证我们的猜想:  

```c  
Ep = &InstancePtr->DeviceConfig.Ep[EpNum];
if(Direction & XUSBPS_EP_DIRECTION_OUT) {
    Ep->Out.HandlerFunc    = CallBackFunc;
    Ep->Out.HandlerRef    = CallBackRef;
}
if(Direction & XUSBPS_EP_DIRECTION_IN) {
    Ep->In.HandlerFunc    = CallBackFunc;
    Ep->In.HandlerRef    = CallBackRef;
}
```  

在这可以看见 **XUsbPs_EpSetHandler** 根据USB端点(Endpoint,EP)不同和方向(Direction)不同而在 **XUsbPs** 中的 **DeviceConfig.Ep** 中设置了相应的handler.  

不过在这要强调一下 Direction == In 和 Direction == Out 的 handler 到底是什么时候调用的.  

- Out 和 In 都是相对于master而言的, 比如PC作为master , 而PS端作为 device, 则 Out 是指PC主动发送信息到PS, In 是指PC向 device 主动发起读取信息. TX 和 RX 则是相对 PS 而言的, RX 对应 Out, TX 对应 In.  
  - PC向PS的USB发送数据 -> **USB 数据接收完毕后触发USB总中断handler** -> 总中断handler通过读取寄存器发现是 Out 则调用 XUsbPs_IntrHandleRX -> XUsbPs_IntrHandleTX 通过对端点号进行扫描调用对应的 Ep->Out.HandlerFunc  
  - PS将数据放到USB上 -> PC在某个时刻对PS的USB发起 In 请求 -> **等待 USB 将数据传递到PC后触发USB总中断handler** -> 总中断handler通过读取寄存器发现是 In 则调用 XUsbPs_IntrHandleTX -> XUsbPs_IntrHandleTX 通过对端点号进行扫描调用对应的 Ep->In.HandlerFunc  

大家可以在 XUsbPs_Ep1EventHandler 定义处的 476 行将以下代码替换为自己的协议, 比如将存在USB buffer中的数据打印出来.当然更建议的是将总的协议封装成一个函数进行调用,比如将USB收到的数据发送到GPIO从而控制外设或者PL的逻辑.  

```c
// XUsbPs_HandleStorageReq(InstancePtr, EpNum,BufferPtr, BufferLen);
BufferPtr[BufferLen - 1] = '\0' ;
xil_printf("BufferPtr = %s \r\n",BufferPtr);
```

这样就会在接收到 PC 发送的数据后将信息打印出来对照啦.  

### 1.3 请根据源码和注释理解 UsbIntrExample  

如果不理解和USB相关的东西, 可以先百度一下看看, 暂时不打算添加这部分内容.  

```c  
static int UsbIntrExample(XScuGic *IntcInstancePtr, XUsbPs *UsbInstancePtr,
                    u16 UsbDeviceId, u16 UsbIntrId)
{
    int    Status;
    u8    *MemPtr = NULL;
    int    ReturnStatus = XST_FAILURE;

    /* 此例子只配置了两个端点(Endpoint)
        Ep0 默认控制端点
        Ep1 BULK端点
     */
    const u8 NumEndpoints = 2;

    XUsbPs_Config        *UsbConfigPtr;
    XUsbPs_DeviceConfig    DeviceConfig;

    /* 根据 UsbDeviceId 查找对应的 Usb 配置信息(就是在一个数组里面根据 UsbDeviceId 查找)
     */
    UsbConfigPtr = XUsbPs_LookupConfig(UsbDeviceId);
    if (NULL == UsbConfigPtr) {
        goto out;
    }


    /* 因为在此例子中未开启虚拟地址, 所以物理地址和虚拟地址相同, 因此我们将物理基地址作为第三个参数. 对于支持虚拟内存的系统, 第三个参数应当使用虚拟地址.  
     */
     // 将 UsbConfigPtr 中的数据复制到 UsbInstancePtr->UsbConfigPtr
    Status = XUsbPs_CfgInitialize(UsbInstancePtr,
                       UsbConfigPtr,
                       UsbConfigPtr->BaseAddress);
    if (XST_SUCCESS != Status) {
        goto out;
    }

    /* 即中断控制器那提到的, 设置USB的总中断handler, 可以参考 1.1节  
     */
    Status = UsbSetupIntrSystem(IntcInstancePtr,
                    UsbInstancePtr,
                    UsbIntrId);
    if (XST_SUCCESS != Status)
    {
        goto out;
    }

    /* 
    Device的配置分为多个阶段进行

    1) 使用 XUsbPs_DeviceConfig 数据结构对需要的端点配置进行设置. 包括端点数量, 每个端点的传输描述符(Transfer Descriptor), 以及 Out(Rx)端点的buffer大小, 每个端点的Buffer可以设置为不同大小.

    2) 使用 XUsbPs_DeviceMemRequired() 请求所需的可DMA的(DMAable)内存大小.

    3) 分配DMAable内存且设置XUsbPs_DeviceConfig数据结构中的DMAMemVirt和DMAMemPhys成员变量.  

    4) 通过调用XUsbPs_ConfigureDevice()完成对 DEVICE 侧的控制器设置.
     */

    /*
    对于此例子, 我们只配置了 Ep0 和 Ep1 

     */
    DeviceConfig.EpCfg[0].Out.Type        = XUSBPS_EP_TYPE_CONTROL;
    DeviceConfig.EpCfg[0].Out.NumBufs    = 2;
    DeviceConfig.EpCfg[0].Out.BufSize    = 64;
    DeviceConfig.EpCfg[0].Out.MaxPacketSize    = 64;
    DeviceConfig.EpCfg[0].In.Type        = XUSBPS_EP_TYPE_CONTROL;
    DeviceConfig.EpCfg[0].In.NumBufs    = 2;
    DeviceConfig.EpCfg[0].In.MaxPacketSize    = 64;

    DeviceConfig.EpCfg[1].Out.Type        = XUSBPS_EP_TYPE_BULK;
    DeviceConfig.EpCfg[1].Out.NumBufs    = 16;
    DeviceConfig.EpCfg[1].Out.BufSize    = 512;
    DeviceConfig.EpCfg[1].Out.MaxPacketSize    = 512;
    DeviceConfig.EpCfg[1].In.Type        = XUSBPS_EP_TYPE_BULK;
    DeviceConfig.EpCfg[1].In.NumBufs    = 16;
    DeviceConfig.EpCfg[1].In.MaxPacketSize    = 512;

    DeviceConfig.NumEndpoints = NumEndpoints;

    MemPtr = (u8 *)&Buffer[0];
    memset(MemPtr,0,MEMORY_SIZE);
    Xil_DCacheFlushRange((unsigned int)MemPtr, MEMORY_SIZE);

    /* 完成DeviceConfig结构的配置和配置器件端的Controller.
     */
    DeviceConfig.DMAMemPhys = (u32) MemPtr;

    Status = XUsbPs_ConfigureDevice(UsbInstancePtr, &DeviceConfig);
    if (XST_SUCCESS != Status) {
        goto out;
    }

    /* Set the handler for receiving frames. */
    Status = XUsbPs_IntrSetHandler(UsbInstancePtr, UsbIntrHandler, NULL,
                        XUSBPS_IXR_UE_MASK);
    if (XST_SUCCESS != Status) {
        goto out;
    }

    /* 为 Ep0 设置中断处理函数, 以处理 PC 发送的 SetUp 包(一般不动这里)
     */
    Status = XUsbPs_EpSetHandler(UsbInstancePtr, 0,
                XUSBPS_EP_DIRECTION_OUT,
                XUsbPs_Ep0EventHandler, UsbInstancePtr);

    /* 为 Ep1 设置中断处理函数, 以处理 PC 发送的 BULK 传输

    在此例子中我们没有为 TX(In) 设置相应的中断处理函数(不代表不会触发中断,只是说PS的USB发送数据完毕后,PS不做特殊处理), 
     */
    Status = XUsbPs_EpSetHandler(UsbInstancePtr, 1,
                XUSBPS_EP_DIRECTION_OUT,
                XUsbPs_Ep1EventHandler, UsbInstancePtr);

    /* 使能USB器件的中断(和在中断控制器中加入中断要区分开) */
    XUsbPs_IntrEnable(UsbInstancePtr, XUSBPS_IXR_UR_MASK |
                       XUSBPS_IXR_UI_MASK);


    /* 启动 USB  */
    XUsbPs_Start(UsbInstancePtr);

    /* 
    等待USB插入
     */
    while (NumReceivedFrames < 1) {
        /* NOP */
    }


    /* 
    返回USB中断初始化是否成功
     */
    ReturnStatus = XST_SUCCESS;

out:
    /* 
    Clean Up. 即使为使能或者设置USB,也应当在初始化失败的情况下Clean up和disable中断以及disable中断子系统.
     */
    XUsbPs_Stop(UsbInstancePtr);
    XUsbPs_IntrDisable(UsbInstancePtr, XUSBPS_IXR_ALL);
    (int) XUsbPs_IntrSetHandler(UsbInstancePtr, NULL, NULL, 0);

    UsbDisableIntrSystem(IntcInstancePtr, UsbIntrId);

    /* 
    释放分配的内存.
     */
    if (NULL != UsbInstancePtr->UserDataPtr) {
        free(UsbInstancePtr->UserDataPtr);
    }
    return ReturnStatus;
}

```  

## 2 总结  

下次写.  
