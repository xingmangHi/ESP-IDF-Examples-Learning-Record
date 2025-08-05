# SPI slave sender SPI从机发送

## 文档简介

本例演示了如何配置从机设备向主机设备进行被动的发送数据

## 构建项目

* 配置CMake(可选)
* 选择目标芯片
* 构建项目

## 代码分析

### app_main函数

1. `spi_bus_config_t` 配置SPI总线驱动
2. `spi_device_interface_config_t` 配置SPI设备
   * `command_bits` 命令默认位数
   * `address_bits` 地址默认位数
   * `dummy_bits` 地址和数据阶段插入位数
   * `clock_speed_hz` SPI时钟速度
   * `duty_cycle_pos` 正时钟的占空比
   * `mode` 模式
   * `spics_io_num` CS引脚
   * `cs_ena_posttrans` 传输后cs应保持活动状态的SPI位周期数
   * `queue_size` 队列大小
3. GPIO配置和SPI数据储存初始化
4. GPIO初始化不作赘述
5. 创建信号量
6. `spi_bus_initialize` 初始化SPI总线。 `spi_bus_add_device` 注册连接到总线的设备
7. 给第一次信号量，启动传输
8. `snprintf` 用于格式化字符串并写入指定缓存区，同时限制最大写入字符数
9. 等待信号量，进行传输
10. `spi_bus_remove_device` 从总线上移除特定设备

```c
//Main application
void app_main(void)
{
    esp_err_t ret;
    spi_device_handle_t handle;

    //Configuration for the SPI bus
    spi_bus_config_t buscfg = {
        .mosi_io_num = GPIO_MOSI,
        .miso_io_num = GPIO_MISO,
        .sclk_io_num = GPIO_SCLK,
        .quadwp_io_num = -1,
        .quadhd_io_num = -1
    };

    //Configuration for the SPI device on the other side of the bus
    spi_device_interface_config_t devcfg = {
        .command_bits = 0,
        .address_bits = 0,
        .dummy_bits = 0,
        .clock_speed_hz = 5000000,
        .duty_cycle_pos = 128,      //50% duty cycle
        .mode = 0,
        .spics_io_num = GPIO_CS,
        .cs_ena_posttrans = 3,      //Keep the CS low 3 cycles after transaction, to stop slave from missing the last bit when CS has less propagation delay than CLK
        .queue_size = 3
    };

    //GPIO config for the handshake line.
    gpio_config_t io_conf = {
        .intr_type = GPIO_INTR_POSEDGE,
        .mode = GPIO_MODE_INPUT,
        .pull_up_en = 1,
        .pin_bit_mask = BIT64(GPIO_HANDSHAKE),
    };

    int n = 0;
    char sendbuf[128] = {0};
    char recvbuf[128] = {0};
    spi_transaction_t t;
    memset(&t, 0, sizeof(t));

    //Create the semaphore.
    rdySem = xSemaphoreCreateBinary();

    //Set up handshake line interrupt.
    gpio_config(&io_conf);
    gpio_install_isr_service(0);
    gpio_set_intr_type(GPIO_HANDSHAKE, GPIO_INTR_POSEDGE);
    gpio_isr_handler_add(GPIO_HANDSHAKE, gpio_handshake_isr_handler, NULL);

    //Initialize the SPI bus and add the device we want to send stuff to.
    ret = spi_bus_initialize(SENDER_HOST, &buscfg, SPI_DMA_CH_AUTO);
    assert(ret == ESP_OK);
    ret = spi_bus_add_device(SENDER_HOST, &devcfg, &handle);
    assert(ret == ESP_OK);

    //Assume the slave is ready for the first transmission: if the slave started up before us, we will not detect
    //positive edge on the handshake line.
    xSemaphoreGive(rdySem);

    while (1) {
        int res = snprintf(sendbuf, sizeof(sendbuf),
                           "Sender, transmission no. %04i. Last time, I received: \"%s\"", n, recvbuf);
        if (res >= sizeof(sendbuf)) {
            printf("Data truncated\n");
        }
        t.length = sizeof(sendbuf) * 8;
        t.tx_buffer = sendbuf;
        t.rx_buffer = recvbuf;
        //Wait for slave to be ready for next byte before sending
        xSemaphoreTake(rdySem, portMAX_DELAY); //Wait until slave is ready
        ret = spi_device_transmit(handle, &t);
        printf("Received: %s\n", recvbuf);
        n++;
    }

    //Never reached.
    ret = spi_bus_remove_device(handle);
    assert(ret == ESP_OK);
}
```

### 回调函数

SPI从设备没有主动发送数据的能力，是主机读从机数据而触发的发送。SPI协议下，主机要接收数据必须发送数据，传输的过程进行数据交换。

该函数是GPIO中断的函数，即定义的**GPIO_HANDSHAKE**引脚触发中断。

1. 由于外部干扰，可能会连续收到多个中断，通过`esp_timer_get_time`获取系统的时间戳并和前一次的时间比较，间隔时间太短就不处理这次中断
2. 中断触发代表主机发起数据请求，在中断中给信号量，让休眠的循环程序发送数据
3. `portYIELD_FROM_ISR` 是 FreeRTOS 中 ISR 专用的“任务切换触发器”，用于确保中断唤醒的高优先级任务第一时间执行，保障系统实时性

```c
/*
This ISR is called when the handshake line goes high.
*/
static void IRAM_ATTR gpio_handshake_isr_handler(void* arg)
{
    //Sometimes due to interference or ringing or something, we get two irqs after each other. This is solved by
    //looking at the time between interrupts and refusing any interrupt too close to another one.
    static uint32_t lasthandshaketime_us;
    uint32_t currtime_us = esp_timer_get_time();
    uint32_t diff = currtime_us - lasthandshaketime_us;
    if (diff < 1000) {
        return; //ignore everything <1ms after an earlier irq
    }
    lasthandshaketime_us = currtime_us;

    //Give the semaphore.
    BaseType_t mustYield = false;
    xSemaphoreGiveFromISR(rdySem, &mustYield);
    if (mustYield) {
        portYIELD_FROM_ISR();
    }
}
```

## 总结

笔者逐渐对信号量和freertos有更深刻的体验和了解，本例中笔者也去查询[资料](https://blog.csdn.net/as480133937/article/details/105764119)对SPI通信的传输有更深刻的了解。SPI从机配置步骤和主机有相似之处，但由于从机并不能主动发送数据，所以很多配置都从简，有限配置的作用笔者还对应不上，等以后再进行尝试。
