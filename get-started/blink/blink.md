# Blink 闪烁例程

## 粗略阅读README文档

文档简介*这个例程演示怎样通过GPIO让LED灯闪烁，或者采用`led_strip`库去驱动信号驱动的LED，如WS2812。该库通过组件管理器安装*

注意事项不作解释

采用**menuconfig**配置系统

* 配置**控制方式**(GPIO/LED_strip)
* 在LED_Strip前提下，选择(RMT/SPI)
* 选择**控制引脚**
* 选择**闪烁周期**

构建、烧录、示例输出
特别注意灯的颜色可能不同
对于LED_strip函数简单说明

## 尝试构建、烧录、监视

### GPIO

> 特别说明：
> $~~$在构建过程中，由于`idf_component.yml`文件的存在，会从[组件注册表](https://docs.espressif.com/projects/vscode-esp-idf-extension/zh_CN/latest/additionalfeatures/install-esp-components.html)安装需要的组件，具体查看[构建系统](https://docs.espressif.com/projects/esp-idf/zh_CN/stable/esp32/api-guides/build-system.html#)

* 选择CMake工具
* 选择芯片（*如果报CMake错尝试把bulid文件夹删除再操作*）
* 根据README打开项目配置
![项目配置](b1.png)
* 选择GPIO和引脚（*笔者先进行GPIO的尝试*） 
* 保存配置并构建项目
* *无报错*选择端口，连接主控板
* 烧录和监视
![监视窗口](b2.png)
前面的启动信息不做重复解释，窗口输出均是`ESP_LOGT`函数输出
`gpio`显示选择引脚是**GPIO18**，模式是上拉模式
程序在高低电平变化时窗口提示
* 万用表测试现象：**GPIO18**引脚高低电平变化，*0V*、*3.235V*交替变化，周期大概为1s，与配置相符
![高低电平](b3.BMP)

### LED_strip
