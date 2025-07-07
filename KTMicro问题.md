1. 打印查看
  `预留Uart接口，通过串口打印，串口打印的波特率为921600`
2. GPIO 功能配置  单击控制播放暂停
3. GPIO 功放功能，初始化配置输出
4. 线控配置
```c
在bll_autop_os.c中的void autop_keys_init(void){}函数中，可以修改按键的阈值
具体的阈值需要通过GPIO3进行串口打印
具体打印语句写在autop_task_handle(void *param){}函数中，LOG_INFO("autop_keys_get_code_handle"\n);
```
1. 功能：开关sidetone，mic on/off，speaker on/off，EQ（DAC,ADC）切换
```c
1.speaker mute/unmute 
 在bll_audio_os.c定义标志变量
 然后在case MSG_DAC_PROC{} 中 执行
 memset(p_st_input->pp_v_buf[0],0,u16_data_num*4); //两个声道，两个数据流
 memset(p_st_input->pp_v_buf[1],0,u16_data_num*4);

2.mic mute/unmute 
 同上，只不过是在case MSG_ADC_PROC{}中 执行的
 memset(p_st_input->pp_v_buf[0],0,u16_data_num*4);

3.sidetone on/off
//打开耳返 bll_autop_os.c
 hal_dac_sidetone_cfg_t st_sidetone;
 hal_dac_sidetone_get(&st_sidetone);
 st_sidetone.u8_en = 1;
 hal_dac_sidetone_set(&st_sidetone);
//关闭耳返
 hal_dac_sidetone_cfg_t st_sidetone;
 hal_dac_sidetone_get(&st_sidetone);
 st_sidetone.u8_en = 0;
 hal_dac_sidetone_set(&st_sidetone);
//或者 customdefines.c中
#define USB_SIDE_TONE_POSITION                  (0) //也可以控制耳返，上电默认打开

4.EQ切换函数  ADC EQ切换
user.c
 void user_dac_eq_coeff_switch(uint8_t para_eq_ind)
{
    memcpy((void *)&g_uCfgReg.play_eq_para.eq_para[0], \
    (void *)&g_stUserDacEqCoeffTab[para_eq_ind], 6*sizeof(apl_eq_param_t));
    eq_update_chanel(2);
}
可以配合提示音在bll_autop_os.c中keyhandle使用

 
```
1. GPIO口实现按键功能
```c
在user.c 中的 void user_init(void){}进行函数定义
```
1. IO口控制功放
```c
将IO口配置为输出即可
```
1. 提示音功能，并且闪避（DAC无输出时提示音不播放）
```c
通过脚本，将提示音打包进sdk，然后使用该函数来进行播放提示音
p_wavInfo->wavStartPlay(eq_index);
在wav.c 和 wav.h 中增加响应的数据，修改枚举类型和数组大小即可

并且bll_audio_os.c    //msg_dac
//获取提示音的采样率
stUSB_Audio_Attr *pattr;
pattr = Usb_Get_Usb_Audio_pAttr(USB_AUDIO_OUT);
p_wavInfo->wavPlayer(p_st_output->pp_v_buf[0], p_st_output->pp_v_buf[1], u16_data_num, pattr->u32SamFreq/1000);
```
1. 无MIC / 无线控 / 无Speaker
```c
无mic 和 无Speaker 可以在customdefines.h 中对宏定义进行修改
#define PTYPE_SPEAKER       (1)
#define PTYPE_MICPHONE      (2)
#define PTYPE_HEADSET       (3)
#define PRODUCT_TYPE        (PTYPE_HEADSET) /* 1: speak, 2: micphone, 3: headset */

线控功能 bll_autop_os.c
void autop_micin_func_cb(void)
{
    // BaseType_t xHigherPriorityTaskWoken = pdFALSE;
    // uint8_t msg = TOP_MSG_KEY_PROC;
    // xQueueSendFromISR(s_st_top_msg_queue, &msg, &xHigherPriorityTaskWoken);
    // portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
    //线控功能
}
把中间具体操作注释掉
```
1. 极性修改
```c
将声道的数据流反向
                {
                    uint16_t u16Cnt;
                    int32_t *p_dat_buf = (int32 *)(p_st_output->pp_v_buf[0]);
                    int32_t *p_dat_buf1 = (int32 *)(p_st_output->pp_v_buf[1]);
                    for(u16Cnt = 0; u16Cnt < u16_data_num; u16Cnt++){
                        p_dat_buf[u16Cnt] = -p_dat_buf[u16Cnt];
                        p_dat_buf1[u16Cnt] = -p_dat_buf1[u16Cnt];
                    }
                }
写在bll_audio_os.c 的 msg_dac_proc中
```
1. 左右声道交换 
```c
左右声道交换类似
                {
                    uint16_t u16Cnt;
                    int32_t *p_dat_buf = (int32 *)(p_st_output->pp_v_buf[0]);
                    int32_t *p_dat_buf1 = (int32 *)(p_st_output->pp_v_buf[1]);
                    for(u16Cnt = 0; u16Cnt < u16_data_num; u16Cnt++){
                        p_dat_buf[u16Cnt] = p_dat_buf1[u16Cnt];
                        p_dat_buf1[u16Cnt] = p_dat_buf[u16Cnt];
                    }
                }
```
2. PC端切歌功能 
```c
devicedefines.h
#define PC_KEYS (0/1)
```
1. 调整speaker和mic的采样率以及位深
```c
 位深和采样率在customdefines.h 中定义
#define PLAY_BIT_CFG                (0b111)  
#define REC_BIT_CFG                 (0b001)  

#define PLAY_SAMPLE_RATE            (0xff)  
#define REC_SAMPLE_RATE             (0x10)
BIT位深参数 111 表示，32bit 24bit 16bit 均开启
SAMPLE采样率参数 FF 表示1111 1111. 代表开启响应的采样率 384000   192000   96000   48000  352800  176400  88200  44100
```
 在soft/application/metis_flash_uac/module/Makefile 
         ``        platform/driver/usb \
          `   application/${KT_TARGET}/user \   //增加这一行

如果编译失败，将build文件夹整个删除下，然后再进行操作。

--- 
---
H20 
1. 修改国标、美规、AUX。插拔检测
`ucfg.h       #define USER_EARPHONE_INS_REM_DET_EN            (1) //插入检测`
2. 按键阻值匹配

