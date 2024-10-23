# U8G2_Porting

### 1芯片支持：

SSD1305, SSD1306, SSD1309, SSD1312, SSD1316, SSD1320, SSD1322, SSD1325, SSD1327, SSD1329, SSD1606, SSD1607, SH1106, SH1107, SH1108, SH1122, T6963, RA8835, LC7981, PCD8544, PCF8812, HX1230, UC1601, UC1604, UC1608, UC1610, UC1611, UC1617, UC1638, UC1701, ST7511, ST7528, ST7565, ST7567, ST7571, ST7586, ST7588, ST75256, ST75320, NT7534, ST7920, IST3020, IST7920, LD7032, KS0108, KS0713, HD44102, T7932, SED1520, SBN1661, IL3820, MAX7219

### 2Porting：

1.

原生的u8g2库支持c/c++；单片机移植著需要考虑语言环境，此处是c那就只考虑csrc，c（语言）src（源码）；

在csrc里面找自己的驱动文件。such as：u8x8 d ssd1306 128x64 noname.c 将其移植到自己的主程序中；

2.

修改修改"u8g2_d_setup.c"这个文件，里面有各种驱动芯片的初始化函数，删除其他函数，只留下与使用的驱动芯片相关的函数。
本文使用的ssd1306，但是与ssd1306相关的有多个函数，例如：

下面都是spi接口的：

u8g2_Setup_ssd1306_128x64_noname_1、
u8g2_Setup_ssd1306_128x64_noname_2、
u8g2_Setup_ssd1306_128x64_noname_f，

下面都是i2c接口的：

u8g2_Setup_ssd1306_i2c_128x64_noname_1、
u8g2_Setup_ssd1306_i2c_128x64_noname_2、
u8g2_Setup_ssd1306_i2c_128x64_noname_f，
后缀1、2、f代表缓冲区大小的不同：
1代表128字节，
2代表256字节，
f代表1024字节；
根据单片机空间的大小选择合适的接口，缓冲区小的，刷新lcd/oled的时候就比较耗时，反之。
此处使用u8g2_Setup_ssd1306_i2c_128x64_noname_f这个接口：

3.

修改“u8g2_d_memory.c”文件，这个文件里面其实就是“u8g2_d_setup.c”文件对应的缓冲区，同上面一样，屏蔽掉没用到的，留下用到的：

4.

字库：u8g2有内置字库，但是写ui时候需要等高等宽的字，方便做后续的ui开发，建议找到一款等高宽的字体；

同时注意字库是占空间的，只保留自己需要使用的字库即可，用不到的屏蔽掉；

5.

u8g2的初始化：u8g2_Setup_ssd1306_i2c_128x64_noname_f(&u8g2,U8G2_R0,u8x8_byte_sw_i2c,u8x8_gpio_and_delay);

上面是u8g2的初始化函数，内部有四个参数：

第一个是u8g2的结构体，后面只需定义即可。

第二给参数指的是图像的显示方向。

第三个参数

u8x8_byte_sw_i2c，（sw：soft write）指的是u8g2是i2c通信的方式。软件i2c通信的话，官方已经写的差不多了，只需要简单的指定IO口就行；

软件i2c的接口在u8x8_gpio_and_delay里面；后面我们会讲





u8x8_byte_hw_i2c，（hw：hard write）指的是u8g2的硬件i2c通信，硬件i2c的话需要用户将i2c发送数据的函数写在此函数内部，

```c
uint8_t u8x8_byte_hw_i2c(u8x8_t *u8x8, uint8_t msg, uint8_t arg_int, void *arg_ptr)
{
  static uint8_t buffer[32];		/* u8g2/u8x8 will never send more than 32 bytes between START_TRANSFER and END_TRANSFER */
  static uint8_t buf_idx;
  uint8_t *data;

  switch(msg){
```

```c
case U8X8_MSG_BYTE_SEND:
  data = (uint8_t *)arg_ptr;      
  while( arg_int > 0 ){
			buffer[buf_idx++] = *data;
			data++;
			arg_int--;
		}      
break;
		
case U8X8_MSG_BYTE_INIT:
  /* add your custom code to init i2c subsystem */
break;
	
case U8X8_MSG_BYTE_START_TRANSFER:
  buf_idx = 0;
break;
	
case U8X8_MSG_BYTE_END_TRANSFER:
  HAL_I2C_Master_Transmit(&hi2c1,u8x8_GetI2CAddress(u8x8), buffer, buf_idx,1000);
//此处修改，我这里使用的是hal库的I2C1，那么只需要把hal内置的i2c发送信息的函数放置在这即可；
break;
	
default:
  return 0;
```
  }
  return 1;
}

6.

最后一个参数u8x8_gpio_and_delay；

这个函数需要用户自己定义并且填充：

```c
uint8_t u8g2_gpio_and_delay_stm32(U8X8_UNUSED u8x8_t *u8x8, U8X8_UNUSED uint8_t msg, U8X8_UNUSED uint8_t arg_int, U8X8_UNUSED void *arg_ptr)
{
	switch(msg)
		{
			case U8X8_MSG_DELAY_MILLI://Function which implements a delay, arg_int contains the amount of ms
				HAL_Delay(arg_int);//填写一毫米的延时函数；此处我使用的hal自带的1ms演示；
			break;
		
			case U8X8_MSG_DELAY_10MICRO://Function which delays 10us
				delay_10us();//此处需要10us，我自己定义了一个软件循环延时，建议加定时器中断，更加准确和便于多任务调用
			break;
		
			case U8X8_MSG_DELAY_100NANO://Function which delays 100ns
				__NOP();//这是一个时钟周期的延时，级别一般都在10纳秒，我在这里仅仅填充为了不报错；
            //时钟频率为 72 MHz： [ \text{延时} \approx \frac{1}{72 \times 10^6} \approx 13.89 \text{纳秒} ]
            //公式为：\text{延时} = \frac{1}{F} \text{秒}
			break;
			case U8X8_MSG_GPIO_I2C_CLOCK://在这里定义你需要的引脚
				if (arg_int) HAL_GPIO_WritePin(GPIOB,GPIO_PIN_6,GPIO_PIN_SET);
				else HAL_GPIO_WritePin(GPIOB,GPIO_PIN_6,GPIO_PIN_RESET);
			break;
			case U8X8_MSG_GPIO_I2C_DATA://在这里定义你需要的引脚
				if (arg_int) HAL_GPIO_WritePin(GPIOB,GPIO_PIN_7,GPIO_PIN_SET);
				else HAL_GPIO_WritePin(GPIOB,GPIO_PIN_7,GPIO_PIN_RESET);
			break;
			default:
				return 0; //A message was received which is not implemented, return 0 to indicate an error
	}
	return 1; // command processed successfully.
}
```

#### 目前为止准备工作结束；

接下来初始化试试吧：（在main函数填充下面代码即可）

```c
u8g2_t u8g2;
 
u8g2_Setup_ssd1306_i2c_128x64_noname_f(&u8g2,U8G2_R0,u8x8_byte_hw_i2c,u8g2_gpio_and_delay_stm32);
u8g2_InitDisplay(&u8g2); // send init sequence to the display, display is in sleep mode after this,
u8g2_SetPowerSave(&u8g2, 0); // wake up display
u8g2_ClearDisplay(&u8g2);
u8g2_SetFont(&u8g2, u8g2_font_wqy16_t_chinese1);
u8g2_DrawCircle(&u8g2,60,30,20,U8G2_DRAW_ALL);
u8g2_DrawUTF8(&u8g2,10,50,"Hello,U8g2");
u8g2_SendBuffer(&u8g2);


```

