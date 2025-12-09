# ESP文件夹配置

## 文件夹整理

如题，笔者在ESP学习实验过程中正好碰到了版本更新，由此去查看了[版本简介](https://docs.espressif.com/projects/esp-idf/zh_CN/v5.4.2/esp32/versions.html)决定重新整理ESP文件夹，结构如下

```bash
 ESP/
 - Project/
    - ESP_Example_test/
    - ESP32-CAM
    - ESP32-MP3
   ...
 - Framework/
    - v5.4.1/
        - .espressidf/
        - esp-idfv5.4.1/
    - v5.5/
        - .espressidf/
        - esp-idfv5.5/
    ...
 - Document/
    - xxx.pdf
    ...
```

## vscode配置 

吹爆VScode，自定义配置真爽

[参考文章](https://blog.csdn.net/qq_61585528/article/details/139158561)

由于笔者电脑上的vscode承担了很多功能，所以扩展很多，笔者参考文章配置了多种文件，用于不同的开发或者记录。
同理，对于各版本的ESP-IDF，单独配置文件，给其不同的工具路径，用以隔离和切换。

## 重新配置ESP环境

* 在顶部搜索栏中敲入`> esp-idf config`选择**配置ESP-IDf扩展**
* 根据博客选择`USE EXISTING SETUP` 选择已有的设置，然后选择现有的文件夹，等待下载
* 新版本也可以这么操作，可以选择**EXPRESS**直接安装新版本(*如果有更新的话*)，或者从github直接克隆到本地文件夹，再用**USE EXISTING SETUP**的方式安装工具文件