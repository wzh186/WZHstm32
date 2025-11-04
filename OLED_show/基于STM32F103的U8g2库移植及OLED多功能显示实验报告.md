# 基于STM32F103的U8g2库移植及OLED多功能显示实验报告
## 一、实验目的
1. 了解I2C协议的基本原理和时序规范，掌握外设与STM32的I2C通信机制。
2. 熟悉0.96寸I2C接口OLED屏（SSD1306控制器）的工作原理，理解汉字点阵显示的核心逻辑。
3. 掌握开源GUI库U8g2在STM32F103平台的移植、裁剪与编译方法，实现图形界面可视化技术的应用。
4. 基于U8g2库实现OLED屏的多场景显示功能，包括demo例程运行、个性化信息显示、滑动效果及动态图案展示。

## 二、实验环境
### 1. 硬件环境
- 主控芯片：STM32F103C8T6最小系统板
- 显示模块：0.96寸I2C接口OLED屏（SSD1306控制器，分辨率128×64）
- 辅助工具：杜邦线、下载器、电脑

### 2. 软件环境
- 开发工具：STM32CubeMX 6.6.0（HAL库工程生成）
- 编译软件：Keil MDK 5.37
- 开源库：U8g2图形库（版本：latest）

## 三、实验原理概述
### 1. I2C协议原理
I2C协议是两线式串行通信协议，通过SCL（时钟线）和SDA（数据线）实现主从设备间的双向通信。通信过程包含起始条件（SCL高电平时SDA下降沿）、数据传输（SCL高电平时SDA保持稳定）、应答位（第9位SCL高电平时SDA拉低）和停止条件（SCL高电平时SDA上升沿），STM32作为主机控制OLED屏（从机）完成数据交互。

### 2. OLED屏工作原理
0.96寸OLED屏采用SSD1306控制器，通过I2C接口接收STM32发送的指令和数据，将像素点状态（亮/灭）存储在内部显存中，再驱动屏幕发光显示。屏幕分辨率为128×64，每个像素点对应显存中的一个比特位，通过控制显存数据实现文本、图形的显示。

### 3. U8g2库工作原理
U8g2是一款开源的图形库，支持多种显示设备和单片机平台。其核心功能包括字体管理、图形绘制、显存操作等，通过底层接口适配不同的硬件平台。在STM32上移植时，需实现I2C数据传输、GPIO控制和延迟函数，使U8g2库能通过HAL库操作OLED屏。

### 4. 汉字点阵显示原理
汉字显示基于点阵编码（如GB2312），每个汉字由固定尺寸的像素点阵（如24×24）表示。U8g2库的中文字体文件包含常用汉字的点阵数据，通过`u8g2_DrawStr`函数调用对应点阵数据，在OLED屏指定坐标绘制汉字。

## 四、实验步骤
### 1. 硬件连接
按I2C通信要求完成STM32与OLED屏的引脚连接，确保电源正负极不接反：
| OLED引脚 | STM32引脚 | 功能说明           |
| -------- | --------- | ------------------ |
| VCC      | 3.3V      | 供电（避免5V烧屏） |
| GND      | GND       | 接地               |
| SCL      | PB10      | I2C2时钟线         |
| SDA      | PB11      | I2C2数据线         |

### 2. STM32CubeMX工程配置（HAL库框架）
1. **RCC配置**：启用外部高速晶振（HSE），提升时钟精度。

   <img src="C:\Users\33070\AppData\Roaming\Typora\typora-user-images\image-20251104153859423.png" alt="image-20251104153859423" style="zoom:50%;" />

2. **SYS配置**：Debug模式设为Serial Wire，防止芯片自锁。

   <img src="C:\Users\33070\Pictures\Screenshots\屏幕截图 2025-11-04 154114.png" alt="屏幕截图 2025-11-04 154114" style="zoom:50%;" />

3. **I2C2配置**：选择Master模式，标准速度100KHz，使能I2C外设。

   <img src="C:\Users\33070\AppData\Roaming\Typora\typora-user-images\image-20251104154322164.png" alt="image-20251104154322164" style="zoom:50%;" />

4. **TIM1配置**：内部时钟，预分频系数71，提供us级延迟（U8g2库必需）。

   <img src="C:\Users\33070\AppData\Roaming\Typora\typora-user-images\image-20251104154403636.png" alt="image-20251104154403636" style="zoom:50%;" />

5. **时钟树配置**：系统时钟设为72MHz。

   <img src="C:\Users\33070\Pictures\Screenshots\屏幕截图 2025-11-04 154428.png" alt="屏幕截图 2025-11-04 154428" style="zoom:50%;" />

6. **工程生成**：勾选“Keep User Code when re-generating”，生成HAL库工程文件。

   <img src="C:\Users\33070\AppData\Roaming\Typora\typora-user-images\image-20251104154927892.png" alt="image-20251104154927892" style="zoom:50%;" />

### 3. U8g2库裁剪与移植
#### （1）U8g2库下载与精简
从GitHub下载U8g2源码（https://github.com/olikraus/u8g2），解压后仅保留核心文件：

我们主要关注U8g2库文件中的csrc文件，先双击打开csrc。

1. 找到u8x8_d_xxx.c命名的文件，我们使用的是u8x8_ssd1306_128x64_noname.c这个文件，只留下这一个文件，其他的类似文件删掉。

2. 找到并打开**u8g2_d_setup.c** 文件，里面有很多函数，我们用IIC接口的话，只需要使用u8g2_Setup_ssd1306_i2c_128x64_noname_f 这个函数，其余的统统删掉。然后保存退出

   ![image-20251104171832532](C:\Users\33070\AppData\Roaming\Typora\typora-user-images\image-20251104171832532.png)

3. 找到并打开**u8g2_d_memory.c** 文件，我们只调用**u8g2_m_16_8_f** 这个函数，其余的全部删掉。否则编译时很可能会提示内存不足。

   ![image-20251104171850467](C:\Users\33070\AppData\Roaming\Typora\typora-user-images\image-20251104171850467.png)

#### （2）将精简后的U8g2库添加至Keil
1. 在Keil工程中新建`U8g2`分组，添加精简后的U8g2库文件（csrc文件夹里的剩余文件）。

   <img src="C:\Users\33070\AppData\Roaming\Typora\typora-user-images\image-20251104172031197.png" alt="image-20251104172031197" style="zoom:50%;" />

2. 创建文件`stm32_u8g2.c`和`stm32_u8g2.h`，放在`Application/User/Core`文件夹下。

   ![image-20251104172116840](C:\Users\33070\AppData\Roaming\Typora\typora-user-images\image-20251104172116840.png)

3. 配置头文件路径（点开魔术棒里的C/C++，在Include Paths里加入头文件路径）。

   <img src="C:\Users\33070\AppData\Roaming\Typora\typora-user-images\image-20251104172530492.png" alt="image-20251104172530492" style="zoom:50%;" />

#### （3）底层接口实现
在`stm32_u8g2.c`中实现U8g2库所需的底层接口：
- `u8x8_byte_hw_i2c`：基于HAL库`HAL_I2C_Master_Transmit`函数，实现I2C数据传输。
- `u8x8_gpio_and_delay`：基于`HAL_Delay`和GPIO控制函数，实现延迟和引脚操作。
- `u8g2Init`：初始化U8g2结构体，绑定底层接口，配置OLED屏参数。

### 4. 功能实现代码编写
#### （1）U8g2 demo例程运行
直接调用U8g2库自带的测试函数，验证移植正确性：
```c
void u8g2Demo(u8g2_t *u8g2) {
    u8g2_FirstPage(u8g2);
    do {
        u8g2_SetFont(u8g2, u8g2_font_ncenB12_tf);
        u8g2_DrawStr(u8g2, 10, 20, "U8g2 Demo");
        u8g2_DrawFrame(u8g2, 5, 30, 118, 30); // 绘制矩形框
        u8g2_DrawCircle(u8g2, 64, 45, 10);    // 绘制圆形
    } while (u8g2_NextPage(u8g2));
    HAL_Delay(3000);
}
```

#### （2）学号显示
写入stm_u8g2.c文件，并在stm_u8g2.h文件中声明函数，最后在main.c开头#include"stm_u8g2.h",主程序就能调用这些函数了：
```c
void displayIdName(u8g2_t *u8g2) {
    u8g2_ClearBuffer(u8g2);
    // 显示学号
    u8g2_SetFont(u8g2, u8g2_font_ncenB12_tf);
    u8g2_DrawStr(u8g2, 5, 20, "ID: 632307030231");
    u8g2_SendBuffer(u8g2);
    HAL_Delay(3000);
}
```

#### （3）上下/左右滑动显示
通过循环更新文本坐标偏移量，实现滑动效果：
```c
// 左右滑动
void slideLeftRight(u8g2_t *u8g2) {
    int x_offset = 128;
    while (x_offset > -120) {
        u8g2_ClearBuffer(u8g2);
        u8g2_SetFont(u8g2, u8g2_font_ncenB14_tf);
        u8g2_DrawStr(u8g2, x_offset, 32, "滑动测试");
        u8g2_SendBuffer(u8g2);
        x_offset -= 2;
        HAL_Delay(20);
    }
}
// 上下滑动
void slideUpDown(u8g2_t *u8g2) {
    int y_offset = 64;
    while (y_offset > -30) {
        u8g2_ClearBuffer(u8g2);
        u8g2_SetFont(u8g2, u8g2_font_ncenB14_tf);
        u8g2_DrawStr(u8g2, 20, y_offset, "上下滑动");
        u8g2_SendBuffer(u8g2);
        y_offset -= 2;
        HAL_Delay(20);
    }
}
```

### 5. main函数整合
在`main.c`中初始化硬件和U8g2库，按顺序调用各功能函数：
```c
#include "main.h"
#include "i2c.h"
#include "tim.h"
#include "gpio.h"
#include "u8g2.h"
#include "stm32_u8g2.h"

int main(void) {
    HAL_Init();
    SystemClock_Config();
    MX_GPIO_Init();
    MX_I2C2_Init();
    
    u8g2_t u8g2;
    u8g2Init(&u8g2); // U8g2初始化
    
    while (1) {
        u8g2Demo(&u8g2);        // 1. demo例程
        displayIdName(&u8g2);   // 2. 学号
        slideLeftRight(&u8g2);  // 3. 左右滑动
        slideUpDown(&u8g2);     // 3. 上下滑动
    }
}
```

## 五、实验结果
1. **U8g2 demo例程**：OLED屏成功显示“U8g2 Demo”文本、矩形框和圆形，验证了U8g2库移植正确。

   

2. **学号显示**：学号“632307030231”清晰显示，无乱码。

3. **滑动显示**：文本从屏幕边缘平滑滑入、滑出，左右和上下滑动效果流畅，速度均匀。

   

## 六、常见问题与解决方法
| 问题现象                     | 原因分析                        | 解决办法                                                 |
| ---------------------------- | ------------------------------- | -------------------------------------------------------- |
| 编译警告“文件末尾缺少换行符” | 源文件未按C语言规范添加结尾换行 | 在对应文件最后一行手动添加换行符                         |
| 滑动效果卡顿                 | 延迟时间过长或步长设置不合理    | 减小`HAL_Delay`值（如改为10-20ms），调整步长为2-3像素    |
| 编译错误“未定义符号”         | 函数未声明或文件未添加到工程    | 1. 在头文件中声明函数；2. 检查文件是否添加到Keil工程分组 |



