6.3  中断

中断的缘故千奇百怪，包括指令出错，定时器，串口接收数据或者GPIO电平变化。以GPIO电平变化为例，我们深入讲讲中断。

### 外部中断

触发源来自外部的中断。而STM32依赖引脚与外部沟通，所以也能看做GPIO电平变化原因

为了展示中断的能力，我们先写一个不是那么规范的代码，在GPIO7的小灯反复闪烁的同时，实现按钮 (浮空输入 + 外部上拉电阻) 控制的GPIO4小灯的开关能力。

#### （1）闪烁小灯代码

```c
int main(void)
{
  HAL_Init();
  SystemClock_Config();
  MX_GPIO_Init();

  while (1)
  {
	  HAL_GPIO_WritePin(GPIOA, GPIO_PIN_7, GPIO_PIN_SET);
	  HAL_Delay(1000);
	  HAL_GPIO_WritePin(GPIOA, GPIO_PIN_7, GPIO_PIN_RESET);
	  HAL_Delay(1000);
  }
}
```

* PA7 上的 LED 以 1 秒周期闪烁。

#### （2）按钮控制代码

本函数被写在stm32f1xx_it.c文件中，“it”代表“interrupt”，也就是中断。

```c
void EXTI15_10_IRQHandler(void)
{
  HAL_Delay(10);
  if(HAL_GPIO_ReadPin(GPIOB, GPIO_PIN_12) == GPIO_PIN_RESET)
	HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_4);

  HAL_GPIO_EXTI_IRQHandler(GPIO_PIN_12);
}

```

* 当 PB12 检测到下降沿（按键按下）时，进入中断。
* 首先延时 10ms（消抖），然后读取 PB12 电平，如果仍然为低，则翻转 PA4 上的 LED。
* 最后调用 HAL 库的通用中断处理函数 `HAL_GPIO_EXTI_IRQHandler`，该函数会清除中断标志并调用用户回调。

使用上述代码时注意将 *EXTI15_10_IRQHandler* 的中断优先级设置得比 *SysTick中断* 更低。`HAL_Delay` 依赖 SysTick 中断实现。

但实际上，在中断里用延时时绝对不被推荐的做法。抛开时间问题不谈，在中断中调用中断本身就很容易导致程序卡死。更好的做法是让中断只记录一个时间戳，主循环或定时器中判断稳定。

就以上的判断过程，相信你已经知晓中断的一般作用了。接下对于这个可能的阻碍我们必须进行解决。比较好的办法是如此前一般利用计时器来处理PA7 上的 LED 闪烁，这样程序完全不会因为Delay阻塞，但在实际动手前，我们可以先解决这个在中断中使用中断的问题：

### 解决方式：

中断中仅设置标志，主循环处理按钮实现代码，并且对小灯闪烁采取非阻塞实现：

```c
volatile uint8_t pb12_press_flag = 0;

void EXTI15_10_IRQHandler(void) {
    HAL_GPIO_EXTI_IRQHandler(GPIO_PIN_12);  // 先清除标志，避免重复进入
    pb12_press_flag = 1;                    // 通知主循环
}

int main(void) {
    HAL_Init();
    SystemClock_Config();
    MX_GPIO_Init();

    uint32_t last_toggle_time = 0; // 上一次翻转的时间戳（ms）

    while (1)
    {
	// 非阻塞的小灯闪烁
        // 获取当前系统时间（ms）
        uint32_t now = HAL_GetTick();

        // 检查是否到达翻转时间
        if ((now - last_toggle_time) >= 1000)
        {
            last_toggle_time = now;               // 更新时间戳
            HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_7);
        }

        // 在这里可以添加其他非阻塞任务
	// 但此处实现略微阻塞的简单软件消抖
        if (pb12_press_flag) {
            pb12_press_flag = 0;
            // 可以在这里加消抖（例如再次读取并延时），但注意不要阻塞太久
            HAL_Delay(10);
            if (HAL_GPIO_ReadPin(GPIOB, GPIO_PIN_12) == GPIO_PIN_RESET) {
                HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_4);
            }

    }
}
```

* 注意：volatile uint8_t pb12_press_flag = 0; 设定一个全局变量，保证pb12_press_flag可以同时被两个不同文件的中断函数和主函数访问。

当然，更好的方式是使用上一篇张讲过的简单时间状态机进行消抖。具体怎样实现呢？评论区告诉我哦。
