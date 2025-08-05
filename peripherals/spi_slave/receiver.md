# SPI slave receiver SPI从机接收

## 文档简介

> 由于文档没有README文件，笔者会在实验后补一份简单介绍

本例展示了SPI从机驱动程序的配置和数据的接收处理

## 构建项目

* 配置CMake(可选)
* 选择目标芯片
* 构建项目

## 分析代码

> 由于没有组件，没有多文件，两个回调函数内容只是改变GPIO电平，不作讲解
> 只对main函数进行分析，主要是SPI从机配置

### app_main函数

1. SPI总线配置，包括各引脚号，写保护和保持信号不启用
2. `spi_slave_interface_config_t` 配置SPI从机
   * `mode` SPI 模式，代表一对（CPOL、CPHA）配置 时钟极性、时钟相位
   * `spics_io_num` CS片选脚
   * `queue_size` 队列大小
   * `post_setup_cb` SPI寄存器加载新数据的回调
   * `post_trans_cb` 事务完成后的回调
3. 引脚GPIO配置，不作赘述
4. 配置数据接收引脚上拉模式
5. `spi_slave_initialize` 将SPI外设初始化为从机设备
6. `spi_bus_dma_memory_alloc` 函数用于为SPI总线分配可用的DMA功能内存[文档解释](https://docs.espressif.com/projects/esp-idf/zh_CN/stable/esp32/api-reference/peripherals/spi_master.html#_CPPv424spi_bus_dma_memory_alloc17spi_host_device_t6size_t8uint32_t)
7. `spi_slave_transaction_t` 作为传输事务配置结构体
8. 在主循环中演示了一种SPI任务从机的方式
   1. 首先将发送的接收都初始化 (*sprintf，将格式化后的字符串写入字符数组*)
   2. 然后绑定内存到传输结构体中
   3. `spi_slave_transmit` 函数进入阻塞状态接收数据，等待数据接收完成
   4. 暂停传输，进行操作，然后再次启动传输

```c
//Main application
void app_main(void)
{
    int n = 0;
    esp_err_t ret;

    //Configuration for the SPI bus
    spi_bus_config_t buscfg = {
        .mosi_io_num = GPIO_MOSI,
        .miso_io_num = GPIO_MISO,
        .sclk_io_num = GPIO_SCLK,
        .quadwp_io_num = -1,
        .quadhd_io_num = -1,
    };

    //Configuration for the SPI slave interface
    spi_slave_interface_config_t slvcfg = {
        .mode = 0,
        .spics_io_num = GPIO_CS,
        .queue_size = 3,
        .flags = 0,
        .post_setup_cb = my_post_setup_cb,
        .post_trans_cb = my_post_trans_cb
    };

    //Configuration for the handshake line
    gpio_config_t io_conf = {
        .intr_type = GPIO_INTR_DISABLE,
        .mode = GPIO_MODE_OUTPUT,
        .pin_bit_mask = BIT64(GPIO_HANDSHAKE),
    };

    //Configure handshake line as output
    gpio_config(&io_conf);
    //Enable pull-ups on SPI lines so we don't detect rogue pulses when no master is connected.
    gpio_set_pull_mode(GPIO_MOSI, GPIO_PULLUP_ONLY);
    gpio_set_pull_mode(GPIO_SCLK, GPIO_PULLUP_ONLY);
    gpio_set_pull_mode(GPIO_CS, GPIO_PULLUP_ONLY);

    //Initialize SPI slave interface
    ret = spi_slave_initialize(RCV_HOST, &buscfg, &slvcfg, SPI_DMA_CH_AUTO);
    assert(ret == ESP_OK);

    char *sendbuf = spi_bus_dma_memory_alloc(RCV_HOST, 129, 0);
    char *recvbuf = spi_bus_dma_memory_alloc(RCV_HOST, 129, 0);
    assert(sendbuf && recvbuf);
    spi_slave_transaction_t t = {0};

    while (1) {
        //Clear receive buffer, set send buffer to something sane
        memset(recvbuf, 0xA5, 129);
        sprintf(sendbuf, "This is the receiver, sending data for transmission number %04d.", n);

        //Set up a transaction of 128 bytes to send/receive
        t.length = 128 * 8;
        t.tx_buffer = sendbuf;
        t.rx_buffer = recvbuf;
        /* This call enables the SPI slave interface to send/receive to the sendbuf and recvbuf. The transaction is
        initialized by the SPI master, however, so it will not actually happen until the master starts a hardware transaction
        by pulling CS low and pulsing the clock etc. In this specific example, we use the handshake line, pulled up by the
        .post_setup_cb callback that is called as soon as a transaction is ready, to let the master know it is free to transfer
        data.
        */
        ret = spi_slave_transmit(RCV_HOST, &t, portMAX_DELAY);

        //spi_slave_transmit does not return until the master has done a transmission, so by here we have sent our data and
        //received data from the master. Print it.
        printf("Received: %s\n", recvbuf);

        //pause the slave to save power, transaction will also be paused
        ret = spi_slave_disable(RCV_HOST);
        if (ret == ESP_OK) {
            printf("slave paused ...\n");
        }
        vTaskDelay(100);    //now is able to sleep or do something to save power, any following transaction will be ignored
        ret = spi_slave_enable(RCV_HOST);
        if (ret == ESP_OK) {
            printf("slave ready !\n");
        }
        n++;
    }
}
```

## 总结

本例进行了从机驱动的实验代码分析，从机驱动的代码不多，本例只是接收代码的部分，熟悉了主要的API效果。
