# Real Time State 实时状态

## 粗略阅读README文档

文档简介本示例演示如何在指定持续时间内获取任务的CPU统计情况，即**Real Time State**

项目配置说明

构建烧录示例输出

示例说明：例程创建了多个任务，任务会运行一定数量的CPU周期，然后阻塞一段时间(*文中采用spin来形容tasks，与笔者所写可能有偏差*)

## 构建烧录监视

* 选定合适的芯片型号
* 选择端口号
* 配置项目
![Enable1](r1.png)
![Enable2](r2.png)
* 点击**构建、烧录和监视**
* 窗口输出
![窗口输出](r3.png)

## 输出分析

在100ticks时间内，有如下任务运行，展示其运行时间和CPU占用率。**Run Time** 表示 **实际占用CPU时间** ，此处可能的单位是 **微秒（μs）** ； **Percentage** 表示该任务占总统计时间的百分比，也可以看作是CPU占用率

|任务名称|运行时间（单位统一）|占总运行时间比例（CPU占用率）|
|:-----:|:-----------------:|:------------------------:|
|stats|495|0%|
|spin1|251992|12%|
|spin4|251064|12%|
|spin3|227436|11%|
|IDLE1|323966|16%|
|IDLE2|216754|10%|
|spin2|251991|12%|
|spin0|250710|12%|
|spin5|227082|11%|
|ipc1|0|0%|
|ipc0|0|0%|
|总计|2001490|96%|

由上图可以看出，整个CPU运行被拆成了很多部分，**每个任务占用一部分**，这就是FREERTOS的实现原理

## 代码分析

### app_main()

> 函数相关注释写在下方代码块内

main函数中程序先进行延时，确保所有**核初始化完成**。然后**创建信号量**的实例。接着通过循环在核中**创建任务**。最后创建一个state任务并**释放了信号量**，猜测用于通知其他任务*state任务已被创建*[信号量官方链接](https://docs.espressif.com/projects/esp-idf/zh_CN/stable/esp32/api-reference/system/freertos_idf.html#id29) [任务创建API](https://docs.espressif.com/projects/esp-idf/zh_CN/stable/esp32/api-reference/system/freertos_additions.html#_CPPv423xTaskCreatePinnedToCore14TaskFunction_tPCKcK8uint32_tPCv11UBaseType_tPC12TaskHandle_tK10BaseType_t)

```c
void app_main(void)
{
    //Allow other core to finish initialization
    vTaskDelay(pdMS_TO_TICKS(100));

    //Create semaphores to synchronize
    sync_spin_task = xSemaphoreCreateCounting(NUM_OF_SPIN_TASKS, 0); //创建计数信号量实例，并返回一个句柄
    sync_stats_task = xSemaphoreCreateBinary(); //创建二进制信号量实例，并返回一个句柄

    //Create spin tasks
    for (int i = 0; i < NUM_OF_SPIN_TASKS; i++) {
        snprintf(task_names[i], configMAX_TASK_NAME_LEN, "spin%d", i); //将格式化的字符串写入task_names[i],用于生成格式化的任务名称
        xTaskCreatePinnedToCore(spin_task, task_names[i], 1024, NULL, SPIN_TASK_PRIO, NULL, tskNO_AFFINITY); //创建固定到特定核心的新任务,takNO_AFFINITY 代表任务没有指定的核心
    }

    //Create and start stats task
    xTaskCreatePinnedToCore(stats_task, "stats", 4096, NULL, STATS_TASK_PRIO, NULL, tskNO_AFFINITY);
    xSemaphoreGive(sync_stats_task);
}
```

### 预定义和任务静态函数

预定义了**SPIN_TASK任务数量**，每个任务**执行50万次空操作**，**spin优先级为2**，**state优先级为3**，每**隔1000ms打印一次**系统操作，ARRAY_SIZE_OFFSET 定义**内部缓冲区**大小。后续定义了全局变量，包括**任务名**数组，静态**信号量**。
`spin_task()`函数获取信号量然后进行空操作；`stats_task()`函数负责释放信号量，用于让spin_task启动还有每次输出提示信息。

```c
#define NUM_OF_SPIN_TASKS   6
#define SPIN_ITER           500000  //Actual CPU cycles used will depend on compiler optimization
#define SPIN_TASK_PRIO      2
#define STATS_TASK_PRIO     3
#define STATS_TICKS         pdMS_TO_TICKS(1000)
#define ARRAY_SIZE_OFFSET   5   //Increase this if print_real_time_stats returns ESP_ERR_INVALID_SIZE

static char task_names[NUM_OF_SPIN_TASKS][configMAX_TASK_NAME_LEN];
static SemaphoreHandle_t sync_spin_task;
static SemaphoreHandle_t sync_stats_task;

static void spin_task(void *arg)
{
    xSemaphoreTake(sync_spin_task, portMAX_DELAY);
    while (1) {
        //Consume CPU cycles
        for (int i = 0; i < SPIN_ITER; i++) {
            __asm__ __volatile__("NOP");
        }
        vTaskDelay(pdMS_TO_TICKS(100));
    }
}

static void stats_task(void *arg)
{
    xSemaphoreTake(sync_stats_task, portMAX_DELAY);

    //Start all the spin tasks
    for (int i = 0; i < NUM_OF_SPIN_TASKS; i++) {
        xSemaphoreGive(sync_spin_task);
    }

    //Print real time stats periodically
    while (1) {
        printf("\n\nGetting real time stats over %"PRIu32" ticks\n", STATS_TICKS);
        if (print_real_time_stats(STATS_TICKS) == ESP_OK) {
            printf("Real time stats obtained\n");
        } else {
            printf("Error getting real time stats\n");
        }
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}
```

### 打印函数

> 此函数用于打印在一段给定时间内的CPU使用情况
>
> 此函数将测量并打印指定时间内任务的CPU使用情况
> 滴答数（即实时统计数据）。这可以通过简单地调用来实现
> uxTaskGetSystemState（）两次以延迟分隔，然后计算延迟前后任务运行时间的差异
>
> 注意：
> 如果在延迟期间增加或移除了任务，则不会打印这些任务的统计数据。
> 高优先级调用时，需考虑任务延时性
> 双核心运行下，每个核心运行时间为50%
>
> 参数：
> `xTicksToWait` 统计测量周期
>
> 返回值：
>
> * ESP_OK 成功
> * ESP_ERR_NO_MEM 内存不足，无法分配内部阵列
> * ESP_ERR_INVALID_SIZE *uxTaskGetSystemState*的数组大小不足。尝试增加*ARRAY_SIZE_OFFSET*
> * ESP_ERR_INVALID_STATE 延时太短
>

[任务API](https://docs.espressif.com/projects/esp-idf/zh_CN/stable/esp32/api-reference/system/freertos_idf.html#id23)

函数首先定义了两个指针，指向**任务状态数组**；两个变量记录数组中**任务状态数量**；两个变量记录两次获取**任务状态的系统时间**，用于计算时间间隔。

先使用 `uxTaskGetNumberOfTasks` 函数获取当前系统任务数量，加上偏移量后采用 `malloc`分配内存。接着用 `uxTaskGetSystemState` 获取系统任务状态和系统时间。 `vTaskDelay` 延时足够时间给任务运行。

重复**获取系统量**的流程，计算**系统时间差**。遍历 start_array 中的每个任务状态，在 end_array 中查找具有相同任务句柄的任务状态。如果找到匹配的任务，计算该任务在两次获取状态之间的运行时间差。同时进行百分比转换。

如果有不匹配的任务，即在延时期间创建或删除的任务，对任务进行打印，并标注创建或删除

```c
/**
 * @brief   Function to print the CPU usage of tasks over a given duration.
 *
 * This function will measure and print the CPU usage of tasks over a specified
 * number of ticks (i.e. real time stats). This is implemented by simply calling
 * uxTaskGetSystemState() twice separated by a delay, then calculating the
 * differences of task run times before and after the delay.
 *
 * @note    If any tasks are added or removed during the delay, the stats of
 *          those tasks will not be printed.
 * @note    This function should be called from a high priority task to minimize
 *          inaccuracies with delays.
 * @note    When running in dual core mode, each core will correspond to 50% of
 *          the run time.
 *
 * @param   xTicksToWait    Period of stats measurement
 *
 * @return
 *  - ESP_OK                Success
 *  - ESP_ERR_NO_MEM        Insufficient memory to allocated internal arrays
 *  - ESP_ERR_INVALID_SIZE  Insufficient array size for uxTaskGetSystemState. Trying increasing ARRAY_SIZE_OFFSET
 *  - ESP_ERR_INVALID_STATE Delay duration too short
 */
static esp_err_t print_real_time_stats(TickType_t xTicksToWait)
{
    TaskStatus_t *start_array = NULL, *end_array = NULL;
    UBaseType_t start_array_size, end_array_size;
    configRUN_TIME_COUNTER_TYPE start_run_time, end_run_time;
    esp_err_t ret;

    //Allocate array to store current task states
    start_array_size = uxTaskGetNumberOfTasks() + ARRAY_SIZE_OFFSET;
    start_array = malloc(sizeof(TaskStatus_t) * start_array_size);
    if (start_array == NULL) {
        ret = ESP_ERR_NO_MEM;
        goto exit;
    }
    //Get current task states
    start_array_size = uxTaskGetSystemState(start_array, start_array_size, &start_run_time);
    if (start_array_size == 0) {
        ret = ESP_ERR_INVALID_SIZE;
        goto exit;
    }

    vTaskDelay(xTicksToWait);

    //Allocate array to store tasks states post delay
    end_array_size = uxTaskGetNumberOfTasks() + ARRAY_SIZE_OFFSET;
    end_array = malloc(sizeof(TaskStatus_t) * end_array_size);
    if (end_array == NULL) {
        ret = ESP_ERR_NO_MEM;
        goto exit;
    }
    //Get post delay task states
    end_array_size = uxTaskGetSystemState(end_array, end_array_size, &end_run_time);
    if (end_array_size == 0) {
        ret = ESP_ERR_INVALID_SIZE;
        goto exit;
    }

    //Calculate total_elapsed_time in units of run time stats clock period.
    uint32_t total_elapsed_time = (end_run_time - start_run_time);
    if (total_elapsed_time == 0) {
        ret = ESP_ERR_INVALID_STATE;
        goto exit;
    }

    printf("| Task | Run Time | Percentage\n");
    //Match each task in start_array to those in the end_array
    for (int i = 0; i < start_array_size; i++) {
        int k = -1;
        for (int j = 0; j < end_array_size; j++) {
            if (start_array[i].xHandle == end_array[j].xHandle) {
                k = j;
                //Mark that task have been matched by overwriting their handles
                start_array[i].xHandle = NULL;
                end_array[j].xHandle = NULL;
                break;
            }
        }
        //Check if matching task found
        if (k >= 0) {
            uint32_t task_elapsed_time = end_array[k].ulRunTimeCounter - start_array[i].ulRunTimeCounter;
            uint32_t percentage_time = (task_elapsed_time * 100UL) / (total_elapsed_time * CONFIG_FREERTOS_NUMBER_OF_CORES);
            printf("| %s | %"PRIu32" | %"PRIu32"%%\n", start_array[i].pcTaskName, task_elapsed_time, percentage_time);
        }
    }

    //Print unmatched tasks
    for (int i = 0; i < start_array_size; i++) {
        if (start_array[i].xHandle != NULL) {
            printf("| %s | Deleted\n", start_array[i].pcTaskName);
        }
    }
    for (int i = 0; i < end_array_size; i++) {
        if (end_array[i].xHandle != NULL) {
            printf("| %s | Created\n", end_array[i].pcTaskName);
        }
    }
    ret = ESP_OK;

exit:    //Common return path
    free(start_array);
    free(end_array);
    return ret;
}
```

## 总结

本例程展示了FREERTOS的原理。创建了几个任务，并直观地展示它们的运行时间，CPU占用率。接触了FREERTOS信号量和系统状态的几个API。但对系统具体运行和架构感受不深，留待后续实验学习。
