# ESP-IDF例程记录

笔者对**ESP开发**、物联网、**万物互联**有深厚的兴趣，因此打算对ESP-IDF中的例程进行完整的尝试，并记录所得。
更偏向于**应用和调试**，而不会去特别深究其底层原理。

## 软件环境

采用VScode+ESP-IDF环境编程，IDF版本为v5.4.1，烧录方式为串口烧录。
新版本为v5.5，在实验中有更改。
为优化编译速度，笔者尝试在编译时**把安全软件检测关掉**(*比如笔者电脑上的火绒有文件检测，笔者尝试关闭后文件编译速度直接飞起*)

## 硬件环境

采用ESP32-S3-WROOM-1芯片进行尝试，*外部电路为本人自己设计的PCB*。由于无法最合适地展示所有现象，会由**电信号变化**来表示。

* 如GPIO变化采用万用表测量高低电平
* PWM变化采用示波器分析频率和占空比
* I2C和其他通讯协议采用逻辑分析仪进行查看

直观展示由主控输出信号的真实形态，加深理解。

## 计划安排

> 笔者有一定的嵌入式开发基础，希望更多的尝试和实验，所以**肯定**会对例程进行修改，融入自己的尝试和想法。

*由于IDF中的example是根据字母顺序排列，笔者又希望更为高效地进行学习*，故例程安排计划如下：

1. `get-started` 作为例程尝试的开始
2. `wifi/getting-started`、`system/freertos` **初次接触**WiFi基础功能，接触RTOS相关内容
3. `peripherals/gpio`、`uart`、`led`c、`adc`、`dac`、`rmt`、`timer`、`i2c`、`spi`、`twai（can总线）`等 作为嵌入式基础的**横向扩展**
4. `bluetooth`、`protocols/http`、`protocols/websocket`、`protocols/mqtt`、`protocols/now`等 了解各种**无线/网络通信协议**
5. *待定*

该计划有笔者自己的安排，也有ai结果的参考，大致思路是`“接触体验-模块学习-尝试结合”`这样**逐渐**融会贯通整个系统。

已完成例程如下：

* get-started : **blink** , **hello_world**
* wifi
  * getting-started : **softAP** , **station**
* systrm
  * freertos : **basic_freertos** , **real_time**
* peripherals
  * gpio : **generic_gpio**
  * uart : **uart_echo**
  * ledc : **ledc_basic** , **ledc_fade**
  * mcpwm : **mcpwm_bdc_speed_control** , **mcpwm_bldc_hall_control**
  * rmt : **led_strip** , **led_strip_simple_encoder** , **dshot_esc** , **stepper_motor** , **ir_nec_transceive**
  * touch_sensor (由于笔者查看touch_element只在v5.1.1的测试分支文档中有提及，不做相关实验)
    * touch_sensor_v2 : **touch_pad_read** , **touch_pad_interrupt**
  * spi_master : **lcd** , **hd_eeprom**
  * spi_slave : **receiver** , **sender**
  * i2c : **i2c_basic** , **i2c_tools** , **i2c_slace_network_sensor**
  * twai : **twai_self_test** , **twai_network_master** ,**twai_network_slave**
* bluetooth
  * ble_get_start: **Bluedroid_Beacon** , **Bluedroid_Connection** , **Bluedroid_GATT_Server** , **NimBLE_Beacon** , **NimBLE_Connection**
