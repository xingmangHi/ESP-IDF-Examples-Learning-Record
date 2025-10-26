# using LwIP SNTP module and time functions 使用SNTP模式时间设置

## 粗略阅读README文档

示例演示如何使用LwIP SNTP 模块从互联网服务器获取时间

配置项目需要连接WiFi或以太网 ，获取时间 ， 时间保持 ， 时间设置 ，时区设置
函数介绍

## 代码分析

> 由于笔者主要记录代码编写的过程和规范，故暂时不进行实际构建、烧录和测试

### 特殊类型和关键词

使用`RTC_DATA_ATTR` 关键字告诉编译器数据放在RTC内存中，在深度睡眠模式保持值
变量用来记录系统启动次数

```c
/* Variable holding number of times ESP32 restarted since first boot.
 * It is placed into RTC memory using RTC_DATA_ATTR and
 * maintains its value when ESP32 wakes from deep sleep.
 */
RTC_DATA_ATTR static int boot_count = 0;
```

### app_main函数

1. 记录变量`boot_count`加1，并输出记录值
2. 初始化变量，`now`存储时间，`timeinfo`存储结构化后的时间
3. `time(&now);` c函数填充now，`localtime_r(&now, &timeinfo);` 把时间转换成结构化的年月日时分秒，依赖配置的时区设置
4. 判断是否在2016年之前，如果是，说明时间没进行过设置 （芯片第一次上电时 RTC 是空的，ESP-IDF 默认把 RTC 设成 1970-01-01 00:00:00 UTC，于是
localtime_r 得到的 tm_year 就是 70）
5. 自定义函数进行联网获取时间并更新RTC，之后time获取RTC就是最新的值
6. 配置中启用平滑时间同步，`gettimeofday`获取时间戳，但读到微秒的精度
7. 在 ESP32 上 tv_sec 是 long（32 bit），但乘以 1 000 000 会暂时溢出，所以先强制转成 int64_t 再做乘法，否则 2038 年后会出错
8. 获取时间并计算出时间戳，然后进行加500ms假设出错
9. `settimeofday` 将时间写入RTC
10. 自定义函数获取时间并写入RTC
11. `setenv("TZ", "EST5EDT,M3.2.0/2,M11.1.0", 1)`把环境变量 TZ 设成“美国东部”规则字符串，最后一个参数为1代表覆盖旧值
12. `tzset`读取TZ变量并解析，刷新全局数据，之后`localtime_r`才能按新规则记录时区
13. `strftime`按本地化格式，把tm结构印成字符串，并输出
14. `setenv("TZ", "CST-8", 1)`把环境变量TZ设成“中国标准”时间
15. 如果SNTP被配置成平滑模式，等待校时线程完成
16. `adjtime(NULL, &outdelta)`把内核剩余未校正的量读到 outdelta（不修改时钟，仅查询）
17. `adjtime`原理为内核记得时钟有偏差，然后每次时钟中断微调系统节拍
18. 日志打印并进入深度睡眠状态

```c
void app_main(void)
{
    ++boot_count;
    ESP_LOGI(TAG, "Boot count: %d", boot_count);

    time_t now;
    struct tm timeinfo;
    time(&now);
    localtime_r(&now, &timeinfo);
    // Is time set? If not, tm_year will be (1970 - 1900).
    if (timeinfo.tm_year < (2016 - 1900)) {
        ESP_LOGI(TAG, "Time is not set yet. Connecting to WiFi and getting time over NTP.");
        obtain_time();
        // update 'now' variable with current time
        time(&now);
    }
#ifdef CONFIG_SNTP_TIME_SYNC_METHOD_SMOOTH
    else {
        // add 500 ms error to the current system time.
        // Only to demonstrate a work of adjusting method!
        {
            ESP_LOGI(TAG, "Add a error for test adjtime");
            struct timeval tv_now;
            gettimeofday(&tv_now, NULL);
            int64_t cpu_time = (int64_t)tv_now.tv_sec * 1000000L + (int64_t)tv_now.tv_usec;
            int64_t error_time = cpu_time + 500 * 1000L;
            struct timeval tv_error = { .tv_sec = error_time / 1000000L, .tv_usec = error_time % 1000000L };
            settimeofday(&tv_error, NULL);
        }

        ESP_LOGI(TAG, "Time was set, now just adjusting it. Use SMOOTH SYNC method.");
        obtain_time();
        // update 'now' variable with current time
        time(&now);
    }
#endif

    char strftime_buf[64];

    // Set timezone to Eastern Standard Time and print local time
    setenv("TZ", "EST5EDT,M3.2.0/2,M11.1.0", 1);
    tzset();
    localtime_r(&now, &timeinfo);
    strftime(strftime_buf, sizeof(strftime_buf), "%c", &timeinfo);
    ESP_LOGI(TAG, "The current date/time in New York is: %s", strftime_buf);

    // Set timezone to China Standard Time
    setenv("TZ", "CST-8", 1);
    tzset();
    localtime_r(&now, &timeinfo);
    strftime(strftime_buf, sizeof(strftime_buf), "%c", &timeinfo);
    ESP_LOGI(TAG, "The current date/time in Shanghai is: %s", strftime_buf);

    if (sntp_get_sync_mode() == SNTP_SYNC_MODE_SMOOTH) {
        struct timeval outdelta;
        while (sntp_get_sync_status() == SNTP_SYNC_STATUS_IN_PROGRESS) {
            adjtime(NULL, &outdelta);
            ESP_LOGI(TAG, "Waiting for adjusting time ... outdelta = %jd sec: %li ms: %li us",
                        (intmax_t)outdelta.tv_sec,
                        outdelta.tv_usec/1000,
                        outdelta.tv_usec%1000);
            vTaskDelay(2000 / portTICK_PERIOD_MS);
        }
    }

    const int deep_sleep_sec = 10;
    ESP_LOGI(TAG, "Entering deep sleep for %d seconds", deep_sleep_sec);
    esp_deep_sleep(1000000LL * deep_sleep_sec);
}
```

### 自定义获取时间函数

1. NVS初始化，LwIP相关初始化工作[wifi配置](../../wifi/getting-started/softAP/softAP.md/#初始化函数)
2. `ESP_NETIF_SNTP_DEFAULT_CONFIG` 宏函数填入默认配置
3. 手动配置：启动为false，不立即启动；允许DHCP覆盖、追加到服务器列表；每次拿IP重新解析`option 42`；设置DHCP优先、用户次之；设置事件触发为“拿到IP”
4. `config.sync_cb = time_sync_notification_cb;`绑定回调函数
5. `esp_netif_sntp_init`将config写入配置
6. `example_connect`公共库文件，进行WiFi连接
7. `LWIP_DHCP_GET_NTP_SRV`宏配置代表DHCP能获取到NTP服务器地址，于是只要启动SNTP服务就可以开始校时
8. 如果协议栈支持IPv6且服务器槽 ≥ 3，设置静态IPv6槽作冗余
9. 如果DHCP不下发NTP，需要手动指定服务器
10. `ESP_NETIF_SNTP_DEFAULT_CONFIG_MULTIPLE` 宏函数指定服务器，`ESP_SNTP_SERVER_LIST` 整理两个服务器地址
11. `config.sync_cb` 绑定回调函数，函数会在时间同步完成后触发
12. 如果启用了平滑校时，需要设置`config.smooth_sync = true;`
13. `esp_netif_sntp_init`根据配置，填入服务器列表，启用SNTP服务自动校时
14. `esp_netif_sntp_sync_wait`等待时间同步完成
15. `time`获取RTC时间，`localtime_r`进行时区转换
16. 断开网络，反初始化SNTP

```c
static void obtain_time(void)
{
    ESP_ERROR_CHECK( nvs_flash_init() );
    ESP_ERROR_CHECK(esp_netif_init());
    ESP_ERROR_CHECK( esp_event_loop_create_default() );

#if LWIP_DHCP_GET_NTP_SRV
    /**
     * NTP server address could be acquired via DHCP,
     * see following menuconfig options:
     * 'LWIP_DHCP_GET_NTP_SRV' - enable STNP over DHCP
     * 'LWIP_SNTP_DEBUG' - enable debugging messages
     *
     * NOTE: This call should be made BEFORE esp acquires IP address from DHCP,
     * otherwise NTP option would be rejected by default.
     */
    ESP_LOGI(TAG, "Initializing SNTP");
    esp_sntp_config_t config = ESP_NETIF_SNTP_DEFAULT_CONFIG(CONFIG_SNTP_TIME_SERVER);
    config.start = false;                       // start SNTP service explicitly (after connecting)
    config.server_from_dhcp = true;             // accept NTP offers from DHCP server, if any (need to enable *before* connecting)
    config.renew_servers_after_new_IP = true;   // let esp-netif update configured SNTP server(s) after receiving DHCP lease
    config.index_of_first_server = 1;           // updates from server num 1, leaving server 0 (from DHCP) intact
    // configure the event on which we renew servers
#ifdef CONFIG_EXAMPLE_CONNECT_WIFI
    config.ip_event_to_renew = IP_EVENT_STA_GOT_IP;
#else
    config.ip_event_to_renew = IP_EVENT_ETH_GOT_IP;
#endif
    config.sync_cb = time_sync_notification_cb; // only if we need the notification function
    esp_netif_sntp_init(&config);

#endif /* LWIP_DHCP_GET_NTP_SRV */

    /* This helper function configures Wi-Fi or Ethernet, as selected in menuconfig.
     * Read "Establishing Wi-Fi or Ethernet Connection" section in
     * examples/protocols/README.md for more information about this function.
     */
    ESP_ERROR_CHECK(example_connect());

#if LWIP_DHCP_GET_NTP_SRV
    ESP_LOGI(TAG, "Starting SNTP");
    esp_netif_sntp_start();
#if LWIP_IPV6 && SNTP_MAX_SERVERS > 2
    /* This demonstrates using IPv6 address as an additional SNTP server
     * (statically assigned IPv6 address is also possible)
     */
    ip_addr_t ip6;
    if (ipaddr_aton("2a01:3f7::1", &ip6)) {    // ipv6 ntp source "ntp.netnod.se"
        esp_sntp_setserver(2, &ip6);
    }
#endif  /* LWIP_IPV6 */

#else
    ESP_LOGI(TAG, "Initializing and starting SNTP");
#if CONFIG_LWIP_SNTP_MAX_SERVERS > 1
    /* This demonstrates configuring more than one server
     */
    esp_sntp_config_t config = ESP_NETIF_SNTP_DEFAULT_CONFIG_MULTIPLE(2,
                               ESP_SNTP_SERVER_LIST(CONFIG_SNTP_TIME_SERVER, "pool.ntp.org" ) );
#else
    /*
     * This is the basic default config with one server and starting the service
     */
    esp_sntp_config_t config = ESP_NETIF_SNTP_DEFAULT_CONFIG(CONFIG_SNTP_TIME_SERVER);
#endif
    config.sync_cb = time_sync_notification_cb;     // Note: This is only needed if we want
#ifdef CONFIG_SNTP_TIME_SYNC_METHOD_SMOOTH
    config.smooth_sync = true;
#endif

    esp_netif_sntp_init(&config);
#endif

    print_servers();

    // wait for time to be set
    time_t now = 0;
    struct tm timeinfo = { 0 };
    int retry = 0;
    const int retry_count = 15;
    while (esp_netif_sntp_sync_wait(2000 / portTICK_PERIOD_MS) == ESP_ERR_TIMEOUT && ++retry < retry_count) {
        ESP_LOGI(TAG, "Waiting for system time to be set... (%d/%d)", retry, retry_count);
    }
    time(&now);
    localtime_r(&now, &timeinfo);

    ESP_ERROR_CHECK( example_disconnect() );
    esp_netif_sntp_deinit();
}
```

### 打印函数

打印服务器名称，或ip4/ip6的地址

```c
static void print_servers(void)
{
    ESP_LOGI(TAG, "List of configured NTP servers:");

    for (uint8_t i = 0; i < SNTP_MAX_SERVERS; ++i){
        if (esp_sntp_getservername(i)){
            ESP_LOGI(TAG, "server %d: %s", i, esp_sntp_getservername(i));
        } else {
            // we have either IPv4 or IPv6 address, let's print it
            char buff[INET6_ADDRSTRLEN];
            ip_addr_t const *ip = esp_sntp_getserver(i);
            if (ipaddr_ntoa_r(ip, buff, INET6_ADDRSTRLEN) != NULL)
                ESP_LOGI(TAG, "server %d: %s", i, buff);
        }
    }
}
```

## 总结

本例程是采用SNTP模式进行时间设置的示例，主要演示了如何获取RTC时间，设置RTC时间，还有配置SNTP，包括网络和服务器。有关SNTP的配置项是需要了解的，如果是手动设置服务器，需要知道`settime`相关函数的使用
