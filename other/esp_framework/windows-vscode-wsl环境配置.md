# windows系统下ESP-IDF环境配置

[参考视频](https://www.bilibili.com/video/BV1MGrTYVE93/?spm_id_from=333.1391.0.0&vd_source=d39eb1c9b3e81d6e121f09fe248dfe02)

## 注意事项

1. WSL中需要安装软件包[Linux系统安装](https://docs.espressif.com/projects/esp-idf/zh_CN/stable/esp32/get-started/linux-macos-setup.html)
2. 通过usbipd进行串口映射，也参考上链接
3. 如果esp有网络相关的变化，在使用时，发现当windows连接到esp的WiFi时，linux中的监视会有问题
4. 配置wsl2网络模式为mirror镜像模式[参考文档](https://blog.csdn.net/TeleostNaCl/article/details/149058218)
5. usbipd进行端口映射，指令如下 `usbipd list` 查看所有端口 ； `usbipd attach --wsl --busid 1-1 ` 最后一串为端口号，进行端口映射 ； `usbipd detach --busid 1-1` 最后一串为端口号，断开端口映射