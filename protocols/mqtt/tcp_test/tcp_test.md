# ESP-MQTT sample application ESP-MQTT简单应用

## 粗略阅读README文档

示例使用MQTT库实现MQTT客户端连接到MQTT代理

硬件需要，项目配置

## 构建和编译

* 选择目标芯片
* 点击**构建项目**

## MQTT介绍

> 资料及介绍参考
> esp官方文档[ESP-MQTT](https://docs.espressif.com/projects/esp-idf/zh_CN/stable/esp32/api-reference/protocols/mqtt.html#id1)
> [MQTT协议详解](https://blog.csdn.net/jackwmj12/article/details/129163012)
> [MQTT Qos等级](https://zhuanlan.zhihu.com/p/80203905)

MQTT 是一种基于**发布/订阅**模式的**轻量级消息传输协议**

在消息传输中分为客户端和服务端两种角色，客户端订阅了服务端某个主题后，当服务端该主题收到其他客户端发布的消息，当前客户端即可接收到消息。

基于TCP/IP网络连接（或SSL、WebSocket），但都需要网络连接

Qos 0/1/2 ：**0**代表最多1次，即消息发送后不确认，也不重试；**1**代表至少1次，如果发送方没有确认手动，会重复发送，会导致消息重复；**2**代表恰好1次，保证消息准确无误地送达一次

## 代码分析

### app_main函数

1. 日志打印：启动，空闲内存，IDF版本
2. `esp_log_level_set` 为给定标志设置日志级别，如先把所有设成info，把各种不同的标识设成**ESP_LOG_VERBOSE**，更大的调试块
3. 初始化网络“三件套” + `example/protocols`内写好的通用连接函数
4. `mqtt_app_start` 自定义函数启动mqtt服务

```c
void app_main(void)
{
    ESP_LOGI(TAG, "[APP] Startup..");
    ESP_LOGI(TAG, "[APP] Free memory: %" PRIu32 " bytes", esp_get_free_heap_size());
    ESP_LOGI(TAG, "[APP] IDF version: %s", esp_get_idf_version());

    esp_log_level_set("*", ESP_LOG_INFO);
    esp_log_level_set("mqtt_client", ESP_LOG_VERBOSE);
    esp_log_level_set("mqtt_example", ESP_LOG_VERBOSE);
    esp_log_level_set("transport_base", ESP_LOG_VERBOSE);
    esp_log_level_set("esp-tls", ESP_LOG_VERBOSE);
    esp_log_level_set("transport", ESP_LOG_VERBOSE);
    esp_log_level_set("outbox", ESP_LOG_VERBOSE);

    ESP_ERROR_CHECK(nvs_flash_init());
    ESP_ERROR_CHECK(esp_netif_init());
    ESP_ERROR_CHECK(esp_event_loop_create_default());

    /* This helper function configures Wi-Fi or Ethernet, as selected in menuconfig.
     * Read "Establishing Wi-Fi or Ethernet Connection" section in
     * examples/protocols/README.md for more information about this function.
     */
    ESP_ERROR_CHECK(example_connect());

    mqtt_app_start();
}
```

### mqtt_app_start函数

1. `esp_mqtt_client_config_t`MQTT客户端结构体配置（可通过menuconfig中进行MQTT配置）
   * `broker_t` 设置地址和安全验证
     * `address_t` [地址配置](https://docs.espressif.com/projects/esp-idf/zh_CN/stable/esp32/api-reference/protocols/mqtt.html#id6)可通过`uri` `path`或多字段组合设置服务器地址
     * `verification_t` 结构体用于进行服务器身份验证
   * `credentials_t` 用于身份验证的客户端凭依
   * `session_t` MQTT会话相关配置
   * `network_t` 网络相关配置
   * `task_t` 允许配置FreeRTOS任务
   * `buffer_t`  输入输出缓冲区大小
2. `CONFIG_BROKER_URL_FROM_STDIN` 作为示例配置项，代表是否从标准输入获取服务器地址(Kconfig编写如下)

    ```config
    menu "Example Configuration"

        config BROKER_URL
            string "Broker URL"
            default "mqtt://mqtt.eclipseprojects.io"
            help
                URL of the broker to connect to

        config BROKER_URL_FROM_STDIN
            bool
            default y if BROKER_URL = "FROM_STDIN"

    endmenu
    ```

3. 如果配置**标准输入**，即通过用户输入
   1. 判断当前结构体 uri 配置是否是占位字符（在函数第一步进行写入），如果不是，跳转分支终止程序
   2. 如果是，初始化字符数并提示
   3. 进入循环，`fgetc`c库函数，用于从指定流获取下一字符
   4. 判断到`\n`，代表输入结束，向字符数组中写入`\0`代表c字符串结束标志，并结束循环
   5. 其他情况向字符数组中写入当前字符
   6. 跳出循环后把整个字符串赋值给 mqtt 结构体的地址
4. `esp_mqtt_client_handle_t` 类型 mqtt 客户端句柄，调用`esp_mqtt_client_init`函数传入配置进行初始化
5. `esp_mqtt_client_register_event(client, ESP_EVENT_ANY_ID, mqtt_event_handler, NULL);` 注册mqtt事件函数，对于**ESP_EVENT_ANY_ID**任意事件，绑定`mqtt_event_handler`函数，没有传入参数
6. `esp_mqtt_client_start`通过句柄启动指定mqtt
7. 

```c
static void mqtt_app_start(void)
{
    esp_mqtt_client_config_t mqtt_cfg = {
        .broker.address.uri = CONFIG_BROKER_URL,
    };
#if CONFIG_BROKER_URL_FROM_STDIN
    char line[128];

    if (strcmp(mqtt_cfg.broker.address.uri, "FROM_STDIN") == 0) {
        int count = 0;
        printf("Please enter url of mqtt broker\n");
        while (count < 128) {
            int c = fgetc(stdin);
            if (c == '\n') {
                line[count] = '\0';
                break;
            } else if (c > 0 && c < 127) {
                line[count] = c;
                ++count;
            }
            vTaskDelay(10 / portTICK_PERIOD_MS);
        }
        mqtt_cfg.broker.address.uri = line;
        printf("Broker url: %s\n", line);
    } else {
        ESP_LOGE(TAG, "Configuration mismatch: wrong broker url");
        abort();
    }
#endif /* CONFIG_BROKER_URL_FROM_STDIN */

    esp_mqtt_client_handle_t client = esp_mqtt_client_init(&mqtt_cfg);
    /* The last argument may be used to pass data to the event handler, in this example mqtt_event_handler */
    esp_mqtt_client_register_event(client, ESP_EVENT_ANY_ID, mqtt_event_handler, NULL);
    esp_mqtt_client_start(client);
}
```

### mqtt_event_handler事件函数

1. 参数分别为：注册到事件的用户参数、事件base、事件id、事件数据
2. 将事件触发的参数获取到函数内部，包括事件数据和 mqtt 客户端句柄
3. 根据不同的事件进程不同的操作
4. `MQTT_EVENT_CONNECTED` MQTT客户端连接至服务器
   1. `esp_mqtt_client_publish` 创建mqtt消息，参数包括：mqtt客户端句柄，**主题**字符串，**数据**字符串，数据长度（设为0自动计算），发布消息Qos，保留位
   2. `esp_mqtt_client_subscribe` 进行订阅，参数包括：mqtt客户端句柄，主题（或主题数组），Qos或订阅数组大小
   3. `esp_mqtt_client_unsubscribe` 取消订阅某主题
5. `MQTT_EVENT_DISCONNECTED` 客户端终止连接，进行日志输出
6. `MQTT_EVENT_SUBSCRIBED` 服务器确认客户端的订阅需求，返回订阅消息的消息id。然后进行一次消息发送
7. `MQTT_EVENT_UNSUBSCRIBED` 客户端确认服务端的退订请求
8. `MQTT_EVENT_PUBLISHED` 服务器确认客户端的发布消息
9. `MQTT_EVENT_DATA` 客户端已收到发布消息 ，输出`event`中对应的数据
10. `MQTT_EVENT_ERROR` 客户端遇到错误

```c
static void mqtt_event_handler(void *handler_args, esp_event_base_t base, int32_t event_id, void *event_data)
{
    ESP_LOGD(TAG, "Event dispatched from event loop base=%s, event_id=%" PRIi32 "", base, event_id);
    esp_mqtt_event_handle_t event = event_data;
    esp_mqtt_client_handle_t client = event->client;
    int msg_id;
    switch ((esp_mqtt_event_id_t)event_id) {
    case MQTT_EVENT_CONNECTED:
        ESP_LOGI(TAG, "MQTT_EVENT_CONNECTED");
        msg_id = esp_mqtt_client_publish(client, "/topic/qos1", "data_3", 0, 1, 0);
        ESP_LOGI(TAG, "sent publish successful, msg_id=%d", msg_id);

        msg_id = esp_mqtt_client_subscribe(client, "/topic/qos0", 0);
        ESP_LOGI(TAG, "sent subscribe successful, msg_id=%d", msg_id);

        msg_id = esp_mqtt_client_subscribe(client, "/topic/qos1", 1);
        ESP_LOGI(TAG, "sent subscribe successful, msg_id=%d", msg_id);

        msg_id = esp_mqtt_client_unsubscribe(client, "/topic/qos1");
        ESP_LOGI(TAG, "sent unsubscribe successful, msg_id=%d", msg_id);
        break;
    case MQTT_EVENT_DISCONNECTED:
        ESP_LOGI(TAG, "MQTT_EVENT_DISCONNECTED");
        break;

    case MQTT_EVENT_SUBSCRIBED:
        ESP_LOGI(TAG, "MQTT_EVENT_SUBSCRIBED, msg_id=%d", event->msg_id);
        msg_id = esp_mqtt_client_publish(client, "/topic/qos0", "data", 0, 0, 0);
        ESP_LOGI(TAG, "sent publish successful, msg_id=%d", msg_id);
        break;
    case MQTT_EVENT_UNSUBSCRIBED:
        ESP_LOGI(TAG, "MQTT_EVENT_UNSUBSCRIBED, msg_id=%d", event->msg_id);
        break;
    case MQTT_EVENT_PUBLISHED:
        ESP_LOGI(TAG, "MQTT_EVENT_PUBLISHED, msg_id=%d", event->msg_id);
        break;
    case MQTT_EVENT_DATA:
        ESP_LOGI(TAG, "MQTT_EVENT_DATA");
        printf("TOPIC=%.*s\r\n", event->topic_len, event->topic);
        printf("DATA=%.*s\r\n", event->data_len, event->data);
        break;
    case MQTT_EVENT_ERROR:
        ESP_LOGI(TAG, "MQTT_EVENT_ERROR");
        if (event->error_handle->error_type == MQTT_ERROR_TYPE_TCP_TRANSPORT) {
            log_error_if_nonzero("reported from esp-tls", event->error_handle->esp_tls_last_esp_err);
            log_error_if_nonzero("reported from tls stack", event->error_handle->esp_tls_stack_err);
            log_error_if_nonzero("captured as transport's socket errno",  event->error_handle->esp_transport_sock_errno);
            ESP_LOGI(TAG, "Last errno string (%s)", strerror(event->error_handle->esp_transport_sock_errno));

        }
        break;
    default:
        ESP_LOGI(TAG, "Other event id:%d", event->event_id);
        break;
    }
}
```

## 总结

和其他的传输协议配置如蓝牙、http类似，esp有关MQTT的使用大体分为3个步骤，MQTT结构体及参数配置，MQTT初始化并绑定事件函数，事件函数具体实现。具体过程在本示例中都有实现，就不具体赘述，有关mqtt，还有许多不同实现方式的示例，后续进行实验。
