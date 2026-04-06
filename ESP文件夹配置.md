# ESP文件夹配置

## EIM进行版本管理

***确保配置Python环境变量***

提前配置 `eim_config.toml` 文件，在eim中选择**加载配置**（如果你确定你的esp环境具体安装路径，并确认.espressif文件夹为空），然后删除原来的非必要文件

***不推荐使用配置导入，还是老实用专家模式选择文件夹吧***

```toml
# EIM 的“根路径”，通常是 .espressif 目录，用作默认基准路径
path = "C:/ESP/Framework/.espressif"

# ESP-IDF 源码所在路径；如果这里指向一个已有的 ESP-IDF Git 仓库，
# EIM 会使用该仓库而不是重新克隆，并忽略 idf_versions 的版本选择
idf_path = "C:/ESP/Framework/v5.5/esp-idf-v5.5"

# 保存 eim_idf.json 的路径（相对或绝对），用于记录安装的 ESP-IDF 版本信息
esp_idf_json_path = "C:/ESP/Framework/.espressif/tools"

# 工具下载缓存目录（dist），存放下载好的压缩包等
tool_download_folder_name = "C:/ESP/Framework/.espressif/dist"

# 工具实际安装目录（tools），解压后的编译器、OpenOCD 等会放在这里
tool_install_folder_name = "C:/ESP/Framework/.espressif/tools"

# Python 虚拟环境目录名，EIM 会在 path 下创建该目录存放 Python 环境
python_env_folder_name = "C:/ESP/Framework/python_env"

# 目标芯片列表，"all" 表示为所有支持的目标安装工具
target = ["all"]

# 要安装的 ESP-IDF 版本列表；当 idf_path 指向已有仓库时，这个字段会被忽略
idf_versions = ["v5.5"]

# tools.json 文件相对 ESP-IDF 安装目录的路径，用于描述需要哪些工具及版本
tools_json_file = "tools/tools.json"

# 导出配置文件时的保存文件名（相对路径），用于以后复用当前安装配置
config_file_save_path = "eim_v5.5_config.toml"

# 是否以非交互模式运行；true 表示完全静默/脚本化安装
non_interactive = false

# 在向导模式下是否显示所有问题；false 表示只问必要问题（此处在非交互模式下通常无效）
wizard_all_questions = false

# 工具下载镜像地址，替代默认的 github.com；这里使用官方默认 GitHub 源
mirror = "https://dl.espressif.com/github_assets"

# ESP-IDF 仓库镜像地址，替代默认的 github.com；这里同样使用 GitHub
idf_mirror = "https://gitee.com/EspressifSystems/esp-idf.git"

# PyPI 镜像地址，替代默认的 https://pypi.org/simple，用于下载 Python 依赖
pypi_mirror = "https://pypi.tuna.tsinghua.edu.cn/simple"

# 克隆 ESP-IDF 仓库时是否递归子模块；true 表示会拉取所有子模块
recurse_submodules = true

# 是否尝试自动安装所有缺失的系统依赖（仅 Windows 支持自动安装）
install_all_prerequisites = true

# 是否跳过前置条件检查；false 表示先检查依赖是否满足（官方不建议随意跳过）
skip_prerequisites_check = false

# 需要额外安装的 ESP-IDF “特性”集合，例如：
# ci: CI 工具依赖；docs: 文档构建工具依赖
idf_features = ["ci", "docs"]
```

在经历较长时间的折腾后，笔者认为，放过自己，知道配置文件和工具文件安装在哪里就可以了，不然你的系统里会多出很多奇怪的文件夹

## 文件夹整理（旧版）

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

***在更新后idf的vscode环境不支持路径配置，而是使用eim统一管理***

## 重新配置ESP环境（旧版）

* 在顶部搜索栏中敲入`> esp-idf config`选择**配置ESP-IDf扩展**
* 根据博客选择`USE EXISTING SETUP` 选择已有的设置，然后选择现有的文件夹，等待下载
* 新版本也可以这么操作，可以选择**EXPRESS**直接安装新版本(*如果有更新的话*)，或者从github直接克隆到本地文件夹，再用**USE EXISTING SETUP**的方式安装工具文件

## WSL中配置IDF路径（旧版）

笔者发现再vscode远程连接wsl后，有关esp-idf的插件配置就不正常，即无法找到已经安装的idf路径！
解决方案如下：

在`.vscode/settings.json`中粘贴如下配置，具体路径自行替换

```json
{

  "idf.idfPath": "/home/sky/esp/framework/v5.5/esp-idf-v5.5",
  "idf.pythonBinPath": "/home/sky/esp/framework/v5.5/.espressif/python_env/idf5.5_py3.10_env/bin/python",
  "idf.toolsPath": "/home/sky/esp/framework/v5.5/.espressif",
  "clangd.path": "/home/sky/esp/framework/v5.5/.espressif/tools/esp-clang/esp-19.1.2_20250312/esp-clang/bin/clangd",
}
```
