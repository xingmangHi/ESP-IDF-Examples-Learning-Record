# esp-who安装记录

## 安装过程

1. 从`github`克隆或直接下载压缩包 `esp-who` 
2. 解压文件到某一目录下
3. 新建项目，复制需要的例程文件直接覆盖
4. 将`components`文件夹整个复制到项目文件中（组件需要在项目CMake中注册，可以不复制而使用路径实现）
5. 如果复制，CMake中设置如下，其中`./`代表相对路径同一文件夹下

```cmake
set(EXTRA_COMPONENT_DIRS ./components/who_task
                         ./components/who_peripherals/who_usb
                         ./components/who_peripherals/who_cam
                         ./components/who_peripherals/who_lcd
                         ./components/who_peripherals/who_spiflash_fatfs
                         ./components/who_frame_cap
                         ./components/who_frame_lcd_disp
                         ./components/who_detect
                         ./components/who_recognition
                         ./components/who_app/who_recognition_app)
```

## 构建项目报错

* 某一元素未找到，可能是在版本更新后该元素被弃用，如果没有其他使用可以直接注释掉
* 如果报错是分区大小不足，在 `menuconfig` 中找到 `Partition Table` ，选择 `Custom paratition table CSV` 允许在项目根目录下新建分区表并使用（需要首先修改`Flash size`，esp32s3是16MB）