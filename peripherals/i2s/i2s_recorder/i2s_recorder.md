# I2S Digital Microphone Recording I2S数字麦克风录音

## 粗略阅读README文档

文件简介本例使用PDM数据格式在I2S外围设备记录数字麦克风捕获的音频文件并存入SD卡

硬件需要，配置项目，构建烧录和示例输出

## 构建项目

* 选择版本
* 选择目标芯片
* 点击**构建项目**

## 代码分析

### Kconfig配置展示

```kconfig
menu "Example Configuration"

    menu "SDCard Configuration"

        config EXAMPLE_SPI_MISO_GPIO
            int "SPI MISO GPIO"
            default 15
            help
                Set the GPIO number used for MISO from SPI.

        config EXAMPLE_SPI_MOSI_GPIO
            int "SPI MOSI GPIO"
            default 14
            help
                Set the GPIO number used for MOSI from SPI.

        config EXAMPLE_SPI_SCLK_GPIO
            int "SPI SCLK GPIO"
            default 18
            help
                Set the GPIO number used for SCLK from SPI.

        config EXAMPLE_SPI_CS_GPIO
            int "SPI CS GPIO"
            default 19
            help
                Set the GPIO number used for CS from SPI.

    endmenu

    menu "I2S MEMS MIC Configuration"

        config EXAMPLE_SAMPLE_RATE
            int "Audio Sample Rate"
            default 44100 if SOC_I2S_SUPPORTS_PDM2PCM
            default 5644800
            help
                Set the audio sample rate frequency. Usually 16000 or 44100 Hz if PCM data format supported.
                Oversample rate usually can be 2048000 or 5644800 Hz if only raw PDM data format supported.

        config EXAMPLE_BIT_SAMPLE
            int "Audio Bit Sample"
            default 16
            help
                Define the number of bits for each sample. Default 16 bits per sample.

        config EXAMPLE_I2S_DATA_GPIO
            int "I2S Data GPIO"
            default 5
            help
                Set the GPIO number used for transmitting/receiving data from I2S.

        config EXAMPLE_I2S_CLK_GPIO
            int "I2S Clock GPIO"
            default 4
            help
                Set the GPIO number used for the clock line from I2S.

    endmenu

    config EXAMPLE_REC_TIME
        int "Example Recording Time in Seconds"
        default 15
        help
            Set the time for recording audio in seconds.

endmenu
```

### 宏定义和全局变量

* `SPI_DMA_CH_AUTO`自动分配SPI的DMA通道
* `NUM_CHANNELS`定义音频录制声道数为1
* `SD_MOUNT_POINT`定义SD卡挂载点路径为`"/sdcard"`，后续所有文件操作都基于此路径
* `SAMPLE_SIZE`定义每次从I2S读取数据缓存区大小，（样本*bit*数x1k）
* 每秒传输字节数 采样率x样本*字节*数x声道数

* `sdmmc_host_t`类型配置调用`SDSPI_HOST_DEFAULT`宏函数进行初始化，SDMMC 主机驱动会尝试以当前卡所支持的最大总线宽度进行通信（SD 卡为 4 线，eMMC 为 8 线），并使用 20 MHz 的通信频率
* `WAVE_HEADER_SIZE`设置wav格式头大小为44字节
* 其他为全局声明，调用时修改

```c
#define SPI_DMA_CHAN        SPI_DMA_CH_AUTO
#define NUM_CHANNELS        (1) // For mono recording only!
#define SD_MOUNT_POINT      "/sdcard"
#define SAMPLE_SIZE         (CONFIG_EXAMPLE_BIT_SAMPLE * 1024)
#define BYTE_RATE           (CONFIG_EXAMPLE_SAMPLE_RATE * (CONFIG_EXAMPLE_BIT_SAMPLE / 8)) * NUM_CHANNELS

// When testing SD and SPI modes, keep in mind that once the card has been
// initialized in SPI mode, it can not be reinitialized in SD mode without
// toggling power to the card.
sdmmc_host_t host = SDSPI_HOST_DEFAULT();
sdmmc_card_t *card;
i2s_chan_handle_t rx_handle = NULL;

static int16_t i2s_readraw_buff[SAMPLE_SIZE];
size_t bytes_read;
const int WAVE_HEADER_SIZE = 44;
```

### app_main函数

main函数调用自定义函数进行配置和操作，在录音完成后关闭I2S通道并释放资源

```c
void app_main(void)
{
    printf("PDM microphone recording example start\n--------------------------------------\n");
    // Mount the SDCard for recording the audio file
    mount_sdcard();
    // Acquire a I2S PDM channel for the PDM digital microphone
    init_microphone();
    ESP_LOGI(TAG, "Starting recording for %d seconds!", CONFIG_EXAMPLE_REC_TIME);
    // Start Recording
    record_wav(CONFIG_EXAMPLE_REC_TIME);
    // Stop I2S driver and destroy
    ESP_ERROR_CHECK(i2s_channel_disable(rx_handle));
    ESP_ERROR_CHECK(i2s_del_channel(rx_handle));
}
```

### 安装SDcard

1. `esp_vfs_fat_sdmmc_mount_config_t`被映射到`esp_vfs_fat_mount_config_t`
   * `format_if_mount_failed` 如果FAT分区不能挂载，且此参数为true，创建分区表并格式化文件系统
   * `max_files` 最大可打开文件数
   * `allocation_unit_size` 和前参数配合，设定格式化时的大小
2. `spi_bus_config_t` SPI总线结构体配置，设定引脚和最大传输大小
3. `spi_bus_initialize`参数为SPI主机ID，配置结构体，DMA通道
4. `SDSPI_DEVICE_CONFIG_DEFAULT`宏函数将初始化没有卡检测和写保护的卡槽，如有，后续单独配置
5. `esp_vfs_fat_sdspi_mount` 参考[官方文档](https://docs.espressif.com/projects/esp-idf/zh_CN/stable/esp32/api-reference/storage/fatfs.html#fatfs-vfs)，是官方提供的便捷函数，进行初始化
6. `sdmmc_card_print_info`打印关于card的信息 `stdout` 框架定义的流（直接抄吧），`card`全局定义的`sdmmc_card_t*`类型变量

```c
void mount_sdcard(void)
{
    esp_err_t ret;
    // Options for mounting the filesystem.
    // If format_if_mount_failed is set to true, SD card will be partitioned and
    // formatted in case when mounting fails.
    esp_vfs_fat_sdmmc_mount_config_t mount_config = {
        .format_if_mount_failed = true,
        .max_files = 5,
        .allocation_unit_size = 8 * 1024
    };
    ESP_LOGI(TAG, "Initializing SD card");

    spi_bus_config_t bus_cfg = {
        .mosi_io_num = CONFIG_EXAMPLE_SPI_MOSI_GPIO,
        .miso_io_num = CONFIG_EXAMPLE_SPI_MISO_GPIO,
        .sclk_io_num = CONFIG_EXAMPLE_SPI_SCLK_GPIO,
        .quadwp_io_num = -1,
        .quadhd_io_num = -1,
        .max_transfer_sz = 4000,
    };
    ret = spi_bus_initialize(host.slot, &bus_cfg, SPI_DMA_CHAN);
    if (ret != ESP_OK) {
        ESP_LOGE(TAG, "Failed to initialize bus.");
        return;
    }

    // This initializes the slot without card detect (CD) and write protect (WP) signals.
    // Modify slot_config.gpio_cd and slot_config.gpio_wp if your board has these signals.
    sdspi_device_config_t slot_config = SDSPI_DEVICE_CONFIG_DEFAULT();
    slot_config.gpio_cs = CONFIG_EXAMPLE_SPI_CS_GPIO;
    slot_config.host_id = host.slot;

    ret = esp_vfs_fat_sdspi_mount(SD_MOUNT_POINT, &host, &slot_config, &mount_config, &card);

    if (ret != ESP_OK) {
        if (ret == ESP_FAIL) {
            ESP_LOGE(TAG, "Failed to mount filesystem.");
        } else {
            ESP_LOGE(TAG, "Failed to initialize the card (%s). "
                     "Make sure SD card lines have pull-up resistors in place.", esp_err_to_name(ret));
        }
        return;
    }

    // Card has been initialized, print its properties
    sdmmc_card_print_info(stdout, card);
}
```

### 初始化麦克风

1. `I2S_CHANNEL_DEFAULT_CONFIG`宏函数进行默认I2S通道配置，参数为端口ID和角色（主/从）
2. `i2s_new_channel`传入配置创建新通道，只启用接收
3. 初始化为PDM RX模式
   1. 宏函数进行默认配置，传入每秒传入字节数（传输速率）
   2. 根据是否支持PDM和PCM转换决定配置转换或原始PDM格式
   3. 设置引脚，时钟脚和数据脚，不反转时钟输出
4. `i2s_channel_init_pdm_rx_mode`将通道初始化为特定模式，此处为PDM格式接收模式
5. `i2s_channel_enable`使能通道

```c
void init_microphone(void)
{
#if SOC_I2S_SUPPORTS_PDM2PCM
    ESP_LOGI(TAG, "Receive PDM microphone data in PCM format");
#else
    ESP_LOGI(TAG, "Receive PDM microphone data in raw PDM format");
#endif  // SOC_I2S_SUPPORTS_PDM2PCM
    i2s_chan_config_t chan_cfg = I2S_CHANNEL_DEFAULT_CONFIG(I2S_NUM_AUTO, I2S_ROLE_MASTER);
    ESP_ERROR_CHECK(i2s_new_channel(&chan_cfg, NULL, &rx_handle));

    i2s_pdm_rx_config_t pdm_rx_cfg = {
        .clk_cfg = I2S_PDM_RX_CLK_DEFAULT_CONFIG(CONFIG_EXAMPLE_SAMPLE_RATE),
        /* The default mono slot is the left slot (whose 'select pin' of the PDM microphone is pulled down) */
#if SOC_I2S_SUPPORTS_PDM2PCM
        .slot_cfg = I2S_PDM_RX_SLOT_PCM_FMT_DEFAULT_CONFIG(I2S_DATA_BIT_WIDTH_16BIT, I2S_SLOT_MODE_MONO),
#else
        .slot_cfg = I2S_PDM_RX_SLOT_RAW_FMT_DEFAULT_CONFIG(I2S_DATA_BIT_WIDTH_16BIT, I2S_SLOT_MODE_MONO),
#endif
        .gpio_cfg = {
            .clk = CONFIG_EXAMPLE_I2S_CLK_GPIO,
            .din = CONFIG_EXAMPLE_I2S_DATA_GPIO,
            .invert_flags = {
                .clk_inv = false,
            },
        },
    };
    ESP_ERROR_CHECK(i2s_channel_init_pdm_rx_mode(rx_handle, &pdm_rx_cfg));
    ESP_ERROR_CHECK(i2s_channel_enable(rx_handle));
}
```

### 录音数据

1. 传入参数，录音秒数
2. `BYTE_RATE * rec_time`将秒转换成字节数
3. `WAV_HEADER_PCM_DEFAULT`调用宏函数生成wav文件头，文件大小为44B+计算所得字节数
4. `stat`检测文件是否存在，存在则用`unlink`删除文件
5. 打开文件，并赋值给指针（由于原可能的文件已删除，a模式追加可以被看做新建）
6. `fwrite`把文件头写入文件中
7. 循环写入数据。每次读取I2S的rx通道，向文件中写入数据
8. 直到数据大小满足计算字节，结束循环
9. `fclose`关闭文件链接
10. `esp_vfs_fat_sdcard_unmount`卸载fat分区
11. 释放SPI总线资源（用于SD卡）

```c
void record_wav(uint32_t rec_time)
{
    // Use POSIX and C standard library functions to work with files.
    int flash_wr_size = 0;
    ESP_LOGI(TAG, "Opening file");

    uint32_t flash_rec_time = BYTE_RATE * rec_time;
    const wav_header_t wav_header =
        WAV_HEADER_PCM_DEFAULT(flash_rec_time, 16, CONFIG_EXAMPLE_SAMPLE_RATE, 1);

    // First check if file exists before creating a new file.
    struct stat st;
    if (stat(SD_MOUNT_POINT"/record.wav", &st) == 0) {
        // Delete it if it exists
        unlink(SD_MOUNT_POINT"/record.wav");
    }

    // Create new WAV file
    FILE *f = fopen(SD_MOUNT_POINT"/record.wav", "a");
    if (f == NULL) {
        ESP_LOGE(TAG, "Failed to open file for writing");
        return;
    }

    // Write the header to the WAV file
    fwrite(&wav_header, sizeof(wav_header), 1, f);

    // Start recording
    while (flash_wr_size < flash_rec_time) {
        // Read the RAW samples from the microphone
        if (i2s_channel_read(rx_handle, (char *)i2s_readraw_buff, SAMPLE_SIZE, &bytes_read, 1000) == ESP_OK) {
            printf("[0] %d [1] %d [2] %d [3]%d ...\n", i2s_readraw_buff[0], i2s_readraw_buff[1], i2s_readraw_buff[2], i2s_readraw_buff[3]);
            // Write the samples to the WAV file
            fwrite(i2s_readraw_buff, bytes_read, 1, f);
            flash_wr_size += bytes_read;
        } else {
            printf("Read Failed!\n");
        }
    }

    ESP_LOGI(TAG, "Recording done!");
    fclose(f);
    ESP_LOGI(TAG, "File written on SDCard");

    // All done, unmount partition and disable SPI peripheral
    esp_vfs_fat_sdcard_unmount(SD_MOUNT_POINT, card);
    ESP_LOGI(TAG, "Card unmounted");
    // Deinitialize the bus after all devices are removed
    spi_bus_free(host.slot);
}
```

## 总结

本例程综合展示了SD卡的初始化配置，文件写入，I2S麦克风读取相互配合实现的录音并储存的效果。
