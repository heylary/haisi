# 🚗基于AI视觉的隧道智慧消防系统设计 🚗

综合海思sample案例，在Hi3516DV300 SDK的基础之上进行开发，主要基于训练好的车辆wk模型在板端进行部署，充分发挥海思IVE、NNIE硬件加速能力，完成AI推理和业务处理。
## 📝 项目简介

隧道车辆检测系统是一个基于 AI 视觉的智能消防系统，旨在实现对隧道内车辆的实时监测和预警。通过结合 Pegasus 套件和 Taurus 套件的先进技术，该系统能够实现对传统火灾报警设备的物联，并通过 AI 视频提供更准确和可靠的隧道车辆检测服务。在Hi3516DV300 SDK基础上进行开发，在利用媒体通路的基础上，通过捕获VPSS帧进行预处理操作，并送至NNIE进行推理，结合AI CPU算子最终得到AI Flag并进行相应业务处理，运用到媒体理论、多线程、IPC通信、IVE、NNIE等思想，海思媒体处理平台主要分为视频输入（VI）、视频处理（VPSS）、视频编码（VENC）、视频解码（VDEC）、视频输出（VO）等模块



## 🎯 主要功能

- 实时监测隧道内的车辆
- 物联网集成，传统火灾报警设备联动(待实现)

## 🌟 项目架构

```
│ BUILD.gn                    # 编译ohos ai_sample需要的gn文件
├─ai_infer_process             # AI前处理、推理、后处理相关接口
├─dependency                   # ai sample依赖的一些功能，如语音播报
├─ext_util					 # 常用的基础接口、可移植操作系统接口posix等
├─mpp_help        		     # 封装的媒体相关接口
├─scenario
│   ├── car_detect
│   │   ├── yolov2_car_detect.c
│   │   └── yolov2_car_detect.h
└─smp			            # ai sample主入口及媒体处理文件
```
ai_infer_process 主要为AI前处理、推理、后处理相关接口，ext_util为常用的基础接口、可移植操作系统接口posix等，mmp为封装的媒体相关接口，scenario内包含我们的作品，smp为主入口及媒体处理文件；

### Vi部分
调用海思官方sdk，需要先获取sensor的参数，通过配置SAMPLE_COMM_VI_GetSensorInfo获取摄像头信息
```c++sensor配置
HI_VOID SAMPLE_COMM_VI_GetSensorInfo(SAMPLE_VI_CONFIG_S *pstViConfig)//获取摄像头信息
{
    HI_S32 i;

    for (i = 0; i < VI_MAX_DEV_NUM; i++) {
        pstViConfig->astViInfo[i].stSnsInfo.s32SnsId = i;
        pstViConfig->astViInfo[i].stSnsInfo.s32BusId = i;
        pstViConfig->astViInfo[i].stSnsInfo.MipiDev = i;
        (hi_void)
            memset_s(&pstViConfig->astViInfo[i].stSnapInfo, sizeof(SAMPLE_SNAP_INFO_S), 
            0, sizeof(SAMPLE_SNAP_INFO_S));
        pstViConfig->astViInfo[i].stPipeInfo.bMultiPipe = HI_FALSE;
        pstViConfig->astViInfo[i].stPipeInfo.bVcNumCfged = HI_FALSE;
    }

    pstViConfig->astViInfo[0].stSnsInfo.enSnsType = SENSOR0_TYPE;
    pstViConfig->astViInfo[1].stSnsInfo.enSnsType = SENSOR1_TYPE;
}
```
 
配置vi还需要配置初始化vi配置，设置vi channel，配置视频缓冲池等操作，部分实现细节如下所示
```c++
/* init ViCfg */
void ViCfgInit(ViCfg* self)
{
    HI_ASSERT(self);
    if (memset_s(self, sizeof(*self), 0, sizeof(*self)) != EOK) {
        HI_ASSERT(0);
    }

    SAMPLE_COMM_VI_GetSensorInfo(self);
    self->s32WorkingViNum = 1;
    self->as32WorkingViId[0] = 0;

    self->astViInfo[0].stSnsInfo.MipiDev =
        SAMPLE_COMM_VI_GetComboDevBySensor(self->astViInfo[0].stSnsInfo.enSnsType, 0);
    self->astViInfo[0].stSnsInfo.s32BusId = 0;
}

/* Set up the VI channel */
void ViCfgSetChn(ViCfg* self, int chnId, PIXEL_FORMAT_E pixFormat,
    VIDEO_FORMAT_E videoFormat, DYNAMIC_RANGE_E dynamicRange)
{
    HI_ASSERT(self);
    HI_ASSERT((int)pixFormat < PIXEL_FORMAT_BUTT);
    HI_ASSERT((int)videoFormat < VIDEO_FORMAT_BUTT);
    HI_ASSERT((int)dynamicRange < DYNAMIC_RANGE_BUTT);

    self->astViInfo[0].stChnInfo.ViChn = chnId;
    self->astViInfo[0].stChnInfo.enPixFormat =
        (int)pixFormat < 0 ? PIXEL_FORMAT_YVU_SEMIPLANAR_420 : pixFormat;
    self->astViInfo[0].stChnInfo.enVideoFormat =
        (int)videoFormat < 0 ? VIDEO_FORMAT_LINEAR : videoFormat;
    self->astViInfo[0].stChnInfo.enDynamicRange =
        (int)dynamicRange < 0 ? DYNAMIC_RANGE_SDR8 : dynamicRange;
}
```
 

### VPSS部分
通过调用系统接口，可将VI和VO等模块进行绑定，其中前者为VPSS的输入源，后者为VPSS的接收者，如图2-9所示，后续仍需设置VPSS group以及设置VPSS channel等操作.

### VO部分
Hi3516DV300支持的显示/回写设备，实现步骤包括配置mipi屏幕参数，获取屏幕高度和宽度，start mipi channel等操作，
 
### AI部分
前向推理部分位于ai_infer_process文件中，本作品的ai推理部分主要设计到yolov2部分，具体代码包含实现步骤 
1)	Yolov2 init，包括硬件参数初始化，软件参数初始化
2)	Creat yolo2 model basad mode file，包括设置配置参数，加载yolov2模型，初始化yolov2模型参数
3)	Destory yolo2 model 
4)	Fetch result，获取模型推理结果
5)	Calculation yuv image，包括NNIE处理，软件参数处理等
