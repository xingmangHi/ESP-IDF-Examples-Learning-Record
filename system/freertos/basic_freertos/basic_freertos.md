# FreeRTOS basic API FREERTOS基础API

## 粗略阅读README文档

文档简介例程演示了FREERTOS在SMP架构中一些有用的API，包括任务创建、队列、互斥锁、任务标记等。

简要介绍文档结构

简单介绍 **任务创建**及示例输出，**通过队列数据传输**及示例输出，**互斥锁、自旋锁、原子操作**的比较和示例输出，**任务通知**通信及示例输出，前几种方式**集成示例**及输出

> 如何使用此示例
> 此示例使用了一个交互式控制台组件，以便您可以选择要通过终端运行的部分。您可以键入'help'以获取命令列表，使用UPDOWN arows浏览命令历史记录；键入命令名时按TAB键自动完成。有关交互式控制台组件的更多信息，请参阅控制台。支持的命令包括：
> .help：获取命令列表
> .create_task：运行创建任务示例
> .queue：运行队列示例
> .lock：运行locks示例
> .task_notiation：运行任务通知示例
> .batch_processing：运行批处理示例
> 一旦组件开始运行，它将在大约5秒内停止运行。如果希望延长运行时间，请修改headerfilinc.h.中宏COMP_LOOP_PERIOD的值。

## 构建烧录和监视

* 选择目标芯片
* 选择端口号
* 点击**构建、烧录和监视**
* 监视输出，似乎是在芯片内搭建了一个系统
![监视输出](b1.png)

### 任务创建

在窗口中输入 `create_task`。从窗口输出可以看到，提示我们任务被创建，然后提示我们任务在核0或核1上运行
 ![任务创建](b2.png)
  
### 队列数据

在窗口中输入 `queue`。从窗口数据结果可以看到队列数据有发送，有接收
![队列数据](b3.png)

### 锁相关

在窗口中输入 `lock` 。首先是 **互斥锁(mutex)** 的输出提示，然后是**自旋锁(spinlock)** 的输出提示，然后是 **原子信号(atomic)** 的输出提示。接着创建了两个互斥任务，分别去设置和读取同一个value，可以感觉到 **同一个value不会同时被两个任务访问** 。
![锁](b4.png)

### 任务通知

在窗口中输入 `task_notification` 。提示我们一个任务发送任务通知，一个任务等待，然后其处理任务通知。多次循环，说明**一次发送只有一次处理。**
![任务通知](b5.png)

### 集成示例

在窗口中输入 `batch_processing` 。首先队列中加入了一组数据，然后读取出一组数据，从提示中可以看到，满足 **队列先进先出(FIFO)** 。然后提示队列中数据为0。

> 文档示例说明
> 在最后一部分中，提供了一个实际演示，其中队列、互斥体和任务通知被集成在一起以实现一个真实的工作流程，从而展示了它们在现实世界场景中的实用性。
> 名为 **rcv数据任务** 的任务模拟接收不规则到达的数据。每次接收到一个数据项时，它都会被推入队列，接收到的项目编号会增加1；一旦任务收集了5个数据项，它就会向 **proc数据任务** 发送任务通知，以处理队列中的这批数据。当后一个任务完成处理时，它将把接收到的项目编号减少5。因为这两个任务都可以修改这个全局编号，所以修改操作受到互斥体的保护。

![集成示例](b6.png)

## 代码分析

> 代码分析借助AI，帮助理解，会有错误

### app_main()

**app_main** 中只调用了 `config_console`函数，该函数*把一个 串口终端（REPL） 搭起来，让用户可以在运行时通过命令行交互式地启动 FreeRTOS 示例的各个子模块。一句话：它把“示例菜单”做成了串口命令。*
具体注释写在代码旁边

```c
static void config_console(void)
{
    esp_console_repl_t *repl = NULL; //运行期控制台实例句柄 REPL
    esp_console_repl_config_t repl_config = ESP_CONSOLE_REPL_CONFIG_DEFAULT(); //采用默认REPL配置
    /* Prompt to be printed before each line. 每行前提示打印
     * This can be customized, made dynamic, etc. 这可以定制，动态化
     */
    repl_config.prompt = PROMPT_STR ">"; //定制提示符
    repl_config.max_cmdline_length = 1024; //单行命令最长1024字节
    esp_console_dev_uart_config_t uart_config = ESP_CONSOLE_DEV_UART_CONFIG_DEFAULT(); //串口底层数据配置，如波特率，引脚等
    ESP_ERROR_CHECK(esp_console_new_repl_uart(&uart_config, &repl_config, &repl)); //把串口驱动与 REPL 绑定，生成 repl 实例

    esp_console_register_help_command(); //注册内建help指令，用户输入help就能看到所以已注册命令

    // register entry functions for each component
    // 注册各命令实例
    register_creating_task();
    register_queue();
    register_lock();
    register_task_notification();
    register_batch_proc_example();

    ESP_ERROR_CHECK(esp_console_start_repl(repl)); //启动
    printf("\n"
           "Please type the component you would like to run.\n");
}

void app_main(void)
{
    config_console();
}
```

### 宏定义和register函数

在 **头文件(.h)** 中 `CONFIG_IDF_TARGET` 是esp-idf构建系统时生成的宏，代表当前芯片型号。后定义任务优先级、循环周期、错误提示字符串。
声明各命令注册的实际函数 (*该头文件在后续各单独文件中有引用，此处统一注册函数* )

```c
#define PROMPT_STR CONFIG_IDF_TARGET 
#define TASK_PRIO_3         3
#define TASK_PRIO_2         2
#define COMP_LOOP_PERIOD    5000
#define SEM_CREATE_ERR_STR                "semaphore creation failed"
#define QUEUE_CREATE_ERR_STR              "queue creation failed"

int comp_creating_task_entry_func(int argc, char **argv);
int comp_queue_entry_func(int argc, char **argv);
int comp_lock_entry_func(int argc, char **argv);
int comp_task_notification_entry_func(int argc, char **argv);
int comp_batch_proc_example_entry_func(int argc, char **argv);
```

注册函数的写法基本相同，`.command` 确定 **启动指令** ；`.help` 确定该命令的 **帮助文本** ; `hint` 确定该命令的 **提示文本** ；`func` 确定该命令对应的 **调用函数** 。最后采用 `esp_console_cmd_register` **进行注册** 。

```c
static void register_creating_task(void)
{
    const esp_console_cmd_t creating_task_cmd = {
        .command = "create_task",
        .help = "Run the example that demonstrates how to create and run pinned and unpinned tasks",
        .hint = NULL,
        .func = &comp_creating_task_entry_func,
    };
    ESP_ERROR_CHECK(esp_console_cmd_register(&creating_task_cmd));
}

static void register_queue(void)
{
    const esp_console_cmd_t queue_cmd = {
        .command = "queue",
        .help = "Run the example that demonstrates how to use queue to communicate between tasks",
        .hint = NULL,
        .func = &comp_queue_entry_func,
    };
    ESP_ERROR_CHECK(esp_console_cmd_register(&queue_cmd));
}

static void register_lock(void)
{
    const esp_console_cmd_t lock_cmd = {
        .command = "lock",
        .help = "Run the example that demonstrates how to use mutex and spinlock to protect a shared resource",
        .hint = NULL,
        .func = &comp_lock_entry_func,
    };
    ESP_ERROR_CHECK(esp_console_cmd_register(&lock_cmd));
}

static void register_task_notification(void)
{
    const esp_console_cmd_t task_notification_cmd = {
        .command = "task_notification",
        .help = "Run the example that demonstrates how to use task notifications to synchronize tasks",
        .hint = NULL,
        .func = &comp_task_notification_entry_func,
    };
    ESP_ERROR_CHECK(esp_console_cmd_register(&task_notification_cmd));
}

static void register_batch_proc_example(void)
{
    const esp_console_cmd_t batch_proc_example_cmd = {
        .command = "batch_processing",
        .help = "Run the example that combines queue, mutex, task notification together",
        .hint = NULL,
        .func = &comp_batch_proc_example_entry_func,
    };
    ESP_ERROR_CHECK(esp_console_cmd_register(&batch_proc_example_cmd));
}
```

### create_task 创建任务

#### 头文件和宏定义

特别注意调用 **"basic_freertos_smp_usage.h"** 因为 **函数声明** 在该头文件中。
宏定义 `SPIN_ITER` 次数为350000，定义核0和核1

```c
#include "freertos/FreeRTOS.h"
#include "esp_log.h"
#include "basic_freertos_smp_usage.h"


#define SPIN_ITER   350000  //actual CPU cycles consumed will depend on compiler optimization
#define CORE0       0
// only define xCoreID CORE1 as 1 if this is a multiple core processor target, else define it as tskNO_AFFINITY
#define CORE1       ((CONFIG_FREERTOS_NUMBER_OF_CORES > 1) ? 1 : tskNO_AFFINITY)
```

#### 创建任务-命令任务函数

该函数实现任务创建，把两个任务创建在核0上，一个任务创建在核1上，还有一个任务不指定核，进行延时一段时间，观察两个的使用情况。
`xTaskCreatePinnedToCore(任务函数，任务名称，分配内存，传入任务的参数，优先级，任务句柄，指定核)`  [FREERTOS附加功能](https://docs.espressif.com/projects/esp-idf/zh_CN/stable/esp32/api-reference/system/freertos_additions.html#_CPPv429xTaskCreateStaticPinnedToCore14TaskFunction_tPCKcK8uint32_tPCv11UBaseType_tPC11StackType_tPC12StaticTask_tK10BaseType_t)

```c
// Creating task example: show how to create pinned and unpinned tasks on CPU cores
int comp_creating_task_entry_func(int argc, char **argv)
{
    timed_out = false;
    // pin 2 tasks on same core and observe in-turn execution,
    // and pin another task on the other core to observe "simultaneous" execution
    int task_id0 = 0, task_id1 = 1, task_id2 = 2, task_id3 = 3;
    xTaskCreatePinnedToCore(spin_task, "pinned_task0_core0", 4096, (void*)task_id0, TASK_PRIO_3, NULL, CORE0);
    xTaskCreatePinnedToCore(spin_task, "pinned_task1_core0", 4096, (void*)task_id1, TASK_PRIO_3, NULL, CORE0);
    xTaskCreatePinnedToCore(spin_task, "pinned_task2_core1", 4096, (void*)task_id2, TASK_PRIO_3, NULL, CORE1);
    // Create a unpinned task with xCoreID = tskNO_AFFINITY, which can be scheduled on any core, hopefully it can be observed that the scheduler moves the task between the different cores according to the workload
    xTaskCreatePinnedToCore(spin_task, "unpinned_task", 4096, (void*)task_id3, TASK_PRIO_2, NULL, tskNO_AFFINITY);

    // time out and stop running after 5 seconds
    vTaskDelay(pdMS_TO_TICKS(COMP_LOOP_PERIOD));
    timed_out = true;
    // delay to let tasks finish the last loop
    vTaskDelay(500 / portTICK_PERIOD_MS);
    return 0;
}
```

#### 创建任务-实际任务函数

内联汇编指令 `__asm__ __volatile__("NOP")`，表示执行一条空操作指令（NOP）。*NOP 指令不会改变任何寄存器或内存状态，但会消耗 CPU 周期*
`spin_task` 函数用作提示性输出，`spin_iteration` 内部执行空操作，占用CPU

```c
static void spin_iteration(int spin_iter_num)
{
    for (int i = 0; i < spin_iter_num; i++) {
        __asm__ __volatile__("NOP");
    }
}

static void spin_task(void *arg)
{
    // convert arg pointer from void type to int type then dereference it
    int task_id = (int)arg; //获取参数，强制转为int，获取任务编号
    ESP_LOGI(TAG, "created task#%d", task_id);
    while (!timed_out) {
        int core_id = esp_cpu_get_core_id();
        ESP_LOGI(TAG, "task#%d is running on core#%d", task_id, core_id);
        // consume some CPU cycles to keep Core#0 a little busy, so task3 has opportunity to be scheduled on Core#1
        spin_iteration(SPIN_ITER);
        vTaskDelay(pdMS_TO_TICKS(150));
    }

    vTaskDelete(NULL); //删除当前任务，释放任务资源
}
```

### queue 队列

#### 头文件和变量定义

限制队列长度为40，其他不作特别解释

```c
#include "freertos/FreeRTOS.h"
#include "esp_log.h"
#include "basic_freertos_smp_usage.h"


static QueueHandle_t msg_queue;
static const uint8_t msg_queue_len = 40;
static volatile bool timed_out;
const static char *TAG = "queue example";
```

#### 队列-命令任务函数

函数进行队列创建，核任务创建，并延时让核运行。
`queueQUEUE_TYPE_SET` 代表普通类型队列 *该函数为SMP架构下的库函数，笔者未找到官方介绍，建议直接查看源文件*

```c
// Queue example: illustrate how queues can be used to synchronize between tasks
int comp_queue_entry_func(int argc, char **argv)
{
    timed_out = false;

    msg_queue = xQueueGenericCreate(msg_queue_len, sizeof(int), queueQUEUE_TYPE_SET); //队列长度，队列中消息类型，队列类型
    if (msg_queue == NULL) {
        ESP_LOGE(TAG, QUEUE_CREATE_ERR_STR);
        return 1;
    }
    xTaskCreatePinnedToCore(print_q_msg, "print_q_msg", 4096, NULL, TASK_PRIO_3, NULL, tskNO_AFFINITY);
    xTaskCreatePinnedToCore(send_q_msg, "send_q_msg", 4096, NULL, TASK_PRIO_3, NULL, tskNO_AFFINITY);

    // time out and stop running after 5 seconds
    vTaskDelay(pdMS_TO_TICKS(COMP_LOOP_PERIOD));
    timed_out = true;
    // delay to let tasks finish the last loop
    vTaskDelay(500 / portTICK_PERIOD_MS);
    return 0;
}
```

![队列宏定义](b7.png)

#### 队列-实际任务函数

```c
static void print_q_msg(void *arg)
{
    int data;  // data type should be same as queue item type
    int to_wait_ms = 1000;  // the maximal blocking waiting time of millisecond
    const TickType_t xTicksToWait = pdMS_TO_TICKS(to_wait_ms);

    while (!timed_out) {
        if (xQueueReceive(msg_queue, (void *)&data, xTicksToWait) == pdTRUE) {
            ESP_LOGI(TAG, "received data = %d", data);
        } else {
            ESP_LOGI(TAG, "Did not received data in the past %d ms", to_wait_ms);
        }
    }

    vTaskDelete(NULL);
}

static void send_q_msg(void *arg)
{
    int sent_num  = 0;

    while (!timed_out) {
        // Try to add item to queue, fail immediately if queue is full
        if (xQueueGenericSend(msg_queue, (void *)&sent_num, portMAX_DELAY, queueSEND_TO_BACK) != pdTRUE) {
            ESP_LOGI(TAG, "Queue full\n");
        }
        ESP_LOGI(TAG, "sent data = %d", sent_num);
        sent_num++;

        // send an item for every 250ms
        vTaskDelay(250 / portTICK_PERIOD_MS);
    }

    vTaskDelete(NULL);
}
```