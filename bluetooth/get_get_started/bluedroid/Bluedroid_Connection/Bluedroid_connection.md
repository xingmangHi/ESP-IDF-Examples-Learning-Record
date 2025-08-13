# Bluedroid Connection 双模蓝牙连接

## 粗略阅读README文档

文档简介本示例作为beacon的扩展，进一步介绍如何作为可连接的外设进行广告宣传，如何捕获GAP事件并对其处理，如何更新参数

后续代码解释在本例解释中分析

## 构建、烧录和监视

* 选择目标芯片
* 选择端口号、烧录方式
* 点击**构建、烧录和监视**
![监视输出](bc1.png)

## 分析代码

### 头文件和静态变量

C语言头文件基础库，FreeRTOS相关库，esp系统提示相关库，NVS非易失性存储库，ESP蓝牙相关库

蓝牙驱动程序可以分为3层：

* `"esp_bt_defs.h"` `"esp_bt_main.h"` `"esp_bt_device.h"` 最底层，驱动蓝牙芯片/功能
* `"esp_gap_ble_api.h"` 链路层，负责广播、扫描、连接参数更新、安全配对、身份解析（IRK）等“链路管理”工作
* `"esp_gatts_api.h"` `"esp_gatt_common_api.h"` 服务层，实现服务端和公共通信交换部分

`esp_ble_adv_params_t` 类型数据 具体解释详见[](../Bluedroid_Beacon/Bluedroid_Beacon.md/#数据分析)

* `adv_int_min` 非定向和低占空比定向广告的最小广告间隔
* `adv_int_max` 非定向和低占空比定向广告的最大广告间隔
* `adv_type`广告类型
* `own_addr_type` 所有者蓝牙设备地址类型
* `channel_map` 广告通道
* `adv_filter_policy` 广告过滤策略

`adv_raw_data` 储存广告发送的数据

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <inttypes.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "freertos/event_groups.h"
#include "esp_system.h"
#include "esp_log.h"
#include "nvs_flash.h"
#include "esp_bt.h"                 // 蓝牙双模（BT/BLE）控制器和主机的总入口

#include "esp_gap_ble_api.h"        // GAP层API，负责广播、扫描、连接参数更新、安全配对等“链路管理”功能
#include "esp_gatts_api.h"          // GATT Clint API（主机端），用来发现服务、读写特征值、注册notify/indicate
#include "esp_bt_defs.h"            // 蓝牙公共数据结构
#include "esp_bt_main.h"            // 蓝牙控制器的初始化 / 反初始化
#include "esp_bt_device.h"          // 驱动本地设备
#include "esp_gatt_common_api.h"    // 介于GAP和GATT之间的公共部分

static const char *CONN_TAG = "CONN_DEMO";
static const char device_name[] = "Bluedroid_Conn";

static esp_ble_adv_params_t adv_params = {
    .adv_int_min = 0x20,  // 20ms
    .adv_int_max = 0x20,  // 20ms
    .adv_type = ADV_TYPE_IND,
    .own_addr_type = BLE_ADDR_TYPE_PUBLIC,
    .channel_map = ADV_CHNL_ALL,
    .adv_filter_policy = ADV_FILTER_ALLOW_SCAN_ANY_CON_ANY,
};

static uint8_t adv_raw_data[] = {
    0x02, ESP_BLE_AD_TYPE_FLAG, 0x06,
    0x0F, ESP_BLE_AD_TYPE_NAME_CMPL, 'B', 'l', 'u', 'e', 'd', 'r', 'o', 'i', 'd', '_', 'C', 'o', 'n', 'n',
    0x02, ESP_BLE_AD_TYPE_TX_PWR, 0x09,
};
```

### app_main函数

> 笔者注，函数中if判断的效率实在较低，完全可以采用ESP_ERROR_CHECK等类似的宏函数进行判断 [错误判断](https://docs.espressif.com/projects/esp-idf/zh_CN/stable/esp32/api-guides/error-handling.html#id1)

1. 非易失性存储初始化，`nvs_flash_erase` 重新分配内存，再`nvs_flash_init` 初始化
2. `esp_bt_controller_mem_release` 清理经典蓝牙内存
3. `BT_CONTROLLER_INIT_CONFIG_DEFAULT` 默认蓝牙控制配置
4. `esp_bt_controller_init` 蓝牙初始化 [API调用](https://docs.espressif.com/projects/esp-idf/zh_CN/stable/esp32/api-reference/bluetooth/controller_vhci.html#_CPPv422esp_bt_controller_initP26esp_bt_controller_config_t)
5. `esp_bt_controller_enable` 打开特定模式的蓝牙
6. `esp_bluedroid_init` bluedroid初始化和分配蓝牙资源 [API查看](https://docs.espressif.com/projects/esp-idf/zh_CN/stable/esp32/api-reference/bluetooth/esp_bt_main.html#_CPPv418esp_bluedroid_initv)
7. `esp_bluedroid_enable` 使能蓝牙
8. `esp_ble_gap_register_callback` 注册GAP事件回调函数，在所有GAP事件触发
9. `esp_ble_gatts_register_callback` 注册GATT服务器应用程序回调 [GATT服务器API](https://docs.espressif.com/projects/esp-idf/zh_CN/stable/esp32/api-reference/bluetooth/esp_gatts.html#gatt-api)
10. `esp_ble_gatts_app_register` 注册GATT服务器应用程序，参数为应用ID
11. `esp_ble_gatt_set_local_mtu` 设置本地所需的MTU大小，不设置则为默认23字节 (*MTU（最大传输单元）是指在蓝牙低功耗（BLE）连接中，单次数据传输所能承载的最大数据量*)
12. `esp_ble_gap_set_device_name` 设置GAP设备名称
13. `esp_ble_gap_config_adv_data_raw` 进行广播数据配置

```c
void app_main(void)
{
    esp_err_t ret;

    //initialize NVS
    ret = nvs_flash_init();
    if (ret == ESP_ERR_NVS_NO_FREE_PAGES || ret == ESP_ERR_NVS_NEW_VERSION_FOUND) {
        ESP_ERROR_CHECK(nvs_flash_erase());
        ret = nvs_flash_init();
    }
    ESP_ERROR_CHECK( ret );

    ESP_ERROR_CHECK(esp_bt_controller_mem_release(ESP_BT_MODE_CLASSIC_BT));
    esp_bt_controller_config_t bt_cfg = BT_CONTROLLER_INIT_CONFIG_DEFAULT();
    ret = esp_bt_controller_init(&bt_cfg);
    if (ret) {
        ESP_LOGE(CONN_TAG, "%s initialize controller failed: %s", __func__, esp_err_to_name(ret));
        return;
    }

    ret = esp_bt_controller_enable(ESP_BT_MODE_BLE);
    if (ret) {
        ESP_LOGE(CONN_TAG, "%s enable controller failed: %s", __func__, esp_err_to_name(ret));
        return;
    }

    ret = esp_bluedroid_init();
    if (ret) {
        ESP_LOGE(CONN_TAG, "%s init bluetooth failed: %s", __func__, esp_err_to_name(ret));
        return;
    }

    ret = esp_bluedroid_enable();
    if (ret) {
        ESP_LOGE(CONN_TAG, "%s enable bluetooth failed: %s", __func__, esp_err_to_name(ret));
        return;
    }

    ret = esp_ble_gap_register_callback(esp_gap_cb);
    if (ret) {
        ESP_LOGE(CONN_TAG, "%s gap register failed, error code = %x", __func__, ret);
        return;
    }

    ret = esp_ble_gatts_register_callback(gatts_event_handler);
    if (ret) {
        ESP_LOGE(CONN_TAG, "%s gatts register failed, error code = %x", __func__, ret);
        return;
    }

    ret = esp_ble_gatts_app_register(APP_ID_PLACEHOLDER);
    if (ret) {
        ESP_LOGE(CONN_TAG, "%s gatts app register failed, error code = %x", __func__, ret);
        return;
    }

    ret = esp_ble_gatt_set_local_mtu(500);
    if (ret) {
        ESP_LOGE(CONN_TAG, "set local  MTU failed, error code = %x", ret);
        return;
    }

    ret = esp_ble_gap_set_device_name(device_name);
    if (ret) {
        ESP_LOGE(CONN_TAG, "set device name failed, error code = %x", ret);
        return;
    }

    ret = esp_ble_gap_config_adv_data_raw(adv_raw_data, sizeof(adv_raw_data));
    if (ret) {
        ESP_LOGE(CONN_TAG, "config adv data failed, error code = %x", ret);
    }
}
```

### GAP回调函数

1. 根据触发事件判断
2. `ESP_GAP_BLE_ADV_DATA_RAW_SET_COMPLETE_EVT` 原始广播**数据设置完成**（*esp_ble_gap_config_adv_data_raw*或类似函数完成），LOG日志打印，并开始广播
3. `ESP_GAP_BLE_ADV_START_COMPLETE_EVT` 控制器**开始广播**或**启动失败**，进行判断并打印相应日志
4. `ESP_GAP_BLE_ADV_STOP_COMPLETE_EVT` 广播**停止**（可以函数停止，或完成后自动停止），进行判断并打印相应日志
5. `ESP_GAP_BLE_UPDATE_CONN_PARAMS_EVT` 主机或从机调用 **update API** ，LL 层链路参数重新协商完成，日志打印参数

```c
static void esp_gap_cb(esp_gap_ble_cb_event_t event, esp_ble_gap_cb_param_t *param)
{
    switch (event) {
    case ESP_GAP_BLE_ADV_DATA_RAW_SET_COMPLETE_EVT:
        ESP_LOGI(CONN_TAG, "Advertising data set, status %d", param->adv_data_raw_cmpl.status);
        esp_ble_gap_start_advertising(&adv_params);
        break;
    case ESP_GAP_BLE_ADV_START_COMPLETE_EVT:
        if (param->adv_start_cmpl.status != ESP_BT_STATUS_SUCCESS) {
            ESP_LOGE(CONN_TAG, "Advertising start failed, status %d", param->adv_start_cmpl.status);
            break;
        }
        ESP_LOGI(CONN_TAG, "Advertising start successfully");
        break;
    case ESP_GAP_BLE_ADV_STOP_COMPLETE_EVT:
        if (param->adv_stop_cmpl.status != ESP_BT_STATUS_SUCCESS) {
            ESP_LOGE(CONN_TAG, "Advertising stop failed, status %d", param->adv_stop_cmpl.status);
        }
        ESP_LOGI(CONN_TAG, "Advertising stop successfully");
        break;
    case ESP_GAP_BLE_UPDATE_CONN_PARAMS_EVT:
        ESP_LOGI(CONN_TAG, "Connection params update, status %d, conn_int %d, latency %d, timeout %d",
                    param->update_conn_params.status,
                    param->update_conn_params.conn_int,
                    param->update_conn_params.latency,
                    param->update_conn_params.timeout);
        break;
    default:
        break;
    }
}
```

### GATT处理程序

1. 判断触发事件
2. `ESP_GATTS_REG_EVT` **Bluedroid 把 app_id 注册到 GATT Server 层，并分配一个唯一的 gatts_if 触发**（esp_ble_gatts_register_callback() 之后，紧接着调用esp_ble_gatts_app_register(app_id)），日志打印状态和appid
3. `ESP_GATTS_CONNECT_EVT` GAP 层有中央设备（手机、PC、另一个 ESP32 等）完成链路建立，底层 Link Layer **收到 CONNECT_REQ 并建立连接之后，GATT Server 向上抛事件**
   1. 初始化参数变量，获取参数变量存入本地参数
   2. 写入其他需要的变量
   3. 日志打印连接设备的conn_id ，蓝牙设备MAC地址(remote_bda)
   4. 通过刚才修改的变量更新连接参数`esp_ble_gap_update_conn_params`
4. `ESP_GATTS_DISCONNECT_EVT` 链路被任何一方主动断开时触发，日志打印设备地址，断开原因，重新进入可连接的广播状态

```c
//because we only have one profile table in this demo, there is only one set of gatts evemt handler
static void gatts_event_handler(esp_gatts_cb_event_t event, esp_gatt_if_t gatts_if, esp_ble_gatts_cb_param_t *param)
{
    switch (event) {
    case ESP_GATTS_REG_EVT:
        ESP_LOGI(CONN_TAG, "GATT server register, status %d, app_id %d", param->reg.status, param->reg.app_id);
        break;
    case ESP_GATTS_CONNECT_EVT:
        esp_ble_conn_update_params_t conn_params = {0};
        memcpy(conn_params.bda, param->connect.remote_bda, sizeof(esp_bd_addr_t));
        conn_params.latency = 0;
        conn_params.max_int = 0x20;
        conn_params.min_int = 0x10;
        conn_params.timeout = 400;
        ESP_LOGI(CONN_TAG, "Connected, conn_id %u, remote "ESP_BD_ADDR_STR"",
                param->connect.conn_id, ESP_BD_ADDR_HEX(param->connect.remote_bda));
        esp_ble_gap_update_conn_params(&conn_params);
        break;
    case ESP_GATTS_DISCONNECT_EVT:
        ESP_LOGI(CONN_TAG, "Disconnected, remote "ESP_BD_ADDR_STR", reason 0x%02x",
                ESP_BD_ADDR_HEX(param->disconnect.remote_bda), param->disconnect.reason);
        esp_ble_gap_start_advertising(&adv_params);
        break;
    default:
        break;
    }
}
```

## [蓝牙架构](https://docs.espressif.com/projects/esp-idf/zh_CN/stable/esp32/api-guides/bt-architecture/index.html)介绍

* 经典蓝牙: 优化用于连续的高吞吐量数据流，适用于音频传输等应用。
* 低功耗蓝牙 (Bluetooth LE): 针对低功耗间歇性数据传输设计，适合用于传感器和可穿戴设备等

> **Bluedroid** 协议栈 和 **NimBLE** 协议栈 是两种协议栈，均可用于蓝牙开发
>
>* Bluedroid 是ESP-IDF 默认的主机协议栈，支持经典蓝牙和低功耗蓝牙
>* NimBLE 是专为低功耗蓝牙设计的轻量级主机协议栈

文档写得非常清晰，笔者就不作可能会歪曲的额外解读，强烈建议学习蓝牙的阶段从头到尾看一遍文档，*无论现在有多少了解。*

如果没看的话此处只摘录和示例相关的GAP和GATT，两者都是主机层的交换协议

* **GAP (通用访问规范)**: 管理设备发现、连接建立，并定义蓝牙设备的角色和模式。
* **ATT/GATT (属性协议/通用属性规范)**: 通过服务和特征实现基于属性的数据交换，主要用于低功耗蓝牙。

## 总结

本例依然只是在代码上进行了蓝牙连接的分析，知道了gatt的回调函数和事件，还仔细查看了ESP-IDF的文档，知道了蓝牙架构和两种协议栈的区别，对后续关于蓝牙的使用和认知有很大的帮助。
