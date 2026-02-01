# I2C Tools I2C工具

> 笔者注：经过笔者实测，该例程跑在ESP32C3的supermini开发板上，会显示警告
> `--- Warning: Writing to serial is timing out. Please make sure that your application supports an interactive console and that you have picked the correct console for serial communication.`
> 该警告会导致命令行无法正常响应，直接导致I2C工具***无法正常运行***

## 粗略阅读README文档

文档简介本示例实现了基于esp32控制台组件的I2C工具，支持配置(GPIO引脚、端口号、频率)，扫描I2C总线设备，读取寄存器，设置寄存器，检查寄存器功能

使用时需要硬件设备，正确接线和软件配置

后附有示例输出

## 构建、烧录和监视

* 选择目标芯片
* 选择端口号
* 配置项目
* 点击**构建、烧录和监视**
![监视输出](it1.png)

由于笔者身边并没有i2c设备，但本例的实际意义在于会使用该工具，故笔者根据示例输出对工具的使用进行介绍

### help

在窗口中键入help并回车，示例给了如下的输出。
help的作用是帮助，把所有在控制台组件注册的命令显示出来，并指示各命令该怎么使用，和各命令的作用

```bash
help
  Print the list of registered commands

i2cconfig  [--port=<0|1>] [--freq=<Hz>] --sda=<gpio> --scl=<gpio>
  Config I2C bus
  --port=<0|1>  Set the I2C bus port number
   --freq=<Hz>  Set the frequency(Hz) of I2C bus
  --sda=<gpio>  Set the gpio for I2C SDA
  --scl=<gpio>  Set the gpio for I2C SCL

i2cdetect
  Scan I2C bus for devices

i2cget  -c <chip_addr> [-r <register_addr>] [-l <length>]
  Read registers visible through the I2C bus
  -c, --chip=<chip_addr>  Specify the address of the chip on that bus
  -r, --register=<register_addr>  Specify the address on that chip to read from
  -l, --length=<length>  Specify the length to read from that data address

i2cset  -c <chip_addr> [-r <register_addr>] [<data>]...
  Set registers visible through the I2C bus
  -c, --chip=<chip_addr>  Specify the address of the chip on that bus
  -r, --register=<register_addr>  Specify the address on that chip to read from
        <data>  Specify the data to write to that data address

i2cdump  -c <chip_addr> [-s <size>]
  Examine registers visible through the I2C bus
  -c, --chip=<chip_addr>  Specify the address of the chip on that bus
  -s, --size=<size>  Specify the size of each read

free
  Get the current size of free heap memory

heap
  Get minimum size of free heap memory that was available during program execu
  tion

version
  Get version of chip and SDK

restart
  Software reset of the chip

deep_sleep  [-t <t>] [--io=<n>] [--io_level=<0|1>]
  Enter deep sleep mode. Two wakeup modes are supported: timer and GPIO. If no
  wakeup option is specified, will sleep indefinitely.
  -t, --time=<t>  Wake up time, ms
      --io=<n>  If specified, wakeup using GPIO with given number
  --io_level=<0|1>  GPIO level to trigger wakeup

light_sleep  [-t <t>] [--io=<n>]... [--io_level=<0|1>]...
  Enter light sleep mode. Two wakeup modes are supported: timer and GPIO. Mult
  iple GPIO pins can be specified using pairs of 'io' and 'io_level' arguments
  . Will also wake up on UART input.
  -t, --time=<t>  Wake up time, ms
      --io=<n>  If specified, wakeup using GPIO with given number
  --io_level=<0|1>  GPIO level to trigger wakeup

tasks
  Get information about running tasks
```

### i2cconfig

该命令对I2C总线进行配置，中括号中的是可选项，其他是必须项，顺序没有特定要求

` i2cconfig  [--port=<0|1>] [--freq=<Hz>] --sda=<gpio> --scl=<gpio> `

* `--port` 指定I2C端口，可选0或1
* `--freq` 指定I2C总线频率
* `--sda` 、 `--scl` 指定SPI总线的GPIO引脚编号

### i2cdetect

借用示例的输出，调用i2cdetect会扫描i2c总线，查找总线上的设备并如图显示

```bash
i2c-tools> i2cdetect
     0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
00: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
10: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
20: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
30: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
40: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
50: -- -- -- -- -- -- -- -- -- -- -- 5b -- -- -- --
60: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
70: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
```

### i2cget

该命令用于**读取**可用的寄存器地址数据，尖括号中的参数需要替换为指定地址，和上一命令配合使用，扫描到总线设备后填入设备地址，以查询寄存器

`i2cget  -c <chip_addr> [-r <register_addr>] [-l <length>]` 

* `-c` 用于指定I2C设备的地址
* `-r` 用于指定要检查的寄存器地址
* `-l` 用于指定内容的长度

借用示例：

```bash
i2c-tools> i2cget -c 0x5b -r 0x00 -l 1
0x10
```

该示例就读入了`0x5b`地址设备的`0x00`寄存器的`1`字节数据

### i2cset

该命令用于**设置/写入**可用的寄存器地址。

`i2cset  -c <chip_addr> [-r <register_addr>] [<data>]...`

* `-c` 用于指定I2C设备的地址
* `-r` 用于指定写入的地址
* `data` 指定写入的数据

借用示例：

```bash
i2c-tools> i2cset -c 0x5b -r 0xF4
I (734717) cmd_i2ctools: Write OK
i2c-tools> i2cset -c 0x5b -r 0x01 0x10
I (1072047) cmd_i2ctools: Write OK
i2c-tools> i2cget -c 0x5b -r 0x00 -l 1
0x98
```

该示例向`0x5b`地址的设备进行操作，指定`0xf4`地址但没有写入数据，可能只是触发设备内部的某种操作，**Write OK**证明操作成功；
后指定向`0x01`地址写入`0x10`也成功，这两个操作可能是触发了设备的某种逻辑，进行测量或其他操作，设备会对`0x00`地址的值进行更改，再读取可以查看

### i2cdump

该命令用于检查总线上可用的寄存器，(*注意：文档说明该函数只支持I2C设备内寄存器长度相同的寄存器)

`i2cdump  -c <chip_addr> [-s <size>]`

* `-c` 用于指定I2C设备地址
* `-s` 用于指定每次读取大小

### 其他

其他函数在示例输出中没有具体展示，笔者以为凭上述函数足以应付大部分情况，故其他函数不作具体分析，可在后续使用时进行实际尝试

## 代码简要分析

> 由于本例的代码重在使用和操作，笔者不会对代码进行逐行逐块分析，相关代码逻辑和调用可见[代码分析](../../../system/freertos/basic_freertos/basic_freertos.md#代码分析)

### app_main函数

函数中进行了文件创建(数据保存)的初始化准备；根据不同烧录方式进行repl控制器组件的新建；新建了I2C总线；绑定了I2Ctools的工具和命令；输出了初始界面；启动repl控制器组件

(*文件系统的操作笔者不懂，暂时不作分析*)

### 命令文件

定义`i2cconfig_args`结构体参数，用于储存总线参数

*这里的 arg_int 是 argtable3 库中定义的结构体，用于解析命令行中的整型参数。*

```c
static struct {
    struct arg_int *port;
    struct arg_int *freq;
    struct arg_int *sda;
    struct arg_int *scl;
    struct arg_end *end;
} i2cconfig_args;
```

具体的I2C配置较多，笔者本例不单独进行分析。

各命令的注册采用`register_xx`的形式，内部配置`esp_console_cmd_t`类型结构体绑定`do_xx`的自定义函数，完成各操作的注册

## 总结

本例是支持ESP的I2C工具组介绍，笔者认为最大的意义在于可以直接对ESP连接的设备进行测试。和SPI相同，在开发过程中大部分情况都是利用I2C通信控制其他设备而不会去过度关注通信具体是怎样，所有该工具的使用意义大于学习意义。
