# ESP_frpc ESP作为frp cline

## 文档简介

esp_frpc是一个使用C语言实现的内网穿透客户端，运行于基于freeRTOS操作系统的ESP8266上，支持tcp连接，编译后的二进制代码长度约500KBytes。

但由于该文件是基于ESP8266和make编写，所以本文档是架构分析、代码分析、下一个文档会进行移植整理和总结

## 代码分析

### 代码架构

```txt
- main
  - config 包括引脚、wifi、frp服务器、代理等
    - config.h
    - config.c
  - control 各种模块的初始化
    - control.h
    - control.c
  - crypto 
    - crypto.h
    - crpyto.c
  - login
    - login.h
    - login.c
  - msg
    - msg.h
    - msg.c
  - sntp
    - sntp.h
    - sntp.c
  - tcpmux
    - tcpmux.h
    - tcpmux.c
  - timer 定时器相关
    - timer.h
    - timer.c
  - webserver web服务管理函数
    - webserver.h
    - webserver.c
  - wifi_ap
    - wifi_ap.h
    - wifi_ap.c
  - main.c
- config
  - nvs_data.csv
  - partitions.csv
  - web_config.html
```

### main-config

#### 头文件(只保留主要部分)

1. 引脚定义和掩码配置（ULL unsigned long long）
2. 结构体配置中以字符类型储存网络相关配置数据，整数类型储存数字量

```c
// GPIO pin definitions
#define RELAY               5       // Relay control pin (1:ON/0:OFF)
#define POWER_LED           12      // Power indicator LED (RED) (0:ON/1:OFF)
#define LINK_LED            13      // Network status LED (BLUE) (0:ON/1:OFF)
#define KEY                 0       // Button input pin

// GPIO pin selection masks
#define GPIO_OUTPUT_PIN_SEL ((1ULL<<RELAY) | (1ULL<<POWER_LED) | (1ULL<<LINK_LED))
#define GPIO_INPUT_PIN_SEL  ((1ULL<<KEY))

// 配置结构体 - 包含所有可配置的参数
typedef struct {
    // WiFi配置
    char wifi_ssid[32];
    char wifi_password[64];
    char wifi_encryption[8];  // WPA2, WPA, WEP, NONE
    
    // frp服务器配置
    char frp_server[64];
    uint16_t frp_port;
    char frp_token[64];
    
    // 代理配置
    char proxy_name[32];
    char proxy_type[8];  // tcp, udp, http, https
    char local_ip[16];
    uint16_t local_port;
    uint16_t remote_port;
    
    // 心跳配置
    uint16_t heartbeat_interval;
    uint16_t heartbeat_timeout;
    
    // 配置版本号
    uint32_t config_version;
    
} device_config_t;

// 默认配置
extern const device_config_t default_config;

// 全局配置变量
extern device_config_t g_device_config;

```

#### config默认配置

* `wifi_ssid` ESP作为AP的WiFi SSID
* `wifi_password` WiFi AP模式的密码设置
* `wifi_encryption` WiFi编码方式
* frp相关
  * `frp_server` 服务器ip地址
  * `frp_port` 服务器监听端口号（客户端与服务器初始连接和控制）
  * `frp_token` frp token（身份验证密钥）
  * `proxy_name` 代理会话名称
  * `proxy_type` 创建的代理类型，决定传输协议（tcp udp http）
  * `local_ip` 本地ip地址 （内网中服务的IP地址）
  * `local_port` 本地端口号
  * `remote_port` frp服务器监听的端口（frp服务器对外暴露的端口号）
* `heartbeat_interval` 发送心跳包的间隔（秒）
* `heartbeat_timeout` 超时配置（秒）
* `config_version` 版本号

```c
// 默认配置
const device_config_t default_config = {
    .wifi_ssid = "ESP8266_Config",
    .wifi_password = "12345678",
    .wifi_encryption = "WPA2",
    .frp_server = "192.168.1.100",
    .frp_port = 7000,
    .frp_token = "52010",
    .proxy_name = "ssh-ubuntu",
    .proxy_type = "tcp",
    .local_ip = "127.0.0.1",
    .local_port = 22,
    .remote_port = 7005,
    .heartbeat_interval = 30,
    .heartbeat_timeout = 90,
    .config_version = 1
};
```

#### 初始化配置系统

`config_init` 初始化函数

1. 从默认配置中拷贝`default_config`
2. 自定义函数从NVS中加载配置
3. 打印配置

`config_load_from_nvs` 从NVS加载配置

1. `nvs_open` 选择使用带有NVS标签的分区（分区名，打开方式[读写/只读]，输出句柄）
2. 错误输出（`esp_err_to_name`是内置库函数，用于把错误转换成字符说明输出）
3. 获取NVS中的版本号，如果和默认配置不一致就不使用内部储存(即如果结构体配置的数据种类和数量都不同，无法使用)
4. 从nvs中根据不同键值获取数据，存入全局变量`g_device_config`
5. 关闭nvs

```c
/**
 * 初始化配置系统
 */
esp_err_t config_init(void)
{
    ESP_LOGI(TAG, "Initializing configuration system...");
    
    // 先加载默认配置
    memcpy(&g_device_config, &default_config, sizeof(device_config_t));
    
    // 尝试从NVS加载配置
    esp_err_t ret = config_load_from_nvs();
    if (ret != ESP_OK) {
        ESP_LOGW(TAG, "Failed to load config from NVS, using default config");
        // 保存默认配置到NVS
        //config_save_to_nvs();
    }
    
    config_print();
    return ESP_OK;
}

/**
 * 从NVS加载配置
 */
 // ... existing code ...
esp_err_t config_load_from_nvs(void)
{
    nvs_handle nvs_handle;
    esp_err_t err;

    err = nvs_open("config", NVS_READONLY, &nvs_handle);
    if (err != ESP_OK) {
        ESP_LOGE(TAG, "Error opening NVS handle: %s", esp_err_to_name(err));
        return err;
    }

    uint32_t version = 0;
    err = nvs_get_u32(nvs_handle, "version", &version);
    if (err != ESP_OK) goto fail;

    if (version != default_config.config_version) {
        nvs_close(nvs_handle);
        return ESP_ERR_INVALID_VERSION;
    }

    size_t len;
    len = sizeof(g_device_config.wifi_ssid);
    err = nvs_get_str(nvs_handle, "wifi_ssid", g_device_config.wifi_ssid, &len);
    if (err != ESP_OK) goto fail;
    len = sizeof(g_device_config.wifi_password);
    err = nvs_get_str(nvs_handle, "wifi_password", g_device_config.wifi_password, &len);
    if (err != ESP_OK) goto fail;
    len = sizeof(g_device_config.wifi_encryption);
    err = nvs_get_str(nvs_handle, "wifi_enc", g_device_config.wifi_encryption, &len);
    if (err != ESP_OK) goto fail;
    len = sizeof(g_device_config.frp_server);
    err = nvs_get_str(nvs_handle, "frp_srv", g_device_config.frp_server, &len);
    if (err != ESP_OK) goto fail;
    err = nvs_get_u16(nvs_handle, "frp_port", &g_device_config.frp_port);
    if (err != ESP_OK) goto fail;
    len = sizeof(g_device_config.frp_token);
    err = nvs_get_str(nvs_handle, "frp_tok", g_device_config.frp_token, &len);
    if (err != ESP_OK) goto fail;
    len = sizeof(g_device_config.proxy_name);
    err = nvs_get_str(nvs_handle, "prx_name", g_device_config.proxy_name, &len);
    if (err != ESP_OK) goto fail;
    len = sizeof(g_device_config.proxy_type);
    err = nvs_get_str(nvs_handle, "prx_type", g_device_config.proxy_type, &len);
    if (err != ESP_OK) goto fail;
    len = sizeof(g_device_config.local_ip);
    err = nvs_get_str(nvs_handle, "loc_ip", g_device_config.local_ip, &len);
    if (err != ESP_OK) goto fail;
    err = nvs_get_u16(nvs_handle, "loc_port", &g_device_config.local_port);
    if (err != ESP_OK) goto fail;
    err = nvs_get_u16(nvs_handle, "rmt_port", &g_device_config.remote_port);
    if (err != ESP_OK) goto fail;
    err = nvs_get_u16(nvs_handle, "hb_itvl", &g_device_config.heartbeat_interval);
    if (err != ESP_OK) goto fail;
    err = nvs_get_u16(nvs_handle, "hb_to", &g_device_config.heartbeat_timeout);
    if (err != ESP_OK) goto fail;
    g_device_config.config_version = version;

    nvs_close(nvs_handle);
    ESP_LOGI(TAG, "Configuration loaded from NVS successfully");
    return ESP_OK;

fail:
    ESP_LOGE(TAG, "Error reading config item: %s", esp_err_to_name(err));
    nvs_close(nvs_handle);
    return err;
}
/**
 * 打印当前配置
 */
void config_print(void)
{
    ESP_LOGI(TAG, "=== Current Configuration ===");
    ESP_LOGI(TAG, "WiFi SSID: %s", g_device_config.wifi_ssid);
    ESP_LOGI(TAG, "WiFi Encryption: %s", g_device_config.wifi_encryption);
    ESP_LOGI(TAG, "frp Server: %s:%d", g_device_config.frp_server, g_device_config.frp_port);
    ESP_LOGI(TAG, "Proxy Name: %s", g_device_config.proxy_name);
    ESP_LOGI(TAG, "Proxy Type: %s", g_device_config.proxy_type);
    ESP_LOGI(TAG, "Local Service: %s:%d", g_device_config.local_ip, g_device_config.local_port);
    ESP_LOGI(TAG, "Remote Port: %d", g_device_config.remote_port);
    ESP_LOGI(TAG, "Heartbeat: %d/%d", g_device_config.heartbeat_interval, g_device_config.heartbeat_timeout);
    ESP_LOGI(TAG, "Config Version: %u", g_device_config.config_version);
    ESP_LOGI(TAG, "================================");
} 
```

#### 配置修改和保存

调用函数把全局变量`g_device_config`修改为默认配置`default_config`。`config_save_to_nvs`将配置存入nvs。

1. 打开nvs
2. 把需要的配置写入nvs缓冲区`nvs_handle`(函数变量)
3. `nvs_commit` 写入nvs分区
4. `nvs_close` 关闭分区并释放占用资源（参数为open提供的句柄）

```c
/**
 * 重置配置为默认值
 */
esp_err_t config_reset_to_default(void)
{
    ESP_LOGI(TAG, "Resetting configuration to default values...");
    
    memcpy(&g_device_config, &default_config, sizeof(device_config_t));
    
    esp_err_t ret = config_save_to_nvs();
    if (ret == ESP_OK) {
        ESP_LOGI(TAG, "Configuration reset successfully");
    }
    
    return ret;
}
esp_err_t config_save_to_nvs(void)
{
    nvs_handle nvs_handle;
    esp_err_t err;

    err = nvs_open("config", NVS_READWRITE, &nvs_handle);
    if (err != ESP_OK) {
        ESP_LOGE(TAG, "Error opening NVS handle: %s", esp_err_to_name(err));
        return err;
    }

    err = nvs_set_u32(nvs_handle, "version", g_device_config.config_version);
    if (err != ESP_OK) goto fail;

    err = nvs_set_str(nvs_handle, "wifi_ssid", g_device_config.wifi_ssid);
    if (err != ESP_OK) goto fail;
    err = nvs_set_str(nvs_handle, "wifi_password", g_device_config.wifi_password);
    if (err != ESP_OK) goto fail;
    err = nvs_set_str(nvs_handle, "wifi_enc", g_device_config.wifi_encryption);
    if (err != ESP_OK) goto fail;
    err = nvs_set_str(nvs_handle, "frp_srv", g_device_config.frp_server);
    if (err != ESP_OK) goto fail;
    err = nvs_set_u16(nvs_handle, "frp_port", g_device_config.frp_port);
    if (err != ESP_OK) goto fail;
    err = nvs_set_str(nvs_handle, "frp_tok", g_device_config.frp_token);
    if (err != ESP_OK) goto fail;
    err = nvs_set_str(nvs_handle, "prx_name", g_device_config.proxy_name);
    if (err != ESP_OK) goto fail;
    err = nvs_set_str(nvs_handle, "prx_type", g_device_config.proxy_type);
    if (err != ESP_OK) goto fail;
    err = nvs_set_str(nvs_handle, "loc_ip", g_device_config.local_ip);
    if (err != ESP_OK) goto fail;
    err = nvs_set_u16(nvs_handle, "loc_port", g_device_config.local_port);
    if (err != ESP_OK) goto fail;
    err = nvs_set_u16(nvs_handle, "rmt_port", g_device_config.remote_port);
    if (err != ESP_OK) goto fail;
    err = nvs_set_u16(nvs_handle, "hb_itvl", g_device_config.heartbeat_interval);
    if (err != ESP_OK) goto fail;
    err = nvs_set_u16(nvs_handle, "hb_to", g_device_config.heartbeat_timeout);
    if (err != ESP_OK) goto fail;

    err = nvs_commit(nvs_handle);
    if (err != ESP_OK) goto fail;

    nvs_close(nvs_handle);
    ESP_LOGI(TAG, "Configuration saved to NVS successfully");
    return ESP_OK;

fail:
    ESP_LOGE(TAG, "Error writing config item: %s", esp_err_to_name(err));
    nvs_close(nvs_handle);
    return err;
}
```

#### 检查配置

通过读取引脚电平，判断按钮是否按下，应该和别的函数配合进入配置模式，`config`中只控制灯的闪烁状态

### main-control(主要内容)

#### 头文件和结构定义

`Control_t` 作为系统状态总控结构体
`msg_type_t` 枚举定义消息类型
`ProxyService_t` 代理服务配置结构体，参数注释如下
`ProxyClient_t` 代理客户端状态结构体，参数注释如下

```c
typedef struct Control {
    int iMainSock;  //main socketfd
    tmux_stream_t  stream;
} Control_t;

typedef enum msg_type {
    TypeLogin                 = 'o',
    TypeLoginResp             = '1',
    TypeNewProxy              = 'p',
    TypeNewProxyResp          = '2',
    TypeCloseProxy            = 'c',
    TypeNewWorkConn           = 'w',
    TypeReqWorkConn           = 'r',
    TypeStartWorkConn         = 's',
    TypeNewVisitorConn        = 'v',
    TypeNewVisitorConnResp    = '3',
    TypePing                  = 'h',
    TypePong                  = '4',
    TypeUDPPacket             = 'u',
    TypeNatHoleVisitor        = 'i',
    TypeNatHoleClient         = 'n',
    TypeNatHoleResp           = 'm',
    TypeNatHoleClientDetectOK = 'd',
    TypeNatHoleSid            = '5',
}msg_type_t;

typedef struct proxy_service {
    char     *proxy_name;           // 代理名称
    char     *proxy_type;           // 代理类型
    char     *ftp_cfg_proxy_name;   // 用于ftp的某个配置
    int      use_encryption;        // 是否启用数据加密传输
    int      use_compression;       // 是否启用数据压缩传输

    char    *local_ip;              // 本地服务ip地址
    int      remote_port;           // frp服务对外暴露端口
    int      remote_data_port;      // 用于数据通道的额外远程端口
    int      local_port;            // 本地服务端口号
}ProxyService_t;

typedef struct proxy_client {
    int iMainSock;          // xfrpc proxy <---> frps 主套接字描述符，储存frps连接状态
    int iLocalSock;         // xfrpc proxy <---> local service 本地套接字描述符，与本地数据连接状态
    struct tmux_stream stream; // 数据流管理结构体
    uint32_t    stream_id;     // 流ID，标识当前代理会话的数据流
    int connected;      // 连接状态标志位
    int work_started;   // 工作状态标志位
    struct  proxy_service   *ps;    // 指向关联结构体的指针
    unsigned char   *data_tail; // storage untreated data 缓冲区指针
    size_t data_tail_size; // 数据大小

}ProxyClient_t;
```

#### 初始化控制相关函数

1. 如果`g_pMainCtl`已经初始化，进行`close`（关闭底层TCP连接，释放资源）和`free`（释放分配的结构体内存），是防御性编程思路，确保资源不泄露，状态不冲突
2. `calloc`分配内存并初始化初值为1
3. 初始化结构体数值

```c
/**
 * Initialize main control structure
 * @return _SUCCESS on success, _FAIL otherwise
 */
int init_main_control() {    
    if (g_pMainCtl && g_pMainCtl->iMainSock) {
        ESP_LOGE(TAG, "error: main control or base socket already exists!");
        close(g_pMainCtl->iMainSock);
        free(g_pMainCtl);
        return _FAIL;
    }
    
    g_pMainCtl = calloc(sizeof(Control_t), 1);
    if (NULL == g_pMainCtl) {
        ESP_LOGE(TAG, "error: main control init failed!");
        return _FAIL;
    }  
    
    g_pMainCtl->iMainSock = -1;
    g_pMainCtl->stream.id = g_session_id;  // Set session ID
    g_pMainCtl->stream.state = INIT;       // Initial stream state

    return _SUCCESS;
}
```

1. 分配内存并设初始为1
2. `strdup`函数分配合适内存并将字符串复制到新空间内，返回内存指针
3. 对于数值直接赋值
4. `dump_common_conf` 函数负责进行日志打印，输出地址和端口号

```c
/**
 * Initialize proxy service configuration
 */
void init_proxy_Service() {
    g_pProxyService = (ProxyService_t *)calloc(sizeof(ProxyService_t), 1);
    if (NULL == g_pProxyService) {
        ESP_LOGE(TAG, "error: init proxy service _FAIL");
        return;
    }

    // Set proxy configuration parameters from NVS config
    g_pProxyService->proxy_name = strdup(g_device_config.proxy_name);
    g_pProxyService->proxy_type = strdup(g_device_config.proxy_type);
    g_pProxyService->local_ip = strdup(g_device_config.local_ip);
    g_pProxyService->local_port = g_device_config.local_port;
    g_pProxyService->remote_port = g_device_config.remote_port;

    dump_common_conf();  // Log configuration details
}
```

初始化流程和上述基本相同。

在 FRP 这类反向代理协议中，一个典型的代理会话通常包含两种类型的通信： 控制流/数据流。
在奇偶分配策略中，偶数id代表客户端到服务端；奇数id代表服务端到客户端

综上，为了扩展性和分配策略，id号每次加2

```c
/**
 * Create new proxy client instance
 * @return Initialized proxy client structure
 */
ProxyClient_t *new_proxy_client() {
    g_session_id += 2;  // Increment session ID
    ProxyClient_t *client = calloc(1, sizeof(ProxyClient_t));
    client->stream_id = g_session_id;       // Assign stream ID
    client->iMainSock = g_pMainCtl->iMainSock;  // Share main socket
    client->stream.id = g_session_id;       // Set stream ID
    client->stream.state = INIT;            // Initial stream state
    return client;
}
```

#### 服务器相关函数

1. `addr_str`字符数组用于存储字符形式的IP地址
2. `addr_family` 被设置为`AF_INET` 表示使用IPv4协议
3. `ip_protocol` 用于指定传输层协议，设为`IPPROTO_IP`即默认的IP协议
4. `inet_addr`函数将服务器IP地址（字符串）转换为网络字节序的32位整数
5. 设置`destAddr`的参数 `htons`把主机字节序转换位网络字节序
6. `inet_ntoa_r`将ip地址转换为字符串格式并写入缓冲区
7. `socket(addr_family, SOCK_STREAM, ip_protocol)` 创建TCP套接字，`SOCK_STREAM`流式套接字，对应TCP协议
8. `connect(MainSock, (struct sockaddr *)&destAddr, sizeof(destAddr));`阻塞调用，尝试连接到目标服务器，目标地址在`destAddr`中
9. `send_window_update` 自定义函数，发送窗口更新消息，通知服务器当前客户端的数据接收能力
10. `login`自定义函数，向服务器发起登录请求
11. `process_data`主循环中调用函数处理来自服务器的消息

```c
/**
 * Establish connection to the remote server
 */
void connect_to_server() {
    char addr_str[128];
    int addr_family;
    int ip_protocol;
    int err;

    struct sockaddr_in destAddr;
    destAddr.sin_addr.s_addr = inet_addr(g_device_config.frp_server);  // Server IP from NVS config
    destAddr.sin_family = AF_INET;                                     // IPv4
    destAddr.sin_port = htons(g_device_config.frp_port);               // Server port from NVS config
    addr_family = AF_INET;
    ip_protocol = IPPROTO_IP;
    inet_ntoa_r(destAddr.sin_addr, addr_str, sizeof(addr_str) - 1);

    // Timer is already initialized in main.c

    int MainSock = socket(addr_family, SOCK_STREAM, ip_protocol);
    if (MainSock < 0) {
        ESP_LOGE(TAG, "Unable to create socket: errno %d", errno);
        return;
    }
    ESP_LOGI(TAG, "Socket created");

    err = connect(MainSock, (struct sockaddr *)&destAddr, sizeof(destAddr));
    if (err != 0) {
        ESP_LOGE(TAG, "Socket unable to connect: errno %d", errno);
        close(MainSock);
        RESET_DEVICE;  // 异常断开直接重启，不需要LED闪烁
        return;
    }

    g_pMainCtl->iMainSock = MainSock;
    ESP_LOGI(TAG, "Successfully connected");

    send_window_update(MainSock, &g_pMainCtl->stream, 0);  // window update
    login(MainSock);  // Perform login procedure
    
    while (1) {  // Main processing loop
        process_data();
    }
}
```

1. `login_request_marshal` 将登录所需的信息打包成JSON协议格式，返回生成的消息长度
2. `send_msg_frp_server`发送登录消息
   * `Sockfd` 已连接的描述符
   * `TypeLogin` 枚举中定义消息类型为 'o' 标识登录请求
   * `lg_msg` 序列化后的消息内容
   * `len` 消息长度
   * `&g_pMainCtl->stream` 关联的流控制结构体
3. `SAFE_FREE` 宏定义函数，释放指针指向的内存并把指针置为NULL

```c
/**
 * Perform login procedure
 * @param Sockfd Socket descriptor
 * @return _SUCCESS on success
 */
int login(int Sockfd) {
    char *lg_msg = NULL;
    uint len = login_request_marshal(&lg_msg);  // Marshal login request
    
    if (!lg_msg) {
        ESP_LOGE(TAG, "error: login_request_marshal failed");
        exit(0);
    }
    
    send_msg_frp_server(Sockfd, TypeLogin, lg_msg, len, &g_pMainCtl->stream);
    ESP_LOGI(TAG, "info: end login procedure");
    
    SAFE_FREE(lg_msg);  // Free allocated message buffer
    return _SUCCESS;
}
```

函数向服务器发送本地代理配置请求提供反向代理通道

1. `new_proxy_service_marshal`将ps指向的代理配置打包成frp协议规定的JSON格式，并返回消息长度
2. `send_enc_msg_frp_server` 发送新建代理的请求（enc代表加密）
   * `g_pMainCtl->iMainSock` 主socket描述符
   * `TypeNewProxy` 枚举中定义消息类型为 'p' 标识是创建代理的请求
   * `new_proxy_msg` 序列化后的消息内容
   * `len` 消息长度
   * `&g_pMainCtl->stream` 关联的流控制结构体
3. `SAFE_FREE`安全释放内存

```c
/**
 * Start proxy services by sending configuration to server
 */
void start_proxy_services() {
    ProxyService_t *ps = g_pProxyService;
    
    char *new_proxy_msg = NULL;
    int len = new_proxy_service_marshal(ps, &new_proxy_msg);  // Marshal proxy config
    if (!new_proxy_msg) {
        ESP_LOGI(TAG, "proxy service request marshal failed");
        return;
    }

    ESP_LOGI(TAG, "control proxy client: [Type %d : proxy_name %s : msg_len %d]", 
             TypeNewProxy, ps->proxy_name, len);
    
    send_enc_msg_frp_server(g_pMainCtl->iMainSock, TypeNewProxy, new_proxy_msg, len, &g_pMainCtl->stream);
    SAFE_FREE(new_proxy_msg);  // Free message buffer
}
```

`new_client_connect` 新建客户端连接

1. `new_proxy_client` 初始化client的结构体
2. `send_window_update` 发送窗口更新消息
3. `new_work_connection` 自定义函数和服务端进行连接

`new_work_connection` 函数

1. 分配`struct work_conn`结构体内存
2. 绑定本地运行id
3. `new_work_conn_marshal`将连接配置转换成JSON格式并返回消息长度
4. `send_msg_frp_server`发送登录消息
   * `iSock` 已连接的描述符
   * `TypeNewWorkConn` 枚举中定义消息类型为 'w' 标识新建工作连接（服务器分配独立的数据通道）
   * `new_work_conn_request_message` 序列化后的消息内容
   * `nret` 消息长度
   * `stream` 关联的流控制结构体
5. `SAFE_FREE` 安全释放内存

```c
/**
 * Handle new client connection through TCP multiplexer
 */
void new_client_connect() {
    g_pClient = new_proxy_client();  // Create client instance
    ESP_LOGI(TAG, "new client through tcp mux: %d", g_pClient->stream_id);
    send_window_update(g_pClient->iMainSock, &g_pClient->stream, 0);  // window Update
    new_work_connection(g_pMainCtl->iMainSock, &g_pClient->stream);   // Establish work connection
}

/**
 * Establish new work connection with server
 * @param iSock Main socket descriptor
 * @param stream Stream context
 */
void new_work_connection(int iSock, struct tmux_stream *stream) {
    assert(iSock);
    
    struct work_conn *work_c = calloc(1, sizeof(struct work_conn));
    work_c->run_id = g_pLogin->run_id;  // Get run ID from login context
    if (!work_c->run_id) {
        ESP_LOGI(TAG, "cannot found run ID");
        SAFE_FREE(work_c);
        return;
    }
    
    char *new_work_conn_request_message = NULL;
    int nret = new_work_conn_marshal(work_c, &new_work_conn_request_message);
    if (0 == nret) {
        ESP_LOGI(TAG, "new work connection request marshal failed!");
        return;
    }

    send_msg_frp_server(iSock, TypeNewWorkConn, new_work_conn_request_message, nret, stream);

    SAFE_FREE(new_work_conn_request_message);
    SAFE_FREE(work_c);
}
```

#### 消息处理函数

1. 初始化函数内变量，结构体数据置零，描述符绑定
2. `read（文件描述符，指向缓冲区的指针，最多读取字节）`函数读取原始字节数据，返回实际读取的字节数（read是POSIX标准定义的系统调用）
3. `tmux_hdr`读取的头部结构，通常包含：消息类型、数据长度、流ID、校验和等
4. `ntohs` 将16位网络字节序转换为主机字节序 `ntohl`将32位网络字节序转换为主机字节序 （网络序固定为大端模式，主机多为小端模式）
5. 根据`streamId`绑定不同数据流，1为主控制流，处理所有与连接相关的控制消息；其他为工作数据流，处理具体代理会话的数据转发
6. `process_flags`根据不同的标识处理流数据格式
7. 如果类型是`DATA`数据传输
   1. 初始化储存缓冲
   2. `read`读取数据
   3. `my_aes_decrypt`是自定义的解码函数
   4. `0 == g_ProxyWork`判断客户端是登录和初始化阶段，0代表未建立代理通道
   5. `handle_login_response` 解析服务器返回登录结果，设置身份验证标志位
   6. 身份验证后根据接收到的数据调用`init_decoder``init_encoder`对加密模块初始化
   7. `TypeReqWorkConn`代表请求建立工作连接，如果未连接，写TCP流数据，发送本地服务注册请求
   8. 有置位则`new_client_connect`创建新的代理客户端并连接，设置`g_ProxyWork = 1`进入代理工作模式
   9. `g_session_id == streamId` 用于ESP向服务器方向的工作流
   10. `linked`代表工作连接已建立，打印原始数据，`tmux_stream_write`向服务器回复确认消息
   11. `strstr(g_RxBuffer, "POWER_ON")`查找返回消息中是否有该字串（即指令匹配），进行操作
   12. `TypeStartWorkConn == mhdr->type`服务器发送消息，类型为工作连接，代表连接建立，设置linked和本地状态
   13. `1 == streamId` 用于主控制流，处理控制消息。`TypePong`类型调用`obtain_time`从服务端获取时间
   14. `send_window_update`向服务器发送窗口更新，告知可继续发送数据
   15. `SAFE_FREE`释放临时解码缓冲区
8. 如果是`PING`类型数据，调用函数处理请求

```c
/**
 * Process incoming data from server
 */
void process_data() {
    struct tcp_mux_header tmux_hdr;
    uint uiHdrLen;
    uint stream_len;
    uint16_t flags;
    int MainSock = g_pMainCtl->iMainSock;
    int streamId;
    int rx_len;
    size_t pt_len;
    struct msg_hdr* mhdr;
    uchar *decrypted = NULL;
    tmux_stream_t cur_stream;
    
    memset(&tmux_hdr, 0, sizeof(tmux_hdr));
    uiHdrLen = read(MainSock, &tmux_hdr, sizeof(tmux_hdr));  // Read header

    if (uiHdrLen < sizeof(tmux_hdr)) {
        ESP_LOGI(TAG, "uiHdrLen [%d] < sizeof tmux_hdr", uiHdrLen);
        RESET_DEVICE;
        return;
    }

    flags = ntohs(tmux_hdr.flags);          // Extract flags
    stream_len = ntohl(tmux_hdr.length);    // Extract data length
    streamId = ntohl(tmux_hdr.stream_id);   // Extract stream ID

    // Select stream context based on ID
    if (1 == streamId) {
        cur_stream = g_pMainCtl->stream;
    } else {
        cur_stream = g_pClient->stream;
    }
    if (!process_flags(flags, &(cur_stream))) {  // Process protocol flags
        return;
    }

    switch (tmux_hdr.type) {
        case DATA: {
            memset(&g_RxBuffer, 0, sizeof(g_RxBuffer));
            rx_len = read(MainSock, g_RxBuffer, stream_len);  // Read payload

            if (0 == rx_len) {  // Connection closed
	            close(MainSock);
            	RESET_DEVICE;  // 异常断开直接重启，不需要LED闪烁
		        return;
            }

            // Handle encrypted data for main stream
            if (decoder && (1 == streamId)) {
                decrypted = calloc(1, stream_len);
                my_aes_decrypt((uchar*)g_RxBuffer, stream_len, decrypted, &pt_len);
                mhdr = (struct msg_hdr*)decrypted;
                ESP_LOGI(TAG, "type: %c", mhdr->type);
                ESP_LOGI(TAG, "data: %s", mhdr->data);
            } else {  // Plaintext handling
                mhdr = (struct msg_hdr*)g_RxBuffer;
                ESP_LOGI(TAG, "type: %c", mhdr->type);
                ESP_LOGI(TAG, "data: %s", mhdr->data);
            }
            
            if (0 == g_ProxyWork) { 
                if (TypeLoginResp == mhdr->type) {  // Login response
                    ESP_LOGI(TAG, "type: TypeLoginResp");
                    handle_login_response(g_RxBuffer, rx_len);
                    g_IsLogged = 1;
                } else if (g_IsLogged && (NULL == decoder)) {  // Init crypto
                    if (rx_len != 16) {  // Check IV length
                        ESP_LOGI(TAG, "info: iv length != 16");
                        break;
                    }
                    init_decoder((uint8_t*)g_RxBuffer);  // Initialize decoder
                    init_encoder((uint8_t*)g_RxBuffer);  // Initialize encoder
                } else if(TypeReqWorkConn == mhdr->type) {  // Work conn request
                    ESP_LOGI(TAG, "mhdr->type == TypeReqWorkConn");
                    if (!client_connected) {
                        tmux_stream_write(MainSock, (char*)encoder->iv, 16, &g_pMainCtl->stream);
                        start_proxy_services();  // Activate proxy
                        client_connected = 1;
                    }
                    new_client_connect();  // Create client connection
                    g_ProxyWork = 1;       // Enable proxy operation
                } else if(TypeNewProxyResp == mhdr->type) {  // Proxy response
                    ESP_LOGI(TAG, "mhdr->type == TypeNewProxyResp");
                }
            } else {
                if (g_session_id == streamId) {  // Client stream
                    if(1 == linked) {  // Data transfer
                        // Debug print received data
                        printf("\n################DATA RECIEVE################\n");
                        for(int i=0;i<stream_len;i++) {
                            if(isprint(g_RxBuffer[i])) printf("%c", g_RxBuffer[i]); 
                            else printf(".");
                        }
                        printf("\n\n");
                        for(int i=0;i<stream_len;i++) {
                            printf("%02X ", g_RxBuffer[i]);
                        }
                        printf("\n############################################\n\n");
                        
                        // Send acknowledgment
                        char buf[32] = {0};
                        sprintf(buf, "%d bytes recieved!\n", stream_len);
                        tmux_stream_write(g_pMainCtl->iMainSock, buf, strlen(buf), &g_pClient->stream);
                        
                        // GPIO control logic (LED and Relay control)
                        if(strstr(g_RxBuffer, "POWER_ON")) {
                            gpio_set_level(POWER_LED, 0);   // Turn ON power LED (active-low)
                            gpio_set_level(RELAY, 1);       // Turn ON relay
                            ESP_LOGI(TAG, "POWER_ON command received - LED and Relay activated");
                        }
                        if(strstr(g_RxBuffer, "POWER_OFF")) {
                            gpio_set_level(POWER_LED, 1);   // Turn OFF power LED (active-low)
                            gpio_set_level(RELAY, 0);       // Turn OFF relay
                            ESP_LOGI(TAG, "POWER_OFF command received - LED and Relay deactivated");
                        }
                    }
                    if(TypeStartWorkConn == mhdr->type) {
                        linked = 1;  // Mark connection ready
                        set_frpc_connection_connected();  // Set NET LED to constant on
                    }
                } else if (1 == streamId) {  // Main control stream
                    ESP_LOGI(TAG, "g_ControlState: StateProxyWork");
                    if (TypePong == mhdr->type) {  // Keep-alive response
                        g_Pongtime = obtain_time();
                        ESP_LOGI(TAG, "msg->type: TypePong");
                    }
                }
                send_window_update(g_pMainCtl->iMainSock, &g_pClient->stream, stream_len);  // Update window
            }
            if (1 == streamId) {
                SAFE_FREE(decrypted);  // Cleanup decryption buffer
            }
            break;
        }
        case PING: {  // Handle keep-alive ping
            handle_tcp_mux_ping(&tmux_hdr);
            break;
        }
    }
}
```

1. `switch`判断状态，对于`LOCAL_CLOSE` `CLOSED` `RESET` 日志打印连接关闭，直接返回
2. `get_send_flags` 自定义函数获取标志符
3. `tcp_mux_send_hdr` 自定义函数发送数据头
4. `send` 函数用于通过已连接的套接字发送数据

```c
/**
 * Write data to TCP multiplexing stream
 * @param Sockfd Socket descriptor
 * @param data Data buffer to send
 * @param length Data length
 * @param pstream Stream context
 * @return _SUCCESS on success, _FAIL otherwise
 */
uint tmux_stream_write(int Sockfd, char *data, uint length, tmux_stream_t *pstream) {
    uint uiRet = _SUCCESS;
        
    switch(pstream->state) {
    case LOCAL_CLOSE:
    case CLOSED:
    case RESET:
        ESP_LOGI(TAG, "stream %d state is closed", pstream->id);
        return 0;
    default:
        break;
    }

    ushort flags = get_send_flags(pstream);  // Get protocol flags
    ESP_LOGI(TAG, "tmux_stream_write stream id %u  length %u", pstream->id, length);

    tcp_mux_send_hdr(Sockfd, flags, pstream->id, length);  // Send header
    if (send(Sockfd, data, length, 0) < 0) {               // Send payload
        ESP_LOGE(TAG, "error: tmux_stream_write send FAIL");
        uiRet = _FAIL;
        RESET_DEVICE;  // 异常断开直接重启，不需要LED闪烁
    }   
    return uiRet;
}
```

### main-crypto

#### 宏定义和结构体

宏定义高级加密标准(AES,Advanced Encryption Standard)。是最常见的对称加密算法
*在AES标准规范中，分组长度只能是128位，也就是说，每个分组为16个字节（每个字节8位）。密钥的长度可以使用128位、192位或256位。*
16字节，每字节8位，共128位，限定数据长度

结构体用于储存加密参数

```c
#define AES_128_KEY_SIZE 16
#define AES_128_IV_SIZE  16

struct frp_coder {
    uint8_t key[16];    // 储存加密算法密钥
    char    *salt;      // 指向一个字符串，用于派生密钥的盐值 (Salt)。用于增加密钥派生函数的复杂性，防止彩虹表攻击
    uint8_t iv[16];     // 储存用于AES加密的初始化向量
    char    *token;     // 储存身份验证令牌
}; 
```

#### 初始化解密器和加密器

1. `mbedtls_cipher_info_t`是mbedtls库中的函数，用于对称加密算法
2. 分配内存并初始化为1
3. `strdup`分配新内存并复制内容，返回指针
4. `mbedtls_md_init`初始化`mbedtls_md_context_t`类型资源，用于管理信息，进行哈希计算
5. `mbedtls_md_info_from_type`获取SHA-1消息摘要算法信息
6. `mbedtls_md_setup`设置`sha1-ctx`上下文以使用算法，并启用HMAC模式（*HMAC (Hash-based Message Authentication Code) 是一种基于哈希的消息认证码，它结合了密钥和消息，提供数据完整性验证和身份认证*）
7. `mbedtls_pkcs5_pbkdf2_hmac`使用PBKDF2算法，通过HMAC-SHA1派生最终加密密钥
   * `&sha1_ctx`设置好的HMAC-SHA-1上下文
   * `decoder->token`提供作为密码的`token`；`decoder->salt`提供盐值
   * `strlen`获取长度
   * 指定迭代次数64、输出密钥长度16字节
   * `decoder->key`输出缓冲区
8. `memcpy`将传入的初始化向量（iv）复制到decoder结构体中
9. `mbedtls_cipher_info_from_type(MBEDTLS_CIPHER_AES_128_CFB128)`获取AES-128-CFB128模式的加密算法信息
10. `mbedtls_cipher_init`初始化`mbedtls_cipher_context_t`类型变量储存解密操作状态
11. `mbedtls_cipher_setup(&dec_ctx, cipher_info)`使用获取的`cipher_info`配置`dec_ctx`上下文，即将解密上下文和算法关联
12. `mbedtls_cipher_setkey`将密钥设置到上下文中，指定操作模式为`MBEDTLS_DECRYPT`解密
13. `mbedtls_cipher_set_iv`将iv设置到上下文中
14. 返回已成功初始化的 `decoder` 结构体指针，此时该结构体包含了用于后续解密的所有必要信息（密钥、IV、以及底层的加密上下文）

```c
/**
 * Initialize decoder structure with given IV 
 * @param iv Initialization vector
 * @return Pointer to initialized decoder structure
 */
struct frp_coder* init_decoder(const uint8_t *iv) {
    const mbedtls_cipher_info_t *cipher_info;
    
    // Allocate and zero-initialize decoder structure
    decoder = calloc(sizeof(struct frp_coder), 1);

    // Copy token and salt into decoder
    decoder->token = strdup(token);
    decoder->salt = strdup(salt);
    
    // Initialize SHA-1 context for PBKDF2
    mbedtls_md_context_t sha1_ctx;
    mbedtls_md_init(&sha1_ctx);

    // Get SHA-1 algorithm information
    const mbedtls_md_info_t *sha1_info = mbedtls_md_info_from_type(MBEDTLS_MD_SHA1);

    // Setup HMAC-based SHA-1 context
    mbedtls_md_setup(&sha1_ctx, sha1_info, 1); // 1 = HMAC mode

    // Derive encryption key using PBKDF2
    mbedtls_pkcs5_pbkdf2_hmac(
        &sha1_ctx,                             // HMAC context
        (const unsigned char *)decoder->token,  // Secret token
        strlen((const char *)decoder->token),   // Token length
        (const unsigned char *)decoder->salt,   // Salt value
        strlen((const char *)decoder->salt),    // Salt length
        64,                                     // Iteration count
        16,                                     // Output key length (16 bytes = 128 bits)
        decoder->key                            // Output key buffer
    );

    // Copy initialization vector
    memcpy(decoder->iv, iv, AES_128_IV_SIZE);
    
    // Initialize AES-128-CFB cipher context
    cipher_info = mbedtls_cipher_info_from_type(MBEDTLS_CIPHER_AES_128_CFB128);
    mbedtls_cipher_init(&dec_ctx);  
    mbedtls_cipher_setup(&dec_ctx, cipher_info);
    mbedtls_cipher_setkey(&dec_ctx, decoder->key, 128, MBEDTLS_DECRYPT);
    mbedtls_cipher_set_iv(&dec_ctx, decoder->iv, 16);
    
    return decoder;
}
```

加密配置基本一致，只是解密最终储存在`dec_ctx`中，加密储存在`enc_ctx`中。`mbedtls_cipher_setkey`中设置`MBEDTLS_ENCRYPT`代表用于加密操作

```c
/**
 * Initialize encoder structure with given IV
 * @param iv Initialization vector
 * @return Pointer to initialized encoder structure
 */
struct frp_coder* init_encoder(const uint8_t *iv) {
    const mbedtls_cipher_info_t *cipher_info;
    
    // Allocate and zero-initialize encoder structure
    encoder = calloc(sizeof(struct frp_coder), 1);
    
    // Copy token and salt into encoder
    encoder->token = strdup(token);
    encoder->salt = strdup(salt);
    
    // Initialize SHA-1 context for PBKDF2
    mbedtls_md_context_t sha1_ctx;
    mbedtls_md_init(&sha1_ctx);
    
    // Select the SHA-1 algorithm and initialize
    const mbedtls_md_info_t *sha1_info = mbedtls_md_info_from_type(MBEDTLS_MD_SHA1);
    
    // Setup HMAC-based SHA-1 context
    mbedtls_md_setup(&sha1_ctx, sha1_info, 1);  // 1 = HMAC mode

    // Derive encryption key using PBKDF2
    mbedtls_pkcs5_pbkdf2_hmac(
        &sha1_ctx,	// HMAC context
        (const unsigned char *)encoder->token,  // Secret token
        strlen((const char *)encoder->token),   // Token length
        (const unsigned char *)encoder->salt,   // Salt value
        strlen((const char *)encoder->salt),    // Salt length
        64,	// Iteration count
        16,	//Output key length (16 bytes = 128 bits)
        encoder->key	// Output key buffer
    );

    // Copy initialization vector
    memcpy(encoder->iv, iv, AES_128_IV_SIZE);
    
    // Initialize AES-128-CFB cipher context
    cipher_info = mbedtls_cipher_info_from_type(MBEDTLS_CIPHER_AES_128_CFB128);
    mbedtls_cipher_init(&enc_ctx);  
    mbedtls_cipher_setup(&enc_ctx, cipher_info);
    mbedtls_cipher_setkey(&enc_ctx, encoder->key, 128, MBEDTLS_ENCRYPT);
    mbedtls_cipher_set_iv(&enc_ctx, encoder->iv, 16);
    
    return encoder;
}
```

#### 解密和加密操作

1. `mbedtls_cipher_update`将输入的密文进行解密
   * `dec_ctx` 是初始化配置的解密结构体
   * `ciphertext` 包含解密数据的缓冲区
   * `ct_len` 缓冲区数据长度
   * `plaintext` 储存解密后的数据
   * `pt_len` 实际写入解密数据的字节数
2. `mbedtls_cipher_finish`用于处理数据尾部
   * `dec_ctx` 是初始化配置的解密结构体
   * `plaintext + *pt_len`指向缓冲区末尾的指针,会在后方追加结果
   * `&finish_olen` 接收最终解密数据长度

```c
/**
 * Decrypt data using AES-128-CFB
 * @param ciphertext Pointer to ciphertext buffer
 * @param ct_len Length of ciphertext
 * @param plaintext Pointer to plaintext buffer
 * @param pt_len Pointer to plaintext length (updated by function)
 * @return 0 on success, error code otherwise
 */
int my_aes_decrypt(unsigned char *ciphertext, size_t ct_len, 
                  unsigned char *plaintext, size_t *pt_len) {
    int ret;
    size_t finish_olen;

    // Process ciphertext in multiple steps
    if ((ret = mbedtls_cipher_update(&dec_ctx, ciphertext, ct_len, 
                                    plaintext, pt_len)) != 0) {
        goto exit;
    }

    // Finalize decryption process
    if ((ret = mbedtls_cipher_finish(&dec_ctx, plaintext + *pt_len, 
                                    &finish_olen)) != 0) {
        goto exit;
    }
 
    *pt_len += finish_olen;

exit:
    return ret;  // Return 0 on success, error code otherwise
}
```

`mbedtls_cipher_update`函数并不区分加密和解密，是由于传入`ctx`的配置不同，进行加密和解密，此处解密，函数输出其实不变，两个传入、两个传出，只是给的参数名不一样

```c
/**
 * Encrypt data using AES-128-CFB
 * @param plaintext Pointer to plaintext buffer
 * @param pt_len Length of plaintext
 * @param ciphertext Pointer to ciphertext buffer
 * @param ct_len Pointer to ciphertext length (updated by function)
 * @return 0 on success, error code otherwise
 */
int my_aes_encrypt(const unsigned char *plaintext, size_t pt_len,
                  unsigned char *ciphertext, size_t *ct_len) {
    int ret;
    size_t finish_olen;

    // Process plaintext in multiple steps
    if ((ret = mbedtls_cipher_update(&enc_ctx, plaintext, pt_len,
                                    ciphertext, ct_len)) != 0) {
        goto exit;
    }

    // Finalize encryption process
    if ((ret = mbedtls_cipher_finish(&enc_ctx, ciphertext + *ct_len,
                                    &finish_olen)) != 0) {
        goto exit;
    }
    
    *ct_len += finish_olen;

exit:
    return ret;  // Return 0 on success, error code otherwise
}
```

### main-login

#### 头文件和结构体定义

宏定义协议版本

结构体参数见注释

```c
#define PROTOCOL_VERESION "0.43.0"

typedef struct login {
    char    *version;       // 客户端协议版本号
    char    *hostname;      // 设备名称
    char    *os;            // 操作系统类型（如 'ESP8266'）
    char    *arch;          // CPU架构
    char    *user;          // 用户名
    char    *privilege_key; // 权限密钥
    int timestamp;          // 时间戳
    char    *run_id;        // 运行ID（由服务器生成，用于会话管理）
    char    *metas;         // 元数据（可自定义）
    int pool_count;         // 计数

    /* fields not need json marshal */
    int logged; //0 not login 1:logged
} login_t;

struct login_resp {
    char    *version;       // 版本
    char    *run_id;        // 运行ID
    char    *error;         // 报错信息
};

// 主要控制结构体
typedef struct Main_Conf {
    char    *server_addr;   /* 服务器地址 default 0.0.0.0 */
    int    server_port;    /* 服务器端口 default 7000 */
    char    *auth_token;    // 身份识别令牌
    int    heartbeat_interval; /* 心跳间隔 default 10 */
    int    heartbeat_timeout;   /* 心跳超时 default 30 */
} MainConfig_t;
```

#### 初始化登录和main

1. 为登录结构体分配内存
2. `strdup` 分配新空间并把字符串复制到新空间内
3. 设置`login`结构体的各项参数
4. `esp_read_mac`获取特定接口的AMC地址，成功后格式化输出
5. `snprintf`格式化输出字符串，并把结果写入缓冲区，再赋给`run_id`
6. 

```c
/**
 * Initialize the login structure with default values and device information.
 * 
 * This function allocates memory for the login structure, sets default values
 * for various fields, retrieves the device MAC address, and uses it to generate
 * a unique run identifier.
 * 
 * @return _SUCCESS on success, _FAIL on failure (e.g., memory allocation error).
 */
int init_login()
{
    char mac_str[13]; // Buffer for MAC address string
    
    // Allocate memory for login structure
    g_pLogin = calloc(sizeof(login_t), 1);
    if (NULL == g_pLogin)
    {
        ESP_LOGE(TAG, "Calloc login_t _FAIL\r\n");
        return _FAIL;
    }

    // Initialize login structure fields
    g_pLogin->version        = strdup(PROTOCOL_VERESION); // Protocol version from define
    g_pLogin->hostname       = NULL; 
    g_pLogin->os             = strdup("freeRTOS");       // Operating system
    g_pLogin->arch           = strdup("Xtensa");          // CPU architecture
    g_pLogin->user           = NULL;
    g_pLogin->timestamp      = 0;                         // Initialize timestamp
    g_pLogin->run_id         = NULL;
    g_pLogin->metas          = NULL;
    g_pLogin->pool_count     = 1;
    g_pLogin->privilege_key  = NULL;
    g_pLogin->logged         = 0;

    uint8_t mac[6]; // Buffer for MAC address
    
    // Read WiFi station MAC address
    esp_err_t ret = esp_read_mac(mac, ESP_MAC_WIFI_STA);
    
    if (ESP_OK == ret) 
    {
        ESP_LOGI(TAG, "MAC: %02X:%02X:%02X:%02X:%02X:%02X",
                mac[0], mac[1], mac[2], mac[3], mac[4], mac[5]);
    } 
    else 
    {
        ESP_LOGE(TAG, "Failed to read MAC\n");
    }

    // Format MAC address as run_id 
    snprintf(mac_str, sizeof(mac_str), "%02X%02X%02X%02X%02X%02X",
         mac[0], mac[1], mac[2], mac[3], mac[4], mac[5]);
    
    // Set run_id as MAC address string
    g_pLogin->run_id = strdup((char*)mac_str);

    // Log initialization results
    ESP_LOGI(TAG, "info: version = %s  run_id = %s", 
                    g_pLogin->version,  g_pLogin->run_id);

    return _SUCCESS;
}
```

初始化`MainConfig_t`结构体，从`device_config`设备配置中读取并写入数据

`dump_common_conf`进行日志打印

```c
/**
 * Initialize the main configuration structure with server and communication parameters.
 * 
 * This function allocates memory for the main configuration structure and sets
 * default values for server address, port, authentication token, and heartbeat parameters.
 * 
 * @return _SUCCESS on success, _FAIL on failure (e.g., memory allocation error).
 */
int init_main_config()
{
    g_pMainConf = (MainConfig_t *)calloc(sizeof(MainConfig_t), 1);
    if (NULL == g_pMainConf)
    {
        ESP_LOGE(TAG, "error: init main config _FAIL\r\n");
        return _FAIL;
    }

    // Set configuration values from NVS config
    g_pMainConf->server_addr = strdup(g_device_config.frp_server);              // Server address
    g_pMainConf->server_port = g_device_config.frp_port;                        // Server port
    g_pMainConf->auth_token = strdup(g_device_config.frp_token);                // Authentication token
    g_pMainConf->heartbeat_interval = g_device_config.heartbeat_interval;       // Heartbeat interval
    g_pMainConf->heartbeat_timeout = g_device_config.heartbeat_timeout;         // Heartbeat timeout

    dump_common_conf(); // Print configuration details

    return _SUCCESS;
}
/**
 * Print the main configuration parameters for debugging purposes.
 * 
 * This function logs the current configuration settings including server address,
 * port, authentication token, heartbeat interval, and timeout.
 */
void dump_common_conf()
{
    if(! g_pMainConf) {
        ESP_LOGE(TAG, "Error: g_pMainConf is NULL");
        return;
    }

    // Log configuration parameters
    ESP_LOGI(TAG, "Main control {server_addr:%s, server_port:%d, auth_token:%s, interval:%d, timeout:%d}",
             g_pMainConf->server_addr, g_pMainConf->server_port, g_pMainConf->auth_token,
             g_pMainConf->heartbeat_interval, g_pMainConf->heartbeat_timeout);
}

```

#### 登录回复函数

1. 进行结构体类型转换
2. `login_resp_unmarshal`解码JSON结构体到结构体中
3. `login_resp_check`自定义函数检查登录回复，检查后释放内存
4. `ntohs` 将16位网络序转换为主机序（可能是大小端转换）
5. `lien` 从接收到的完整消息中减去登录响应消息体和消息头

```c
/**
 * Process the login response from the server.
 * 
 * This function parses the login response message, checks its validity,
 * and handles any additional data following the login response.
 * 
 * @param buf Pointer to the received message buffer.
 * @param len Length of the received message buffer.
 * @return _SUCCESS if login is successful, _FAIL otherwise.
 */
int handle_login_response(const char* buf, int len) 
{
    msg_hdr_t* mhdr = (struct msg_hdr*)buf; // Parse message header
    ESP_LOGI(TAG, "handle_login_response");

    // Unmarshal login response data
    struct login_resp* lres = login_resp_unmarshal((const char*)mhdr->data);
    if (!lres) {
        return _FAIL;
    }

    // Validate login response
    if (!login_resp_check(lres)) {
        ESP_LOGI(TAG, "login failed");
        free(lres);
        return _FAIL;
    }
    free(lres);

    // Calculate remaining data length
    int login_len = ntohs(mhdr->length); // Convert network byte order to host
    int lien = len - login_len - sizeof(struct msg_hdr);

    ESP_LOGI(TAG, "login success! login_len %d len %d lien %d", login_len, len, lien);

    if (lien < 0) {
        ESP_LOGI(TAG, "handle_login_response: lien < 0");
        return _FAIL;
    }
    return _SUCCESS;
}
```

1. 检查运行ID，然后检查消息，报错并重置登录状态
2. 根据传入的结构体进行日志输出，有ID代表连接成功
3. 重置`run_id`传入接收结构体的ID号
4. 把状态改为已登录

```c
/**
 * Validate the login response from the server.
 * 
 * This function checks if the login response contains valid information
 * such as a valid run ID. If valid, it updates the global login structure
 * with the received information and sets the login status to success.
 * 
 * @param lr Pointer to the login response structure.
 * @return 1 if login response is valid, 0 otherwise.
 */
int login_resp_check(struct login_resp* lr) 
{
    // Check if run_id is valid
    if ((NULL == lr->run_id) || strlen(lr->run_id) <= 1) {
        if (lr->error && strlen(lr->error) > 0) {
            ESP_LOGI(TAG, "login response error: %s", lr->error);
        }
        ESP_LOGI(TAG, "login failed!");
        g_pLogin->logged = 0; // Update login status
        return 0;
    }

    // Update login information with server response
    ESP_LOGI(TAG, "login response: run_id: [%s], version: [%s]",
          lr->run_id, lr->version);
    SAFE_FREE(g_pLogin->run_id);           // Free existing run_id
    g_pLogin->run_id = strdup(lr->run_id); // Set new run_id
    g_pLogin->logged = 1;                 // Set logged-in status

    return 1;
}
```

### main-msg

#### 宏定义和特殊结构体

`SAFE_JSON_STRING`宏定义，如果结构体为空，记为`""`，符合JSON格式

`SAFE_FREE`安全释放，调用free并把指针置NULL

`__attribute__((__packed__)) msg_hdr` 前置关键字是GCC编译器特定熟悉，用于禁用内存对齐填充（默认编译器会在结构体成员间插入额外字节使对齐。使用关键字结构体成员会紧密排列，节省空间）

* `type` 标识消息的类型或类别
* `length` 储存消息负载长度
* `data` 柔性数组成员，标识结构体后有未指定大小的内存区域

`work_conn` 暂时只储存运行ID，可扩展

```c
#define SAFE_JSON_STRING(str) (str ? str : "")
#define SAFE_FREE(ptr) do { if (ptr) { free(ptr); ptr = NULL; } } while (0)

typedef struct __attribute__((__packed__)) msg_hdr {
    char    type;
    uint64_t    length;
    char    data[];
} msg_hdr_t;

struct work_conn {
    char *run_id;
};
```

#### 发送消息函数

1. 检查描述符
2. 分配并初始化内存，长度为消息长度加消息头长度
3. 设置`type`和长度`length`，长度进行64位网络序和主机序转换
4. 复制数据到`req_msg`中
5. `tmux_stream_write`把消息通过已连接的网络套接字传输到frp服务器
6. 释放函数内变量的内存

```c
/**
 * @brief Send plain text message to FRP server
 * @param Sockfd: Socket file descriptor for communication
 * @param type: Message type (defined in msg_type_t enum)
 * @param pmsg: Pointer to message payload data
 * @param msg_len: Length of message payload
 * @param stream: Pointer to tmux stream structure for I/O operations
 * @return _SUCCESS(1)/_FAIL(0) on operation result
 */
int send_msg_frp_server(int Sockfd, 
                     const msg_type_t type, 
                     const char *pmsg, 
                     const uint msg_len, 
                     tmux_stream_t *stream)
{
    if (Sockfd < 0) {
        ESP_LOGE(TAG, "error: send_msg_frp_server failed, Sockfd < 0");
        return 0;
    }
    ESP_LOGE(TAG, "send plain msg ----> [%c: %s]", type, pmsg);
    
    // Allocate memory for message header + payload
    size_t len = msg_len + sizeof(msg_hdr_t);
    msg_hdr_t *req_msg = calloc(len, 1);
    
    if (NULL == req_msg) {
        ESP_LOGE(TAG, "error: req_msg init failed");
        return _FAIL;
    }   
    
    // Construct message header
    req_msg->type = type;
    req_msg->length = ntoh64((uint64_t)msg_len);  // Convert to network byte order
    memcpy(req_msg->data, pmsg, msg_len);          // Copy payload
    
    // Send through TMUX stream
    tmux_stream_write(Sockfd, (char *)req_msg, len, stream);
    free(req_msg);

    return _SUCCESS;
}
```

1. 检查描述符
2. 分配内存和写入数据
3. `my_aes_encrypt`自定义函数进行加密
4. `tmux_stream_write` 函数发送数据
5. 释放内存

```c
/**
 * @brief Send encrypted message to FRP server
 * @param Sockfd: Socket file descriptor for communication
 * @param type: Message type (from msg_type enum)
 * @param msg: Pointer to message payload data
 * @param msg_len: Length of message payload
 * @param stream: Pointer to tmux stream structure for I/O operations
 */
void send_enc_msg_frp_server(int Sockfd,
             const enum msg_type type, 
             const char *msg, 
             const size_t msg_len, 
             struct tmux_stream *stream)
{
    size_t ct_len;
    
    if (Sockfd < 0) {
        ESP_LOGE(TAG, "error: send_msg_frp_server failed, Sockfd < 0");
        return;
    }

    // Create message header + payload
    struct msg_hdr *req_msg = calloc(msg_len+sizeof(struct msg_hdr), 1);
    assert(req_msg);
    req_msg->type = type;
    req_msg->length = ntoh64((uint64_t)msg_len);
    memcpy(req_msg->data, msg, msg_len);

    // Encrypt the entire message
    uint8_t *enc_msg = calloc(msg_len+sizeof(struct msg_hdr), 1);
    my_aes_encrypt((uint8_t *)req_msg, msg_len+sizeof(struct msg_hdr), enc_msg, &ct_len);

    // Send encrypted data
    tmux_stream_write(Sockfd, (char*)enc_msg, ct_len, stream);  

    // Cleanup resources
    free(enc_msg);    
    free(req_msg);
}
```

#### 编码转换相关函数

1. 释放已有的密钥
2. `get_auth_key`自定义函数生成新的密钥
3. 通过分配内存和复制的方式把密钥给结构体
4. `cJSON_CreateObject` 调用cJSON库函数创建新的JSON对象`{}`
5. 如果操作出错，安全释放内存并返回 0
6. `cJSON_AddStringToObject`向json对象中添加字符串（对象指针，键名，内容）
7. `cJSON_PrintUnformatted`将cJSON对象转换为紧凑的JSON字符串（更适合网络传输或储存）
8. 日志打印指针值，`nret`存储json数据长度 `msg`通过`strdup`把`tmp`内容复制并储存
9. `cJSON_free`释放函数生成的字符串
10. `cJSON_Delete`删除完整的cJSON结构

```c
/**
 * Marshal login request to JSON format
 * @param msg Output parameter for generated JSON string
 * @return Length of generated JSON string (0 on failure)
 */
uint32_t login_request_marshal(char **msg) 
{
    uint32_t nret = 0;
    uint32_t freeHeap;
    *msg = NULL;

    // Free existing privilege key
    SAFE_FREE(g_pLogin->privilege_key);
    
    // Generate new authentication key
    char *auth_key = get_auth_key(g_pMainConf->auth_token, &g_pLogin->timestamp);
    if (!auth_key) {
        ESP_LOGE(TAG, "get_auth_key fail");
        return 0;
    }
    
    // Store new privilege key
    g_pLogin->privilege_key = strdup(auth_key);
    if (!g_pLogin->privilege_key) {
        ESP_LOGE(TAG, "privilege_key fail");
        SAFE_FREE(auth_key);
        return 0;
    }

    // Create JSON object for login request
    cJSON *j_login_req = cJSON_CreateObject();
    if (!j_login_req) {
        ESP_LOGE(TAG, "cJSON_CreateObject fail");
        SAFE_FREE(auth_key);
        return 0;
    }
    
    // Add all required fields to JSON
    freeHeap = esp_get_free_heap_size();
    ESP_LOGE(TAG, "Free Heap: %u bytes\n", freeHeap);
    
    cJSON_AddStringToObject(j_login_req, "version", SAFE_JSON_STRING(g_pLogin->version));
    cJSON_AddStringToObject(j_login_req, "hostname", SAFE_JSON_STRING(g_pLogin->hostname));
    cJSON_AddStringToObject(j_login_req, "os", SAFE_JSON_STRING(g_pLogin->os));
    cJSON_AddStringToObject(j_login_req, "arch", SAFE_JSON_STRING(g_pLogin->arch));
    cJSON_AddStringToObject(j_login_req, "user", SAFE_JSON_STRING(g_pLogin->user));
    cJSON_AddStringToObject(j_login_req, "privilege_key", SAFE_JSON_STRING(g_pLogin->privilege_key));
    
    char buf[32] = {0};
    sprintf(buf , "%d" , g_pLogin->timestamp);
    cJSON_AddRawToObject(j_login_req, "timestamp", buf);
    cJSON_AddStringToObject(j_login_req, "run_id", SAFE_JSON_STRING(g_pLogin->run_id));
    sprintf(buf , "%d" , g_pLogin->pool_count);
    cJSON_AddRawToObject(j_login_req, "pool_count", buf);
    cJSON_AddNullToObject(j_login_req, "metas");
    
    // Generate JSON string
    char *tmp = cJSON_PrintUnformatted(j_login_req);
    ESP_LOGE(TAG, "tmp = %p\n", tmp);
    if (tmp && strlen(tmp) > 0) {
        nret = strlen(tmp);
        *msg = strdup(tmp);
        if (!*msg) {
            nret = 0;
        }
    }

    // Cleanup resources
    if (tmp) {
        cJSON_free(tmp);
    }
    cJSON_Delete(j_login_req);
    SAFE_FREE(auth_key);
    return nret;
}
```

1. 分配内存并初始化
2. `cJSON_Parse`将json数据包解析成cJSON结构体
3. `cJSON_GetObjectItem` 从json对象中查找子对象
4. `cJSON_IsString(l_version)` 检查子对象类型是/不是字符串
5. `strdup(l_version->valuestring)` 检查通过后将`l_version`指向的字符串内容复制一份并返回指针
6. `cJSON_Delete`操作完成后删除整个cJSON结构

```c
/**
 * @brief Unmarshal login response JSON into structure
 * @param jres: Pointer to JSON response string
 * @return Pointer to login_resp structure (needs free()), NULL on failure
 */
struct login_resp* login_resp_unmarshal(const char* jres) 
{
    struct login_resp* lr = (struct login_resp*)calloc(1, sizeof(struct login_resp));
    if (NULL == lr) {
        return NULL;
    }

    cJSON* j_lg_res = cJSON_Parse(jres);
    if (NULL == j_lg_res) {
        free(lr);
        return NULL;
    }

    cJSON* l_version = cJSON_GetObjectItem(j_lg_res, "version");
    if (!cJSON_IsString(l_version)) {
        goto END_ERROR;
    }
    lr->version = strdup(l_version->valuestring);

    cJSON* l_run_id = cJSON_GetObjectItem(j_lg_res, "run_id");
    if (!cJSON_IsString(l_run_id)) {
        goto END_ERROR;
    }
    lr->run_id = strdup(l_run_id->valuestring);
    cJSON_Delete(j_lg_res);
    return lr;

END_ERROR:
    cJSON_Delete(j_lg_res);
    free(lr);
    return NULL;
}
```

1. 检查传入参数
2. `cJSON_CreateObject`创建cJSON对象
3. `cJSON_AddStringToObject`向cJSON对象中添加字符串，`cJSON_AddNullToObject`向cJSON对象中添加键值对，值为`null`，`cJSON_AddBoolToObject`将布尔值添加到cJSON对象中，`cJSON_AddRawToObject`向对象中添加键值对，值以原始字符串形式提供
4. `cJSON_Print`将cJSON对象转换为**格式化**的JSON字符串，有缩进、换行、空格等
5. 同样进行判断，然后存入msg中
6. 释放内存

```c
/**
 * @brief Marshal new proxy service configuration into JSON
 * @param np_req: Pointer to proxy service configuration structure
 * @param msg: Pointer to store generated JSON string (needs free())
 * @return 1 on success, 0 on failure
 */
int new_proxy_service_marshal(const struct proxy_service *np_req, char **msg)
{
    char *tmp = NULL;
    int nret = 0;
    
    if (!np_req || !msg) {
        //printf("Error: Invalid input\n");
        return 0;
    }
    
    cJSON *j_np_req = cJSON_CreateObject();
    if (!j_np_req) {
        //printf("\r\ncJSON_CreateObject fail\r\n");
        return 0;
    }
    
    if (np_req->proxy_name) {
        //printf("Add proxy_name: %s\n", np_req->proxy_name);
        cJSON_AddStringToObject(j_np_req, "proxy_name", np_req->proxy_name);
    } else {
        cJSON_AddNullToObject(j_np_req, "proxy_name");
    }
    
    if (np_req->proxy_type) {
        //printf("Add proxy_type: %s\n", np_req->proxy_type);
        cJSON_AddStringToObject(j_np_req, "proxy_type", np_req->proxy_type);
    } else {
        cJSON_AddNullToObject(j_np_req, "proxy_type");
    }

    cJSON_AddBoolToObject(j_np_req, "use_encryption", np_req->use_encryption);
    cJSON_AddBoolToObject(j_np_req, "use_compression", np_req->use_compression);
    
    char buffer[32];
    if (np_req->remote_port != -1) {
        snprintf(buffer, sizeof(buffer), "%d", np_req->remote_port);
        cJSON_AddRawToObject(j_np_req, "remote_port", buffer);
        //printf("Add remote_port: %d\n", np_req->remote_port);
    } else {
        cJSON_AddNullToObject(j_np_req, "remote_port");
    }
    
    tmp = cJSON_Print(j_np_req);

    if (!tmp) {
        //printf("Error: Failed to print JSON (out of memory)\n");
        cJSON_Delete(j_np_req);
        return 0;
    }

    if (tmp && strlen(tmp) > 0) {
        nret = strlen(tmp);
        *msg = (char *)malloc(nret + 1);
        if (*msg) {
            strcpy(*msg, tmp);
        } else {
            nret = 0;
        }
    }
    
    free((void *)tmp);
    cJSON_Delete(j_np_req);
    
    return nret;
}
```

1. `cJSON_CreateObject`新建cJSON对象
2. `cJSON_AddStringToObject`写入字符串
3. `cJSON_PrintUnformatted`将cJSON对象转成紧凑的JSON字符串
4. `tmp`验证是否转换完成，并转存
5. `cJSON_Delete`删除整个cJSON结构
6. `free` 释放内存

```c
/**
 * @brief Marshal new work connection information into JSON
 * @param work_c: Pointer to work connection structure
 * @param msg: Pointer to store generated JSON string (needs free())
 * @return Length of JSON string on success, 0 on failure
 */
int new_work_conn_marshal(const struct work_conn *work_c, char **msg) 
{
    int nret = 0;
    cJSON *j_new_work_conn = cJSON_CreateObject();
    if (!j_new_work_conn) {
        return 0;
    }

    // Add the run_id to the JSON object
    cJSON_AddStringToObject(j_new_work_conn, "run_id", work_c->run_id ? work_c->run_id : "");

    // Convert the cJSON object to a JSON string
    char *tmp = cJSON_PrintUnformatted(j_new_work_conn);
    if (tmp && strlen(tmp) > 0) {
        nret = strlen(tmp);
        *msg = strdup(tmp);
        assert(*msg);
    }

    // Clean up
    cJSON_Delete(j_new_work_conn);
    free(tmp);

    return nret;
}
```

#### 哈希操作相关函数

1. 从服务器获取时间戳
2. 如果有token，就使用token+时间戳作为种子
3. 没有就只有时间戳

4. `esp_md5_init`初始化MD5哈希上下文
5. `esp_md5_update`将原始数据添加到MD5哈希中
6. `esp_md5_final`完成MD5哈希计算并将16字节哈希值写入缓冲区
7. （上述三个函数应该在`<esp8266/esp_md5.h>`库中）
8. 把输出的值进行格式化存入输出数组中

```c
/**
 * Generate authentication key using MD5 hash
 * @param token Authentication token (can be NULL)
 * @param timestamp Output parameter for current timestamp
 * @return MD5 hash string (needs to be freed by caller)
 */
char * get_auth_key(const char *token, int *timestamp)
{
    char seed[128] = {0};
    *timestamp = obtain_time();  // Get current timestamp
    
    // Create seed string: token + timestamp or just timestamp
    if (token)
        snprintf(seed, 128, "%s%d", token, *timestamp);
    else
        snprintf(seed, 128, "%d", *timestamp);
    
    return calc_md5(seed, strlen(seed));  // Calculate MD5 of seed
}
/**
 * Calculate MD5 hash of input data
 * @param data Input data buffer
 * @param datalen Length of input data
 * @return MD5 hash string (33-byte buffer with null terminator)
 */
char * calc_md5(const char *data, int datalen) {
    unsigned char digest[16] = {0};
    char *out = malloc(33);
    if (NULL == out) {
        return NULL;
    }

    // Calculate MD5 hash
    struct MD5Context md5;
    esp_md5_init(&md5);
    esp_md5_update(&md5, (const uint8_t *)data, datalen);
    esp_md5_final(&md5, digest);
    
    // Convert to hex string
    for (int n = 0; n < 16; ++n) {
        snprintf(&(out[n*2]), 3, "%02x", (unsigned int)digest[n]);
    }
    out[32] = '\0';

    return out;
}
```

### main-sntp

#### 获取时间戳函数

循环直到达到指定时间戳或超过重试限制

`time_t time(time_t *seconds)`函数返回自（1970-01-01 00：00：00）起经过的时间（秒）

```c
/**
 * Obtain the current time from the SNTP server.
 * Attempts to retrieve the current time with retries if the time is invalid.
 * 
 * @return The current Unix timestamp on success, -1 on failure.
 */
time_t obtain_time(void)
{
    time_t now = 0;         // Stores current timestamp
    int retry = 0;          // Retry counter
    const int retry_count = 10;  // Maximum retry attempts

    // Loop until valid time is obtained or retry limit reached
    // 1600000000 = September 13, 2020 (arbitrary valid timestamp threshold)
    while (now < 1600000000 && ++retry < retry_count) 
    {
        ESP_LOGI(TAG, "Waiting for system time to be set... (%d/%d)", retry, retry_count);
        vTaskDelay(2000 / portTICK_PERIOD_MS);  // Delay for 2 seconds (FreeRTOS)
        time(&now);  // Update current time value
    }

    if (now < 1600000000) 
    {
        ESP_LOGE(TAG, "Failed to get SNTP time");
        RESET_DEVICE;
        return -1;  // Return error code
    }

    ESP_LOGI(TAG, "The current time: %ld", now);
    
    return now;  // Return valid Unix timestamp
}
```

1. `sntp_setoperatingmode`设置SNTP（Simple Network Time Protocol，简单网络时间协议）服务的运行模式 `SNTP_OPMODE_POLL`代表轮询
2. `sntp_setservername`设置SNTP客户端连接的NTP服务器地址，`"ntp.aliyun.com"`由阿里云提供，是公开可用的NTP时间源
3. `sntp_init`初始化SNTP客户端
4. 在[v5.5版本](https://docs.espressif.com/projects/esp-idf/zh_CN/stable/esp32/api-reference/system/system_time.html#sntp)编程指南有集成函数

```c
/**
 * Initialize the SNTP (Simple Network Time Protocol) client.
 * Configures the SNTP operating mode to polling and sets the NTP server.
 */
void init_sntp(void)
{
    ESP_LOGI(TAG, "Initializing SNTP");
    sntp_setoperatingmode(SNTP_OPMODE_POLL);  // Set SNTP operating mode to periodic polling
    sntp_setservername(0, "ntp.aliyun.com");  // Set primary NTP server address
    sntp_init();  // Initialize SNTP service
}
```

### main-tcpmux

#### 类型定义

```c
#define _SUCCESS 0
typedef unsigned char uchar;

enum tcp_mux_state {
    INIT = 0,       // 初始状态，连接未建立
    SYN_SEND,       // 已发送同步（SYN）包，等待对方确认
    SYN_RECEIVED,   // 接收到同步包
    ESTABLISHED,    // 连接成功建立，可以进行数据传输
    LOCAL_CLOSE,    // 本地主动发起关闭连接
    REMOTE_CLOSE,   // 对方发起关闭连接
    CLOSED,         // 连接已关闭
    RESET           // 连接被重置
};

typedef enum tcp_mux_flag {
    ZERO,       // 无特殊标志
    SYN,        // 同步标志
    ACK = 1<<1, // 确认标志
    FIN = 1<<2, // 结束标志
    RST = 1<<3, // 重置标志
}tcp_mux_flag_t;

typedef enum tcp_mux_type {
    DATA,           // 数据类型
    WINDOW_UPDATE,  // 窗口更新
    PING,           // 心跳/探测
    GO_AWAY,        // 告知对方将关闭连接
}tcp_mux_type_t;

typedef struct __attribute__((__packed__)) tcp_mux_header
{
    uchar    version;   // 版本号
    uchar    type;      // 消息类型
    ushort   flags;     // 标志位
    uint     stream_id; // 流ID
    uint     length;    // 数据长度
}tcp_mux_header_t;


typedef struct tmux_stream {
    uint    id;                 // 流ID
    enum tcp_mux_state state;   // 当前状态

}tmux_stream_t;
```

#### 编码函数

1. 检查参数
2. 写入版本号、类型等参数
3. `htons``htonl`从主机序变成网络序(大小端转换)
4. 日志打印

```c
/**
 * Encode TCP multiplexing header fields into a structured format.
 * This function populates the TCP multiplexing header with the provided
 * type, flags, stream ID, and length information, ensuring proper byte ordering.
 * 
 * @param type Type of the TCP multiplexing message
 * @param flags Control flags for the message
 * @param uiStreamId Stream identifier
 * @param uiLength Length of the payload
 * @param ptmux_hdr Pointer to the header structure to be filled
 */
void tcp_mux_encode(tcp_mux_type_t type, tcp_mux_flag_t flags, uint uiStreamId, uint uiLength, tcp_mux_header_t *ptmux_hdr)
{
    if (NULL == ptmux_hdr)
    {
        ESP_LOGE(TAG, "ptmux_hdr is NULL\r\n");
    }   
    // Set header fields with proper byte ordering
    ptmux_hdr->version = proto_version;
    ptmux_hdr->type = type;
    ptmux_hdr->flags = htons(flags);
    ptmux_hdr->stream_id = htonl(uiStreamId);
    ptmux_hdr->length = htonl(uiLength);
    
    ESP_LOGI(TAG, "info: ptmux_hdr version = %u, type = %u, flags = %u, stream_id = %u, length = %u",
             proto_version, type, flags, uiStreamId, uiLength);
}
```

#### 获取标志位

根据`pStream`的状态将标志位置位

* 如果是`INIT`初始化，将SYN同步位置位，状态改为发送同步包
* 如果是`SYN_RECEIVED`接收同步包，将应答位置位，状态改为已连接

```c
/**
 * Get the flags to be sent based on the current state of the stream.
 * This function determines the appropriate flags (e.g., SYN, ACK) to send
 * based on the current state of the stream and updates the stream's state.
 * 
 * @param pStream Pointer to the stream structure
 * @return Flags to be sent in the TCP multiplexing header
 */
ushort get_send_flags(struct tmux_stream *pStream)
{
    ushort flags = 0;
    ESP_LOGI(TAG, "get_send_flags: state %d, stream_id %d", pStream->state, pStream->id);
    
    switch (pStream->state) 
    {
    case INIT:
        flags |= SYN;                // Set SYN flag for initial state
        pStream->state = SYN_SEND;   // Transition to SYN_SEND state
        break;            
    case SYN_RECEIVED:
        flags |= ACK;                // Set ACK flag after receiving SYN
        pStream->state = ESTABLISHED; // Transition to ESTABLISHED state
        break;
    default:
        break;
    } 

    return flags;
}
```

#### 发送消息相关函数

1. 获取标志位，ID号
2. `tcp_mux_encode` 将数据编码成网络序
3. 调用`send`函数将数据发送到服务器
4. 日志打印

```c
/**
 * Send a window update message to adjust the receive window size.
 * This function constructs and sends a window update message with the
 * specified length, including appropriate flags based on the stream state.
 * 
 * @param iSockfd Socket file descriptor
 * @param pStream Pointer to the stream structure
 * @param uiLength Length of the window update
 * @return Success or failure code
 */
int send_window_update(int iSockfd, tmux_stream_t *pStream, uint uiLength)
{    
    ushort flags = get_send_flags(pStream);    
    tcp_mux_header_t tmux_hdr;
    uint uiStreamId = pStream->id;

    memset(&tmux_hdr, 0, sizeof(tmux_hdr));
    tcp_mux_encode(WINDOW_UPDATE, flags, uiStreamId, uiLength, &tmux_hdr);
    
    // Send the encoded header
    if(send(iSockfd, (uchar *)&tmux_hdr, sizeof(tmux_hdr), 0)<0) 
    {
        ESP_LOGE(TAG, "error: window update send FAIL");
        RESET_DEVICE;
    }
    
    ESP_LOGI(TAG, "send window update: flags %d, stream_id %d, length %u", flags, pStream->id, uiLength);

    return _SUCCESS;
}
```

1. 将数据编码成网络序
2. 通过`send`函数发送

```c
/**
 * Send a TCP multiplexing header for data transmission.
 * This function constructs and sends a data header with the specified
 * flags, stream ID, and payload length.
 * 
 * @param iSockfd Socket file descriptor
 * @param flags Control flags for the data message
 * @param stream_id Stream identifier
 * @param length Length of the data payload
 */
void tcp_mux_send_hdr(int iSockfd, ushort flags, uint stream_id, uint length)
{
    struct tcp_mux_header tmux_hdr;
    
    memset(&tmux_hdr, 0, sizeof(tmux_hdr));
    tcp_mux_encode(DATA, flags, stream_id, length, &tmux_hdr);
    if(send(iSockfd, (char *)&tmux_hdr, sizeof(tmux_hdr), 0)<0) 
    {
        ESP_LOGE(TAG, "error: tcp mux hdr send_FAIL");
        RESET_DEVICE;
    }
    ESP_LOGI(TAG, "tcp mux header,stream_id:%d len:%d", stream_id, length);
}
```

1. 把网络序转换成主机序（大小端转换）
2. `if (SYN == (flags & SYN))`判断flags中的SYN位是否置位，是代表新连接
3. `tcp_mux_encode`自定义函数将信息编码到变量中，即把参数编码为符合TCP的数据
   * `type`: `PING` 表示这是一个 PING 消息
   * `flags`: `ACK` 表示这是一个确认响应
   * `length`: `0` 此响应消息不携带额外的数据负载
   * `stream_id`: `ping_id` (这是从原始 PING 消息中提取的、用于标识这个特定会话的 ID)
4. `send`函数将编码好的消息通过描述符发送到服务器
5. `RESET_DEVICE`宏函数进行重启设备

```c
/**
 * Handle a TCP multiplexing ping request.
 * This function processes ping requests and sends back appropriate responses
 * when a SYN flag is detected in the ping message.
 * 
 * @param pTmux_hdr Pointer to the TCP multiplexing header of the ping message
 */
void handle_tcp_mux_ping(struct tcp_mux_header *pTmux_hdr)
{    
    uint16_t flags = ntohs(pTmux_hdr->flags);
    uint32_t ping_id = ntohl(pTmux_hdr->length);

    if (SYN == (flags & SYN)) 
    {        
        struct tcp_mux_header Tmux_hdr_send = {0};

        // Prepare and send ping response
        tcp_mux_encode(PING, ACK, 0, ping_id, &Tmux_hdr_send);
        
        ESP_LOGI(TAG, "PING");        
        
        if (send(g_pMainCtl->iMainSock, &Tmux_hdr_send, sizeof(Tmux_hdr_send), 0) < 0)
        {
            ESP_LOGI(TAG, "error: handle tcp mux ping send FAIL");
            RESET_DEVICE;
            return;
        }        
    }
}
```

#### 状态变化

* 如果ACK应答位被置位，说明连接已建立，将状态改为已连接
* 如果FIN完成位被置位，说明连接已结束，将参数设置为断开连接状态
  * 对于连接/初始化状态，将状态改为远端关闭
  * 对于本地关闭状态，改为以关闭，并将`close_stream`变量置1，代表本地连接断开
  * 其他状态报错
* 如果RST重启标志位置位，将状态改为重启，并将变量置1

```c
/**
 * Process control flags received in a TCP multiplexing header.
 * This function handles different flags (ACK, FIN, RST) and updates
 * the stream state accordingly, managing connection state transitions.
 * 
 * @param flags Control flags from the header
 * @param stream Pointer to the stream structure
 * @return Non-zero on success, zero on error
 */
int process_flags(uint16_t flags, struct tmux_stream *stream)
{
    uint32_t close_stream = 0;
    if (ACK == (flags & ACK)) {
        // Handle ACK flag
        if (SYN_SEND == stream->state) stream->state = ESTABLISHED;
    } else if (FIN == (flags & FIN)) {
        // Handle FIN flag (connection termination)
        g_ProxyWork = 0;
        linked = 0;
        set_frpc_connection_lost();  // 正常断开时设置LED闪烁
        
        switch(stream->state) {
        case SYN_SEND:
        case SYN_RECEIVED:
        case ESTABLISHED:
            stream->state = REMOTE_CLOSE;
            break;
        case LOCAL_CLOSE:
            stream->state = CLOSED;
            close_stream = 1;
            break;
        default:
            ESP_LOGI(TAG, "unexpected FIN flag in state %d", stream->state);
            assert(0);
            return 0;
        }
    } else if (RST == (flags & RST)) {
        // Handle RST flag (connection reset)
        stream->state = RESET;
        close_stream = 1;
    }

    if (close_stream) {
        ESP_LOGI(TAG, "free stream %d", stream->id);        
    }

    return 1;
}
```

### main-timer

#### 连接状态

```c
// 连接状态管理
typedef enum {
    CONNECTION_DISCONNECTED,    // 未连接或其他状态 - LED不亮
    CONNECTION_CONNECTED,       // 连接成功 - LED常亮
    CONNECTION_LOST            // 连接丢失 - LED快速闪烁
} connection_state_t;
static connection_state_t frpc_connection_state = CONNECTION_DISCONNECTED;
```

#### 新建软件定时器函数

1. 定义常量： 定时器名称，定时器间隔（每0.1s），启用自动重载
2. `xTimerCreate` FreeRTOS的软件定时器
   * `timerName` 定时器名称
   * `timerPeriod` 定时周期
   * `autoReload` 自动重载是否启用
   * 初始占空比
   * `TimerCallback` 定时器回调函数
3. 如果创建成功则启动定时器

```c
/**
 * Function to create and start the timer
 * Initializes and starts the timer with 0.1 second period for LED control and periodic pings.
 */
void CreateTimer() 
{
    const char* timerName = "FrpcTimer";                 // Timer identifier
    const TickType_t timerPeriod = pdMS_TO_TICKS(100);   // Convert 100ms to ticks (0.1 second)
    const UBaseType_t autoReload = pdTRUE;               // Enable auto-reload mode
    
    // Create software timer
    FrpcTimer = xTimerCreate(
        timerName,       // Timer name for debugging
        timerPeriod,      // Timer period (0.1 seconds)
        autoReload,       // Auto-reload when expired
        (void*)0,         // No parameters passed to callback
        TimerCallback     // Callback function pointer
    );

    // Check timer creation result
    if (NULL == FrpcTimer) {
        ESP_LOGI(TAG, "FrpcTimer create fail!");
    } else {
        ESP_LOGI(TAG, "FrpcTimer create success!");
        
        // Attempt to start the timer
        if (xTimerStart(FrpcTimer, 0) != pdPASS) {  // 0 = don't wait if queue is full
            ESP_LOGI(TAG, "FrpcTimer start fail!");
        } else {
            ESP_LOGI(TAG, "FrpcTimer start success!");
        }
    }
}
```

#### 定时器回调函数

函数每0.1s调用

1. 定时器和PING计数器递增
2. 根据控制模式和frp连接状态设置GPIO电平以控制LED
3. 每30秒进行PING逻辑
   1. 检查是否有可行的控制连接
   2. 计算是否超时，超时代表故障
   3. 未超时，发送ping到服务器(无有效数据，只是说明客户端还存在)
4. 每1000次运行重置计数器

```c
/**
 * Timer callback function definition
 * This function is called every 0.1 seconds.
 */
void TimerCallback(TimerHandle_t xTimer) 
{
    tickcnt++;  // 滴答计数器递增
    ping_tick_count++;  // Ping计数器递增

    // NET LED控制逻辑（非webserver模式下）
    if (!config_mode) {
        switch (frpc_connection_state) {
            case CONNECTION_DISCONNECTED:
                // 未连接状态 - LED不亮
                gpio_set_level(LINK_LED, 1);  // 关闭LED
                break;
                
            case CONNECTION_CONNECTED:
                // 连接成功 - LED常亮
                gpio_set_level(LINK_LED, 0);  // 点亮LED
                break;
                
            case CONNECTION_LOST:
                // 断线状态 - 快速闪烁（每2个滴答=0.2秒切换一次）
                if (tickcnt % 2 == 0) {
                    led_state = !led_state;
                    gpio_set_level(LINK_LED, led_state ? 0 : 1);
                }
                break;
        }
    }

    // Ping逻辑 - 每300个tick（30秒）执行一次
    if (ping_tick_count >= 300) {
        ping_tick_count = 0;  // 重置ping计数器
        
        // 检查是否有有效的控制结构和socket
        if (g_pMainCtl && g_pMainCtl->iMainSock > 0) {
            // 获取当前时间
            time_t current_time = obtain_time();
            
            // 计算距离上次pong的时间间隔
            int interval = current_time - g_Pongtime;
            
            // 检查是否超时（40秒阈值）
            if (g_Pongtime && interval > 40) {
                ESP_LOGI(TAG, "Time out");
                // 超时会导致后续的RESET_DEVICE，不需要LED闪烁
                return;
            }
            
            // 发送周期性ping到FRPS服务器
            ESP_LOGI(TAG, "ping frps");
            char *ping_msg = "{}";
            send_enc_msg_frp_server(
                g_pMainCtl->iMainSock,      // Main socket descriptor
                TypePing,                   // Message type: PING
                ping_msg,                   // Empty JSON payload
                strlen(ping_msg),           // Message length
                &g_pMainCtl->stream        // Stream context
            );
        }
    }

    // 滴答计数器清零 - 防止溢出
    if (tickcnt == 1000) {
        tickcnt = 0;
    }
}
```

#### 其他函数

```c
/**
 * Set FRPC connection state to connected
 */
void set_frpc_connection_connected(void) 
{
    frpc_connection_state = CONNECTION_CONNECTED;
    ESP_LOGI(TAG, "FRPC connection established - NET LED on");
}

/**
 * Set FRPC connection state to lost
 */
void set_frpc_connection_lost(void) 
{
    frpc_connection_state = CONNECTION_LOST;
    ESP_LOGI(TAG, "FRPC connection lost - NET LED fast blink");
}

/**
 * Set FRPC connection state to disconnected
 */
void set_frpc_connection_disconnected(void) 
{
    frpc_connection_state = CONNECTION_DISCONNECTED;
    ESP_LOGI(TAG, "FRPC disconnected - NET LED off");
}

/**
 * Get current tick count
 */
uint32_t get_tick_count(void) 
{
    return tickcnt;
}
```

### main-webserver

#### webserver前置操作

1. 变量存储html数据数组、数据大小
2. 静态常量的`httpd_uri_t`类型数组存储多个URL路由表
   * `uri` 访问相对地址
   * `method` 访问方式
   * `handler` 函数句柄，绑定访问
   * `user_ctx` 指向用户上下文的指针
3. 按照路由配置，如果有符合的路由，在前面的先调用，`"/*"`算是通配所有其他路由，都指向`not found` 404 端口

```c
// 全局html数组声明
extern uint8_t g_web_html[8192];
extern size_t g_web_html_size;

// URL路由表
static const httpd_uri_t uri_table[] = {
    {
        .uri = "/",
        .method = HTTP_GET,
        .handler = get_config_page,
        .user_ctx = NULL
    },
    {
        .uri = "/update",
        .method = HTTP_POST,
        .handler = post_config_update,
        .user_ctx = NULL
    },
    {
        .uri = "/*",
        .method = HTTP_GET,
        .handler = not_found_handler,
        .user_ctx = NULL
    }
};
```

1. `HTTPD_DEFAULT_CONFIG`调用宏函数进行默认http配置
2. `lru_purge_enable` 最近最少连接，启用代表允许服务器关闭不常用的连接
3. `max_uri_handlers` 最大允许uri数量
4. `httpd_start` 启动http服务器
5. `httpd_register_uri_handler`传入路由表，注册URI处理程序

[URL和URI的区别](https://zhuanlan.zhihu.com/p/38120321)

```c
/**
 * 初始化Web服务器
 */
esp_err_t webserver_init(void)
{
    ESP_LOGI(TAG, "Initializing web server...");
    
    // 配置HTTP服务器
    httpd_config_t config = HTTPD_DEFAULT_CONFIG();
    config.lru_purge_enable = true;
    config.max_uri_handlers = 16;
    
    // 启动HTTP服务器
    esp_err_t ret = httpd_start(&server, &config);
    if (ret != ESP_OK) {
        ESP_LOGE(TAG, "Failed to start HTTP server: %s", esp_err_to_name(ret));
        return ret;
    }
    
    // 注册URI处理器
    for (int i = 0; i < sizeof(uri_table) / sizeof(uri_table[0]); i++) {
        ret = httpd_register_uri_handler(server, &uri_table[i]);
        if (ret != ESP_OK) {
            ESP_LOGE(TAG, "Failed to register URI handler %d: %s", i, esp_err_to_name(ret));
            return ret;
        }
    }
    
    // ESP8266不支持错误处理器注册，使用通配符路由处理404
    
    ESP_LOGI(TAG, "Web server initialized successfully");
    return ESP_OK;
}
```

#### 页面处理函数

`httpd_resp_set_type`设置http响应的响应类型，`httpd_resp_set_hdr`设置http响应的响应头，`httpd_resp_send`返回全部数据，此处是经过处理的html数据，会在浏览器端解析

```c
// HTTP GET处理函数 - 返回配置页面
static esp_err_t get_config_page(httpd_req_t *req)
{
    ESP_LOGI(TAG, "GET / - Serving configuration page");
    
    // 设置响应头
    httpd_resp_set_type(req, "text/html");
    httpd_resp_set_hdr(req, "Content-Encoding", "identity");
    
    // 直接返回全局html数组内容
    httpd_resp_send(req, (const char*)g_web_html, g_web_html_size);
    
    return ESP_OK;
}
```

1. 获取数据长度，如果过长就返回**HTTP 500**代表无法处理
2. 分配函数内缓冲区储存数据
3. `httpd_req_recv` 从响应中读取数据，返回实际读取的数据大小
4. 在读取数组最后加上 '\0'
5. `handle_config_update` 自定义函数，传入获取的数据，进行状态更新
6. 处理后释放
7. 更新成功后返回`text`格式的重启界面，并进行设备重启
8. 配置不成功返回错误界面（html中`<a>`块用于跳转）

```c
// HTTP POST处理函数 - 处理配置更新
static esp_err_t post_config_update(httpd_req_t *req)
{
    ESP_LOGI(TAG, "POST /update - Processing configuration update");
    
    // 获取POST数据长度
    size_t content_len = req->content_len;
    if (content_len > 1024) {
        ESP_LOGE(TAG, "POST data too large: %d", content_len);
        httpd_resp_send_500(req);
        return ESP_FAIL;
    }
    
    // 分配缓冲区
    char* post_data = malloc(content_len + 1);
    if (!post_data) {
        ESP_LOGE(TAG, "Failed to allocate memory for POST data");
        httpd_resp_send_500(req);
        return ESP_FAIL;
    }
    
    // 读取POST数据
    int recv_len = httpd_req_recv(req, post_data, content_len);
    if (recv_len <= 0) {
        ESP_LOGE(TAG, "Failed to receive POST data");
        free(post_data);
        httpd_resp_send_500(req);
        return ESP_FAIL;
    }
    post_data[recv_len] = '\0';
    
    ESP_LOGI(TAG, "Received POST data: %s", post_data);
    
    // 处理配置更新
    esp_err_t ret = handle_config_update(post_data, recv_len);
    free(post_data);
    
    if (ret == ESP_OK) {
        // 返回成功页面
        const char* success_html = 
            "<!DOCTYPE html>\n"
            "<html><head><meta charset=\"UTF-8\"><title>配置更新成功</title></head>\n"
            "<body><h2>配置更新成功！</h2><p>设备将在3秒后重启...</p>\n"
            "<script>setTimeout(function(){window.location.href='/';}, 3000);</script>\n"
            "</body></html>";
        
        httpd_resp_set_type(req, "text/html");
        httpd_resp_send(req, success_html, strlen(success_html));
        
        // 延迟重启设备
        vTaskDelay(3000 / portTICK_PERIOD_MS);
        esp_restart();
    } else {
        // 返回错误页面
        const char* error_html = 
            "<!DOCTYPE html>\n"
            "<html><head><meta charset=\"UTF-8\"><title>配置更新失败</title></head>\n"
            "<body><h2>配置更新失败！</h2><p><a href=\"/\">返回配置页面</a></p></body></html>";
        
        httpd_resp_set_type(req, "text/html");
        httpd_resp_send(req, error_html, strlen(error_html));
    }
    
    return ESP_OK;
}
```

访问地址不存在，输出404报错界面，`httpd_resp_set_status`设置HTTP状态为**404**未找到

```c
// 404处理函数 - 作为通用处理器
static esp_err_t not_found_handler(httpd_req_t *req)
{
    ESP_LOGI(TAG, "404 - Not Found: %s", req->uri);
    
    const char* not_found_html = 
        "<!DOCTYPE html>\n"
        "<html><head><meta charset=\"UTF-8\"><title>404 - 页面未找到</title></head>\n"
        "<body><h2>404 - 页面未找到</h2><p><a href=\"/\">返回主页</a></p></body></html>";
    
    httpd_resp_set_type(req, "text/html");
    httpd_resp_set_status(req, "404 Not Found");
    httpd_resp_send(req, not_found_html, strlen(not_found_html));
    
    return ESP_OK;
}
```

#### HTTP其他函数

POST请求数据获取后的处理函数

1. 初始化变量储存数据，并在末尾加上'\0'便于字符串处理
2. `strtok`根据分隔符将字符串分割成一系列字串，第一次调用获取第一个字串
3. `while (token)` 每次循环获取下一字串，没有更多字串返回`NULL`跳出循环
4. `strchr`在字符串中找到对应符号位置，返回指向该位置的指针，将该位写为`\0`，对后续解码进行分割
5. `url_decode`进行URL解码，分别获取`key`和`value`
6. 进行键值对匹配，成功后进行修改（即本地更新配置）
7. `free`释放资源
8. `config_save_to_nvs` 储存配置到NVS（修改后的配置和默认不同，但版本号不变，确保配置项相同，能够使用）
9. `config_print` 打印当前配置

```c
/**
 * 处理配置更新
 */
esp_err_t handle_config_update(const char* post_data, size_t data_len)
{
    ESP_LOGI(TAG, "Processing configuration update...");
    
    // 解析POST数据（application/x-www-form-urlencoded格式）
    char* data_copy = malloc(data_len + 1);
    if (!data_copy) {
        ESP_LOGE(TAG, "Failed to allocate memory for data copy");
        return ESP_ERR_NO_MEM;
    }
    memcpy(data_copy, post_data, data_len);
    data_copy[data_len] = '\0';
    
    // 解析表单数据
    char* token = strtok(data_copy, "&");
    while (token) {
        char* equal_sign = strchr(token, '=');
        if (equal_sign) {
            *equal_sign = '\0';
            char* key = url_decode(token);
            char* value = url_decode(equal_sign + 1);
            
            if (key && value) {
                ESP_LOGI(TAG, "Config: %s = %s", key, value);
                
                // 更新配置
                // 只处理非空值，空值保持原配置不变
                if (strcmp(key, "wifi_ssid") == 0 && strlen(value) > 0) {
                    strncpy(g_device_config.wifi_ssid, value, sizeof(g_device_config.wifi_ssid) - 1);
                    ESP_LOGI(TAG, "Updated WiFi SSID: %s", value);
                } else if (strcmp(key, "wifi_pass") == 0 && strlen(value) > 0) {
                    strncpy(g_device_config.wifi_password, value, sizeof(g_device_config.wifi_password) - 1);
                    ESP_LOGI(TAG, "Updated WiFi password");
                } else if (strcmp(key, "wifi_enc") == 0 && strlen(value) > 0) {
                    strncpy(g_device_config.wifi_encryption, value, sizeof(g_device_config.wifi_encryption) - 1);
                    ESP_LOGI(TAG, "Updated WiFi encryption: %s", value);
                } else if (strcmp(key, "frp_server") == 0 && strlen(value) > 0) {
                    strncpy(g_device_config.frp_server, value, sizeof(g_device_config.frp_server) - 1);
                    ESP_LOGI(TAG, "Updated FRP server: %s", value);
                } else if (strcmp(key, "frp_port") == 0 && strlen(value) > 0) {
                    g_device_config.frp_port = atoi(value);
                    ESP_LOGI(TAG, "Updated FRP port: %d", g_device_config.frp_port);
                } else if (strcmp(key, "frp_token") == 0 && strlen(value) > 0) {
                    strncpy(g_device_config.frp_token, value, sizeof(g_device_config.frp_token) - 1);
                    ESP_LOGI(TAG, "Updated FRP token");
                } else if (strcmp(key, "proxy_name") == 0 && strlen(value) > 0) {
                    strncpy(g_device_config.proxy_name, value, sizeof(g_device_config.proxy_name) - 1);
                    ESP_LOGI(TAG, "Updated proxy name: %s", value);
                } else if (strcmp(key, "proxy_type") == 0 && strlen(value) > 0) {
                    strncpy(g_device_config.proxy_type, value, sizeof(g_device_config.proxy_type) - 1);
                    ESP_LOGI(TAG, "Updated proxy type: %s", value);
                } else if (strcmp(key, "local_ip") == 0 && strlen(value) > 0) {
                    strncpy(g_device_config.local_ip, value, sizeof(g_device_config.local_ip) - 1);
                    ESP_LOGI(TAG, "Updated local IP: %s", value);
                } else if (strcmp(key, "local_port") == 0 && strlen(value) > 0) {
                    g_device_config.local_port = atoi(value);
                    ESP_LOGI(TAG, "Updated local port: %d", g_device_config.local_port);
                } else if (strcmp(key, "remote_port") == 0 && strlen(value) > 0) {
                    g_device_config.remote_port = atoi(value);
                    ESP_LOGI(TAG, "Updated remote port: %d", g_device_config.remote_port);
                } else if (strcmp(key, "heartbeat_interval") == 0 && strlen(value) > 0) {
                    g_device_config.heartbeat_interval = atoi(value);
                    ESP_LOGI(TAG, "Updated heartbeat interval: %d", g_device_config.heartbeat_interval);
                } else if (strcmp(key, "heartbeat_timeout") == 0 && strlen(value) > 0) {
                    g_device_config.heartbeat_timeout = atoi(value);
                    ESP_LOGI(TAG, "Updated heartbeat timeout: %d", g_device_config.heartbeat_timeout);
                }
            }
            
            free(key);
            free(value);
        }
        token = strtok(NULL, "&");
    }
    
    free(data_copy);
    
    // 保存配置到NVS
    esp_err_t ret = config_save_to_nvs();
    if (ret != ESP_OK) {
        ESP_LOGE(TAG, "Failed to save configuration to NVS");
        return ret;
    }
    
    ESP_LOGI(TAG, "Configuration updated successfully");
    config_print();
    
    return ESP_OK;
} 
```

### main-wifi_ap

#### WiFi AP模式配置

流程基本都有注释，且是根据编程指南编写，但IDF在v5版本，WiFi配置流程有变化。

且，该函数是WiFi通用配置初始化，针对AP和station模式需要额外配置

```c
/**
 * 初始化WiFi热点
 */
esp_err_t wifi_ap_init(void)
{
    ESP_LOGI(TAG, "Initializing WiFi Access Point...");
    
    // 初始化TCP/IP适配器
    tcpip_adapter_init();
    
    // 创建默认事件循环
    ESP_ERROR_CHECK(esp_event_loop_create_default());
    
    // 初始化WiFi
    wifi_init_config_t cfg = WIFI_INIT_CONFIG_DEFAULT();
    ESP_ERROR_CHECK(esp_wifi_init(&cfg));
    
    // 注册事件处理器
    ESP_ERROR_CHECK(esp_event_handler_register(WIFI_EVENT,
                                               ESP_EVENT_ANY_ID,
                                               &wifi_event_handler,
                                               NULL));
    
    // 设置WiFi模式为AP
    ESP_ERROR_CHECK(esp_wifi_set_mode(WIFI_MODE_AP));
    
    ESP_LOGI(TAG, "WiFi AP initialized successfully");
    return ESP_OK;
}
```

1. 初始化`wifi_config_t`的AP配置（由于配置需要可变，所以进行单独写入）
2. `esp_wifi_set_config`函数设置wifi配置
3. 后续流程有注释，不作赘述，但在v5版本流程需要重写

```c
/**
 * 启动WiFi热点
 */
esp_err_t wifi_ap_start(void)
{
    ESP_LOGI(TAG, "Starting WiFi Access Point...");
    
    // 配置AP参数
    wifi_config_t wifi_config = {
        .ap = {
            .ssid = "",
            .ssid_len = 0,
            .password = "",
            .max_connection = 4,
            .authmode = WIFI_AUTH_WPA_WPA2_PSK,
            .channel = 1,
            .beacon_interval = 100,
        },
    };
    
    // 使用默认的SSID和密码（不使用用户配置）
    strncpy((char*)wifi_config.ap.ssid, "ESP8266_Config", sizeof(wifi_config.ap.ssid) - 1);
    strncpy((char*)wifi_config.ap.password, "12345678", sizeof(wifi_config.ap.password) - 1);
    wifi_config.ap.ssid_len = strlen((char*)wifi_config.ap.ssid);
    
    // 设置认证模式为WPA2（固定）
    wifi_config.ap.authmode = WIFI_AUTH_WPA2_PSK;
    
    // 设置WiFi配置
    ESP_ERROR_CHECK(esp_wifi_set_config(WIFI_IF_AP, &wifi_config));
    
    // 启动WiFi
    ESP_ERROR_CHECK(esp_wifi_start());
    
    // 等待WiFi启动完成
    vTaskDelay(1000 / portTICK_PERIOD_MS);
    
    // 停止DHCP服务
    ESP_ERROR_CHECK(tcpip_adapter_dhcps_stop(TCPIP_ADAPTER_IF_AP));
    
    // 配置AP的IP地址
    tcpip_adapter_ip_info_t ip_info;
    IP4_ADDR(&ip_info.ip, 192, 168, 4, 1);
    IP4_ADDR(&ip_info.gw, 192, 168, 4, 1);
    IP4_ADDR(&ip_info.netmask, 255, 255, 255, 0);
    esp_err_t ret = tcpip_adapter_set_ip_info(TCPIP_ADAPTER_IF_AP, &ip_info);
    if (ret != ESP_OK) {
        ESP_LOGE(TAG, "Failed to set AP IP info: %s", esp_err_to_name(ret));
        return ret;
    }
    
    // 重新启动DHCP服务
    ESP_ERROR_CHECK(tcpip_adapter_dhcps_start(TCPIP_ADAPTER_IF_AP));
    
    ESP_LOGI(TAG, "WiFi AP started with SSID: %s", wifi_config.ap.ssid);
    ESP_LOGI(TAG, "AP IP address: %s", ap_ip);
    
    return ESP_OK;
}
```

调用函数停止WiFi

```c
/**
 * 停止WiFi热点
 */
esp_err_t wifi_ap_stop(void)
{
    ESP_LOGI(TAG, "Stopping WiFi Access Point...");
    
    ESP_ERROR_CHECK(esp_wifi_stop());
    ESP_ERROR_CHECK(esp_wifi_deinit());
    
    ESP_LOGI(TAG, "WiFi AP stopped");
    return ESP_OK;
}
```

#### WiFi station模式配置

配置基本按照流程，具体如果是既AP又Station的配置还是按照v5的编程指南[WiFi模式](https://docs.espressif.com/projects/esp-idf/zh_CN/stable/esp32/api-guides/wifi.html#id38)中进行配置

```c
/**
 * 初始化WiFi Station模式
 */
esp_err_t wifi_sta_init(void)
{
    ESP_LOGI(TAG, "Initializing WiFi Station...");
    
    // 初始化TCP/IP适配器
    tcpip_adapter_init();
    
    // 创建默认事件循环（如果还没有创建）
    esp_err_t ret = esp_event_loop_create_default();
    if (ret != ESP_OK && ret != ESP_ERR_INVALID_STATE) {
        ESP_ERROR_CHECK(ret);
    }
    
    // 初始化WiFi
    wifi_init_config_t cfg = WIFI_INIT_CONFIG_DEFAULT();
    ESP_ERROR_CHECK(esp_wifi_init(&cfg));
    
    // 注册事件处理器
    ESP_ERROR_CHECK(esp_event_handler_register(WIFI_EVENT,
                                               ESP_EVENT_ANY_ID,
                                               &wifi_sta_event_handler,
                                               NULL));
    ESP_ERROR_CHECK(esp_event_handler_register(IP_EVENT,
                                               IP_EVENT_STA_GOT_IP,
                                               &wifi_sta_event_handler,
                                               NULL));
    
    // 设置WiFi模式为Station
    ESP_ERROR_CHECK(esp_wifi_set_mode(WIFI_MODE_STA));
    
    ESP_LOGI(TAG, "WiFi Station initialized successfully");
    return ESP_OK;
}
```

连接和断开连接的代码如下

```c
/**
 * 连接到WiFi网络
 */
esp_err_t wifi_sta_connect(const char* ssid, const char* password)
{
    ESP_LOGI(TAG, "Connecting to WiFi SSID: %s", ssid);
    
    wifi_config_t wifi_config = {0};
    
    // 设置SSID和密码
    strncpy((char*)wifi_config.sta.ssid, ssid, sizeof(wifi_config.sta.ssid) - 1);
    strncpy((char*)wifi_config.sta.password, password, sizeof(wifi_config.sta.password) - 1);
    
    // 设置WiFi配置
    ESP_ERROR_CHECK(esp_wifi_set_config(WIFI_IF_STA, &wifi_config));
    
    // 启动WiFi
    ESP_ERROR_CHECK(esp_wifi_start());
    
    // 等待连接建立（最多等待10秒）
    int retry_count = 0;
    while (!sta_connected && retry_count < 100) {
        vTaskDelay(100 / portTICK_PERIOD_MS);
        retry_count++;
    }
    
    if (sta_connected) {
        ESP_LOGI(TAG, "Successfully connected to WiFi");
        return ESP_OK;
    } else {
        ESP_LOGE(TAG, "Failed to connect to WiFi within timeout");
        return ESP_FAIL;
    }
}

/**
 * 断开WiFi连接
 */
esp_err_t wifi_sta_disconnect(void)
{
    ESP_LOGI(TAG, "Disconnecting from WiFi...");
    sta_connected = false;
    ESP_ERROR_CHECK(esp_wifi_stop());
    ESP_ERROR_CHECK(esp_wifi_deinit());
    ESP_LOGI(TAG, "WiFi Station disconnected");
    return ESP_OK;
}
```

### main.c

1. 串口配置并写入
2. NVS初始化并调用`config_init`从NVS中获取配置
3. 初始化GPIO、创建软件定时器
4. 检查是否进入配置模式（根据按钮是否按下决定）
5. 从NVS中获取分割的web页面并拼接
   1. 打开分区
   2. 每次访问1200字节，和分割时一致
   3. 循环中动态生成键名
   4. 从NVS中获取二进制数据
   5. 每次在后方写入
6. 配置模式中初始化wifi AP，初始化web server，循环打印状态
7. 正常模式初始化网络，连接WiFi，进行组件初始化，连接服务器

```c
void app_main()
{
    // Configure UART parameters for serial communication
    uart_config_t uart_config = {
        .baud_rate = 115200,         // Set baud rate to 115200
        .data_bits = UART_DATA_8_BITS, // 8-bit data format
        .parity    = UART_PARITY_DISABLE, // No parity check
        .stop_bits = UART_STOP_BITS_1, // 1 stop bit
        .flow_ctrl = UART_HW_FLOWCTRL_DISABLE // No hardware flow control
    };
    // Apply UART configuration to UART0 (console port)
    uart_param_config(UART_NUM_0, &uart_config);

    // Initialize Non-Volatile Storage (NVS)
    ESP_ERROR_CHECK(nvs_flash_init());
    
    // Initialize configuration system
    ESP_ERROR_CHECK(config_init());
    
    // Initialize GPIO pins
    init_gpio_pins();
    
    // Initialize timer (needed for config mode detection)
    CreateTimer();
    
    // Check if should enter config mode
    config_mode = check_config_mode();


    
    // 启动时从NVS读取webpart1-6（只读模式）
    nvs_handle nvs_handle_web;
    esp_err_t err_web = nvs_open("config", NVS_READONLY, &nvs_handle_web);
    if (err_web == ESP_OK) {
        size_t part_size = 1200;  // 修改为1200字节，匹配split -b 1200
        size_t len = part_size;
        for (int i = 1; i <= 6; ++i) {  // 读取6个部分
            char key[32];
            snprintf(key, sizeof(key), "webpart%d", i);
            err_web = nvs_get_blob(nvs_handle_web, key, g_web_html + (i-1)*part_size, &len);
            if (err_web != ESP_OK) {
                ESP_LOGE("MAIN", "Failed to read %s from NVS: %s", key, esp_err_to_name(err_web));
            }
            len = part_size;
        }
        nvs_close(nvs_handle_web);
    } else {
        ESP_LOGE("MAIN", "Failed to open NVS for web html: %s", esp_err_to_name(err_web));
    }

    if (config_mode) {
        // 配置模式：启动WiFi热点和Web服务器
        ESP_LOGI("MAIN", "=== Entering Configuration Mode ===");
        ESP_LOGI("MAIN", "Tick count: %u", get_tick_count());
        
        // 进入webserver模式时设置LED状态
        gpio_set_level(LINK_LED, 0);  // 点亮LINK LED表示进入配置模式
        
        // Initialize WiFi Access Point
        ESP_ERROR_CHECK(wifi_ap_init());
        ESP_ERROR_CHECK(wifi_ap_start());

        // Initialize and start Web Server
        ESP_ERROR_CHECK(webserver_init());
        ESP_ERROR_CHECK(webserver_start());
        
        ESP_LOGI("MAIN", "=== ESP8266 Configuration System Ready ===");
        ESP_LOGI("MAIN", "AP SSID: ESP8266_Config (default)");
        ESP_LOGI("MAIN", "AP Password: 12345678 (default)");
        ESP_LOGI("MAIN", "AP IP Address: %s", wifi_ap_get_ip());
        ESP_LOGI("MAIN", "Web Server: http://%s", wifi_ap_get_ip());
        ESP_LOGI("MAIN", "Configure WiFi settings for FRP connection");
        ESP_LOGI("MAIN", "==========================================");

        // 配置模式主循环 - 基于tick计数
        uint32_t last_tick = get_tick_count();
        while (1) {
            // 每秒打印一次状态
            uint32_t current_tick = get_tick_count();
            if ((current_tick - last_tick) >= 10) {  // 10个滴答 = 1秒
                ESP_LOGI("MAIN", "Config mode running... tick: %u", current_tick);
                last_tick = current_tick;
            }
            vTaskDelay(10 / portTICK_PERIOD_MS);
        }
    } else {
        // 正常模式：准备启动frpc客户端
        ESP_LOGI("MAIN", "=== Entering Normal Mode (FRP Client) ===");
        ESP_LOGI("MAIN", "Tick count: %u", get_tick_count());
        ESP_LOGI("MAIN", "Target WiFi SSID: %s", g_device_config.wifi_ssid);
        ESP_LOGI("MAIN", "FRP Server: %s:%d", g_device_config.frp_server, g_device_config.frp_port);
        ESP_LOGI("MAIN", "==========================================");
        
        
        // 初始化TCP/IP网络接口
        ESP_ERROR_CHECK(esp_netif_init());
        
        // 创建默认事件循环
        ESP_ERROR_CHECK(esp_event_loop_create_default());
        
        // 初始化并连接WiFi Station
        ESP_LOGI("MAIN", "Initializing WiFi Station...");
        ESP_ERROR_CHECK(wifi_sta_init());
        
        ESP_LOGI("MAIN", "Connecting to WiFi: %s", g_device_config.wifi_ssid);
        esp_err_t wifi_ret = wifi_sta_connect(g_device_config.wifi_ssid, g_device_config.wifi_password);
        if (wifi_ret != ESP_OK) {
            ESP_LOGE("MAIN", "Failed to connect to WiFi, entering error state");
            gpio_set_level(LINK_LED, 0);  // 点亮网络LED表示错误
            while(1) {
                vTaskDelay(1000 / portTICK_PERIOD_MS);
            }
        }
        
        // WiFi连接成功，点亮网络LED
        gpio_set_level(LINK_LED, 0);
        ESP_LOGI("MAIN", "WiFi connected successfully");
        
        // 初始化自定义硬件组件
        ESP_LOGI("MAIN", "Initializing FRP client components...");
        initialize();
        
        // 建立与远程服务器的连接
        ESP_LOGI("MAIN", "Connecting to FRP server...");
        connect_to_server();
    }
}
```

### html页

html页面承担一个表单的作用，负责直观地修改配置，同时有基础的样式和js逻辑代码

## 总结

以ESP作为FRP客户端有几个要素。一个web页面和http服务器作为承载，一个加密和解密的组件，一个JSON格式转换，一个管理整机状态，一个发送和响应消息。整个组成ESP-FRPC的完整流程。后续移植有一个大体的参照