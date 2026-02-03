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


* 对于外部函数无法访问私有变量，可以在类内采用`friend`定义友元函数（友元不是成员，需要传递对象指针）
* 或者定义一个成员函数进行任务操作，把实际任务函数中`while`循环中的操作写在该函数中

```cpp
class APP_MPU6050{
    public:
    friend void mpu6050_task_function(void *arg);
};
void mpu6050_task_function(void *arg){
    APP_MPU6050* instance = static_cast<APP_MPU6050*>(arg);
    while(1){}
}
```

* 在抽象类和派生类之间共用资源时，可采用静态参数和静态函数进行配置。即用静态参数进行需要共享资源的一次初始化
* `static`变量作为静态量，和静态函数
* `cpp`源文件中，进行静态成员和宏定义
* 在main函数中进行类函数调用
* 在派生类中需要对重写的基类函数进行重新声明
* `override`关键字用于“声明”函数是重写父类虚函数

```cpp
class SDcard {
    private:
        static bool s_initialized;  // 静态标志位，标记是否已初始化
        static sdmmc_card_t* s_card;  // 静态卡句柄，所有实例共享
        static sdmmc_host_t s_host;   // 静态主机配置，所有实例共享
        
        const char* mount_point ;   // 文件路径字符串
        const char* TAG;            // 日志提示模组名
        bool isReady;               // 初始化状态
        sdmmc_card_t* card;         // 驱动句柄
        sdmmc_host_t host;          // 主机配置
        uint16_t file_number;       // 文件序号
        files_t files;              // 文件句柄
    public:
        SDcard(const char* tag); // 构造函数
        virtual ~SDcard();      // 虚析构函数

        // 基础SD卡操作
        static void app_file_init_once();  // 修改为静态方法，只初始化一次
        virtual bool app_file_check_file(const char* filename, size_t* file_size = nullptr); // 检查文件是否已存在        
        virtual esp_err_t app_file_open_file(); // 新建文件并打开
        virtual esp_err_t app_file_write_file(int index,void* data);   // 根据索引写对应文件
        virtual esp_err_t app_file_save_file(int index);  // 根据索引保存文件并释放指针
        virtual bool app_file_check_module(); // 检查组件状态
        // 必须重构的纯虚函数
        virtual void app_file_create_task()= 0 ; // 创建freertos任务
        virtual void run_once() = 0 ; // 放在while循环中的实际任务
};

class IMU_Data :public SDcard{
    private:
        mpu6050_processed_data_t* imu_data_buffer;  // IMU数据缓冲区
        size_t buffer_size;                         // 缓冲区大小
        const char* imu_filename_prefix;            // 文件名前缀
    public:
        IMU_Data(const char* tag);                  // 构造函数
        ~IMU_Data();
        // 基类函数重写
        esp_err_t app_file_open_file() override;
        esp_err_t app_file_write_file(int index,void* data) override;
        // IMU数据操作
        void imu_data_init(size_t size);            // 初始化数据缓冲区
        void imu_data_get(mpu6050_processed_data_t* data); // 获取当前帧数据
        void imu_data_add_to_file();                // 将数据结构化写入文件 
};

class MIC_Data:public SDcard{
    private:
        uint8_t* mic_audio_buffer;      // 麦克风数据缓冲区
        size_t buffer_size;             // 缓冲区大小
        uint32_t sample_rate;           // 采样率
        uint8_t bit_depth;              // 位深度
        const char* mic_filename_prefix;// 麦克风文件名前缀
    public:
        MIC_Data(const char* tag);
        ~MIC_Data();

        // 麦克风数据操作
        void mic_data_init(size_t size);    // 初始化音频缓冲区
        void mic_data_get();
        void mic_data_add_to_file();        // 写入文件系统
};
```

```cpp
SDcard::app_file_init_once(); // 全局初始化
```

## 构建相关

* 如果把`app_main()`函数的文件改为了c++文件，在编译时函数会被名称修饰，导致构建项目时找不到该函数，需要进行声明
* 在整个任务流程中，如果`app_main`函数结束，日志会提示`main_task:Returned from app_main()`，导致main中的**函数变量被删除**，影响其他任务

```cpp
extern "C" void app_main(void) {
    APP_MPU6050 mpu6050("app_mpu6050"); // 类变量需放在外层，防止因main函数结束受影响
}
```




* 采用静态函数创建freertos任务函数，希望获取到类内成员，通过`this`将当前对象进行传递
* 同时采用c++推荐的`static_cast<type>`方式进行类型转换

```cpp
static void mpu6050_task_function(void *arg) {
    APP_MPU6050* instance = static_cast<APP_MPU6050*>(arg);
    while(1){
        
    }
}
xTaskCreate(mpu6050_task_function, "mpu6050 task ", 4096, this, 5,
                mpu6050_handler);
```

* `xTaskCreate`函数接受参数为`TaskFunction_t`类型任务函数，普通的类内函数在c++编译下会有this参数，无法正常使用
* 静态成员函数没有this参数，类似c函数
* 且保持了面向对象的封装

```cpp
        // 静态任务入口
        static void collection_task_entry(void* arg);
        static void write_task_entry(void* arg);
```

```cpp
/**
 * @brief 静态任务入口 - 文件写入任务
 */
void IMU_Data::write_task_entry(void* arg) {
  IMU_Data* instance = static_cast<IMU_Data*>(arg);
  instance->write_task_loop();
}

/**
 * @brief 创建IMU数据处理任务（双任务版本）
 * @note 创建两个任务：
 *       - collection_task: 高优先级(5)，专门快速采集数据
 *       - write_task: 低优先级(2)，专门处理慢速SD写入
 */
void IMU_Data::app_file_create_task() {
  // 启动数据采集任务（高优先级）
  if (s_collection_task == nullptr) {
    xTaskCreate(collection_task_entry, "imu_collect", 4096, this, 5, &s_collection_task);
    ESP_LOGI(TAG, "IMU collection task created successfully (priority 5)");
  } else {
    ESP_LOGW(TAG, "IMU collection task already exists");
  }
}
```