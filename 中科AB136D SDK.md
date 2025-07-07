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
