# ESP-IDF 采用C++进行开发时的注意事项

## 程序相关

* 在进行各种组件初始化时会对结构体进行填充，但c++对此非常严格，如下方配置时需要严格根据结构体定义时元素顺序进行赋值，不然会报错

```cpp
    i2c_config_t config = {
        .mode = I2C_MODE_MASTER,
        .sda_io_num = I2C_MASTER_SDA_IO,
        .scl_io_num = I2C_MASTER_SCL_IO,
        .sda_pullup_en = GPIO_PULLUP_ENABLE,
        .scl_pullup_en = GPIO_PULLUP_ENABLE,
        .clk_flags = I2C_SCLK_SRC_FLAG_FOR_NOMAL
    };
```

## 构建相关

* 如果把`app_main()`函数的文件改为了c++文件，在编译时函数会被名称修饰，导致构建项目时找不到该函数，需要进行声明

```cpp
extern "C" void app_main(void) {
}
```