# SPI Host Driver SPI主机驱动LCD

> 笔者注，从这个文件开始，实验示例采用v5.5版本
> 本例由于笔者并无可供实验用的LCD，故编译后直接进行代码分析

## 粗略阅读README文档

文档简介示例旨在展示如何使用SPI主机驱动程序API，即主要演示API的使用。

硬件需要，构建烧录和示例输出

## 代码分析

> 本示例导入了外部组件库，笔者会酌情进行解读
> 笔者分析完了主体代码，发现组件和其他有些函数为了lcd的数据发送和适配，在实际操作lcd的示例或实验中再作解释

### 全局/局部变量解读

#### main.c文件

`lcd_init_cmd_t` 结构体储存lcd指定数据。包括**命令寄存器地址**、**发送的命令**、特殊作用位(*bit6~0: 真正有效的参数个数;bit7: 置 1 表示发完这条命令后需要延时;特殊值 0xFF 表示整个初始化列表到此结束)

`type_lcd_t` 枚举用于后续使用时指定驱动芯片，1即ILI9341驱动，2即ST7789驱动

**DRAM_ATTR**关键字强制把这些表数据放入DMA可访问的DRAM中，数组中每个元素对应一个`lcd_init_cmd_t`类型指令。两个数组针对两种不同驱动的初始化

```c
/*
 The LCD needs a bunch of command/argument values to be initialized. They are stored in this struct.
*/
typedef struct {
    uint8_t cmd;
    uint8_t data[16];
    uint8_t databytes; //No of data in data; bit 7 = delay after set; 0xFF = end of cmds.
} lcd_init_cmd_t;

typedef enum {
    LCD_TYPE_ILI = 1,
    LCD_TYPE_ST,
    LCD_TYPE_MAX,
} type_lcd_t;

//Place data into DRAM. Constant data gets placed into DROM by default, which is not accessible by DMA.
DRAM_ATTR static const lcd_init_cmd_t st_init_cmds[] = {
    /* Memory Data Access Control, MX=MV=1, MY=ML=MH=0, RGB=0 */
    {0x36, {(1 << 5) | (1 << 6)}, 1},
    /* Interface Pixel Format, 16bits/pixel for RGB/MCU interface */
    {0x3A, {0x55}, 1},
    /* Porch Setting */
    {0xB2, {0x0c, 0x0c, 0x00, 0x33, 0x33}, 5},
    /* Gate Control, Vgh=13.65V, Vgl=-10.43V */
    {0xB7, {0x45}, 1},
    /* VCOM Setting, VCOM=1.175V */
    {0xBB, {0x2B}, 1},
    /* LCM Control, XOR: BGR, MX, MH */
    {0xC0, {0x2C}, 1},
    /* VDV and VRH Command Enable, enable=1 */
    {0xC2, {0x01, 0xff}, 2},
    /* VRH Set, Vap=4.4+... */
    {0xC3, {0x11}, 1},
    /* VDV Set, VDV=0 */
    {0xC4, {0x20}, 1},
    /* Frame Rate Control, 60Hz, inversion=0 */
    {0xC6, {0x0f}, 1},
    /* Power Control 1, AVDD=6.8V, AVCL=-4.8V, VDDS=2.3V */
    {0xD0, {0xA4, 0xA1}, 2},
    /* Positive Voltage Gamma Control */
    {0xE0, {0xD0, 0x00, 0x05, 0x0E, 0x15, 0x0D, 0x37, 0x43, 0x47, 0x09, 0x15, 0x12, 0x16, 0x19}, 14},
    /* Negative Voltage Gamma Control */
    {0xE1, {0xD0, 0x00, 0x05, 0x0D, 0x0C, 0x06, 0x2D, 0x44, 0x40, 0x0E, 0x1C, 0x18, 0x16, 0x19}, 14},
    /* Sleep Out */
    {0x11, {0}, 0x80},
    /* Display On */
    {0x29, {0}, 0x80},
    {0, {0}, 0xff}
};

DRAM_ATTR static const lcd_init_cmd_t ili_init_cmds[] = {
    /* Power control B, power control = 0, DC_ENA = 1 */
    {0xCF, {0x00, 0x83, 0X30}, 3},
    /* Power on sequence control,
     * cp1 keeps 1 frame, 1st frame enable
     * vcl = 0, ddvdh=3, vgh=1, vgl=2
     * DDVDH_ENH=1
     */
    {0xED, {0x64, 0x03, 0X12, 0X81}, 4},
    /* Driver timing control A,
     * non-overlap=default +1
     * EQ=default - 1, CR=default
     * pre-charge=default - 1
     */
    {0xE8, {0x85, 0x01, 0x79}, 3},
    /* Power control A, Vcore=1.6V, DDVDH=5.6V */
    {0xCB, {0x39, 0x2C, 0x00, 0x34, 0x02}, 5},
    /* Pump ratio control, DDVDH=2xVCl */
    {0xF7, {0x20}, 1},
    /* Driver timing control, all=0 unit */
    {0xEA, {0x00, 0x00}, 2},
    /* Power control 1, GVDD=4.75V */
    {0xC0, {0x26}, 1},
    /* Power control 2, DDVDH=VCl*2, VGH=VCl*7, VGL=-VCl*3 */
    {0xC1, {0x11}, 1},
    /* VCOM control 1, VCOMH=4.025V, VCOML=-0.950V */
    {0xC5, {0x35, 0x3E}, 2},
    /* VCOM control 2, VCOMH=VMH-2, VCOML=VML-2 */
    {0xC7, {0xBE}, 1},
    /* Memory access control, MX=MY=0, MV=1, ML=0, BGR=1, MH=0 */
    {0x36, {0x28}, 1},
    /* Pixel format, 16bits/pixel for RGB/MCU interface */
    {0x3A, {0x55}, 1},
    /* Frame rate control, f=fosc, 70Hz fps */
    {0xB1, {0x00, 0x1B}, 2},
    /* Enable 3G, disabled */
    {0xF2, {0x08}, 1},
    /* Gamma set, curve 1 */
    {0x26, {0x01}, 1},
    /* Positive gamma correction */
    {0xE0, {0x1F, 0x1A, 0x18, 0x0A, 0x0F, 0x06, 0x45, 0X87, 0x32, 0x0A, 0x07, 0x02, 0x07, 0x05, 0x00}, 15},
    /* Negative gamma correction */
    {0XE1, {0x00, 0x25, 0x27, 0x05, 0x10, 0x09, 0x3A, 0x78, 0x4D, 0x05, 0x18, 0x0D, 0x38, 0x3A, 0x1F}, 15},
    /* Column address set, SC=0, EC=0xEF */
    {0x2A, {0x00, 0x00, 0x00, 0xEF}, 4},
    /* Page address set, SP=0, EP=0x013F */
    {0x2B, {0x00, 0x00, 0x01, 0x3f}, 4},
    /* Memory write */
    {0x2C, {0}, 0},
    /* Entry mode set, Low vol detect disabled, normal display */
    {0xB7, {0x07}, 1},
    /* Display function control */
    {0xB6, {0x0A, 0x82, 0x27, 0x00}, 4},
    /* Sleep out */
    {0x11, {0}, 0x80},
    /* Display on */
    {0x29, {0}, 0x80},
    {0, {0}, 0xff},
};
```

#### decode_image.c

**asm**：GCC 的 “asm label” 扩展：把 C 标识符 和 汇编层面/链接器层面的符号名 强行绑定在一起。
换句话说，C 里叫 image_jpg_start，但在 .o 文件和最终 ELF 里实际引用的是 _binary_image_jpg_start 这个符号

![定义解释](l1.png)

```c
//Reference the binary-included jpeg file
extern const uint8_t image_jpg_start[] asm("_binary_image_jpg_start");
extern const uint8_t image_jpg_end[] asm("_binary_image_jpg_end");
//Define the height and width of the jpeg file. Make sure this matches the actual jpeg
//dimensions.
```

#### pretty_effect.c

`int8_t` 偏移范围-127~128 ；数组个数320和240与LCD实际像素数一致，对应每一行/列。在实际驱动中，数组用于预储存一行/列的数据。
![数组解释](l2.png)

```c
//Instead of calculating the offsets for each pixel we grab, we pre-calculate the valueswhenever a frame changes, then re-use
//these as we go through all the pixels in the frame. This is much, much faster.
static int8_t xofs[320], yofs[240];
static int8_t xcomp[320], ycomp[240];
```

### app_main()函数

1. 初始化配置变量
2. `spi_bus_config_t` 总线配置[结构体](https://docs.espressif.com/projects/esp-idf/zh_CN/stable/esp32/api-reference/peripherals/spi_master.html#_CPPv416spi_bus_config_t)
   * `miso_io_num` **主机输入从机输出**引脚
   * `mosi_io_num` **主机输出从机输入**引脚
   * `sclk_io_num` SPI用于**时钟输出引脚**，如果不使用配置为-1
   * `quadwp_io_num` 用于**写保护信号的GPIO引脚**，不使用配置为-1
   * `quadhd_io_num` 用于**保持信号的GPIO引脚**，不使用配置为-1
   * `max_transfer_sz`最大传输大小
3. `spi_device_interface_config_t` 设备配置[结构体](https://docs.espressif.com/projects/esp-idf/zh_CN/stable/esp32/api-reference/peripherals/spi_master.html#_CPPv429spi_device_interface_config_t)
   * `clock_speed_hz` SPI时钟速度
   * `mode` SPI 模式，代表一对（CPOL、CPHA）配置
    ![相关说明](l3.png)
   * `spics_io_num` **设备的CS引脚**
   * `queue_size` 队列事务的大小
   * `pre_cb` **传输开始前调用的回调**
4. `spi_bus_initialize` 函数**初始化SPI总线**
5. `spi_bus_add_device` 注册**连接到总线的设备**
6. 自定义函数进行初始化和初始启动LCD

```c
void app_main(void)
{
    esp_err_t ret;
    spi_device_handle_t spi;
    spi_bus_config_t buscfg = {
        .miso_io_num = PIN_NUM_MISO,
        .mosi_io_num = PIN_NUM_MOSI,
        .sclk_io_num = PIN_NUM_CLK,
        .quadwp_io_num = -1,
        .quadhd_io_num = -1,
        .max_transfer_sz = PARALLEL_LINES * 320 * 2 + 8
    };
    spi_device_interface_config_t devcfg = {
#ifdef CONFIG_LCD_OVERCLOCK
        .clock_speed_hz = 26 * 1000 * 1000,     //Clock out at 26 MHz
#else
        .clock_speed_hz = 10 * 1000 * 1000,     //Clock out at 10 MHz
#endif
        .mode = 0,                              //SPI mode 0
        .spics_io_num = PIN_NUM_CS,             //CS pin
        .queue_size = 7,                        //We want to be able to queue 7 transactions at a time
        .pre_cb = lcd_spi_pre_transfer_callback, //Specify pre-transfer callback to handle D/C line
    };
    //Initialize the SPI bus
    ret = spi_bus_initialize(LCD_HOST, &buscfg, SPI_DMA_CH_AUTO);
    ESP_ERROR_CHECK(ret);
    //Attach the LCD to the SPI bus
    ret = spi_bus_add_device(LCD_HOST, &devcfg, &spi);
    ESP_ERROR_CHECK(ret);
    //Initialize the LCD
    lcd_init(spi);
    //Initialize the effect displayed
    ret = pretty_effect_init();
    ESP_ERROR_CHECK(ret);

    //Go do nice stuff.
    display_pretty_colors(spi);
}
```

### LCD初始化函数

1. GPIO初始化配置，将DC(数据/命令选择)，RST(复位)，BCKL(背光控制)对应引脚配置为上拉输出模式
2. 设置GPIO引脚初始电平并等待完成
3. `lcd_get_id` 自定义函数获取lcd的型号判断(0/1)
4. 根据ID确定驱动型号
5. **CONFIG**起始的宏定义一般在项目配置，即menuconfig中配置，不同配置对应不同代码操作
6. while循环发送数组配置的不同驱动的初始化，即一串地址加命令(*根据驱动的要求，完成初始化*)
7. 启动背光

```c
//Initialize the display
void lcd_init(spi_device_handle_t spi)
{
    int cmd = 0;
    const lcd_init_cmd_t* lcd_init_cmds;

    //Initialize non-SPI GPIOs
    gpio_config_t io_conf = {};
    io_conf.pin_bit_mask = ((1ULL << PIN_NUM_DC) | (1ULL << PIN_NUM_RST) | (1ULL << PIN_NUM_BCKL));
    io_conf.mode = GPIO_MODE_OUTPUT;
    io_conf.pull_up_en = true;
    gpio_config(&io_conf);

    //Reset the display
    gpio_set_level(PIN_NUM_RST, 0);
    vTaskDelay(100 / portTICK_PERIOD_MS);
    gpio_set_level(PIN_NUM_RST, 1);
    vTaskDelay(100 / portTICK_PERIOD_MS);

    //detect LCD type
    uint32_t lcd_id = lcd_get_id(spi);
    int lcd_detected_type = 0;
    int lcd_type;

    printf("LCD ID: %08"PRIx32"\n", lcd_id);
    if (lcd_id == 0) {
        //zero, ili
        lcd_detected_type = LCD_TYPE_ILI;
        printf("ILI9341 detected.\n");
    } else {
        // none-zero, ST
        lcd_detected_type = LCD_TYPE_ST;
        printf("ST7789V detected.\n");
    }

#ifdef CONFIG_LCD_TYPE_AUTO
    lcd_type = lcd_detected_type;
#elif defined( CONFIG_LCD_TYPE_ST7789V )
    printf("kconfig: force CONFIG_LCD_TYPE_ST7789V.\n");
    lcd_type = LCD_TYPE_ST;
#elif defined( CONFIG_LCD_TYPE_ILI9341 )
    printf("kconfig: force CONFIG_LCD_TYPE_ILI9341.\n");
    lcd_type = LCD_TYPE_ILI;
#endif
    if (lcd_type == LCD_TYPE_ST) {
        printf("LCD ST7789V initialization.\n");
        lcd_init_cmds = st_init_cmds;
    } else {
        printf("LCD ILI9341 initialization.\n");
        lcd_init_cmds = ili_init_cmds;
    }

    //Send all the commands
    while (lcd_init_cmds[cmd].databytes != 0xff) {
        lcd_cmd(spi, lcd_init_cmds[cmd].cmd, false);
        lcd_data(spi, lcd_init_cmds[cmd].data, lcd_init_cmds[cmd].databytes & 0x1F);
        if (lcd_init_cmds[cmd].databytes & 0x80) {
            vTaskDelay(100 / portTICK_PERIOD_MS);
        }
        cmd++;
    }

    ///Enable backlight
    gpio_set_level(PIN_NUM_BCKL, LCD_BK_LIGHT_ON_LEVEL);
}
```

### lcd相关函数

`lcd_cmd` 函数采用轮询，即直接发送的方式发送cmd控制命令(*对于采用协议驱动的模块，一般有数据码和命令码两种*)

1. memset设置t中所有数据为0
2. 设置t中发送的数据，此函数只发送命令码
3. 采用轮询方式发送
4. 确保发送完成

`lcd_data` 函数采用轮询，即直接发送的方式发送data数据码，具体实现过程与上一代码基本相同

`lcd_spi_pre_transfer_callback` 函数作用在传输开始前，用于配置DC脚

`lcd_get_id` 抢占总线并进行发送接收判断

1. `spi_device_acquire_bus` 函数占用总线，用于连续发送或其他，此时与其他设备的传输处于待处理状态
2. `lcd_cmd` 发送命令码
3. 发送数据码并获取接收值(*spi的发送也会进行一次读取*)
4. 释放总线
5. 返回接收值

```c
/* Send a command to the LCD. Uses spi_device_polling_transmit, which waits
 * until the transfer is complete.
 *
 * Since command transactions are usually small, they are handled in polling
 * mode for higher speed. The overhead of interrupt transactions is more than
 * just waiting for the transaction to complete.
 */
void lcd_cmd(spi_device_handle_t spi, const uint8_t cmd, bool keep_cs_active)
{
    esp_err_t ret;
    spi_transaction_t t;
    memset(&t, 0, sizeof(t));       //Zero out the transaction
    t.length = 8;                   //Command is 8 bits
    t.tx_buffer = &cmd;             //The data is the cmd itself
    t.user = (void*)0;              //D/C needs to be set to 0
    if (keep_cs_active) {
        t.flags = SPI_TRANS_CS_KEEP_ACTIVE;   //Keep CS active after data transfer
    }
    ret = spi_device_polling_transmit(spi, &t); //Transmit!
    assert(ret == ESP_OK);          //Should have had no issues.
}

/* Send data to the LCD. Uses spi_device_polling_transmit, which waits until the
 * transfer is complete.
 *
 * Since data transactions are usually small, they are handled in polling
 * mode for higher speed. The overhead of interrupt transactions is more than
 * just waiting for the transaction to complete.
 */
void lcd_data(spi_device_handle_t spi, const uint8_t *data, int len)
{
    esp_err_t ret;
    spi_transaction_t t;
    if (len == 0) {
        return;    //no need to send anything
    }
    memset(&t, 0, sizeof(t));       //Zero out the transaction
    t.length = len * 8;             //Len is in bytes, transaction length is in bits.
    t.tx_buffer = data;             //Data
    t.user = (void*)1;              //D/C needs to be set to 1
    ret = spi_device_polling_transmit(spi, &t); //Transmit!
    assert(ret == ESP_OK);          //Should have had no issues.
}

//This function is called (in irq context!) just before a transmission starts. It will
//set the D/C line to the value indicated in the user field.
void lcd_spi_pre_transfer_callback(spi_transaction_t *t)
{
    int dc = (int)t->user;
    gpio_set_level(PIN_NUM_DC, dc);
}

uint32_t lcd_get_id(spi_device_handle_t spi)
{
    // When using SPI_TRANS_CS_KEEP_ACTIVE, bus must be locked/acquired
    spi_device_acquire_bus(spi, portMAX_DELAY);

    //get_id cmd
    lcd_cmd(spi, 0x04, true);

    spi_transaction_t t;
    memset(&t, 0, sizeof(t));
    t.length = 8 * 3;
    t.flags = SPI_TRANS_USE_RXDATA;
    t.user = (void*)1;

    esp_err_t ret = spi_device_polling_transmit(spi, &t);
    assert(ret == ESP_OK);

    // Release bus
    spi_device_release_bus(spi);

    return *(uint32_t*)t.rx_data;
}
```

### 发送相关函数

`send_lines` 函数发送多行，采用中断模式提高平均效率(*和轮询多次发送相比*)

1. 初始化储存，其中`trans`变量为static内部静态类型，只在第一次调用函数时初始化
2. for循环初始化每一行数据，偶数位配置成命令码格式，奇数位配置成数据码格式，长度位32
3. 依次指定数组每一位数据
4. `spi_device_queue_trans` 将事务添加到队列中，等待中断传输

> 笔者注，相比assert，采用ESP-IDF中的**ESP_ERROR_CHECK**等宏函数进行错误判断更符合实际使用情况(*assert会直接退出程序*)

`send_line_finish` 函数和`send_lines`函数配合使用，等待发送完成。在官方编程指南中，也采用持续调用**spi_device_get_trans_result**函数的方式等待发送完成，没有对应中断触发(*ISR在许多过程中被禁用)

```c
/* To send a set of lines we have to send a command, 2 data bytes, another command, 2 more data bytes and another command
 * before sending the line data itself; a total of 6 transactions. (We can't put all of this in just one transaction
 * because the D/C line needs to be toggled in the middle.)
 * This routine queues these commands up as interrupt transactions so they get
 * sent faster (compared to calling spi_device_transmit several times), and at
 * the mean while the lines for next transactions can get calculated.
 */
static void send_lines(spi_device_handle_t spi, int ypos, uint16_t *linedata)
{
    esp_err_t ret;
    int x;
    //Transaction descriptors. Declared static so they're not allocated on the stack; we need this memory even when this
    //function is finished because the SPI driver needs access to it even while we're already calculating the next line.
    static spi_transaction_t trans[6];

    //In theory, it's better to initialize trans and data only once and hang on to the initialized
    //variables. We allocate them on the stack, so we need to re-init them each call.
    for (x = 0; x < 6; x++) {
        memset(&trans[x], 0, sizeof(spi_transaction_t));
        if ((x & 1) == 0) {
            //Even transfers are commands
            trans[x].length = 8;
            trans[x].user = (void*)0;
        } else {
            //Odd transfers are data
            trans[x].length = 8 * 4;
            trans[x].user = (void*)1;
        }
        trans[x].flags = SPI_TRANS_USE_TXDATA;
    }
    trans[0].tx_data[0] = 0x2A;         //Column Address Set    设置列地址
    trans[1].tx_data[0] = 0;            //Start Col High        起始列高位
    trans[1].tx_data[1] = 0;            //Start Col Low         起始列低位
    trans[1].tx_data[2] = (320 - 1) >> 8;   //End Col High      结束列高位
    trans[1].tx_data[3] = (320 - 1) & 0xff; //End Col Low       结束列低位
    trans[2].tx_data[0] = 0x2B;         //Page address set      设置页地址
    trans[3].tx_data[0] = ypos >> 8;    //Start page high       起始页高位
    trans[3].tx_data[1] = ypos & 0xff;  //start page low        起始页低位
    trans[3].tx_data[2] = (ypos + PARALLEL_LINES - 1) >> 8; //end page high  结束页高位
    trans[3].tx_data[3] = (ypos + PARALLEL_LINES - 1) & 0xff; //end page low 结束页低位
    trans[4].tx_data[0] = 0x2C;         //memory write          储存器写入
    trans[5].tx_buffer = linedata;      //finally send the line data 最后发送列数据
    trans[5].length = 320 * 2 * 8 * PARALLEL_LINES;  //Data length, in bits 数据长度，多少位
    trans[5].flags = 0; //undo SPI_TRANS_USE_TXDATA flag        撤销发送标志

    //Queue all transactions.
    for (x = 0; x < 6; x++) {
        ret = spi_device_queue_trans(spi, &trans[x], portMAX_DELAY);
        assert(ret == ESP_OK);
    }

    //When we are here, the SPI driver is busy (in the background) getting the transactions sent. That happens
    //mostly using DMA, so the CPU doesn't have much to do here. We're not going to wait for the transaction to
    //finish because we may as well spend the time calculating the next line. When that is done, we can call
    //send_line_finish, which will wait for the transfers to be done and check their status.
}

static void send_line_finish(spi_device_handle_t spi)
{
    spi_transaction_t *rtrans;
    esp_err_t ret;
    //Wait for all 6 transactions to be done and get back the results.
    for (int x = 0; x < 6; x++) {
        ret = spi_device_get_trans_result(spi, &rtrans, portMAX_DELAY);
        assert(ret == ESP_OK);
        //We could inspect rtrans now if we received any info back. The LCD is treated as write-only, though.
    }
}
```

## 总结

本例进行了spi主机的配置和数据发送，主要对SPI协议进行熟悉，了解具体发送配置，包括轮询发送和中断传输。具体使用还是比较简单的，但针对不同的驱动和设备，需要进行单独的驱动编写，即采用SPI发送适合驱动的数据以配合具体模块或芯片。
