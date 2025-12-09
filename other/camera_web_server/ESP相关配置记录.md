# ESP相关配置记录

本文不是在实验该例程中撰写，但都是在进行esp32cam该开发板使用过程中遇到的问题，所以放在这里

## Kconfig

* 在main组件中的文件为`Kconfig.projbuild`，其他组件中为`Kconfig`具体的编写直接抄各个例程中的配置
* 写完后需要重新构建项目才能识别
* 采用IDF框架，导入esp32-camera后需要在menuconfig中开启PSRAM，并设置Flash和PSRAM频率为80MHz
* 有关PSRAM配置![PSRAM配置](psram1.png)

## CMakeLists

* 项目文件CMake必须包含下面三行

```cmake
cmake_minimum_required(VERSION 3.16)
include($ENV{IDF_PATH}/tools/cmake/project.cmake)
project(ESP-MP3)
```

* 组件CMake以下方为例
  * SRCS 代表源代码
  * INCLUDSE_DIR 代表包含的头文件地址文件夹，如果是整个组件文件就写 "."
  * PRIV_REQUIRES 代表私有依赖，写组件需要的库文件（用的不是很明白，但**如果需要项目内组件必须写**）
  * EMBED_TXTFILES 需要的文件，在本项目中用于链接证书

```cmake
idf_component_register(SRCS "HTTP_server.c"
                    INCLUDE_DIRS "include"
                    PRIV_REQUIRES esp_https_server esp_wifi WiFi_config
                    EMBED_TXTFILES "certs/servercert.pem"
                                   "certs/prvtkey.pem")
```

* 组件头文件不建议包含库函数，如果有特殊的类型使用，在CMake的PRIV_REQUIRES写需要的库文件名
* 如果组件A包含组件B，而组件B在**头文件的定义中**需要`PRIV_REQUIRES`某组件，那么，需要在组件A的CMake中`PRIV_REQUIRES`该组件，下方代码就是在main组件中调用HTTP_server但没有在CMake中写需要依赖

```c
In file included from D:/ESP/Project/ESP-MP3/main/main.c:8:
D:/ESP/Project/ESP-MP3/components/HTTP_server/include/HTTP_server.h:4:10: fatal error: esp_http_server.h: No such file or directory 
    4 | #include <esp_http_server.h>
      |          ^~~~~~~~~~~~~~~~~~~
compilation terminated.
```

## WiFi配置

* 采用如图方式选择WiFi模式![esp-wifi](esp-wifi1.png)
* 报错时检查头文件包含，如`"freertos/event_groups.h"`使用时需要`"freertos/FreeRTOS.h"`作为前置

## HTTP-SEVER配置

* 创建服务端时采用全局变量进行存储，且最好不要跨文件，可能会指针错误。
* 每次创建前检查是否已存在，防止重新创建
* 如果创建https的服务端，需要导入`<esp_https_server.h>`并且需要证书配置，启动和停止采用`httpd_ssl_start``httpd_ssl_stop`，宏函数默认配置有SSL，事件处理为HTTPS事件
* 想要使用URL正则表达式包括通配符，需要进行`conf.uri_match_fn = httpd_uri_match_wildcard;`将通配符匹配函数赋给函数指针，使能通配符匹配
* 外部html文件建议通过上CMake外部文件包含，然后在代码中进行`extern const unsigned char web_html_start[] asm("_binary_web_server_html_start");`获取头指针，`extern const unsigned char web_html_end[] asm("_binary_web_server_html_end");`获取尾指针，总长度是**尾指针-头指针**

## ISR中断相关

* 中断中不能使用如`ESP_LOG`相关的日志打印函数，其他rtos任务函数除非中断特殊版本，否则也会超时触发看梦狗崩溃
* 任务句柄最好作为全局变量，需要传入中断中，如果是函数内变量，在函数结束后该句柄会变成野指针，会触发崩溃

## camera相关

> 代码大部分为提出需求后由AI进行编写

* 在数据传输时出现黑边，花屏等问题，添加帧校验判断，特别的在判断后需要添加**continue**，跳过后续的帧处理，否则写入无效帧后web端无法正常解析

```c++
        // 帧校验判断
        // 头
        if (fb->len < 2 || fb->buf[0] != 0xFF || fb->buf[1] != 0xD8)
        {
            Serial.println("[CameraAsync] Invalid JPEG frame, skipping...");
            camera_manager::release(fb);
            continue;
        }
        // 尾
        if (fb->buf[fb->len - 2] != 0xFF || fb->buf[fb->len - 1] != 0xD9)
        {
            Serial.println("[CameraAsync] JPEG end marker missing");
            camera_manager::release(fb);
            continue;
        }
```

* 动态分配内存可能存在内存碎片化而无法分配大块内存，本产品使用静态内存池分配
* 在通信大小不够时，建议采用分块发送的方式，但不可避免的，显示数据会有颜色错误，割裂等情况（至少我暂时没办法）

## I2S相关

* camera组件连接摄像头会使用I2S0进行数据传输，如果有其他需要用的模块会冲突（修正，以OV2640为例，使用的是SCCB协议，不一定用I2S，在esp32-camera组件中使用的就是I2C）
* 如果采用中断配置，只需要把`intr_alloc_flags`项进行配置，中断无回调函数，只会在DMA接收一定量数据后自动放入队列，依靠队列的配合传输数据
* INMP441具体的使用参考手册或网上可行的教程，判断是否可用靠I2S位数配置，麦克风是24bit数据

## 其他

* 经过评估，无法在esp32运行frpc，性能有限，且系统等资源限制。