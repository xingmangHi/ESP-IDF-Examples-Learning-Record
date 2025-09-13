# ESP32-CAM web server ESP32-CAM web服务端

## 文档简介

本文是对于esp32-cam的ESP-IDF v4版本例程`camera_web_server`的分析理解，为后续v5版本重构做准备

## 代码分析

### part-wifi

#### 宏定义

WiFi站模式的SSID和Password，最大重试次数
WiFiAP模式的SSID和Password，最大站连接，IP地址设置，通道设置

```c
#define EXAMPLE_ESP_WIFI_SSID      CONFIG_ESP_WIFI_SSID
#define EXAMPLE_ESP_WIFI_PASS      CONFIG_ESP_WIFI_PASSWORD
#define EXAMPLE_ESP_MAXIMUM_RETRY  CONFIG_ESP_MAXIMUM_RETRY
#define EXAMPLE_ESP_WIFI_AP_SSID   CONFIG_ESP_WIFI_AP_SSID
#define EXAMPLE_ESP_WIFI_AP_PASS   CONFIG_ESP_WIFI_AP_PASSWORD
#define EXAMPLE_MAX_STA_CONN       CONFIG_MAX_STA_CONN
#define EXAMPLE_IP_ADDR            CONFIG_SERVER_IP
#define EXAMPLE_ESP_WIFI_AP_CHANNEL CONFIG_ESP_WIFI_AP_CHANNEL
```

#### 系统事件处理函数

1. 作为AP，有连接，日志打印
2. 作为AP，断开连接，日志打印
3. 作为站，启动，调用`esp_wifi_connect`连接函数
4. 作为站，获取到IP，`ip4addr_ntoa`函数将IP地址转换为ASCII字符串
5. 作为站，断开连接，如果重连次数小于最大重连次数，尝试重连。有日志打印
6. `mdns_handle_system_event` 应用程序需要从系统事件处理程序调用函数，使mDNS服务正常运行

```c
static esp_err_t event_handler(void *ctx, system_event_t *event)
{
    switch(event->event_id) {
    case SYSTEM_EVENT_AP_STACONNECTED:
        ESP_LOGI(TAG, "station:" MACSTR " join, AID=%d",
                 MAC2STR(event->event_info.sta_connected.mac),
                 event->event_info.sta_connected.aid);
        break;
    case SYSTEM_EVENT_AP_STADISCONNECTED:
        ESP_LOGI(TAG, "station:" MACSTR "leave, AID=%d",
                 MAC2STR(event->event_info.sta_disconnected.mac),
                 event->event_info.sta_disconnected.aid);
        break;
    case SYSTEM_EVENT_STA_START:
        esp_wifi_connect();
        break;
    case SYSTEM_EVENT_STA_GOT_IP:
        ESP_LOGI(TAG, "got ip:%s",
                 ip4addr_ntoa(&event->event_info.got_ip.ip_info.ip));
        s_retry_num = 0;
        break;
    case SYSTEM_EVENT_STA_DISCONNECTED:
        {
            if (s_retry_num < EXAMPLE_ESP_MAXIMUM_RETRY) {
                esp_wifi_connect();
                s_retry_num++;
                ESP_LOGI(TAG,"retry to connect to the AP");
            }
            ESP_LOGI(TAG,"connect to the AP fail");
            break;
        }
    default:
        break;
    }
    mdns_handle_system_event(ctx, event);
    return ESP_OK;
}
```

#### AP和站模式初始化函数

`wifi_init_softap`作为AP初始化

1. 如果设置IP为`"192.168.4.1"`,`sscanf`函数从字符串中读取数字并赋值给abcd
2. `IP4_ADDR`宏函数初始化IPv4地址，包括IP、网关、子网掩码
3. `tcpip_adapter_dhcps_stop` 停止DHCP服务，`tcpip_adapter_set_ip_info`设置接口的IP地址信息，`tcpip_adapter_dhcps_start`启用DHCP服务（**DHCP（动态主机配置协议**是一种网络管理协议，用于自动分配IP地址和其他网络配置参数给网络中的设备。）
4. 把设置的SSID、Password和最大连接写入WiFi配置
5. 设置连接模式为`WPA2_PSK`（目前绝大部分wifi都是这个模式）
6. 写入通道数
7. `esp_wifi_set_config`配置WiFi

`wifi_init_sta`函数中把menuconfig中的SSID和Password写入，然后调用`esp_wifi_set_config`进行配置

```c
void wifi_init_softap()
{
    if (strcmp(EXAMPLE_IP_ADDR, "192.168.4.1"))
    {
        int a, b, c, d;
        sscanf(EXAMPLE_IP_ADDR, "%d.%d.%d.%d", &a, &b, &c, &d);
        tcpip_adapter_ip_info_t ip_info;
        IP4_ADDR(&ip_info.ip, a, b, c, d);
        IP4_ADDR(&ip_info.gw, a, b, c, d);
        IP4_ADDR(&ip_info.netmask, 255, 255, 255, 0);
        ESP_ERROR_CHECK(tcpip_adapter_dhcps_stop(WIFI_IF_AP));
        ESP_ERROR_CHECK(tcpip_adapter_set_ip_info(WIFI_IF_AP, &ip_info));
        ESP_ERROR_CHECK(tcpip_adapter_dhcps_start(WIFI_IF_AP));
    }
    wifi_config_t wifi_config;
    memset(&wifi_config, 0, sizeof(wifi_config_t));
    snprintf((char*)wifi_config.ap.ssid, 32, "%s", EXAMPLE_ESP_WIFI_AP_SSID);
    wifi_config.ap.ssid_len = strlen((char*)wifi_config.ap.ssid);
    snprintf((char*)wifi_config.ap.password, 64, "%s", EXAMPLE_ESP_WIFI_AP_PASS);
    wifi_config.ap.max_connection = EXAMPLE_MAX_STA_CONN;
    wifi_config.ap.authmode = WIFI_AUTH_WPA_WPA2_PSK;
    if (strlen(EXAMPLE_ESP_WIFI_AP_PASS) == 0) {
        wifi_config.ap.authmode = WIFI_AUTH_OPEN;
    }
    if (strlen(EXAMPLE_ESP_WIFI_AP_CHANNEL)) {
        int channel;
        sscanf(EXAMPLE_ESP_WIFI_AP_CHANNEL, "%d", &channel);
        wifi_config.ap.channel = channel;
    }

    ESP_ERROR_CHECK(esp_wifi_set_config(ESP_IF_WIFI_AP, &wifi_config));

    ESP_LOGI(TAG, "wifi_init_softap finished.SSID:%s password:%s",
             EXAMPLE_ESP_WIFI_AP_SSID, EXAMPLE_ESP_WIFI_AP_PASS);
}

void wifi_init_sta()
{
    wifi_config_t wifi_config;
    memset(&wifi_config, 0, sizeof(wifi_config_t));
    snprintf((char*)wifi_config.sta.ssid, 32, "%s", EXAMPLE_ESP_WIFI_SSID);
    snprintf((char*)wifi_config.sta.password, 64, "%s", EXAMPLE_ESP_WIFI_PASS);

    ESP_ERROR_CHECK(esp_wifi_set_config(ESP_IF_WIFI_STA, &wifi_config) );

    ESP_LOGI(TAG, "wifi_init_sta finished.");
    ESP_LOGI(TAG, "connect to ap SSID:%s password:%s",
             EXAMPLE_ESP_WIFI_SSID, EXAMPLE_ESP_WIFI_PASS);
}
```

#### wifi-main函数

1. 宏函数设置WiFi默认配置
2. 设置模式为：AP和STA / AP / STA
3. nvs初始化
4. `tcpip_adapter_init`初始化底层TCP/IP堆栈
5. `esp_event_loop_init`创建系统事件任务
6. `esp_wifi_init`WiFi驱动程序安装，`esp_wifi_set_mode` 模式设置
7. `wifi_init_softap` `wifi_init_sta` 调用函数进行配置（由此发现两种配置不冲突）
8. `esp_wifi_start` 启动WiFi驱动
9. `esp_wifi_set_ps` 设置WiFi省电类型

```c
void app_wifi_main()
{
    wifi_init_config_t cfg = WIFI_INIT_CONFIG_DEFAULT();
    wifi_mode_t mode = WIFI_MODE_NULL;

    if (strlen(EXAMPLE_ESP_WIFI_AP_SSID) && strlen(EXAMPLE_ESP_WIFI_SSID)) {
        mode = WIFI_MODE_APSTA;
    } else if (strlen(EXAMPLE_ESP_WIFI_AP_SSID)) {
        mode = WIFI_MODE_AP;
    } else if (strlen(EXAMPLE_ESP_WIFI_SSID)) {
        mode = WIFI_MODE_STA;
    }

    esp_err_t ret = nvs_flash_init();
    if (ret == ESP_ERR_NVS_NO_FREE_PAGES || ret == ESP_ERR_NVS_NEW_VERSION_FOUND) {
      ESP_ERROR_CHECK(nvs_flash_erase());
      ret = nvs_flash_init();
    }
    ESP_ERROR_CHECK(ret);

    if (mode == WIFI_MODE_NULL) {
        ESP_LOGW(TAG,"Neither AP or STA have been configured. WiFi will be off.");
        return;
    }

    tcpip_adapter_init();
    ESP_ERROR_CHECK(esp_event_loop_init(event_handler, NULL));
    ESP_ERROR_CHECK(esp_wifi_init(&cfg));
    ESP_ERROR_CHECK(esp_wifi_set_mode(mode));

    if (mode & WIFI_MODE_AP) {
        wifi_init_softap();
    }

    if (mode & WIFI_MODE_STA) {
        wifi_init_sta();
    }
    ESP_ERROR_CHECK(esp_wifi_start());
    ESP_ERROR_CHECK(esp_wifi_set_ps(WIFI_PS_NONE));
}
```

#### wifi-迁移重构

经过分析和资料对比，tcpip有关函数现在被netif替代，系统事件函数也进行了更改，WiFi模式配置基本没什么变化，但可以写得更整齐一些

### part-camera

#### 结构体定义

在`sensor`头文件和源文件中，定义了传感器的抽象层。

* 为不同信号定义了宏 （NT99141、OV9650、OV7670……）
* 定义像素格式的枚举类型，即为不同像素格式设置标识符
* 定义帧尺寸的枚举类型，不同帧尺寸有各种标识
* 定义纵横比的枚举类型
* 定义增益上限的枚举类型
* 定义比例设置结构体，设置最大宽度，最大高度，起始点终点坐标，偏移量等
* 定义分辨率结构体，储存宽度、高度、纵横比
* 定义传感器ID结构体，包括制造商ID，产品ID和版本号
* 定义相机状态结构体，包括帧尺寸，缩放，亮度，对比度……
* 定义传感器结构体，包含ID，I2C从地址，像素格式，相机状态，一系列函数指针（用于具体操作传感器）

#### esp_camera.h

该文件是ESP32-CAM相机模块的驱动头文件

1. 定义相机配置结构体，包括引脚配置、XCLK频率、LEDC定时器和通道、像素格式、帧尺寸、JPEG质量等参数
2. 定义相机帧缓冲区结构体，储存捕获的图像帧数据。包括像素数据、指针、缓冲区字节数、宽度、高度、像素格式、时间戳
3. 定义错误码
4. 声明相机操作的具体函数

#### camera-main函数

1. 配置引脚13和引脚14为输入、上拉、不开中断
2. 设置ledc定时器配置`ledc_timer_config_t`
   * `duty_resolution` 占空比分辨率，即范围从0到2的n次
   * `freq_hz` 频率
   * `speed_mode` 速度模式（根据型号有限制）
   * `timer_num` 定时器选择
3. 设置ledc通道配置
   * `channel` 通道
   * `duty` 初始占空比（取决于分辨率）
   * `gpio_num` 绑定输出引脚
   * `speed_mode` 速度模式
   * `hpoint` 用于高速模式调整信号相位
   * `timer_sel` 定时选择
4. 配置ledc
5. 如果不启用LED，上述配置都不进行
6. 从头文件中导入宏定义配置`camera_config_t`
7. `esp_camera_init` 自定义函数进行初始化
8. `esp_camera_sensor_get` 自定义函数获取传感器数据
9. 对图像进行操作

```c
void app_camera_main ()
{
#if CONFIG_CAMERA_MODEL_ESP_EYE || CONFIG_CAMERA_MODEL_ESP32_CAM_BOARD
    /* IO13, IO14 is designed for JTAG by default,
     * to use it as generalized input,
     * firstly declair it as pullup input */
    gpio_config_t conf;
    conf.mode = GPIO_MODE_INPUT;
    conf.pull_up_en = GPIO_PULLUP_ENABLE;
    conf.pull_down_en = GPIO_PULLDOWN_DISABLE;
    conf.intr_type = GPIO_INTR_DISABLE;
    conf.pin_bit_mask = 1LL << 13;
    gpio_config(&conf);
    conf.pin_bit_mask = 1LL << 14;
    gpio_config(&conf);
#endif

#ifdef CONFIG_LED_ILLUMINATOR_ENABLED
    gpio_set_direction(CONFIG_LED_LEDC_PIN,GPIO_MODE_OUTPUT);
    ledc_timer_config_t ledc_timer = {
        .duty_resolution = LEDC_TIMER_8_BIT,            // resolution of PWM duty
        .freq_hz         = 1000,                        // frequency of PWM signal
        .speed_mode      = LEDC_LOW_SPEED_MODE,  // timer mode
        .timer_num       = CONFIG_LED_LEDC_TIMER        // timer index
    };
    ledc_channel_config_t ledc_channel = {
        .channel    = CONFIG_LED_LEDC_CHANNEL,
        .duty       = 0,
        .gpio_num   = CONFIG_LED_LEDC_PIN,
        .speed_mode = LEDC_LOW_SPEED_MODE,
        .hpoint     = 0,
        .timer_sel  = CONFIG_LED_LEDC_TIMER
    };
    #ifdef CONFIG_LED_LEDC_HIGH_SPEED_MODE
    ledc_timer.speed_mode = ledc_channel.speed_mode = LEDC_HIGH_SPEED_MODE;
    #endif
    switch (ledc_timer_config(&ledc_timer)) {
        case ESP_ERR_INVALID_ARG: ESP_LOGE(TAG, "ledc_timer_config() parameter error"); break;
        case ESP_FAIL: ESP_LOGE(TAG, "ledc_timer_config() Can not find a proper pre-divider number base on the given frequency and the current duty_resolution"); break;
        case ESP_OK: if (ledc_channel_config(&ledc_channel) == ESP_ERR_INVALID_ARG) {
            ESP_LOGE(TAG, "ledc_channel_config() parameter error");
          }
          break;
        default: break;
    }
#endif

    camera_config_t config;
    config.ledc_channel = LEDC_CHANNEL_0;
    config.ledc_timer = LEDC_TIMER_0;
    config.pin_d0 = Y2_GPIO_NUM;
    config.pin_d1 = Y3_GPIO_NUM;
    config.pin_d2 = Y4_GPIO_NUM;
    config.pin_d3 = Y5_GPIO_NUM;
    config.pin_d4 = Y6_GPIO_NUM;
    config.pin_d5 = Y7_GPIO_NUM;
    config.pin_d6 = Y8_GPIO_NUM;
    config.pin_d7 = Y9_GPIO_NUM;
    config.pin_xclk = XCLK_GPIO_NUM;
    config.pin_pclk = PCLK_GPIO_NUM;
    config.pin_vsync = VSYNC_GPIO_NUM;
    config.pin_href = HREF_GPIO_NUM;
    config.pin_sscb_sda = SIOD_GPIO_NUM;
    config.pin_sscb_scl = SIOC_GPIO_NUM;
    config.pin_pwdn = PWDN_GPIO_NUM;
    config.pin_reset = RESET_GPIO_NUM;
    config.xclk_freq_hz = 20000000;
    config.pixel_format = PIXFORMAT_JPEG;
    //init with high specs to pre-allocate larger buffers
    config.frame_size = FRAMESIZE_QSXGA;
    config.jpeg_quality = 10;
    config.fb_count = 2;

    // camera init
    esp_err_t err = esp_camera_init(&config);
    if (err != ESP_OK) {
        ESP_LOGE(TAG, "Camera init failed with error 0x%x", err);
        return;
    }

    sensor_t * s = esp_camera_sensor_get();
    s->set_vflip(s, 1);//flip it back
    //initial sensors are flipped vertically and colors are a bit saturated
    if (s->id.PID == OV3660_PID) {
        s->set_brightness(s, 1);//up the blightness just a bit
        s->set_saturation(s, -2);//lower the saturation
    }
    //drop down frame size for higher initial frame rate
    s->set_framesize(s, FRAMESIZE_HD);
}
```

#### SCCB[使用OV摄像头协议](https://blog.csdn.net/qq_39842609/article/details/122274827)

SCCB协议与IIC协议十分相似，不过IIC是PHILIPS的专利，所以OmnVision在IIC的基础上做了点小改动。SCCB最主要是阉割了IIC的连续读写的功能，即每读写完一个字节就主机必须发送一个NA信号。

##### 头文件和宏定义

需要注意的是，预编译中对`ARDUINO_ARCH_ESP32` 和 `CONFIG_ARDUHAL_ESP_LOG`进行了特别判断，用于不同环境下的log实现

SCCB协议的宏定义类比I2C，有频率、读写位、检查相关，还储存了从机地址

`#define LITTLETOBIG(x)          ((x<<8)|(x>>8))` 该宏函数用于将16位整数的字节数据从小端格式转换为大端格式

```c
#include <stdbool.h>
#include <freertos/FreeRTOS.h>
#include <freertos/task.h>
#include "sccb.h"
#include <stdio.h>
#include "sdkconfig.h"
#if defined(ARDUINO_ARCH_ESP32) && defined(CONFIG_ARDUHAL_ESP_LOG)
#include "esp32-hal-log.h"
#else
#include "esp_log.h"
static const char* TAG = "sccb";
#endif

#define LITTLETOBIG(x)          ((x<<8)|(x>>8))

#include "driver/i2c.h"

#define SCCB_FREQ               100000           /*!< I2C master frequency*/
#define WRITE_BIT               I2C_MASTER_WRITE /*!< I2C master write */
#define READ_BIT                I2C_MASTER_READ  /*!< I2C master read */
#define ACK_CHECK_EN            0x1              /*!< I2C master will check ack from slave*/
#define ACK_CHECK_DIS           0x0              /*!< I2C master will not check ack from slave */
#define ACK_VAL                 0x0              /*!< I2C ack value */
#define NACK_VAL                0x1              /*!< I2C nack value */
#if CONFIG_SCCB_HARDWARE_I2C_PORT1
const int SCCB_I2C_PORT         = 1;
#else
const int SCCB_I2C_PORT         = 0;
#endif
static uint8_t ESP_SLAVE_ADDR   = 0x3c;
```

##### 初始化

函数中配置i2c结构体变量，设为主机模式，开启上拉，绑定引脚

`i2c_param_config` 函数写入配置 `i2c_driver_install`安装驱动程序

该函数用于**对I2C进行初始化，配置为主机模式并开启上拉，为后续数据传输做准备**

```c
int SCCB_Init(int pin_sda, int pin_scl)
{
    ESP_LOGI(TAG, "pin_sda %d pin_scl %d\n", pin_sda, pin_scl);
    //log_i("SCCB_Init start");
    i2c_config_t conf;
    conf.mode = I2C_MODE_MASTER;
    conf.sda_io_num = pin_sda;
    conf.sda_pullup_en = GPIO_PULLUP_ENABLE;
    conf.scl_io_num = pin_scl;
    conf.scl_pullup_en = GPIO_PULLUP_ENABLE;
    conf.master.clk_speed = SCCB_FREQ;

    i2c_param_config(SCCB_I2C_PORT, &conf);
    i2c_driver_install(SCCB_I2C_PORT, conf.mode, 0, 0, 0);
    return 0;
}
```

##### 查找从机地址

1. `i2c_cmd_link_create` 创建**命令链接**
2. `i2c_master_start` 启动
3. `i2c_master_write_byte` 将“写入字节”命令排队到命令列表。参数为：命令列表，需要发送的字节，使能ACK信号（由于I2C协议的特殊性，是地址+数据）
4. 有数据写入的话，采用`i2c_master_write`函数（此处只是寻找有相应的地址，不进行操作）
5. `i2c_master_stop` 停止
6. `i2c_master_cmd_begin` 执行**命令链接**，并检查写入状态
7. `i2c_cmd_link_delete` 释放命令链接使用的资源
8. 如果有地址能写入，就储存地址并返回

该函数用于**遍历可能的I2C地址，向其中写入数据，检查哪个地址有效**

```c
uint8_t SCCB_Probe()
{
    uint8_t slave_addr = 0x0;
    while(slave_addr < 0x7f) {
        i2c_cmd_handle_t cmd = i2c_cmd_link_create();
        i2c_master_start(cmd);
        i2c_master_write_byte(cmd, ( slave_addr << 1 ) | WRITE_BIT, ACK_CHECK_EN);
        i2c_master_stop(cmd);
        esp_err_t ret = i2c_master_cmd_begin(SCCB_I2C_PORT, cmd, 1000 / portTICK_RATE_MS);
        i2c_cmd_link_delete(cmd);
        if( ret == ESP_OK) {
            ESP_SLAVE_ADDR = slave_addr;
            return ESP_SLAVE_ADDR;
        }
        slave_addr++;
    }
    return ESP_SLAVE_ADDR;
}
```

##### 读取数据

1. 发送从机地址和要读取的寄存器地址
2. 发送从机地址并进行读取
3. 每次命令链接完成后释放内存

该函数用于**读取从机某个寄存器的值，每次读取一个字节**

```c
uint8_t SCCB_Read(uint8_t slv_addr, uint8_t reg)
{
    uint8_t data=0;
    esp_err_t ret = ESP_FAIL;
    i2c_cmd_handle_t cmd = i2c_cmd_link_create();
    i2c_master_start(cmd);
    i2c_master_write_byte(cmd, ( slv_addr << 1 ) | WRITE_BIT, ACK_CHECK_EN);
    i2c_master_write_byte(cmd, reg, ACK_CHECK_EN);
    i2c_master_stop(cmd);
    ret = i2c_master_cmd_begin(SCCB_I2C_PORT, cmd, 1000 / portTICK_RATE_MS);
    i2c_cmd_link_delete(cmd);
    if(ret != ESP_OK) return -1;
    cmd = i2c_cmd_link_create();
    i2c_master_start(cmd);
    i2c_master_write_byte(cmd, ( slv_addr << 1 ) | READ_BIT, ACK_CHECK_EN);
    i2c_master_read_byte(cmd, &data, NACK_VAL);
    i2c_master_stop(cmd);
    ret = i2c_master_cmd_begin(SCCB_I2C_PORT, cmd, 1000 / portTICK_RATE_MS);
    i2c_cmd_link_delete(cmd);
    if(ret != ESP_OK) {
        ESP_LOGE(TAG, "SCCB_Read Failed addr:0x%02x, reg:0x%02x, data:0x%02x, ret:%d", slv_addr, reg, data, ret);
    }
    return data;
}
```

##### 写入数据

向命令链接中写入 从机地址、寄存器地址、写入的数据并进行发送，完成后释放内存

该函数用于**向从机某个寄存器中写入数据**

```c
uint8_t SCCB_Write(uint8_t slv_addr, uint8_t reg, uint8_t data)
{
    esp_err_t ret = ESP_FAIL;
    i2c_cmd_handle_t cmd = i2c_cmd_link_create();
    i2c_master_start(cmd);
    i2c_master_write_byte(cmd, ( slv_addr << 1 ) | WRITE_BIT, ACK_CHECK_EN);
    i2c_master_write_byte(cmd, reg, ACK_CHECK_EN);
    i2c_master_write_byte(cmd, data, ACK_CHECK_EN);
    i2c_master_stop(cmd);
    ret = i2c_master_cmd_begin(SCCB_I2C_PORT, cmd, 1000 / portTICK_RATE_MS);
    i2c_cmd_link_delete(cmd);
    if(ret != ESP_OK) {
        ESP_LOGE(TAG, "SCCB_Write Failed addr:0x%02x, reg:0x%02x, data:0x%02x, ret:%d", slv_addr, reg, data, ret);
    }
    return ret == ESP_OK ? 0 : -1;
}
```

##### 读取16位地址的数据

函数用于**读取寄存器地址为16位的数据**，并进行了大小端转换

```c
uint8_t SCCB_Read16(uint8_t slv_addr, uint16_t reg)
{
    uint8_t data=0;
    esp_err_t ret = ESP_FAIL;
    uint16_t reg_htons = LITTLETOBIG(reg);
    uint8_t *reg_u8 = (uint8_t *)&reg_htons;
    i2c_cmd_handle_t cmd = i2c_cmd_link_create();
    i2c_master_start(cmd);
    i2c_master_write_byte(cmd, ( slv_addr << 1 ) | WRITE_BIT, ACK_CHECK_EN);
    i2c_master_write_byte(cmd, reg_u8[0], ACK_CHECK_EN);
    i2c_master_write_byte(cmd, reg_u8[1], ACK_CHECK_EN);
    i2c_master_stop(cmd);
    ret = i2c_master_cmd_begin(SCCB_I2C_PORT, cmd, 1000 / portTICK_RATE_MS);
    i2c_cmd_link_delete(cmd);
    if(ret != ESP_OK) return -1;
    cmd = i2c_cmd_link_create();
    i2c_master_start(cmd);
    i2c_master_write_byte(cmd, ( slv_addr << 1 ) | READ_BIT, ACK_CHECK_EN);
    i2c_master_read_byte(cmd, &data, NACK_VAL);
    i2c_master_stop(cmd);
    ret = i2c_master_cmd_begin(SCCB_I2C_PORT, cmd, 1000 / portTICK_RATE_MS);
    i2c_cmd_link_delete(cmd);
    if(ret != ESP_OK) {
        ESP_LOGE(TAG, "W [%04x]=%02x fail\n", reg, data);
    }
    return data;
}
```

##### 写16位地址数据

```c
uint8_t SCCB_Write16(uint8_t slv_addr, uint16_t reg, uint8_t data)
{
    static uint16_t i = 0;
    esp_err_t ret = ESP_FAIL;
    uint16_t reg_htons = LITTLETOBIG(reg);
    uint8_t *reg_u8 = (uint8_t *)&reg_htons;
    i2c_cmd_handle_t cmd = i2c_cmd_link_create();
    i2c_master_start(cmd);
    i2c_master_write_byte(cmd, ( slv_addr << 1 ) | WRITE_BIT, ACK_CHECK_EN);
    i2c_master_write_byte(cmd, reg_u8[0], ACK_CHECK_EN);
    i2c_master_write_byte(cmd, reg_u8[1], ACK_CHECK_EN);
    i2c_master_write_byte(cmd, data, ACK_CHECK_EN);
    i2c_master_stop(cmd);
    ret = i2c_master_cmd_begin(SCCB_I2C_PORT, cmd, 1000 / portTICK_RATE_MS);
    i2c_cmd_link_delete(cmd);
    if(ret != ESP_OK) {
        ESP_LOGE(TAG, "W [%04x]=%02x %d fail\n", reg, data, i++);
    }
    return ret == ESP_OK ? 0 : -1;
}
```

#### XCLK(OV摄像头管脚)

XCLK是OV摄像头的外部时钟输入端口，PCLK是摄像头像素同步时钟输出信号

`xclk_timer_conf` 函数中进行了ledc定时器配置，设置为高速、分辨率为2（0-4）

`camera_enable_out_clock` 使能时钟

1. `periph_module_enable` 启用ledc外设模块（*在v5版本已弃用*）
2. `xclk_timer_conf` 调用函数进行ledc定时器配置
3. 进行通道配置，引脚绑定、模式配置、占空比设置、偏移相位

`camera_disable_out_clock` 失能时钟

该文件用于**利用ledc模块进行pwm输出向ov摄像头提供时钟信号**

```c
esp_err_t xclk_timer_conf(int ledc_timer, int xclk_freq_hz)
{
    ledc_timer_config_t timer_conf;
    timer_conf.duty_resolution = 2;
    timer_conf.freq_hz = xclk_freq_hz;
    timer_conf.speed_mode = LEDC_HIGH_SPEED_MODE;
#if ESP_IDF_VERSION_MAJOR >= 4
    timer_conf.clk_cfg = LEDC_AUTO_CLK;
#endif
    timer_conf.timer_num = (ledc_timer_t)ledc_timer;
    esp_err_t err = ledc_timer_config(&timer_conf);
    if (err != ESP_OK) {
        ESP_LOGE(TAG, "ledc_timer_config failed for freq %d, rc=%x", xclk_freq_hz, err);
    }
    return err;
}

esp_err_t camera_enable_out_clock(camera_config_t* config)
{
    periph_module_enable(PERIPH_LEDC_MODULE);

    esp_err_t err = xclk_timer_conf(config->ledc_timer, config->xclk_freq_hz);
    if (err != ESP_OK) {
        ESP_LOGE(TAG, "ledc_timer_config failed, rc=%x", err);
        return err;
    }

    ledc_channel_config_t ch_conf;
    ch_conf.gpio_num = config->pin_xclk;
    ch_conf.speed_mode = LEDC_HIGH_SPEED_MODE;
    ch_conf.channel = config->ledc_channel;
    ch_conf.intr_type = LEDC_INTR_DISABLE;
    ch_conf.timer_sel = config->ledc_timer;
    ch_conf.duty = 2;
    ch_conf.hpoint = 0;
    err = ledc_channel_config(&ch_conf);
    if (err != ESP_OK) {
        ESP_LOGE(TAG, "ledc_channel_config failed, rc=%x", err);
        return err;
    }
    return ESP_OK;
}

void camera_disable_out_clock()
{
    periph_module_disable(PERIPH_LEDC_MODULE);
}
```

#### camera源文件

##### 头文件

头文件中除esp库外的文件基本已解释，对于不同型号传感器的`ovxxx`文件，内部调用SCCB函数和抽象层传感器文件，基本不需要修改

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include "time.h"
#include "sys/time.h"
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "freertos/semphr.h"
#include "soc/soc.h"
#include "soc/gpio_sig_map.h"
#include "soc/i2s_reg.h"
#include "soc/i2s_struct.h"
#include "soc/io_mux_reg.h"
#include "driver/gpio.h"
#include "driver/rtc_io.h"
#include "driver/periph_ctrl.h"
#include "esp_intr_alloc.h"
#include "esp_system.h"
#include "nvs_flash.h"
#include "nvs.h"
#include "sensor.h"
#include "sccb.h"
#include "esp_camera.h"
#include "camera_common.h"
#include "xclk.h"
#if CONFIG_OV2640_SUPPORT
#include "ov2640.h"
#endif
#if CONFIG_OV7725_SUPPORT
#include "ov7725.h"
#endif
#if CONFIG_OV3660_SUPPORT
#include "ov3660.h"
#endif
#if CONFIG_OV5640_SUPPORT
#include "ov5640.h"
#endif
#if CONFIG_NT99141_SUPPORT
#include "nt99141.h"
#endif
#if CONFIG_OV7670_SUPPORT
#include "ov7670.h"
#endif
```

##### camera-common

`rom/lldesc.h` 用于定义和管理硬件描述符，帮助开发者更高效地进行硬件数据传输管理，尤其是在DMA操作中

`dma_elem_t` 联合体定义DMA数据传输格式，可以是四字节结构体，也可以是32字节数据

`i2s_sampling_mode_t` 枚举定义了三种不同的I2S采样模式

```c
#include "esp_system.h"
#if ESP_IDF_VERSION_MAJOR >= 4 // IDF 4+
#if CONFIG_IDF_TARGET_ESP32 // ESP32/PICO-D4
#include "esp32/rom/lldesc.h"
#else 
#error Target CONFIG_IDF_TARGET is not supported
#endif
#else // ESP32 Before IDF 4.0
#include "rom/lldesc.h"
#endif

typedef union {
    struct {
        uint8_t sample2;
        uint8_t unused2;
        uint8_t sample1;
        uint8_t unused1;
    };
    uint32_t val;
} dma_elem_t;

typedef enum {
    /* camera sends byte sequence: s1, s2, s3, s4, ...
     * fifo receives: 00 s1 00 s2, 00 s2 00 s3, 00 s3 00 s4, ...
     */
    SM_0A0B_0B0C = 0,
    /* camera sends byte sequence: s1, s2, s3, s4, ...
     * fifo receives: 00 s1 00 s2, 00 s3 00 s4, ...
     */
    SM_0A0B_0C0D = 1,
    /* camera sends byte sequence: s1, s2, s3, s4, ...
     * fifo receives: 00 s1 00 00, 00 s2 00 00, 00 s3 00 00, ...
     */
    SM_0A00_0B00 = 3,
} i2s_sampling_mode_t;
```

##### 相机帧结构体

和`sensor`中结构体中类似，包含像素数据指针，数据长度、像素宽度、高度、像素格式、时间戳，包含一个指向next的指针，用于组成链表

```c
typedef struct camera_fb_s {
    uint8_t * buf;
    size_t len;
    size_t width;
    size_t height;
    pixformat_t format;
    struct timeval timestamp;
    size_t size;
    uint8_t ref;
    uint8_t bad;
    struct camera_fb_s * next;
} camera_fb_int_t;
```

#### I2S相关函数

##### 每采样数据

```c
static size_t i2s_bytes_per_sample(i2s_sampling_mode_t mode)
{
    switch(mode) {
    case SM_0A00_0B00:
        return 4;
    case SM_0A0B_0B0C:
        return 4;
    case SM_0A0B_0C0D:
        return 2;
    default:
        assert(0 && "invalid sampling mode");
        return 0;
    }
}
```

##### 

```c
static inline void IRAM_ATTR i2s_conf_reset()
{
    const uint32_t lc_conf_reset_flags = I2S_IN_RST_M | I2S_AHBM_RST_M
                                         | I2S_AHBM_FIFO_RST_M;
    I2S0.lc_conf.val |= lc_conf_reset_flags;
    I2S0.lc_conf.val &= ~lc_conf_reset_flags;

    const uint32_t conf_reset_flags = I2S_RX_RESET_M | I2S_RX_FIFO_RESET_M
                                      | I2S_TX_RESET_M | I2S_TX_FIFO_RESET_M;
    I2S0.conf.val |= conf_reset_flags;
    I2S0.conf.val &= ~conf_reset_flags;
    while (I2S0.state.rx_fifo_reset_back) {
        ;
    }
}
```
