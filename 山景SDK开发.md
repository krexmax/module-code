#### 调音工具导出
1. 根据音效框图里的source和sink修改删减effect_mode.c
2. 导出来roboeffect_config.h文件只替换到SDK中上半段的宏定义音效使能开关，中段的宏开关和下段的函数不要动，并且sdk自带的头文件不要去掉
3. 如果导出来的user_effect_flow_Karaoke.c文件比较短，没有param代码，则在ACPworkBench中，将版本设置为2.23.3，再重新导出，这样就不需要导出param文件，只需要导出.c .h文件
4. 导出的user_effect_flow_Karaoke.c和 user_effect_flow_Karaoke.h直接替换掉原本的代码
5. 打开system_config.h里的Karaoke的使能
6. 删除effect_mode_config.c里面不使用的模式的定义

#### 按键控制音效_1
1. _user_effect_parameter.h_ 的枚举类型中增加枚举变量
~~~ C
typedef enum _AUDIOEFFECT_EFFECT_CONTROL
{
	EQ_MODE_ADJUST = 0,
	MIC_BASS_ADJUST,
	MIC_TREB_ADJUST,
	MUSIC_BASS_ADJUST,
	MUSIC_TREB_ADJUST,
	MIC_VOLUME_ADJUST,
	MUSIC_VOLUME_ADJUST,
	ECHO_PARAM,
	MIC_SILENCE_DETECTOR_PARAM,
	MUSIC_SILENCE_DETECTOR_PARAM,
	_3D_ENABLE,
	APPMODE_PREGAIN,

	MONITOR_ON, //新增加的枚举变量，一定不能放在最下面

	AUDIOEFFECT_EFFECT_CONTROL_MAX, //该变量一定要作为最底下的变量
}AUDIOEFFECT_EFFECT_CONTROL;

~~~
2. _effect_mode.c_ 中的数组增加数组变量
~~~ C
const uint8_t karaoke_effect_ctrl[AUDIOEFFECT_EFFECT_CONTROL_MAX] =
{
	[EQ_MODE_ADJUST] = KARAOKE_eq0_ADDR,
	[_3D_ENABLE] = KARAOKE_3D_ADDR,
	[ECHO_PARAM] = KARAOKE_echo0_ADDR,
	[MUSIC_VOLUME_ADJUST] = KARAOKE_gain_control0_ADDR,
	[MIC_VOLUME_ADJUST] = KARAOKE_gain_control1_ADDR,
	[MIC_SILENCE_DETECTOR_PARAM] = KARAOKE_silence_detector0_ADDR,

	[MONITOR_ON] = KARAOKE_gain_control5_ADDR,
	//数组变量名是在枚举中定义的名字，值是对照音效框图填写
};

~~~
3. _app_message.h_ 中增加message id
~~~ C 
typedef enum 
{
///***
	MSG_MIC_BASS_UP,
	MSG_MIC_BASS_DW,
	MSG_MUSIC_TREB_UP,
	MSG_MUSIC_TREB_DW,
	MSG_MUSIC_BASS_UP,
	MSG_BT_CONNECT_CTRL,   
	MSG_BT_CLEAR_PAIRED_LIST, 
	MSG_BT_CONNECT_MODE,
	MSG_BT_OPEN_ACCESS,
	MSG_RGB_MODE,

	MSG_MONITOR, //自定义的message id 
///***

} MessageId;
~~~

4. _key.c_ 中处理按键触发时发送的message
~~~ C
//adc1 key

	{MSG_NONE,						MSG_PLAY_PAUSE,   				MSG_DEEPSLEEP,      		MSG_NONE,                        MSG_NONE},
	{MSG_NONE,						MSG_PRE,          				MSG_FB_START,               MSG_FB_START,              MSG_FF_FB_END},
	{MSG_NONE,						MSG_NEXT,	        			MSG_FF_START,               MSG_FF_START,              MSG_FF_FB_END},
	{MSG_NONE,						MSG_MUSIC_VOLDOWN,	    	MSG_MUSIC_VOLDOWN,   		     MSG_MUSIC_VOLDOWN,			     MSG_NONE},
	{MSG_NONE,						MSG_MONITOR,//单击adckey1 K22 MSG_MUSIC_VOLUP,     		     MSG_MUSIC_VOLUP,     		     MSG_NONE},
	{MSG_NONE,						MSG_EQ,	        				MSG_3D,        				MSG_NONE,                   MSG_NONE},
	{MSG_NONE,						MSG_MUTE,         				MSG_VB,        				MSG_NONE,                   MSG_NONE},
	{MSG_NONE,						MSG_EFFECTMODE,   				MSG_VOCAL_CUT, 				MSG_NONE,                   MSG_NONE},
~~~
5. _audio_msg_process.c_ 中处理获取到的message
~~~ C
uint8_t AudioCommonMsgProcess(uint16_t Msg)
{
	uint8_t 	refresh_addr = 0;

	switch(Msg)
	{
	case MSG_MONITOR:
		refresh_addr = GetEffectControlIndex(MONITOR_ON);
		APP_DBG("MSG_MONITOR     %d\n",refresh_addr);//打印
		if(refresh_addr)
		{
			bool enable = roboeffect_get_effect_status(AudioEffect.context_memory, refresh_addr);
			AudioEffect_effect_enable(refresh_addr, !enable);
		}
				break; //进行按键单击控制音效的打开与关闭
~~~
#### 按键控制音效_2
0. 以控制gain_control5 的mute和gain参数为例
1. 修改文件版本`"\\MVsB5_BT_Audio_SDK_V0.7.1\BT_Audio_APP\app_src\components\audio\effect_control\run.bat"`中，把路径中参数roboeffect_library_info改为v2.23.3
2. 修改文件名称`D:\1_Development\3_MountainView\2_SDK\MVsB5_BT_Audio_SDK_V0.7.1\BT_Audio_APP\app_src\components\audio\music_parameter\karaoke`中的user_effect_flow_karaoke.c中要把该文件名中的Karaoke改为karaoke（不知道原理是什么）
3. 打开文件`"D:\1_Development\3_MountainView\2_SDK\MVsB5_BT_Audio_SDK_V0.7.1\BT_Audio_APP\app_src\components\audio\effect_control\params_msg_mapping.xlsx"`增加最后一行并设置参数为
   MSG_MONITOR        消息名称
   UP_LOOP                  循环模式（切换至向数组大值，并且循环）
   <0,1>                          模式数组大小为2
   gain5_control              随便设置的变量名称，没特殊意义
   karaoke                      消息作用的框图，karaoke小写不要修改
   gain_control5              消息实际作用的音效参数地址，需要与音效对应
   \[ALL\]                         固定配置，与<0,1>相配合
4. bat文件和excel文件配置好后，点击bat文件执行，会自动生成auto_gen_msg_process.c文件，该文件内处理了消息函数
~~~ C
int16_t monitor_data_0[] = {0 , 10*10};
int16_t monitor_data_1[] = {1 , 0}; 
//代码的上面会自动生成数组但是没有定义，直接把数组定义出来
static const int16_t *monitor_table[] = {
	monitor_data_0,
	monitor_data_1,
};
~~~
#### ADCKEY按键汇总
按键端口打印出来的KeyMsg(1, 0x7F82, 15, 2);
其中四个数值分别是msg.source, KeyMsg, msg.index, msg.type
msg.source表示按键输入的类型adkey = 1, IRkey = 2, IOkey = 3, ADC_level key = 4, code key = 5
KeyMsg表示MsgId的数值，判断会触发哪个消息
msg.index表示索引号，15就表示是ADKEY_TAB[][]二维数组中的第15个元素
msg.type表示按键是长按还是短按，配合msg.index使用

#### 新增Karaoke模式
1. _effect_mode.c_ 中增加 _`_const AUDIOEFFECT_EFFECT_PARA **** _effect_para`_
2. _user_effect_flow_Karaoke.c_ 中 _`char *parameter_group_name_Karaoke[] = {"", ""},`_ 增加需要的模式名称，并且增加两个音效参数数组
   _`const unsigned char user_effect_parameters_Karaoke_****[] = {};`_
   _`const unsigned char user_module_parameters_Karaoke_****[] = {};`_
   AcpWorkBench导出文件直接导入就行
3. _`user_effect_flow_Karaoke.h`_ 中
   _`extern const unsigned char user_effect_parameters_Karaoke_****[];`_
   _`extern const unsigned char user_module_parameters_Karaoke_****[];`_
   AcpWorkBench导出文件直接复制粘贴
4. _`effect_mode_config.c`_ 中
   _`extern const AUDIOEFFECT_EFFECT_PARA HunXiang_effect_para;`_ 以及
   _`{EFFECT_MODE_HunXiang,	EffectModeStateReady,	&karaoke_mode,	&HunXiang_effect_para,	karaoke_effect_ctrl,	msg_process_karaoke},`_
5. _`ctrlvars.h`_ 中
~~~ c
typedef enum _EFFECT_MODE
{
	EFFECT_MODE_DEFAULT = 0, 
	EFFECT_MODE_MIC, 
	EFFECT_MODE_MICUSBAI,
	EFFECT_MODE_BYPASS,
    EFFECT_MODE_HunXiang,
    
	EFFECT_MODE_****,

	EFFECT_MODE_COUNT,
} EFFECT_MODE;
~~~
6. 注意模式名称的大小写

#### 音效图参数
例如：
0xaa, /\*gain_control5\*/  gain_control5的地址
0x05,/\*length\*/              长度
0x01, /\*enable\*/            是否使能参数范围（0 | 1）
0x00, 0x00, /\*mute\*/     是否静音，只有前面0x00有作用，参数范围（0x00 | 0x01）
0x4c, 0x02, /\*gain\*/     AcpWorkBench中设置为5.88，当作588，dec588 = hex 24C，2 4C分段存储

音效图导入进入编译烧录后，开发板CPU占用率过高，录音声不正确，并且在此基础上打开混响或3D音效，dac输出也会不正常
//gain_control5功能一直反复开关，导致占用率高，重新修改功能
//排查发现是因为MSG_MONITOR消息放置位置靠前导致出现错误
#### USB基础信息
VID PID `otg_device_standard_request.h` 中的宏定义修改，注意PID数值与实际有偏差，一般情况下是3的偏差

ProductName  ManufactureName  SerialNumber
`otg_device_standard_request.c`中修改
```C
const char *gDeviceProductString ="RY912-GM";			    //max length: 32bytes
const char *gDeviceString_Manu ="RYDZ";						//max length: 32bytes
const char *gDeviceString_SerialNumber ="202504141621";	    //max length: 32bytes
```



#### 小记
mode_task_api.c     void CommonMsgProccess(uint16_t Msg){}根据消息切换模式
打开gpio功能
```c
/**GPIO按键**/
#define CFG_RES_IO_KEY_SCAN
```
io口具体配置sys_gpio.h
```c
//------GPIO KEY检测IO选择------------------------------------------------//
#ifdef CFG_GPIO_KEY1_EN
#define GPIO_KEY1_PORT             A
#define GPIO_KEY1                  GPIO_INDEX16  A16 配置A16口
#endif

```
io触发的消息按键在key.c中的iokey数组中配置，但是第0位数组元素是无法使用的

新增加的iokey要在以下地方进行配置
```c
iokey.c
const uint32_t GPIOKey_Init_Tab[][5] =
{
	#ifdef CFG_SOFT_POWER_KEY_EN
	{POWER_KEY_OE,POWER_KEY_IE,POWER_KEY_PU,POWER_KEY_PD,POWER_KEY_PIN},
	#endif
	#ifdef CFG_GPIO_KEY1_EN
	{GPIO_KEY1_OE,GPIO_KEY1_IE,GPIO_KEY1_PU,GPIO_KEY1_PD,GPIO_KEY1},
	{GPIO_KEY2_OE,GPIO_KEY2_IE,GPIO_KEY2_PU,GPIO_KEY2_PD,GPIO_KEY2},
	#endif
	#ifdef CFG_GPIO_KEY2_EN
	{GPIO_KEY2_OE,GPIO_KEY2_IE,GPIO_KEY2_PU,GPIO_KEY2_PD,GPIO_KEY2},
	#endif	   
};

static uint8_t IOChannelKeyGet(void){
if(!GPIO_RegOneBitGet(GPIO_KEY1_IN,GPIO_KEY1))
	{
		KeyIndex = 1;
	}
	if(!GPIO_RegOneBitGet(GPIO_KEY2_IN,GPIO_KEY2))
		{
			KeyIndex = 2;
		}
}

sys_gpio.h
//------GPIO KEY检测IO选择------------------------------------------------//
#ifdef CFG_GPIO_KEY1_EN
#define GPIO_KEY1_PORT             A
#define GPIO_KEY1                  GPIO_INDEX16  A16 配置A16口
#endif

#ifdef CFG_GPIO_KEY2_EN
#define GPIO_KEY2_PORT             A
#define GPIO_KEY2                  GPIO_INDEX26
#endif

 //------GPIO KEY检测IO寄存器配置------------------------------------------------// 
#define GPIO_KEY1_IE               STRING_CONNECT(GPIO,GPIO_KEY1_PORT,GPIE)        
#define GPIO_KEY1_OE               STRING_CONNECT(GPIO,GPIO_KEY1_PORT,GPOE) 
#define GPIO_KEY1_PU               STRING_CONNECT(GPIO,GPIO_KEY1_PORT,GPPU) 
#define GPIO_KEY1_PD               STRING_CONNECT(GPIO,GPIO_KEY1_PORT,GPPD)        
#define GPIO_KEY1_IN               STRING_CONNECT(GPIO,GPIO_KEY1_PORT,GPIN)        
#define GPIO_KEY1_OUT              STRING_CONNECT(GPIO,GPIO_KEY1_PORT,GPOUT) 


#define GPIO_KEY2_IE               STRING_CONNECT(GPIO,GPIO_KEY2_PORT,GPIE)
#define GPIO_KEY2_OE               STRING_CONNECT(GPIO,GPIO_KEY2_PORT,GPOE)
#define GPIO_KEY2_PU               STRING_CONNECT(GPIO,GPIO_KEY2_PORT,GPPU)
#define GPIO_KEY2_PD               STRING_CONNECT(GPIO,GPIO_KEY2_PORT,GPPD)
#define GPIO_KEY2_IN               STRING_CONNECT(GPIO,GPIO_KEY2_PORT,GPIN)
#define GPIO_KEY2_OUT              STRING_CONNECT(GPIO,GPIO_KEY2_PORT,GPOUT)
```

appconfig.h
//	#define CFG_IDLE_MODE_POWER_KEY	//power key
//	#define CFG_IDLE_MODE_DEEP_SLEEP //deepsleep 关闭
还有蓝牙模式、光纤同轴的模式不需要的也关闭掉
Idle模式不能关闭，也不需要关闭，关闭板子会不识别

volup实现不同模式下不同的功能，例如normal模式下音量的加减，hunxiang模式下耳返大小的改变，首先将其他函数中的msg volup消息接收给注释关闭掉（usb_audio_mode.c、audio_msg_process.c），然后在自动生成的auto_gen_msg_process.c中的switch接收    msg_volup（excel中自定义）并进行处理。case语句中首先使用if语句进行判断当前的模式
```c
要加#Include “main_task.h”
#include "remind_sound.h"
		case MSG_MUSIC_VOLDOWN:
					if(mainAppCt.EffectMode == EFFECT_MODE_Normal){
						PCAudioVolDn();
						break;
					}
					else if(mainAppCt.EffectMode == EFFECT_MODE_sevenone){
						PCAudioVolDn();
						break;
					}
					else{
						flush_address = karaoke_gain_control5_vol_down_loop_msg_music_voldown_ALL();
						break;
					}
		case MSG_MUSIC_VOLUP:
					if(mainAppCt.EffectMode == EFFECT_MODE_Normal){
						PCAudioVolUp();
						break;
					}
					else if(mainAppCt.EffectMode == EFFECT_MODE_sevenone){
						PCAudioVolUp();
						break;
					}
					else{
						flush_address = karaoke_gain_control5_vol_up_loop_msg_music_volup_ALL();
						break;
					}

不同挡位提示音
case MSG_MUSIC_VOLDOWN:
			if(mainAppCt.EffectMode == EFFECT_MODE_Normal){
				PCAudioVolDn();
				break;
			}
			else if(mainAppCt.EffectMode == EFFECT_MODE_sevenone){
				PCAudioVolDn();
				break;
			}
			else{
				flush_address = karaoke_gain_control5_vold_up_loop_msg_music_voldown_ALL();

				if(vold == 1)
					RemindSoundServiceItemRequest(SOUND_REMIND_3, REMIND_PRIO_NORMAL);
				else if(vold == 2)
					RemindSoundServiceItemRequest(SOUND_REMIND_4, REMIND_PRIO_NORMAL);
				else
					RemindSoundServiceItemRequest(SOUND_REMIND_5, REMIND_PRIO_NORMAL);
				break;
			}
```

chip_config.h 选择正确的软件芯片

```c
mode_task_api.c 中切换模式并播放提示音
case MSG_EFFECTMODE:
			EffectMode = mainAppCt.EffectMode;
			if(EffectMode == EFFECT_MODE_Normal){
				RemindSoundServiceItemRequest(SOUND_REMIND_0, REMIND_PRIO_NORMAL);
			}
			else if(EffectMode == EFFECT_MODE_HunXiang){
				RemindSoundServiceItemRequest(SOUND_REMIND_1, REMIND_PRIO_NORMAL);
			}
			else
				RemindSoundServiceItemRequest(SOUND_REMIND_2, REMIND_PRIO_NORMAL);
```

Aio7 设备不同模式下，初始时的耳返开关
例如ktv模式默认打开耳返，所以第一次按下是耳返关，
而原声模式和7.1模式默认关闭耳返，所以第一次按下是耳返开。

控制耳返开关的数值是
```c
autogenmsgprocess.c
extern int16_t monitor;

static uint8_t karaoke_gain_control5_monitor_up_loop_msg_monitor_ALL(void)
{
	//param set type is LIST
	if(++monitor >= 2) monitor = 0; // monitor数值在01之间切换
	roboeffect_set_effect_parameter(AudioEffect.context_memory, KARAOKE_gain_control5_ADDR, 255, monitor_table[monitor]);
	return (uint8_t)KARAOKE_gain_control5_ADDR;
}


case MSG_MONITOR:
	
	
	flush_address = karaoke_gain_control5_monitor_up_loop_msg_monitor_ALL();
//可以看到控制耳返开关的变量是monitor，monitor的数值与提示音进行绑定，可以不需要关注提示音与耳返开关的对应关系，将monitor的定义放在maintaskapi.c中，也就是将monitor的数值与切换后的模式进行绑定
	if(monitor == 1) // 1--耳返开 0--耳返关
		RemindSoundServiceItemRequest(SOUND_REMIND_ERFANKAI,REMIND_PRIO_NORMAL);
	else
		RemindSoundServiceItemRequest(SOUND_REMIND_ERFANGUA,REMIND_PRIO_NORMAL);
	break;

mode_task_api.c
int16_t monitor = 1;//因为初始模式是ktv模式，所以耳返开关是要耳返关->耳返开

case MSG_EFFECTMODE:
			EffectMode = mainAppCt.EffectMode;// normal hunxing sevenone
			if(EffectMode == EFFECT_MODE_Normal){
				RemindSoundServiceItemRequest(SOUND_REMIND_KTVMOSHI, REMIND_PRIO_NORMAL);
				monitor = 1;
			}
			//按下按键后，播放KTVmode的提示音，并切换到ktv模式，所以耳返是关->开，因此
			将monitor设置为1，其余两个模式就设置为0.其余两个模式设置为0目的是为了去除记忆模式
			不然sevenone的耳返开，直接切换到normal时按下耳返开关效果是耳返关。
			else if(EffectMode == EFFECT_MODE_HunXiang){
				RemindSoundServiceItemRequest(SOUND_REMIND_7DIAN1MO, REMIND_PRIO_NORMAL);
				monitor = 0;
			}
			else{
				RemindSoundServiceItemRequest(SOUND_REMIND_YUANSHEN, REMIND_PRIO_NORMAL);
				monitor = 0;
			}


```
设置上电默认模式
main_task.c
	mainAppCt.EffectMode = EFFECT_MODE_Nromal;

关于耳返大中小三档适配的功能，如果三种模式都有耳返开关，那么就直接判断monitor的值，来看耳返是否开关就可以。如果耳返关闭状态monitor=0，那么pcaudiovolup()，反之就直接执行耳返大中小的逻辑，并播放对应的提示音。下面是正确逻辑
```c

#include "remind_sound.h"
#include "main_task.h"


static uint8_t karaoke_gain_control5_vold_up_loop_msg_music_voldown_ALL(void)
{
	//param set type is LIST
	if(++vold >= 3) vold = 0;
	roboeffect_set_effect_parameter(AudioEffect.context_memory, KARAOKE_gain_control5_ADDR, 255, vold_table[vold]);
	return (uint8_t)KARAOKE_gain_control5_ADDR;
}
#endif /*CFG_FLOWCHART_KARAOKE_ENABLE*/

#ifdef CFG_FLOWCHART_KARAOKE_ENABLE
static uint8_t karaoke_gain_control5_volp_up_loop_msg_music_volup_ALL(void)
{
	//param set type is LIST
	if(--vold < 0) vold = 2;
	roboeffect_set_effect_parameter(AudioEffect.context_memory, KARAOKE_gain_control5_ADDR, 255, vold_table[vold]);
	return (uint8_t)KARAOKE_gain_control5_ADDR;
}

case MSG_MUSIC_VOLDOWN:
			if(monitor == 0){
				PCAudioVolDn();
			}
			else{
				flush_address = karaoke_gain_control5_vold_up_loop_msg_music_voldown_ALL();
				if(vold == 0)
				RemindSoundServiceItemRequest(SOUND_REMIND_ERFANDA,REMIND_PRIO_NORMAL);
				else if(vold == 1)
					RemindSoundServiceItemRequest(SOUND_REMIND_ERFANZHO,REMIND_PRIO_NORMAL);
				else
					RemindSoundServiceItemRequest(SOUND_REMIND_ERFANXIA,REMIND_PRIO_NORMAL);
			}
			break;
case MSG_MUSIC_VOLUP:
			if(monitor == 0)	PCAudioVolUp();
			else{
				flush_address = karaoke_gain_control5_volp_up_loop_msg_music_volup_ALL();
				if(vold == 0)
				RemindSoundServiceItemRequest(SOUND_REMIND_ERFANDA,REMIND_PRIO_NORMAL);
				else if(vold == 1)
					RemindSoundServiceItemRequest(SOUND_REMIND_ERFANZHO,REMIND_PRIO_NORMAL);
				else
					RemindSoundServiceItemRequest(SOUND_REMIND_ERFANXIA,REMIND_PRIO_NORMAL);
			}
			break;
```
上电提示音  可以用if(1)去除掉提示音
```c
usb_aduio_mode.c
//if(RemindSoundServiceItemRequest(SOUND_REMIND_SHENGKAM, REMIND_PRIO_NORMAL) == FALSE)
	if(1)
```

`#define		ADC_VAL(val)				(val*4096/330)  val是电压值，ADC_VAL(val)算出来的是具体的adc值`

PP 0          vol+ 0.186           vol-  0.406
(ADC_VAL(0) + ADC_VAL(18.6))/2,
(ADC_VAL(18.6) + ADC_VAL(40.6))/2,
假设上面两个公式结果分别为x,y
则x y分出了三个区间，
小于x的区间为pp键，x - y的区间为vol+键，大于y的区间为vol-键

IO口复用ADCKEY
```c
sys_gpio.h
/**ADC按键**/
#ifdef CFG_RES_ADC_KEY_SCAN
	#define CFG_RES_POWERKEY_ADC_EN		  //power key脚上adc key功能使能
		#define CFG_RES_ADC_KEY_PORT_CH2		ADC_CHANNEL_AD6_A31//配置GPIO口复用
		#define CFG_RES_ADC_KEY_CH2_ANA_EN		GPIO_A_ANA_EN
		#define CFG_RES_ADC_KEY_CH2_ANA_MASK	GPIO_INDEX31
	
	#if(CFG_CHIP_SEL==CFG_CHIP_BP1564A2)
		#define CFG_RES_ADC_KEY_PORT_CH1		ADC_CHANNEL_AD5_A30
		#define CFG_RES_ADC_KEY_CH1_ANA_EN		GPIO_A_ANA_EN
		#define CFG_RES_ADC_KEY_CH1_ANA_MASK	GPIO_INDEX30
		#define CFG_PARA_WAKEUP_GPIO_ADCKEY		WAKEUP_GPIOA30 //同步设置唤醒端口

		#define CFG_RES_ADC_KEY_PORT_CH2		ADC_CHANNEL_AD6_A31//配置GPIO口复用
		#define CFG_RES_ADC_KEY_CH2_ANA_EN		GPIO_A_ANA_EN
		#define CFG_RES_ADC_KEY_CH2_ANA_MASK	GPIO_INDEX31
	#endif
#endif //CFG_RES_ADC_KEY_SCAN
```
adc_key.c中
`#define     NORMAL_ADKEY                (0)    ///1 = 标准的ADKEY值处理，0 = 用户自定义ADKEY值处理`
设置为0，这样使用的是下面的数组参数
```c
#if NORMAL_ADKEY
	#define USER_ADCKEY_VAL_TAB	NULL
#else
	#define ADKEY_0    (ADC_VAL(0)+ADC_VAL(18.9))/2
	#define ADKEY_1    (ADC_VAL(18.9)+ADC_VAL(40.5))/2
	#define ADKEY_2    (ADC_VAL(40.5)+ADC_VAL(100))/2
//	#define ADKEY_3    (ADC_VAL(85)+ADC_VAL(110))/2
//	#define ADKEY_4    (ADC_VAL(110)+ADC_VAL(140))/2
//	#define ADKEY_5    (ADC_VAL(140)+ADC_VAL(165))/2
//	#define ADKEY_6    (ADC_VAL(165)+ADC_VAL(180))/2
//	#define ADKEY_7    (ADC_VAL(180)+ADC_VAL(220))/2
//	#define ADKEY_8    (ADC_VAL(220)+ADC_VAL(230))/2
//	#define ADKEY_9    (ADC_VAL(230)+ADC_VAL(260))/2
//	#define ADKEY_10   (ADC_VAL(260)+ADC_VAL(290))/2

	const uint16_t UserADKey_Tab[ADC_KEY_COUNT]=
	{
		ADKEY_0,
		ADKEY_1,
		ADKEY_2,
//		ADKEY_3,
//		ADKEY_4,
//		ADKEY_5,
//		ADKEY_6,
//		ADKEY_7,
//		ADKEY_8,
//		ADKEY_9,
//		ADKEY_10,
	};

	#define USER_ADCKEY_VAL_TAB	UserADKey_Tab
#endif

其中，0 18.5 40.5 是真实量出来的，而100是随便写的，目的就是做出三个区间范围
类似
pp      vol+        vol-
0       18.5        40.6 
0    x          y          z
0xyz切割出了三个区间，表示三种按键

并且要把PWR_ADCKEY_VAL_TAB给注释掉
const struct
{
	uint8_t 			channel;
	uint8_t 			key_count;
	uint8_t 			key_index_offset;
	const uint16_t * 	key_adc_val_tab;
} AdcKeyScanMap[] =
{
#ifdef CFG_RES_POWERKEY_ADC_EN
	//{ADC_CHANNEL_BK_ADKEY, 		PWR_ADCKEY_COUNT, 	0, 				PWR_ADCKEY_VAL_TAB},
#endif
#ifdef CFG_RES_ADC_KEY_PORT_CH1
	{CFG_RES_ADC_KEY_PORT_CH1,	ADC_KEY_COUNT,		ADC_KEY_COUNT,	USER_ADCKEY_VAL_TAB},
#endif
#ifdef CFG_RES_ADC_KEY_PORT_CH2
	{CFG_RES_ADC_KEY_PORT_CH2,	ADC_KEY_COUNT,		ADC_KEY_COUNT*2,USER_ADCKEY_VAL_TAB},
#endif
};


```
语音助手：
`首先，按下pp键时，会调用一次PCAudioPP()函数，函数嵌套到底就是`
```c
这个函数
bool OTG_DeviceAudioSendPcCmd(uint8_t Cmd)
{
	OTG_DeviceInterruptSend(0x01,&Cmd, 1,1000);
	Cmd = 0;
	OTG_DeviceInterruptSend(0x01,&Cmd, 1,1000);
	return TRUE;
}
```
Cmd初始值为BIT(6)，那么其实
发送OTG_DeviceInterruptSend(0x01,&Cmd, 1,1000);到手机让手机知道按下PP键，
	如果一直发送PP键就会让手机触发语音助手
发送OTG_DeviceInterruptSend(0x01,&Cmd, 1,1000);//cmd=0让手机知道释放了PP键

SHORT_RELEASE处理pp键的逻辑没有问题，因此不需要改动
长按触发是LONGPress 那么此时触发MSG_PLAYPAUSE2，来调用
OTG_DeviceInterruptSend(0x01,&Cmd, 1,1000) BIT(6),
并且松手后触发LONG_PRESS_RELEASE
将cmd=0后再次发送
OTG_DeviceInterruptSend(0x01,&Cmd,1,1000)，完成一次按键的触发
```C
Key.c
//KEY_PRESSED					SHORT_RELEASE                 LONG_PRESS              KEY_HOLD                     LONG_PRESS_RELEASE

//power adc key
{MSG_NONE,						MSG_PLAY_PAUSE,				MSG_PLAY_PAUSE2,  MSG_NONE,					    MSG_PLAY_PAUSE3},

usb_audio_mode.c
void UsbDevicePlayRun(uint16_t msgId)
{
	switch(msgId)
	{
		case MSG_PLAY_PAUSE2://长按触发
			Cmd = BIT(6);
			OTG_DeviceInterruptSend(0x01,&Cmd, 1,1000);
			break;
		case MSG_PLAY_PAUSE:
//			APP_DBG("Play Pause\n");
//			Cmd = 0;
//			OTG_DeviceInterruptSend(0x01,&Cmd, 1,1000);
			PCAudioPP();
			break;
		case MSG_PLAY_PAUSE3://长按松手
			Cmd = 0;
			OTG_DeviceInterruptSend(0x01,&Cmd, 1,1000);
			break;

```
关闭dblclick的宏开关，播放器会自动识别双击三击功能
//#define  CFG_FUNC_DBCLICK_MSG_EN

蓝牙模式
app_config.h里打开
```
#define  CFG_APP_BT_MODE_EN
#define  CFG_FUNC_USB_AUDIO_MIX_MODE //USB Audio mix需要开启USB_AUDIO_MODE
```


闪避功能
闪避功能体现在提示音上，提示音分为两种，一种是音效提示音（耳返大小之类）一种是模式切换提示音，模式切换提示音分为两种
1、先播放提示音再切换模式——有问题，提示音触发播放就切换掉了模式，前置mute无法触发，不起作用
2、先切换模式再播放提示音——可以正常触发前置mute提示音，所以需要把提示音判断播放的代码放在MSG_EFFECTMODE的最下面
mute gain -> remind play -> unmute gain 没有效果，因为执行的速度与顺序不匹配
触发流程后，短暂的mute后就触发提示音的播放，然后unmute，但触发remind play后，remind playing时与unmute gain的状态是重合的。因为我没有正确的知道提示音什么时候开始播放，什么时候算播放结束。不是简单的放在RemindSoundServiceItemRequest()这个播放提示音的函数首尾部就可以了。当然，提示音的触发后开始播放是从这个函数首部开始进行的，但是提示音播放结束不是在这里，而是在触发DBG("Mute Remind\n");这里。

提示音演示模式，在音效图中的REMIND_SOURCE的
![[Pasted image 20250427103035.png]]
其中，n10连接到n17节点通过DAC_SINK输出，gain_control9连接到n13节点，通过APP_SINK输出，通过APP_SINK输出的节点就可以在直播或通话时被对方听到，可以通过录音进行测试，能被录制进录音中说明有效果

#### 左右声道交换
`可以在上位机的HardWare Config中的DAC0的Output Mode中选择Stereo1 或 Stereo2来感受，但是默认的Output Mode需要在代码中进行修改，要去ctrlvars.c中的gCtrlvars.HwCt.DAC0Ct.dac_out_mode = MODE1;进行修改。其中 MODE0是Stereo1 MODE1是Stereo2`