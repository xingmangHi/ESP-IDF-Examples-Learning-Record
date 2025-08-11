# TWAI network slave TWAI网络从机

## 文档简介

本示例演示了TWAI从机的发送和接收数据，但由于CAN协议的特殊性，从机和主机并没有太多区别

## 构建、烧录和监视

* 选择目标芯片
* 选择端口号
* 点击**构建项目**

## 代码分析

> 由于CAN通信是多设备的一种协议，从机和主机的区别并不大，绝大部分代码和[网络主机代码](../twai_network_master/twai_network_master.md/#代码分析)一致，故本例主要进行对数据处理部分进行专门分析

### 队列数据类型

数据主要靠队列来传输，本例中队列数据类型如下，均为枚举，故队列主要用于传递信号，而不是传递数据

```c
typedef enum {
    TX_SEND_PING_RESP,
    TX_SEND_DATA,
    TX_SEND_STOP_RESP,
    TX_TASK_EXIT,
} tx_task_action_t;

typedef enum {
    RX_RECEIVE_PING,
    RX_RECEIVE_START_CMD,
    RX_RECEIVE_STOP_CMD,
    RX_TASK_EXIT,
} rx_task_action_t;

tx_task_queue = xQueueCreate(1, sizeof(tx_task_action_t));
rx_task_queue = xQueueCreate(1, sizeof(rx_task_action_t));
```

### 数据储存，变换和发送

下方代码就是数据接收、发送的主要代码。

接收的步骤为先初始化变量，然后调用`twai_receive`函数进行接收，数据满足`twai_message_t`类型，后续的数据处理可根据结构体内部成员进行操作

发送的步骤为先配置`twai_message_t`类型变量，填入数据等，然后调用`twai_transmit`函数进行发送。在相关示例中我没有看到类似的回调函数和中断等（在v5.5版本，接收报文必须在接收函数回调中进行[v5.5接收报文](https://docs.espressif.com/projects/esp-idf/zh_CN/v5.5/esp32/api-reference/peripherals/twai.html#id7) 更加灵活，事件配合更加精确）

对于发送数据的配置如最后一段循环演示，采用移位和与运算将32位数据分给4个8位数据储存（如果有长度变化需要另外更改）

```c
// receive
twai_message_t rx_msg;
twai_receive(&rx_msg, portMAX_DELAY);

//send
static const twai_message_t ping_resp = {
    // Message type and format settings
    .extd = 0,              // Standard Format message (11-bit ID)
    .rtr = 0,               // Send a data frame
    .ss = 0,                // Not single shot
    .self = 0,              // Not a self reception request
    .dlc_non_comp = 0,      // DLC is less than 8
    // Message ID and payload
    .identifier = ID_SLAVE_PING_RESP,
    .data_length_code = 0,
    .data = {0},
};
twai_transmit(&ping_resp, portMAX_DELAY);

// Data bytes of data message will be initialized in the transmit task
static twai_message_t data_message = {
    // Message type and format settings
    .extd = 0,              // Standard Format message (11-bit ID)
    .rtr = 0,               // Send a data frame
    .ss = 0,                // Not single shot
    .self = 0,              // Not a self reception request
    .dlc_non_comp = 0,      // DLC is less than 8
    // Message ID and payload
    .identifier = ID_SLAVE_DATA,
    .data_length_code = 4,
    .data = {1, 2, 3, 4},
};
twai_transmit(&data_message, portMAX_DELAY);

for (int i = 0; i < 4; i++) {
    data_message.data[i] = (sensor_data >> (i * 8)) & 0xFF;
}
```

## 总结

本例并未对代码进行逐块或逐行分析，只单独对数据部分进行解释，但如前所述，大部分代码和主机相同，故本示例实验就简单带过
