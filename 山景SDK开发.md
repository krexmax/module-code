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
#### USB描述符基础信息
VID PID `otg_device_standard_request.h` 中的宏定义修改，注意PID数值与实际有偏差，一般情况下是3的偏差

ProductName  ManufactureName  SerialNumber
`otg_device_standard_request.c`中修改
```C
const char *gDeviceProductString ="RY912-GM";			    //max length: 32bytes
const char *gDeviceString_Manu ="RYDZ";						//max length: 32bytes
const char *gDeviceString_SerialNumber ="202504141621";	    //max length: 32bytes
```

#### 笔记
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
\#define CFG_IDLE_MODE_POWER_KEY	//power key
\#define CFG_IDLE_MODE_DEEP_SLEEP //deepsleep 关闭
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
![Pasted image 20250427103035.png](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAABSEAAAHcCAIAAACmqvgJAAAgAElEQVR4Ae3dvcssyWEv4LElXwc2CFnnD7CPFDiQfQwLciC0kpys8bXl6OwKDhJGoERgr5AMRwrEdXCXtRNnhoVXmzk1wsFllSjfQBbY8MbCiVJFm5gDe5mp7p7+mpqemZ7uqupHCGne/qiueqp6pn+nenp2r/yHAAECBAgQIECAAAECBAgQmENgN0chyiBAgAABAgQIECBAgAABAgReydgGAQECBAgQIECAAAECBAgQmEdAxp7HUSkECBAgQIAAAQIECBAgQEDGNgYIECBAgAABAgQIECBAgMA8AjL2PI5KIUCAAAECBAgQIECAAAECMrYxQIAAAQIECBAgQIAAAQIE5hGQsedxVAoBAgQIECBAgAABAgQIEJCxjQECBAgQIECAAAECBAgQIDCPgIw9j6NSCBAgQIAAAQIECBAgQICAjG0MECBAgAABAgQIECBAgACBeQRk7HkclUKAAAECBAgQIECAAAECBGRsY4AAAQIECBAgQIAAAQIECMwjIGPP46gUAgQIECBAgAABAgQIECAgYxsDBAgQIECAAAECBAgQIEBgHgEZex5HpRAgQIAAAQIECBAgQIAAARnbGCBAgAABAgQIECBAgAABAvMIyNjzOCqFAAECBAgQIECAAAECBAjI2MYAAQIECBAgQIAAAQIECBCYR0DGnsdRKQQIECBAgAABAgQIECBAQMY2BggQIECAAAECBAgQIECAwDwCMvY8jkohQIAAAQIECBAgQIAAAQIytjFAgAABAgQIECBAgAABAgTmEZCx53FUCgECBAgQIECAAAECBAgQkLGNAQIECBAgQIAAAQIECBAgMI+AjD2Po1IIECBAgAABAgQIECBAgICMbQwQIECAAAECBAgQIECAAIF5BGTseRyVQoAAAQIECBAgQIAAAQIEZGxjgAABAgQIECBAgAABAgQIzCMgY8/jqBQCBAgQIECAAAECBAgQICBjGwMECBAgQIAAAQIECBAgQGAeARl7HkelECBAgAABAgQIECBAgAABGdsYIECAAAECBAgQIECAAAEC8wjI2PM4KoUAAQIECBAgQIAAAQIECMjYxgABAgQIECBAgAABAgQIEJhHQMaex1EpBAgQIECAAAECBAgQIEBAxjYGCBAgQIAAAQIECBAgQIDAPAIy9jyOSiFAgAABAgQIECBAgAABAjK2MUCAAAECBAgQIECAAAECBOYRkLHncVQKAQIECBAgQIAAAQIECBCQsY0BAgQIECBAgAABAgQIECAwj4CMPY+jUggQIECAAAECBAgQIECAgIxtDBAgQIAAAQIECBAgQIAAgXkEZOx5HJVCgAABAgQIECBAgAABAgRkbGOAAAECBAgQIECAAAECBAjMIyBjz+OoFAIECBAgQIAAAQIECBAgIGMbAwQIECBAgAABAgQIECBAYB4BGXseR6UQIECAAAECBAgQIECAAAEZ2xggQIAAAQIECBAgQIAAAQLzCMjY8zgqhQABAgQIECBAgAABAgQIyNjGAAECBAgQIECAAAECBAgQmEdAxp7HUSkECBAgQIAAAQIECBAgQEDGNgYIECBAgAABAgQIECBAgMA8AjL2PI5KIUCAAAECBAgQIECAAAECMrYxQIAAAQIECBAgQIAAAQIE5hGQsedxVAoBAgQIECBAgAABAgQIEJCxjQECBAgQIECAAAECBAgQIDCPgIw9j6NSCBAgQIAAAQIECBAgQICAjG0MECBAgAABAgQIECBAgACBeQRk7HkclUKAAAECBAgQIECAAAECBGRsY4AAAQIECBAgQIAAAQIECMwjIGPP46gUAgQIECBAgAABAgQIECAgYxsDBAgQIECAAAECBAgQIEBgHgEZex5HpRAgQIAAAQIECBAgQIAAARnbGCBAgAABAgQIECBAgAABAvMIyNjzOCqFAAECBAgQIECAAAECBAjI2MYAAQIECBAgQIAAAQIECBCYR0DGnsdRKQQIECBAgAABAgQIECBAQMY2BggQIECAAAECBAgQIECAwDwCMvY8jkohQIAAAQIECBAgQIAAAQIytjFAgAABAgQIECBAgAABAgTmEZCx53FUCgECBAgQIECAAAECBAgQkLGNAQIECBAgQIAAAQIECBAgMI+AjD2Po1IIECBAgAABAgQIECBAgICMbQwQIECAAAECBAgQIECAAIF5BGTseRyVQoAAAQIECBAgQIAAAQIEZGxjgAABAgQIECBAgAABAgQIzCMgY8/jqBQCBAgQIECAAAECBAgQICBjGwMECBAgQIAAAQIECBAgQGAeARl7HkelECBAgAABAgQIECBAgAABGdsYIECAAAECBAgQIECAAAEC8wjI2PM4KoUAAQIE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ARk7zX5RKwIECBAgQIAAAQIECBDIT0DGzq/P1JgAAQIECBAgQIAAAQIE0hSQsdPsF7UiQIAAAQIECBAgQIAAgfwEZOz8+kyNCRAgQIAAAQIECBAgQCBNARk7zX5RKwIECBAgQIAAAQIECBDIT0DGzq/P1JgAAQIECBAgQIAAAQIE0hSQsdPsF7UiQIAAAQIECBAgQIAAgfwEZOz8+kyNCRAgQIAAAQIECBAgQCBNARk7zX5RKwIECBAgQIAAAQIECBDIT0DGzq/P1JgAAQIECBAgQIAAAQIE0hSQsdPsF7UiQIAAAQIECBAgQIAAgfwEZOz8+kyNCRAgQIAAAQIECBAgQCBNARk7zX5RKwIECBAgQIAAAQIECBDIT0DGzq/P1JgAAQIECBAgQIAAAQIE0hSQsdPsF7UiQIAAAQIECBAgQIAAgfwEZOz8+kyNCRAgQIAAAQIECBAgQCBNARk7zX5RKwIECBAgQIAAAQIECBDIT0DGzq/P1JgAAQIECBAgQIAAAQIE0hSQsdPsF7UiQIAAAQIECBAgQIAAgfwEZOz8+kyNCRAgQIAAAQIECBAgQCBNARk7zX5RKwIECBAgQIAAAQIECBDIT0DGzq/P1JgAAQIECBAgQIAAAQIE0hSQsdPsF7UiQIAAAQIECBAgQIAAgfwEZOz8+kyNCRAgQIAAAQIECBAgQCBNARk7zX5RKwIECBAgQIAAAQIECBDIT0DGzq/P1JgAAQIECBAgQIAAAQIE0hSQsdPsF7UiQIAAAQIECBAgQIAAgfwEZOz8+kyNCRAgQIAAAQIECBAgQCBNARk7zX5RKwIECBAgQIAAAQIECBDIT0DGzq/P1JgAAQIECBAgQIAAAQIE0hSQsdPsF7UiQIAAAQIECBAgQIAAgfwEZOz8+kyNCRAgQIAAAQIECBAgQCBNARk7zX5RKwIECBAgQIAAAQIECBDIT0DGzq/P1JgAAQIECBAgQIAAAQIE0hSQsdPsF7UiQIAAAQIECBAgQIAAgfwEZOz8+kyNCRAgQIAAAQIECBAgQCBNARk7zX5RKwIECBAgQIAAAQIECBDIT0DGzq/P1JgAAQIECBAgQIAAAQIE0hSQsdPsF7UiQIAAAQIECBAgQIAAgfwEZOz8+kyNCRAgQIAAAQIECBAgQCBNARk7zX5RKwIECBAgQIAAAQIECBDIT0DGzq/P1JgAAQIECBAgQIAAAQIE0hSQsdPsF7UiQIAAAQIECBAgQIAAgfwEZOz8+kyNCRAgQIAAAQIECBAgQCBNARk7zX5RKwIECBAgQIAAAQIECBDIT0DGzq/P1JgAAQIECBAgQIAAAQIE0hSQsdPsF7UiQIAAAQIECBAgQIAAgfwEZOz8+kyNCRAgQIAAAQIECBAgQCBNARk7zX5RKwIECBAgQIAAAQIECBDIT0DGzq/P1JgAAQIECBAgQIAAAQIE0hSQsdPsF7UiQIAAAQIECBAgQIAAgfwEZOz8+kyNCRAgQIAAAQIECBAgQCBNARk7zX5RKwIECBAgQIAAAQIECBDIT0DGzq/P1JgAAQIECBAgQIAAAQIE0hSQsdPsF7UiQIAAAQIECBAgQIAAgfwEZOz8+kyNCRAgQIAAAQIECBAgQCBNARk7zX5RKwIECBAgQIAAAQIECBDIT0DGzq/P1JgAAQIECBAgQIAAAQIE0hSQsdPsF7UiQIAAAQIECBAgQIAAgfwEZOz8+kyNCRAgQIAAAQIECBAgQCBNARk7zX5RKwIECBAgQIAAAQIECBDIT0DGzq/P1JgAAQIECBAgQIAAAQIE0hSQsdPsF7UiQIAAAQIECBAgQIAAgfwEZOz8+kyNCRAgQIAAAQIECBAgQCBNARk7zX5RKwIECBAgQIAAAQIECBDIT0DGzq/P1JgAAQIECBAgQIAAAQIE0hSQsdPsF7UiQIAAAQIECBAgQIAAgfwEZOz8+kyNCRAgQIAAAQIECBAgQCBNARk7zX5RKwIECBAgQIAAAQIECBDIT0DGzq/P1JgAAQIECBAgQIAAAQIE0hSQsdPsF7UiQIAAAQIECBAgQIAAgfwEZOz8+kyNCRAgQIAAAQIECBAgQCBNARk7zX5RKwIECBAgQIAAAQIECBDIT0DGzq/P1JgAAQIECBAgQIAAAQIE0hSQsdPsF7UiQIAAAQIECBAgQIAAgfwEZOz8+kyNCRAgQIAAAQIECBAgQCBNARk7zX5RKwIECBAgQIAAAQIECBDIT0DGzq/P1JgAAQIECBAgQIAAAQIE0hSQsdPsF7UiQIAAAQIECBAgQIAAgfwEZOz8+kyNCRAgQIAAAQIECBAgQCBNARk7zX5RKwIECBAgQIAAAQIECBDIT0DGzq/P1JgAAQIECBAgQIAAAQIE0hSQsdPsF7UiQIAAAQIECBAgQIAAgfwEZOz8+kyNCRAgQIAAAQIECBAgQCBNARk7zX5RKwIECBAgQIAAAQIECBDIT0DGzq/P1JgAAQIECBAgQIAAAQIE0hSQsdPsF7UiQIAAAQIECBAgQIAAgfwEZOz8+kyNCRAgQIAAAQIECBAgQCBNARk7zX5RKwIECBAgQIAAAQIECBDIT0DGzq/P1JgAAQIECBAgQIAAAQIE0hSQsdPsF7UiQIAAAQIECBAgQIAAgfwEZOz8+kyNCRAgQIAAAQIECBAgQCBNARk7zX5RKwIECBAgQIAAAQIECBDIT0DGzq/P1JgAAQIECBAgQIAAAQIE0hSQsdPsF7UiQIAAAQIECBAgQIAAgfwEZOz8+kyNCRAgQIAAAQIECBAgQCBNARk7zX5RKwIECBAgQIAAAQIECBDIT0DGzq/P1JgAAQIECBAgQIAAAQIE0hSQsdPsF7UiQIAAAQIECBAgQIAAgfwE/j/Z4XbirCoK3AAAAABJRU5ErkJggg==)
其中，n10连接到n17节点通过DAC_SINK输出，gain_control9连接到n13节点，通过APP_SINK输出，通过APP_SINK输出的节点就可以在直播或通话时被对方听到，可以通过录音进行测试，能被录制进录音中说明有效果

#### 左右声道交换
`可以在上位机的HardWare Config中的DAC0的Output Mode中选择Stereo1 或 Stereo2来感受，但是默认的Output Mode需要在代码中进行修改，要去ctrlvars.c中的gCtrlvars.HwCt.DAC0Ct.dac_out_mode = MODE1;进行修改。其中 MODE0是Stereo1 MODE1是Stereo2`

#### 芯片配置 
`chip_config.h里面修改芯片型号，并且可以修改单端和差分模式`
`默认是差分配置，把差分配置的两个宏定义关闭就会被处理为单端配置`
`#define	CHIP_DAC_USE_DIFF `
`#define	CHIP_DAC_USE_PVDD16`
