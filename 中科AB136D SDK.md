#### 功能函数实现
1. [[|MIC静音]] 



AB136D1 KU_PLAY 控制侧音模式
先把K_PLAY删除以及把KL_PLAY原函数注释掉，然后，将mic2src_sta的上下宏判断删除，把原先的mic2src_sta函数移植到KL_PLAY中，将uda_set_isocin_flag()函数修改为usb_mic2src1_en = 0/1;
问题是首次上电KL_PLAY没有反应

vol+-的长按功能是由KH_VOL_UP决定的，但是不能简单的把KH_VOL_UP注释掉，会导致长按的时候虽然音量条没有变化但是音量还是会变化。需要把case KH_VOL_UP后break退出才行

\\\宏定义控制引脚输出高低电平
\#define PB0_INIT_H()       
{GPIOBFEN &= ~BIT(0); \
GPIOBDE |= BIT(0); \
GPIOBDIR &= ~BIT(0); \
GPIOBSET |= BIT(0);}

\#define PB0_INIT_L()      
{GPIOBFEN &= ~BIT(0); \
GPIOBDE |= BIT(0); \
GPIOBDIR &= ~BIT(0); \
GPIOBCLR |= BIT(0);}

解析：
GPIOBFEN &= ~BIT(0);            //0，将该引脚当作通用GPIO口使用；1，当作其他功能IO口
GPIOBDE |= BIT(0);                 //数字IO使能: 0为模拟IO, 1 为数字IO
GPIOBDIR &= ~BIT(0);            //控制IO口的方向，0为输出，1为输入
GPIOBSET |= BIT(0)；            //将该引脚配置为输出高
GPIOBCLR |= BIT(0);              //将该引脚配置为输出低

![[Pasted image 20250701172251.png]]
![[Pasted image 20250701173415.png]]
![[Pasted image 20250701173449.png]]
        msg_enqueue(key);


放在func_usbdev_enter()函数中 uda_set_mic_mute(1);可以实现静音

# SPI 推 LED幻彩灯
首先明确LED的时序
![[Pasted image 20250717194618.png]]
` 数据传输速率     800KHz`
1/800 000 = 1.25us。取1.2us为一个bit的时间
0.3 + 0.9 = 1.2us  //IC 内置规则

AB136D1中的 spibaud 的计算公式为 SPIBAUD = SYSCLK/BAUD - 1;
其中，SPIBAUD是用于配置 SPI 外设的分频器值
BAUD 是SPI的波特率

因为SPI传输的单位是字节，因此配置为8bit 来 模拟一个ws2812的bit 是可以的
MCU内部的数据处理后，通过SPI外设发送出去就是电平数据

所以SPI单bit的发送周期为1.2 / 8 = 0.15us  
那么 BAUD = 1 / 0.15us = 6.66MHz

规格书中知道ws2812每bit都是通过电平的时间长短进行的判断，0.3 = 0.15 `*` 2;  0.9 = 0.15 * 6
所以ws2812所
规定的逻辑0 使用1100 0000 表示   0xC0
规定的逻辑1 使用1111 1100表示     0xFC

然后发送单色LED灯的函数为
```c
void rgb_spi_single_color_blue(void)//蓝色 GRB数据格式
{
//    rgb_spi1_init(RGB_BAUD);
    WDT_CLR();
    rgb_data = 0x000000ff;
    for(u8 i=0;i<LED_NUM;i++)
    {
        data_fill[i] = rgb_data;
    }
    memset(rgb_buf,0xc0,sizeof(rgb_buf));//清除rgb数据BUF
    for(u8 i=0;i<LED_NUM;i++)
    {
      led_rgb_5050_data_fill(data_fill[i],i);//填充rgb数据BUF
    }
    spi1_senddata();
}
```
第一个for循环，将32位的rgb_data数据赋值到u32 data_fill[ ] 数组中。
然后，通过memset函数，将数组rgb_buf[ ]全部元素赋值LOGIC_0 。相当于清零

之后第二个for循环，调用led_rgb_5050_data_fill( )函数进行数据处理
```c
void led_rgb_5050_data_fill(u32 grb_data, u8 led_num)
{
     grb_data <<= 8;//左移8bit，高位对齐
     for(u8 i = 0;i < 24;i++)
     {
        if(grb_data & BIT(31))
        {
            rgb_buf[led_num*24+i] |= LOGIC_1;
        }
        else
        {
            rgb_buf[led_num*24+i] |= LOGIC_0;
        }
        grb_data <<= 1;
     }
}
```
for循环中，grb_data的数据是上一层函数的data_fill[ ]数组的元素，也就是上面赋值设置的0x000000FF的颜色。因此if else判断是将该32位数据与0x10000000进行与运算，BIT(31)的其他位都是0，如果0x000000FF的首位是0的话，全部32位数据都是0，所以可以通过&BIT(31)的运算来对数据最高位进行判断
下面的或运算 LOGIC_1 = 0xFC; 
因为在指定到这个语句之前就已经将rgb_buf[ ]数组进行了初始化，memset(rgb_buf,0xc0,sizeof(rgb_buf));
所有的数值都是0xC0，
所以，如果数据最高位是1，那么就进行 0x1100 0000 | 0x1111 1100的运算，运算的结果就是0x1111 1100 = 0xFC
同理 0x1100 0000 | 0x1100 0000 = 0x1100 0000 = 0xC0（0x1100 0000 是因为已经初始化过0xC0）
其实如果在rgb_buf[ ]数组的初始化时赋值数据不是0xC0,而是0x00的话，不需要或运算，直接复制到rgb_buf[ ]中就可以了