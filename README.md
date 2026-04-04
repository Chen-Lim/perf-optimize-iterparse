# Apple Health Pro 🍎
## A Studio-Grade Data Engine for Apple Health Export.

### Apple Health Pro 是一款专为数据分析师、健康极客和开发者设计的跨平台桌面工具。它能够高效解析 Apple 健康导出的巨型 XML 压缩包，并将其转化为结构清晰、开箱即用的专业级 CSV 数据集。

# ✨核心功能
#### ⚡️高性能流式解析： 采用 iterparse 迭代技术，能够处理数 GB 级别的 export.xml 文件而不占用过多内存，告别程序崩溃。

#### 🔍多维度来源过滤： 自动识别所有数据来源（Apple Watch、iPhone、各种第三方 App），支持用户按需勾选，精准提取目标数据。

#### 📦智能数据分类： 自动将凌乱的健康记录归纳为 7 大标准维度：

1、心脏指标 (Heart Metrics)

2、身体成分 (Body Composition)

3、活动与能量 (Activity & Energy)

4、睡眠分析 (Sleep Analysis)

5、步态与机能 (Mobility & Gait)

6、生殖健康 (Reproductive Health)

7、生命体征与呼吸 (Vitals & Respiratory)

#### 📊自动分块导出： 针对超大型数据集如分钟级心率，系统会自动按 80 万行/文件进行分块处理，确保 Excel 和各类 AI 应用能流畅打开。

# 🚀操作指南
#### 1. 准备数据
在 iPhone 上打开 “健康” App -> 点击右上角头像 -> 选择 “导出所有健康数据”。导出完成后，你会获得一个名为 导出.zip 的文件。

#### 2. 加载与索引
打开 Apple Health Pro，点击底部的按钮 SELECT DATA ARCHIVE (.ZIP)。程序将自动扫描压缩包并索引所有数据源。

#### 3. 选择来源
在 IDENTIFIED SOURCES 列表中，勾选你想要提取的数据来源（例如仅选择 Apple Watch 的记录以排除手机重复计算）。默认状态为全不选。

#### 4. 执行导出
点击 EXECUTE EXPORT。CSV 文件将自动生成在 export.zip 所在的同级目录下，并按分类命名。

# 📥下载与安装
无需配置 Python 环境，请直接前往 Releases 页面下载对应系统的安装程序。

Windows 用户： 下载 HealthPro_Setup_v8.2.0.exe 运行安装。

macOS 用户： 下载 HealthPro_v8.2.0.dmg，将应用拖入 Applications 文件夹。

注意：由于未签名，初次运行请右键点击应用图标并选择“打开”。

# 🛠开发与构建 (For Developers)
如果你希望在本地运行或自行修改代码：

Bash

#### 克隆项目
git clone https://github.com/leecdiang/Apple-Health-Pro.git

#### 安装依赖
pip install -r requirements.txt

#### 运行程序
python health_app.py

# ⚖️ 许可说明 (License)
本项目采用 MIT License 授权。
© 2026 LEEcDiang. All rights reserved.
