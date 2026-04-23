# 基于 Spark 的医美数据分析系统

## 项目简介

本系统对医美项目销售数据进行**采集与预处理**、**存储**、**聚类与趋势分析**及**可视化展示**，为医美平台提供数据支撑。数据来源为包含平台、城市、机构、项目品类、价格、销量、评价等维度的 CSV 表格。

采用 **数据仓库分层思想**（ODS → DWD → DWS → ADS）：以 K-Means 聚类结果为 DWD+ 层数据产物，再派生 **8 张主题宽表** 形成 DWS 层，使用 Spark SQL 的 CTE / 窗口函数 / 多表 JOIN / 多条件 CASE WHEN 等构造复杂业务语义分析，最终聚合成 JSON 供看板读取。

> 本次迭代的完整改动记录见 [`CHANGELOG.md`](./CHANGELOG.md)（含列名中英对照、8 张聚类派生表的 Spark SQL 技术点、前后端全部改动点）。

## 功能模块

| 模块 | 说明 |
|------|------|
| 数据采集与预处理 | 使用 **pandas** 读取 CSV（UTF-8 英文列名，兼容 GBK 自动回退），去重、缺失值填充、异常值处理（IQR），日期异常（如 `########`）置空；由采集日期派生 `stats_month`；输出 Parquet |
| 数据存储 | 对 Parquet/HDFS 数据做一致性检查并输出元信息（行数、列数、缺失列、可用月份分布等） |
| 数据分析 | **Spark** 读取 Parquet：① K-Means 聚类（VectorAssembler + StandardScaler + KMeans，K=3）；② 多维 Spark SQL 趋势统计；③ **基于聚类结果派生 8 张主题宽表**（`cluster_profile` / `cluster_city_rank` / `cluster_category_preference` / `cluster_channel_efficiency` / `cluster_price_band` / `cluster_top_projects` / `cluster_doctor_matching` / `cluster_month_delta`） |
| 数据可视化 | Flask Web **系统**（**注册/登录/个人中心/头像上传**），左侧导航 9 个页面：数据概览、地区分析、销售趋势、地区偏好、品类分析、价格与折扣、渠道分析、聚类分析、**聚类深度分析**（画像卡片 + 热力图 + 堆叠柱 + 月度环比折线）、关于系统；ECharts + JSON 快照 |

## 环境要求

- Python 3.8+
- JDK 8 或 11（Spark 依赖）
- 建议内存 4GB 以上

### Windows 下 HADOOP_HOME / winutils（数据分析写 Parquet 时需要）

1. **下载 winutils**  
   从 [winutils 镜像](https://github.com/cdarlint/winutils) 下载与 Hadoop 版本匹配的 `winutils.exe` 和 `hadoop.dll`（例如 hadoop-3.0.2 对应 `hadoop-3.0.2/bin`）。
2. **设置 HADOOP_HOME 与 Path**  
   将上述文件放到某目录的 `bin` 下，例如 `D:\ProgramFiles\hadoop\bin`，然后：
   - 新建环境变量 **HADOOP_HOME** = `D:\ProgramFiles\hadoop`（不要带 `\bin`）。
   - 建议将 **%HADOOP_HOME%\bin** 加入系统 **Path**，否则可能出现 `UnsatisfiedLinkError: NativeIO$Windows.access0`。
3. **自动检测**  
   若未设置 HADOOP_HOME，程序会尝试 `D:\ProgramFiles\hadoop`、`D:\hadoop` 等路径，仅当存在 `bin\winutils.exe` 时才使用。

## 安装与运行

### 1. 安装依赖

```bash
pip install -r requirements.txt
```

### Docker 启动 HDFS

如果本机没有 `hdfs` 命令，可用 Docker 启动最小 HDFS 集群：

```bash
cd project
docker compose -f docker-compose.hdfs.yml up -d
```

- NameNode 管理页：`http://127.0.0.1:9870`
- HDFS RPC 地址：`127.0.0.1:9000`

通过容器内命令上传 CSV（无需本机安装 Hadoop）：

```bash
cd "F:\... 已完成\A71 基于Spark的医美数据分析系统的设计与实现\project"

# 1) 复制到容器时用英文名
docker cp ".\data\medical_beauty_data.csv" hdfs-namenode:/tmp/mb.csv

# 2) 先确认容器里文件存在
docker exec hdfs-namenode ls -l /tmp/mb.csv

# 3) 上传到 HDFS
docker exec hdfs-namenode hdfs dfs -mkdir -p /user/medical_beauty/raw
docker exec hdfs-namenode hdfs dfs -put -f /tmp/mb.csv /user/medical_beauty/raw/medical_beauty_data.csv

# 4) 验证
docker exec hdfs-namenode hdfs dfs -ls /user/medical_beauty/raw
```

### 2. 放置数据

将医美销售数据 CSV（**UTF-8 编码，英文列名**）放在项目下：

```
project/data/medical_beauty_data.csv
```

CSV 前 28 列（列顺序固定，不依赖列头精确匹配，但建议使用如下英文列名，避免 HDFS / Docker 等环境的编码问题）：

| # | 列名 | 含义 |
|---|---|---|
| 1 | platform | 平台 |
| 2 | city | 城市 |
| 3 | district | 商圈/区县 |
| 4 | store_type | 门店类型 |
| 5 | institution | 机构名称 |
| 6 | category | 项目品类 |
| 7 | subcategory | 项目子类 |
| 8 | spec | 规格 |
| 9 | project_name | 项目名称 |
| 10 | list_price | 标价(元) |
| 11 | actual_price | 到手价(元) |
| 12 | discount_rate | 折扣率 |
| 13 | monthly_sales | 月销量 |
| 14 | review_count | 评价数 |
| 15 | rating | 评分 |
| 16 | doctor_name | 医生姓名 |
| 17 | doctor_title | 医生头衔 |
| 18 | payment_method | 支付方式 |
| 19 | installment | 是否分期 |
| 20 | channel | 渠道 |
| 21 | activity_tag | 活动标签 |
| 22 | approval_no | 批准文号 |
| 23 | device_brand | 器械品牌 |
| 24 | device_model | 型号/机型 |
| 25 | udi_match | UDI匹配 |
| 26 | listing_date | 上架日期 |
| 27 | collection_date | 采集日期 |
| 28 | new_customer | 新客标识 |

**说明**：预处理会按位置取前 28 列，赋予上表英文列名；列内的中文值（城市、机构名、项目品类等）保持原样。如需从老版 GBK CSV（中文列头）迁移，可执行：

```bash
python -c "import csv; \
enc = next((e for e in ['gbk','gb18030','utf-8'] if (lambda: open('data/中国医美消费数据.csv', encoding=e).read(4096))()), None); \
print('src encoding:', enc)"
```

项目已自带一份转换好的 `data/medical_beauty_data.csv`。

#### HDFS 存储模式

默认从本地 `data/*.csv` 读入、Parquet 写 `output/`。若希望**原始 CSV、清洗 Parquet、聚类 Parquet 均放在 HDFS**，请在运行前设置环境变量（Windows 示例）：

```bash
# PowerShell（当前会话生效）
$env:USE_HDFS="1"
$env:HDFS_NAMENODE_HOST="你的NameNode主机名"
$env:HDFS_NAMENODE_PORT="9000"
$env:HDFS_USER="hdfs"
$env:HDFS_WORKDIR="user/medical_beauty"

# CMD（当前窗口生效）
set USE_HDFS=1
set HDFS_NAMENODE_HOST=你的NameNode主机名
set HDFS_NAMENODE_PORT=9000
set HDFS_USER=hdfs
set HDFS_WORKDIR=user/medical_beauty
```

并先将 CSV 上传到 HDFS（若使用 Docker，可直接执行上文容器内命令）。路径需与默认 `HDFS_WORKDIR` + `raw/medical_beauty_data.csv` 一致，或通过 `HDFS_RAW_REL` 等覆盖（见 `config.py`）。

**说明**：`trend_stats.json`、`cluster_centers.json` 写入本地 `project/output/`，`dashboard_users.json` 写入 `project/database/` 供 Flask 看板读取；中间大数据链路走 HDFS。需本机可访问 HDFS（`HADOOP_HOME`、`core-site.xml` 等）且已安装 **pyarrow**。

### 3. 一键运行全流程

在 `project` 目录下执行：

```bash
cd project

# 1) 复制到容器时用英文名
docker cp ".\data\medical_beauty_data.csv" hdfs-namenode:/tmp/mb.csv

# 2) 先确认容器里文件存在
docker exec hdfs-namenode ls -l /tmp/mb.csv

# 3) 上传到 HDFS
docker exec hdfs-namenode hdfs dfs -mkdir -p /user/medical_beauty/raw
docker exec hdfs-namenode hdfs dfs -put -f /tmp/mb.csv /user/medical_beauty/raw/medical_beauty_data.csv

# 4) 验证
docker exec hdfs-namenode hdfs dfs -ls /user/medical_beauty/raw

python run_all.py
```

将依次执行 4 步：

1. **数据采集与预处理** —— pandas 清洗 CSV → `output/clean_data.parquet`
2. **存储检查** —— Parquet/HDFS 一致性校验（列齐全 / 月份分布）
3. **K-Means 聚类 + Spark SQL 趋势统计** —— 生成 `output/clustered_data.parquet`、`output/cluster_centers.json`、`output/trend_stats.json`
4. **聚类数据仓库派生** —— 基于聚类结果生成 8 张主题宽表到 `output/cluster_tables/*.parquet`，聚合到 `output/cluster_insights.json`

若 CSV 为 GBK 编码，预处理会自动尝试 UTF-8/GBK/GB18030 解码。

### 4. 启动可视化看板

```bash
cd project
python -m src.visualize
```

浏览器访问：**http://127.0.0.1:5000/** → **登录**；无账号可先 **注册**（`/register`），密码哈希存入 `database/dashboard_users.json`。

- **个人中心**（`/profile`）：修改**昵称**、**邮箱**、**上传头像**（PNG/JPG/GIF/WEBP，≤2MB）；**注册用户**可在此**修改密码**，内置管理员密码仍在 `config` / 环境变量中配置。

- **默认管理员**：用户名 `admin`，密码 `admin123`（`config.py` 或环境变量 `DASHBOARD_USER` / `DASHBOARD_PASS`）。保留名 `admin`、`root` 及管理员用户名不可被注册占用。
- **看板功能**：左侧切换「数据概览 / 地区分析 / 销售趋势 / 地区偏好 / 品类分析 / 价格与折扣 / 渠道分析 / 聚类分析 / **聚类深度分析** / 关于系统」。各分析页含图表、数据表与按钮（刷新、导出 CSV、地区页可切换 TOP15/全部）。**聚类深度分析**页展示基于聚类结果派生的主题宽表：聚类画像卡片（带业务标签 高端精品型/大众爆款型/长尾引流型/中端主力型）+ 6 张图（聚类×城市堆叠柱、品类偏好度热力图、折扣分桶堆叠柱、月度销量折线、渠道加权销量、医生头衔分布）+ 4 张数据表 + 5 个 CSV 导出。
- **API 端点**：
  - `GET /api/analytics?month=YYYY-MM` —— 主趋势统计，支持按月筛选
  - `GET /api/cluster` —— 聚类中心（标准化特征空间）
  - `GET /api/cluster_insights` —— 8 张聚类主题宽表的聚合结果 + 元信息
- **会话密钥**：生产环境请设置 `FLASK_SECRET_KEY`。
- 未登录访问 `/` 或 `/api/*` 会跳转登录或返回 401。

### 5. 单独运行各模块

- 仅预处理：`python -m src.preprocess`
- 仅存储检查（需先有 `output/clean_data.parquet` 或 HDFS 对应路径）：`python -m src.storage`
- 完整分析（聚类 + 趋势 + 聚类仓库派生）：`python -m src.analysis`
- **仅跑聚类仓库派生**（前提：`output/clustered_data.parquet` 已存在）：
  ```bash
  python -c "from src.analysis import run_cluster_warehouse; print(run_cluster_warehouse())"
  ```
- 仅启动看板：`python -m src.visualize`
- K 值选择辅助（肘部法则曲线）：`python -m src.elbow_plot`

## 配置说明

- `src/config.py`：
  - 数据路径（`RAW_CSV = data/medical_beauty_data.csv`）、Parquet 路径、JSON 用户文件路径
  - CSV 编码（`CSV_ENCODING = "utf-8"`，若源文件是 GBK 可改为 `gbk` / `gb18030` 或留空由程序自动试）
  - 聚类数 K（`KMEANS_K = 3`）
  - 看板登录账号与 `SECRET_KEY`
  - 列名常量（英文，`COL_PLATFORM` / `COL_CITY` / ...，对应 CSV 列头，与 CSV 数据文件中英对照见上表）
- HDFS 切换通过环境变量 `USE_HDFS=1` 启用

## 目录结构

```
medical-beauty-spark-analysis/
├── data/                        # 原始 CSV
│   └── medical_beauty_data.csv  # UTF-8，英文列名
├── doc/                         # 项目说明
│   └── 项目说明.md              # 含数据仓库分层设计 + 8 张聚类派生表对照
├── src/
│   ├── __init__.py
│   ├── config.py                # 配置、英文列名常量、HDFS 切换
│   ├── preprocess.py            # pandas 清洗 → clean_data.parquet
│   ├── storage.py               # Parquet/HDFS 一致性检查
│   ├── analysis.py              # 聚类 + 趋势统计 + 聚类仓库派生（8 张主题表）
│   ├── visualize.py             # Flask Web 服务
│   ├── dashboard_auth.py        # 看板用户注册/校验（JSON 持久化）
│   ├── hdfs_io.py               # pyarrow HadoopFileSystem 读写
│   ├── elbow_plot.py            # 肘部法则辅助脚本（手动选 K）
│   ├── templates/               # login.html、register.html、profile.html、dashboard.html
│   └── static/
│       ├── css/dashboard_app.css
│       ├── js/dashboard_app.js
│       ├── img/default_avatar.svg
│       └── uploads/avatars/     # 用户头像（运行时生成）
├── output/                      # 运行后生成
│   ├── clean_data.parquet       # 预处理产物
│   ├── clustered_data.parquet   # 带聚类标签的明细
│   ├── cluster_centers.json     # 聚类中心（标准化空间）
│   ├── trend_stats.json         # 15 段趋势聚合
│   ├── cluster_insights.json    # 8 张主题表的聚合结果
│   └── cluster_tables/          # 8 张聚类派生 Parquet
│       ├── cluster_profile.parquet
│       ├── cluster_city_rank.parquet
│       ├── cluster_category_preference.parquet
│       ├── cluster_channel_efficiency.parquet
│       ├── cluster_price_band.parquet
│       ├── cluster_top_projects.parquet
│       ├── cluster_doctor_matching.parquet
│       └── cluster_month_delta.parquet
├── database/
│   └── dashboard_users.json     # 看板用户 JSON（运行后生成）
├── run_all.py                   # 一键运行 4 步全流程
├── requirements.txt
├── docker-compose.hdfs.yml      # 最小 HDFS Docker 集群
├── CHANGELOG.md                 # 本次迭代改动记录
└── README.md
```

## 文档

- [项目说明](doc/项目说明.md) —— 数据仓库分层设计、8 张聚类主题表对照、技术栈总结
- [CHANGELOG](CHANGELOG.md) —— 本次迭代的完整改动记录

## 说明

- **聚类特征**：`list_price`、`actual_price`、`discount_rate`、`monthly_sales`、`review_count`、`rating`；K 值可在 `config.KMEANS_K` 中调整（配合 `python -m src.elbow_plot` 的肘部法则曲线选择）。
- **业务标签自动打标**：基于聚类中心均值的 `RANK() OVER` 自动判断"高端精品型 / 大众爆款型 / 长尾引流型 / 中端主力型"，见 `src/analysis.py::run_cluster_warehouse` 第 1 张表 `cluster_profile`。
- **日期处理**：上架日期列中的无效值（如 Excel 显示的 `########`）在预处理时会被置为空，不影响后续分析。
- **统计月份**：`stats_month` 由 `collection_date` 派生（`YYYY-MM`）；若源数据仅有单月，预处理会把一半行改成上月以便看板演示月份切换（`src/preprocess.py::_split_single_month_into_two_month_labels`）。
