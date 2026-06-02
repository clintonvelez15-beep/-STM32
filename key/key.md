6.2 按钮处理方式

我们对待不同形态的按钮应将接受输入的GPIO设置为相对应的模式，举例如下：

#### 1. PB12 - 浮空输入 + 外部上拉电阻

```c
GPIO_InitStruct.Pin = GPIO_PIN_12;
GPIO_InitStruct.Mode = GPIO_MODE_INPUT;
GPIO_InitStruct.Pull = GPIO_NOPULL;   // 浮空
HAL_GPIO_Init(GPIOB, &GPIO_InitStruct);
```

PB12 引脚外部连接了一个上拉电阻（例如 10kΩ 接 VCC），同时按钮一端接 PB12，另一端接地（GND）。

* **无按键时** ：上拉电阻将 PB12 拉高 → `HAL_GPIO_ReadPin` 读取为 `GPIO_PIN_SET`（高电平）。
* **按键按下时** ：PB12 被短路到 GND → 读取为 `GPIO_PIN_RESET`（低电平）。
* **软件不启用内部上拉** （`GPIO_NOPULL`），完全依赖外部电路，避免与内部上拉冲突。

#### 2. PB13 – 软件上拉输入

```c
GPIO_InitStruct.Pin = GPIO_PIN_13;
GPIO_InitStruct.Mode = GPIO_MODE_INPUT;
GPIO_InitStruct.Pull = GPIO_PULLUP;   // 内部上拉
HAL_GPIO_Init(GPIOB, &GPIO_InitStruct);
```

按键同样是一端接 PB13，另一端GND。但是**外部没有上拉电阻** ，完全使用 STM32 内部的上拉电阻（约 40kΩ）。

* **无按键时** ：内部上拉使 PB13 保持高电平 → 读取为 `GPIO_PIN_SET`。
* **按键按下时** ：引脚被拉低 → 读取为 `GPIO_PIN_RESET`。

接下来给处主循环代码讲解：

```c
while (1)
{
    // 按键1（PB12）控制 LED1（PA7）
    if(HAL_GPIO_ReadPin(GPIOB, GPIO_PIN_12) == GPIO_PIN_RESET)
        HAL_GPIO_WritePin(GPIOA, GPIO_PIN_7, GPIO_PIN_SET);
    else
        HAL_GPIO_WritePin(GPIOA, GPIO_PIN_7, GPIO_PIN_RESET);

    // 按键2（PB13）控制 LED2（PB0）
    if(HAL_GPIO_ReadPin(GPIOB, GPIO_PIN_13) == GPIO_PIN_RESET){
        HAL_GPIO_TogglePin(GPIOB, GPIO_PIN_0);            // 翻转 LED2 状态
        while( HAL_GPIO_ReadPin(GPIOB, GPIO_PIN_13) == GPIO_PIN_RESET){}  // 死等按键释放
    }
} 
```

### PB12 的逻辑（点动控制）

* **按下** → PA7 输出高电平（点亮 LED）
* **松开** → PA7 输出低电平（熄灭 LED）

LED 状态完全跟随按键，按下就亮，松开就灭，没有“记忆”功能。

### PB13 的逻辑（边缘触发 + 阻塞等待）

* 当检测到 PB13  **被按下** （低电平）时：
  * 执行 `HAL_GPIO_TogglePin(GPIOB, GPIO_PIN_0)` → **翻转** PB0 引脚的电平（`HAL_GPIO_TogglePin`函数作用）。
  * 然后进入一个空的 `while` 循环，条件为 `HAL_GPIO_ReadPin(GPIOB, GPIO_PIN_13) == GPIO_PIN_RESET`。
    * 按钮按下尚未松开那一段时间内，按钮一直处于低电平，`HAL_GPIO_TogglePin`被多次执行
    * 只要按键还处于按下状态循环就一直空转，确保电平只反转一次
    * **但cpu在按着开关的时间段内便一直卡死在该循环** 。

所以，也有了一个简单的方案：

#### （1）简单软件实现

使用一个静态变量记录 PB13 的上一次电平，每次主循环读取当前电平，检测下降沿或上升沿，然后执行相应动作。

```c
// 在主函数外或开头定义状态变量
static uint8_t lastPb13State = GPIO_PIN_SET;   // 上一次 PB13 电平（默认高）
static uint8_t led2State = 0;                  // 记录 PB0 的 LED 状态

while (1)
{
    uint8_t pb13 = HAL_GPIO_ReadPin(GPIOB, GPIO_PIN_13);
    // 检测下降沿—— 只执行一次翻转
    if (lastPb13State == GPIO_PIN_SET && pb13 == GPIO_PIN_RESET) {
        led2State = !led2State;
        HAL_GPIO_WritePin(GPIOB, GPIO_PIN_0, led2State ? GPIO_PIN_SET : GPIO_PIN_RESET);
    }
    // 更新上一次状态（为下一次比较做准备）
    lastPb13State = pb13;

    // 其他按键（PB12）的处理可以并行，不会阻塞
    // ...（PB12 的代码保持不变）
}
```

#### （2）**时间触发状态机**

除此之外，当然还有状态机的做法，结合上述方法和定时器，将按键扫描放在一个固定时间间隔的定时器中断中（例如每 10ms）。这样主循环只负责业务逻辑，按键响应非常稳定。

不过这要求在主函数之外实现**中断服务函数**（使用按键扫描函数）与**按键扫描函数**（状态机，同时包含按钮实现）

```c
// 定时器中断服务函数（每 10ms 自动进入一次）
void TIM2_IRQHandler() {
    if (TIM2->SR & TIM_SR_UIF) {
        TIM2->SR &= ~TIM_SR_UIF;        // 清除中断标志
        Key_Scan();                     // 按键扫描（包含状态机）
    }
}

// 按键扫描函数（使用状态机，非阻塞）
void Key_Scan() {
    static uint8_t lastPb13 = GPIO_PIN_SET;
    uint8_t cur = HAL_GPIO_ReadPin(GPIOB, GPIO_PIN_13);
    if (lastPb13 == GPIO_PIN_SET && cur == GPIO_PIN_RESET) {
        // 高电平时读到低电平（按下事件）：设置一个标志位，或者直接执行动作（但不建议在中断里做长操作）
        key_press_flag = 1;
    }
    lastPb13 = cur;
}

// 主循环
int main() {
    HAL_Init();
    SystemClock_Config();
    MX_GPIO_Init();
    MX_TIM2_Init();         // 初始化定时器并启动
    while (1) {
        // 业务逻辑：只根据标志位处理按键动作
        if (key_press_flag) {
            key_press_flag = 0;
            HAL_GPIO_TogglePin(GPIOB, GPIO_PIN_0);
        }
        // 其他业务代码...
    }
}
```

让我们仔细看一下

```c
// 定时器中断服务函数（每 10ms 自动进入一次）
void TIM2_IRQHandler() {
    if (TIM2->SR & TIM_SR_UIF) {
        TIM2->SR &= ~TIM_SR_UIF;        // 清除中断标志
        Key_Scan();                     // 按键扫描（包含状态机）
    }
}
```

首先创建TIM2 定时器的中断服务函数，当 TIM2 产生中断时，CPU 会自动跳转到这个函数执行。

接下来if语句检查 UIF 标志位是否为 1，是则进入 if 。

* `TIM2->SR` 是 TIM2 的 **状态寄存器** （Status Register）。
* `TIM_SR_UIF` 是 **更新中断标志** （Update Interrupt Flag），当定时器计数溢出（达到设定的重装载值）时，硬件会自动将这一位置 1。
* `if` 语句检查 UIF 是否为 1，确认本次中断是否由“定时器溢出”引起（避免误触发其他类型的中断）。
* 将 UIF 位清零，告诉硬件“我已响应此中断”。
* 调用按键扫描函数。函数内部实现状态机，读取当前按键电平，更新按键状态，检测按下/释放事件
* 主函数在知道以后，便反转引脚

所以说到底，这里的状态机是什么呢？

状态机是一个数学模型，用来描述一个对象在其生命周期内所经历的各种状态，以及在这些状态下对外部事件的响应和状态之间的转移规则。

所以说，上面的代码要说算，当然算。不过不是很好玩。具体的状态机我们后面慢慢讲解。


### 进阶：按钮的消抖

机械按钮（轻触开关）内部由弹性金属片构成，在按下或释放的瞬间，触点会发生多次弹跳而不是理想的一次性闭合或断开。所以一般无论有没有硬件进行消抖，都需要软件消抖增强代码健壮性

### 1. 延时消抖（最简单、阻塞CPU）

```c
if (HAL_GPIO_ReadPin(GPIOB, GPIO_PIN_12) == GPIO_PIN_RESET) {
    HAL_Delay(10);                     // 等待抖动过去
    if (HAL_GPIO_ReadPin(GPIOB, GPIO_PIN_12) == GPIO_PIN_RESET) {
        // 确认为有效按下
        HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_7);
        while (HAL_GPIO_ReadPin(GPIOB, GPIO_PIN_12) == GPIO_PIN_RESET); // 等待释放
    }
}
```

检测到电平变化后，延时10ms以上再读一次以确保错过抖动时间，若状态一致则认为稳定。但 `HAL_Delay` 会阻塞整个程序，期间无法响应其他任务。 **仅适用于简单实验** 。

### 2. 定时器法

```c
// 在 SysTick_Handler 中（每 1ms 进入一次）
static uint8_t keyFilter = 0;
keyFilter = (keyFilter << 1) | HAL_GPIO_ReadPin(GPIOB, GPIO_PIN_12);
if ((keyFilter & 0x1F) == 0x00) {   // 连续多次为0才认为按下
    // 按键有效
}
```

可以简单记录上一次读到的电平，每隔 10ms 在定时器中断中读取一次按键，然后比较。由于 10ms 采样一次，高频抖动会被忽略。

这是一种**数字滤波**思路，需要理解位操作。

### 3. 非阻塞状态机消抖

利用主循环的周期性查询，通过记录电平持续稳定的时间来判定有效动作。

代码略复杂，需要定时器或系统滴答，还需要写状态机。

而我不想那么快写状态机，所以这个我们先空着。欸嘿，评论区看见你们的做法呀。
