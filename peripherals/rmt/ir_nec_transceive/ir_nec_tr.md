# IR NEC Encoding and Decoding 红外遥控NEC协议编解码

> 笔者并没有红外遥控设备，故本例重点放在代码分析

## 粗略阅读README文档

文档简介示例创建TX信道定期发送NEC信号，RX信道接收信号并解码

硬件连接，需要把TX和RX脚分别接红外发送器和接收器

构建烧录示例输出

## 构建烧录和监视

* 选择目标芯片
* 选择端口号
* 配置项目
* 点击**构建、烧录和监视**
![监视输出](int1.png)
由于没有实际发送，监视窗口没有实际数值

## 分析代码

### 头文件、宏定义和结构体

#### main.c

头文件导入的驱动库的RMT的TX和RX其他不做解释

宏定义时钟分辨率，输入输出引脚 ，`EXAMPLE_IR_NEC_DECODE_MARGIN` 给定时钟容差，*即使发送端晶振或接收头略有偏差，只要测量值落在期望值 ±200 µs 范围内，仍被判定为合法*
关于NEC时钟的具体定义写在代码注释

```c
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "freertos/queue.h"
#include "esp_log.h"
#include "driver/rmt_tx.h"
#include "driver/rmt_rx.h"
#include "ir_nec_encoder.h"

#define EXAMPLE_IR_RESOLUTION_HZ     1000000 // 1MHz resolution, 1 tick = 1us
#define EXAMPLE_IR_TX_GPIO_NUM       18
#define EXAMPLE_IR_RX_GPIO_NUM       4
#define EXAMPLE_IR_NEC_DECODE_MARGIN 200     // Tolerance for parsing RMT symbols into bit stream

/**
 * @brief NEC timing spec
 */
#define NEC_LEADING_CODE_DURATION_0  9000   // NEC一帧起始引导码 高9ms
#define NEC_LEADING_CODE_DURATION_1  4500   // NEC一帧起始引导码 低4.5ms
#define NEC_PAYLOAD_ZERO_DURATION_0  560    // NEC逻辑0脉冲对 560us高
#define NEC_PAYLOAD_ZERO_DURATION_1  560    // NEC逻辑0脉冲对 560us低
#define NEC_PAYLOAD_ONE_DURATION_0   560    // NEC逻辑1脉冲对 560us高
#define NEC_PAYLOAD_ONE_DURATION_1   1690   // NEC逻辑1脉冲对 1690us低
#define NEC_REPEAT_CODE_DURATION_0   9000   // NEC连发码 高9ms
#define NEC_REPEAT_CODE_DURATION_1   2250   // NEC连发码 低2.25ms
```

#### ir_nec_encoder.h

反正重复和C++兼容不作赘述

`ir_nec_scan_code_t` 结构体定义NEC码，包括地址和命令
`ir_nec_encoder_config_t` 结构体定义使用编码器的参数，时钟分辨率

```c
/**
 * @brief IR NEC scan code representation
 */
typedef struct {
    uint16_t address;
    uint16_t command;
} ir_nec_scan_code_t;

/**
 * @brief Type of IR NEC encoder configuration
 */
typedef struct {
    uint32_t resolution; /*!< Encoder resolution, in Hz */
} ir_nec_encoder_config_t;
```

#### ir_nec_encoder.c

配置针对NEC的结构体编码器，笔者此处不作赘述 [其他文档中解释](../led_strip/led_strip.md#led_strip_encoderc)

```c
typedef struct {
    rmt_encoder_t base;           // the base "class", declares the standard encoder interface
    rmt_encoder_t *copy_encoder;  // use the copy_encoder to encode the leading and ending pulse
    rmt_encoder_t *bytes_encoder; // use the bytes_encoder to encode the address and command data
    rmt_symbol_word_t nec_leading_symbol; // NEC leading code with RMT representation
    rmt_symbol_word_t nec_ending_symbol;  // NEC ending code with RMT representation
    int state;
} rmt_ir_nec_encoder_t;
```

### app_main 函数

> 关于NEC信号的具体解释详见[文章](https://blog.csdn.net/Ivan804638781/article/details/111225949)

1. 创建RMT[RX通道](https://docs.espressif.com/projects/esp-idf/zh_CN/stable/esp32/api-reference/peripherals/rmt.html#rmt-rx)，配置与TX通道基本一致
2. 新建**队列**储存RX通道数据 `rmt_rx_done_event_data_t` 为RX接收数据内部定义结构体
![接收数据结构体内容](int2.png)
3. `rmt_rx_register_event_callbacks` 注册**回调函数**
4. 创建TX通道，配置不作分析
5. `rmt_apply_carrier` 函数进行[载波调制](https://docs.espressif.com/projects/esp-idf/zh_CN/stable/esp32/api-reference/peripherals/rmt.html#rmt-carrier-modulation-and-demodulation)
6. `rmt_new_ir_nec_encoder` 函数进行**自定义编码器创建和配置**
7. **使能RMT的RX和TX通道**
8. `raw_symbols` 数组储存一个完整的NEC帧 ，`rx_data` 作为接收数据结构体
9. `rmt_receive` 配置并第一次启动接收 ，`receive_config` 作为噪声或者毛刺配置，小于、大于某值的信号被视作噪声
10. 主循环中以1s为等待读取队列数据，有接收的话进行解码并重新启用RX接收，没有的话发送一帧数据

```c
void app_main(void)
{
    ESP_LOGI(TAG, "create RMT RX channel");
    rmt_rx_channel_config_t rx_channel_cfg = {
        .clk_src = RMT_CLK_SRC_DEFAULT,
        .resolution_hz = EXAMPLE_IR_RESOLUTION_HZ,
        .mem_block_symbols = 64, // amount of RMT symbols that the channel can store at a time
        .gpio_num = EXAMPLE_IR_RX_GPIO_NUM,
    };
    rmt_channel_handle_t rx_channel = NULL;
    ESP_ERROR_CHECK(rmt_new_rx_channel(&rx_channel_cfg, &rx_channel));

    ESP_LOGI(TAG, "register RX done callback");
    QueueHandle_t receive_queue = xQueueCreate(1, sizeof(rmt_rx_done_event_data_t));
    assert(receive_queue);
    rmt_rx_event_callbacks_t cbs = {
        .on_recv_done = example_rmt_rx_done_callback,
    };
    ESP_ERROR_CHECK(rmt_rx_register_event_callbacks(rx_channel, &cbs, receive_queue));

    // the following timing requirement is based on NEC protocol
    rmt_receive_config_t receive_config = {
        .signal_range_min_ns = 1250,     // the shortest duration for NEC signal is 560us, 1250ns < 560us, valid signal won't be treated as noise
        .signal_range_max_ns = 12000000, // the longest duration for NEC signal is 9000us, 12000000ns > 9000us, the receive won't stop early
    };

    ESP_LOGI(TAG, "create RMT TX channel");
    rmt_tx_channel_config_t tx_channel_cfg = {
        .clk_src = RMT_CLK_SRC_DEFAULT,
        .resolution_hz = EXAMPLE_IR_RESOLUTION_HZ,
        .mem_block_symbols = 64, // amount of RMT symbols that the channel can store at a time
        .trans_queue_depth = 4,  // number of transactions that allowed to pending in the background, this example won't queue multiple transactions, so queue depth > 1 is sufficient
        .gpio_num = EXAMPLE_IR_TX_GPIO_NUM,
    };
    rmt_channel_handle_t tx_channel = NULL;
    ESP_ERROR_CHECK(rmt_new_tx_channel(&tx_channel_cfg, &tx_channel));

    ESP_LOGI(TAG, "modulate carrier to TX channel");
    rmt_carrier_config_t carrier_cfg = {
        .duty_cycle = 0.33,
        .frequency_hz = 38000, // 38KHz
    };
    ESP_ERROR_CHECK(rmt_apply_carrier(tx_channel, &carrier_cfg));

    // this example won't send NEC frames in a loop
    rmt_transmit_config_t transmit_config = {
        .loop_count = 0, // no loop
    };

    ESP_LOGI(TAG, "install IR NEC encoder");
    ir_nec_encoder_config_t nec_encoder_cfg = {
        .resolution = EXAMPLE_IR_RESOLUTION_HZ,
    };
    rmt_encoder_handle_t nec_encoder = NULL;
    ESP_ERROR_CHECK(rmt_new_ir_nec_encoder(&nec_encoder_cfg, &nec_encoder));

    ESP_LOGI(TAG, "enable RMT TX and RX channels");
    ESP_ERROR_CHECK(rmt_enable(tx_channel));
    ESP_ERROR_CHECK(rmt_enable(rx_channel));

    // save the received RMT symbols
    rmt_symbol_word_t raw_symbols[64]; // 64 symbols should be sufficient for a standard NEC frame
    rmt_rx_done_event_data_t rx_data;
    // ready to receive
    ESP_ERROR_CHECK(rmt_receive(rx_channel, raw_symbols, sizeof(raw_symbols), &receive_config));
    while (1) {
        // wait for RX done signal
        if (xQueueReceive(receive_queue, &rx_data, pdMS_TO_TICKS(1000)) == pdPASS) {
            // parse the receive symbols and print the result
            example_parse_nec_frame(rx_data.received_symbols, rx_data.num_symbols);
            // start receive again
            ESP_ERROR_CHECK(rmt_receive(rx_channel, raw_symbols, sizeof(raw_symbols), &receive_config));
        } else {
            // timeout, transmit predefined IR NEC packets
            const ir_nec_scan_code_t scan_code = {
                .address = 0x0440,
                .command = 0x3003,
            };
            ESP_ERROR_CHECK(rmt_transmit(tx_channel, nec_encoder, &scan_code, sizeof(scan_code), &transmit_config));
        }
    }
}
```

### 