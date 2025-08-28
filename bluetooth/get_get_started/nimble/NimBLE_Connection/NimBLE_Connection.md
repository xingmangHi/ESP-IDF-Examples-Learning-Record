# NimBLE Connection NimBLE协议栈连接

## 粗略阅读README文档

文档简介本示例从NimBLE扩展而来，进一步介绍如何作为可连接的外围设计进行广告宣传，如何捕获GAP事件并对其进行处理，如何更新连接参数

编译烧录，代码介绍

## 编译、烧录和监视

* 选择目标芯片
* 选择端口号
* 选择烧录方式
* 点击**构建、烧录和监视**
* ![监视输出](nc1.png)

  * `BLE_INIT` BT 蓝牙控制编译器版本
  * 蓝牙功能配置 ： ADV(广告)启用，BLE5.0禁用，DMT(直接扫描模式)启用，扫描启用，CCA(清除信道评估)禁用，SMP(安全管理协议)启用，连接启用
  * 蓝牙设备MAC地址
  * 物理层版本，创建时间
  * nimble主机任务创建
  * GAP初始化：停止广告
  * 连接设备地址
  * GAP初始化：进行广告
  * 发现模式设置为2（可发现可连接）
  * 广告通道0，本地地址类型0，广告过滤策略0，广告最小间隔800，广告最大间隔816
  * 开始广告

## 代码分析

> 笔者注：项目导入外部库为LED灯带驱动库，笔者在此不作分析
> common头文件中列出了nimble库所用到的不同类型头文件，笔者也不作赘述
> gap层基础分析见[gap相关程序](../NimBLE_Beacon/NimBLE_Beacon.md/#gap相关程序)
> 附[官方低功耗蓝牙指南](https://docs.espressif.com/projects/esp-idf/zh_CN/stable/esp32/api-guides/ble/index.html#id1)强烈建议查看

### app_main

1. NVS非易失性存储初始化
2. `nimble_port_init` 初始化主机和控制器堆栈
3. `gap_init` 自定义函数进行GAP层初始化
4. `nimble_host_config_init` 自定义函数绑定回调函数
5. 创建主机任务并运行

```c
void app_main(void) {
    /* Local variables */
    int rc = 0;
    esp_err_t ret = ESP_OK;

    /* LED initialization */
    led_init();

    /* NVS flash initialization */
    ret = nvs_flash_init();
    if (ret == ESP_ERR_NVS_NO_FREE_PAGES ||
        ret == ESP_ERR_NVS_NEW_VERSION_FOUND) {
        ESP_ERROR_CHECK(nvs_flash_erase());
        ret = nvs_flash_init();
    }
    if (ret != ESP_OK) {
        ESP_LOGE(TAG, "failed to initialize nvs flash, error code: %d ", ret);
        return;
    }

    /* NimBLE stack initialization */
    ret = nimble_port_init();
    if (ret != ESP_OK) {
        ESP_LOGE(TAG, "failed to initialize nimble stack, error code: %d ",
                 ret);
        return;
    }

    /* GAP service initialization */
    rc = gap_init();
    if (rc != 0) {
        ESP_LOGE(TAG, "failed to initialize GAP service, error code: %d", rc);
        return;
    }

    /* NimBLE host configuration initialization */
    nimble_host_config_init();

    /* Start NimBLE host task thread and return */
    xTaskCreate(nimble_host_task, "NimBLE Host", 4*1024, NULL, 5, NULL);
    return;
}
```

### 回调函数和主机任务

* `reset`函数触发在主机因为错误重启时，此处只进行了日志打印
* `sync` 函数触发在主机和控制器同步时，进行广告初始化
* 第三个函数为API函数，大意是在储存状态错误时的回调，直接绑定的库函数

```c
/*
 *  Stack event callback functions
 *      - on_stack_reset is called when host resets BLE stack due to errors
 *      - on_stack_sync is called when host has synced with controller
 */
static void on_stack_reset(int reason) {
    /* On reset, print reset reason to console */
    ESP_LOGI(TAG, "nimble stack reset, reset reason: %d", reason);
}

static void on_stack_sync(void) {
    /* On stack sync, do advertising initialization */
    adv_init();
}
static void nimble_host_config_init(void) {
    /* Set host callbacks */
    ble_hs_cfg.reset_cb = on_stack_reset;
    ble_hs_cfg.sync_cb = on_stack_sync;
    ble_hs_cfg.store_status_cb = ble_store_util_status_rr;

    /* Store host configuration */
    ble_store_config_init();
}

```

`nimble_port_run` 按照官方文档描述“启动NimBLE主机层FreeRTOS线程”

```c
static void nimble_host_task(void *param) {
    /* Task entry log */
    ESP_LOGI(TAG, "nimble host task has been started!");

    /* This function won't return until nimble_port_stop() is executed */
    nimble_port_run();

    /* Clean up at exit */
    vTaskDelete(NULL);
}
```

### GAP层区别

> 笔者写完前两个part发现和Beacon例程好像没什么区别，于是感觉GAP层应该有些不同，下述

广播函数`start_advertising`增加设置广播间隔代码

```c
    /* Set advertising interval */
    rsp_fields.adv_itvl = BLE_GAP_ADV_ITVL_MS(500);
    rsp_fields.adv_itvl_is_present = 1;
      /* Set advertising interval */
    adv_params.itvl_min = BLE_GAP_ADV_ITVL_MS(500);
    adv_params.itvl_max = BLE_GAP_ADV_ITVL_MS(510);
```

该函数为内部函数，函数中只进行日志记录，调用`format_addr`格式化地址输出为"XX:XX:XX:XX:XX:XX"的形式。同时在日志中显示各种数据

```c
static void print_conn_desc(struct ble_gap_conn_desc *desc) {
    /* Local variables */
    char addr_str[18] = {0};

    /* Connection handle */
    ESP_LOGI(TAG, "connection handle: %d", desc->conn_handle);

    /* Local ID address */
    format_addr(addr_str, desc->our_id_addr.val);
    ESP_LOGI(TAG, "device id address: type=%d, value=%s",
             desc->our_id_addr.type, addr_str);

    /* Peer ID address */
    format_addr(addr_str, desc->peer_id_addr.val);
    ESP_LOGI(TAG, "peer id address: type=%d, value=%s", desc->peer_id_addr.type,
             addr_str);

    /* Connection info */
    ESP_LOGI(TAG,
             "conn_itvl=%d, conn_latency=%d, supervision_timeout=%d, "
             "encrypted=%d, authenticated=%d, bonded=%d\n",
             desc->conn_itvl, desc->conn_latency, desc->supervision_timeout,
             desc->sec_state.encrypted, desc->sec_state.authenticated,
             desc->sec_state.bonded);
}
```

该函数为GAP层事件处理函数，事件类型和bluedroid协议示例中类似，在广告开始`ble_gap_adv_start`函数中绑定

1. 判断触发事件类型
2. `BLE_GAP_EVENT_CONNECT`GAP层有中央设备完成链路连接（*按照官方文档说法，完成连接后，手机等创建连接的一方为中央设备，原先发送广播的一方为外围设备*）
   1. 日志记录连接状态
   2. 判断状态码为0
   3. `ble_gap_conn_find`函数根据连接句柄获取连接描述符信息`ble_gap_conn_desc`结构体包含
        * `conn_handle`连接句柄
        * `our_id_addr` `our_ota_addr`本地设备地址 （ota代表无线over-the-air）
        * `peer_id_addr` `peer_ota_addr`远端设备地址
        * `conn_itvl`连接间隔
        * `conn_latency`延迟
        * `supervision_timeout`监督超时
        * `sec_state`安全状态（加密、认证、绑定状态）
   4. 调用`print_conn_desc`函数解码并在日志输出描述符
   5. 打开LED灯
   6. `ble_gap_upd_params`类型结构体，定义
        * `itvl_min`最小连接间隔
        * `itvl_max`最大设备连接
        * `latency`连接延迟（外围设备可以忽略的连接事件数）
        * `supervision_timeout` 监督超时（单位10ms）
   7. `ble_gap_update_params` 函数为本设备向远端设备发起更新参数请求，远端设备决定是否接受，并通过事件通知双方更新结果
   8. 如果连接状态码不为0，代表连接状态不正常，重新打开广告
3. `BLE_GAP_EVENT_DISCONNECT`断开连接事件，进行日志记录，关闭LED，重新开启广告
4. `BLE_GAP_EVENT_CONN_UPDATE` 连接更新，即连接参数更新事件
   1. 日志打印状态为连接更新
   2. `ble_gap_conn_find` 函数根据句柄获取连接描述符
   3. `print_conn_desc` 函数格式化输出描述符相关信息

```c
/*
 * NimBLE applies an event-driven model to keep GAP service going
 * gap_event_handler is a callback function registered when calling
 * ble_gap_adv_start API and called when a GAP event arrives
 */
static int gap_event_handler(struct ble_gap_event *event, void *arg) {
    /* Local variables */
    int rc = 0;
    struct ble_gap_conn_desc desc;

    /* Handle different GAP event */
    switch (event->type) {

    /* Connect event */
    case BLE_GAP_EVENT_CONNECT:
        /* A new connection was established or a connection attempt failed. */
        ESP_LOGI(TAG, "connection %s; status=%d",
                 event->connect.status == 0 ? "established" : "failed",
                 event->connect.status);

        /* Connection succeeded */
        if (event->connect.status == 0) {
            /* Check connection handle */
            rc = ble_gap_conn_find(event->connect.conn_handle, &desc);
            if (rc != 0) {
                ESP_LOGE(TAG,
                         "failed to find connection by handle, error code: %d",
                         rc);
                return rc;
            }

            /* Print connection descriptor and turn on the LED */
            print_conn_desc(&desc);
            led_on();

            /* Try to update connection parameters */
            struct ble_gap_upd_params params = {.itvl_min = desc.conn_itvl,
                                                .itvl_max = desc.conn_itvl,
                                                .latency = 3,
                                                .supervision_timeout =
                                                    desc.supervision_timeout};
            rc = ble_gap_update_params(event->connect.conn_handle, &params);
            if (rc != 0) {
                ESP_LOGE(
                    TAG,
                    "failed to update connection parameters, error code: %d",
                    rc);
                return rc;
            }
        }
        /* Connection failed, restart advertising */
        else {
            start_advertising();
        }
        return rc;

    /* Disconnect event */
    case BLE_GAP_EVENT_DISCONNECT:
        /* A connection was terminated, print connection descriptor */
        ESP_LOGI(TAG, "disconnected from peer; reason=%d",
                 event->disconnect.reason);

        /* Turn off the LED */
        led_off();

        /* Restart advertising */
        start_advertising();
        return rc;

    /* Connection parameters update event */
    case BLE_GAP_EVENT_CONN_UPDATE:
        /* The central has updated the connection parameters. */
        ESP_LOGI(TAG, "connection updated; status=%d",
                 event->conn_update.status);

        /* Print connection descriptor */
        rc = ble_gap_conn_find(event->conn_update.conn_handle, &desc);
        if (rc != 0) {
            ESP_LOGE(TAG, "failed to find connection by handle, error code: %d",
                     rc);
            return rc;
        }
        print_conn_desc(&desc);
        return rc;
    }

    return rc;
}
```

## 总结

本例中再次熟悉了NimBLE协议栈的BLE初始化，包括主机配置，GAP配置，广播配置，主要是进行了连接事件的回调函数编写，实际操作比bluedroid简单一些，同时查看了官方手册关于低功耗蓝牙的相关文档，对蓝牙操作有更多的了解。