6.1流水灯

采用STM32F103C8T6最小开发板，软件STM32cubeIDE进行开发

主要修改代码如下：

```
c
  /* USER CODE BEGIN 2 */
  int a=0;  //在while循环外的先定义一个状态机计数器

  /* USER CODE END 2 */

int main(void)
{

  SystemClock_Config();

  MX_GPIO_Init();
  /* USER CODE BEGIN 2 */
  int a=0;
  /* USER CODE END 2 */

  /* Infinite loop */
  /* USER CODE BEGIN WHILE */
  while (1)
  {
    /* USER CODE END WHILE */
	  if (a <= 2)
		  HAL_GPIO_WritePin(GPIOA, GPIO_PIN_7, GPIO_PIN_SET);
	  else
		  HAL_GPIO_WritePin(GPIOA, GPIO_PIN_7, GPIO_PIN_RESET);
	  if (a>=1 && a<=3)
		  HAL_GPIO_WritePin(GPIOA, GPIO_PIN_4, GPIO_PIN_SET);
	  else
		  HAL_GPIO_WritePin(GPIOA, GPIO_PIN_4, GPIO_PIN_RESET);
	  if (a>=2 && a<=5)
		  HAL_GPIO_WritePin(GPIOA, GPIO_PIN_1, GPIO_PIN_SET);
	  else
		  HAL_GPIO_WritePin(GPIOA, GPIO_PIN_1, GPIO_PIN_RESET);
	  a++;

	    if(a==6){
		  HAL_GPIO_WritePin(GPIOA, GPIO_PIN_7, GPIO_PIN_RESET);
		  HAL_GPIO_WritePin(GPIOA, GPIO_PIN_4, GPIO_PIN_RESET);
		  HAL_GPIO_WritePin(GPIOA, GPIO_PIN_1, GPIO_PIN_RESET);
		  a=0;
	  }
	  HAL_Delay(1000);
    /* USER CODE BEGIN 3 */
  }
  /* USER CODE END 3 */
}
```

以上是采用HAL库完成的流水灯，同时采用了HAL_Delay函数。

倘若到此为止便还是低效了一些。接下来我们试着理解HAL库操作的底层，试着寄存器电灯，并且找到HAL_Delay以外的办法。毕竟它内部是一个死循环，不断检查一个叫 `uwTick` 的变量是否达到你指定的毫秒数，此时间段内完全占用CPU，无法做任何其他事情。虽说也可以采取中断的方式解决，但防止滥用HAL_Delay总是好事

#### 首先，理解寄存器基础操作

寄存器操作，说白了就是直接往芯片手册上写的特定内存地址读写数据 。在 C 语言里，我们用指针来完成这件事。

指针 + 地址 = 寄存器

```
c
#define 寄存器名称  (*(volatile uint32_t *) 寄存器地址)
```

* `寄存器地址`：芯片手册上给出的一个十六进制数，比如 GPIOA 的时钟使能寄存器是 `0x40021018`。
* `(uint32_t *)`：把这个数字强制转换成指向 32 位地址的指针。
* `volatile`：告诉编译器，这个地址上的值可能随时被硬件改变，千万别优化掉我的读写操作！ 每次用到都必须重新从地址读或写到地址。
* `*`：解引用指针，让你能像普通变量一样读写这个地址里的内容。

所谓的 `GPIOA->BSRR`，其实就是在底层头文件里用类似方式定义好的。

#### 其次，Delay代替方案

非阻塞方法，即使用一个变量记录上次时间，然后每次循环检查当前时间和上次时间的差值，如果超过阈值就执行任务并更新时间记录。这样循环可以快速运行，不断检查时间，同时也能执行其他代码。

HAL_Delay的本质：

```
c
uint32_t start = HAL_GetTick();   // 记录开始时的时间
while ((HAL_GetTick() - start) < 1000) {
    // 空转，什么都不做，一直等到 1000ms 过去
}
```

改进之后：

```
c
// 定义在 while(1) 外面
uint32_t last_tick = 0;

while (1) {
    // 每次循环都看一眼时间，时间到了便执行代码
    if (HAL_GetTick() - last_tick >= 1000) {  
        last_tick = HAL_GetTick();   // 更新上次执行的时间点
        // 这里放你的流水灯逻辑
    }
    // CPU 马上去干别的事，比如读按键、检查串口（以下为实例，实际上也的确可以用中断代替）
    Check_Button();   // 按键扫描
    Check_UART();     // 串口数据处理
}
```

综上，我们可以开始对源代码进行改进了

### 对应代码参考如下：

```
c
/* USER CODE BEGIN 2 */
int a = 0;
uint32_t last_tick = 0;   // 加上这一行，记录上一次执行的时间点
/* USER CODE END 2 */

 while (1)
  {
    /* USER CODE END WHILE */
	  if (HAL_GetTick() - last_tick >= 1000)
	      {
	        last_tick = HAL_GetTick();

	        // ---- PA7 ----
	        if (a <= 2)
	          GPIOA->BSRR = (1 << 7);   // 输出高电平
	        else
	          GPIOA->BRR  = (1 << 7);   // 输出低电平

	        // ---- PA4 ----
	        if (a >= 1 && a <= 3)
	          GPIOA->BSRR = (1 << 4);
	        else
	          GPIOA->BRR  = (1 << 4);

	        // ---- PA1 ----
	        if (a >= 2 && a <= 5)
	          GPIOA->BSRR = (1 << 1);
	        else
	          GPIOA->BRR  = (1 << 1);

	        a++;
	        if (a == 6){
	        	GPIOA->BRR  = (1 << 7);
	        	GPIOA->BRR  = (1 << 4);
	        	GPIOA->BRR  = (1 << 1);
	        	  a = 0;
	        }


    /* USER CODE BEGIN 3 */

	      }
  /* USER CODE END 3 */
  }
}
```

该可以直接覆盖此前HAL库代码

至于为什么要学寄存器电灯……只是做一个了解。就像开车的人自动挡就够了，但是学学原理还是要摸摸手动挡的

懂寄存器的人能直接看到硬件内部的真实状态，有时候会给你很多惊喜的。

所以，即使使用HAL库，应该用什么样的代码代替HAL_Delay呢？评论区看见你哦！

hh，开玩笑的，实际上这样改动一下就行了。

不过，也希望看见你自己动动脑。毕竟学会原理是一回事，AI打工是另一回事：

```
c
int a = 0;
uint32_t last_tick = 0;

int main(void) {
    HAL_Init();
    SystemClock_Config();
    MX_GPIO_Init();

    int a = 0;
    uint32_t last_tick = 0;

    while (1) {
        if (HAL_GetTick() - last_tick >= 1000) {
            last_tick = HAL_GetTick();

            if (a <= 2)      HAL_GPIO_WritePin(GPIOA, GPIO_PIN_7, GPIO_PIN_SET);
            else             HAL_GPIO_WritePin(GPIOA, GPIO_PIN_7, GPIO_PIN_RESET);

            if (a>=1 && a<=3) HAL_GPIO_WritePin(GPIOA, GPIO_PIN_4, GPIO_PIN_SET);
            else              HAL_GPIO_WritePin(GPIOA, GPIO_PIN_4, GPIO_PIN_RESET);

            if (a>=2 && a<=5) HAL_GPIO_WritePin(GPIOA, GPIO_PIN_1, GPIO_PIN_SET);
            else              HAL_GPIO_WritePin(GPIOA, GPIO_PIN_1, GPIO_PIN_RESET);

            a++;
            if(a==6){
		HAL_GPIO_WritePin(GPIOA, GPIO_PIN_7, GPIO_PIN_RESET);
		HAL_GPIO_WritePin(GPIOA, GPIO_PIN_4, GPIO_PIN_RESET);
		HAL_GPIO_WritePin(GPIOA, GPIO_PIN_1, GPIO_PIN_RESET);
		a=0;
		}
        }
    }
}
```
