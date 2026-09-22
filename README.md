# AR_backend

基于 Django 的增强现实（AR）视觉定位后端服务：上传场景数据集与查询图片后，通过图像检索与特征匹配估算查询相机在场景模型坐标系下的位姿。

## 项目背景 / 用途

在 AR 应用中，设备需要知道"我现在在场景中的什么位置、朝向哪里"，才能把虚拟内容正确叠加到真实画面上。本项目提供这一环节的服务端实现：

- 预先采集一个场景（如一段室内/室外空间），用三维扫描工具导出带位姿与深度的**彩色图像 + 相机内外参 + 深度图**，构建为"场景数据集"；
- 运行时前端（如手机 / AR 眼镜）上传一张**查询图片**；
- 后端在场景库中检索最相似的历史图像，对匹配像素做特征匹配，再由 2D-3D 对应关系求解相机位姿（PnP），把结果转换到目标（Unity）坐标系后返回。

项目同时承担数据集的**上传、存储、配置管理与文件分发**职责，因此除了位姿求解接口外，还包含一整套数据集/场景管理接口。

> 说明：仓库现有的顶层 `README.md` 仍是 Replit 的 Python 模板说明，与本项目业务无关。

## 功能特性

### 数据集与场景管理
- 批量上传数据集压缩包（`.zip` / `.tar` / `.tar.gz` / `.tgz`），自动解压、校验 `config.json` 与 `info.json`，并写入数据库
- 数据集重名检测；处理失败时自动回滚已解压目录
- 上传单张 / 多张图片、单个 / 多个视频，支持自定义存储子目录与重命名策略
- 场景列表查询、按 ID 读取场景配置（JSON）、更新配置（保留上一版配置作为备份）、按名称删除场景及其关联文件
- 按 ID 下载单个数据集文件，或按场景 ID 打包下载全部文件（ZIP）

### 视觉定位（核心）
- 基于 **Patch-NetVLAD** 的图像检索，从场景库中召回候选匹配图像
- 基于 **DISK + LightGlue** 的局部特征提取与匹配（另有 XFeat / ORB 的备用实现）
- 图像归一化预处理：EXIF 方向校正、灰度化、直方图均衡、高斯模糊、按目标分辨率缩放
- 深度图反投影：像素 + 深度 → 相机坐标系 3D 点 → 模型坐标系 3D 点
- **PnPRANSAC** 位姿求解，使用候选图像的已知位姿作为初值
- 多坐标系转换（数据集坐标系 / OpenCV 坐标系 / 目标坐标系），并叠加 EXIF 旋转矩阵以适配 Unity
- 最佳 / 次佳匹配双结果保留与筛选策略（按内点率与内点数加权）
- 匹配结果可视化输出（内点连线图、特征点图、深度图）
- 可选附带 GT 位姿误差评估（位移误差、旋转误差、旋转半径误差、旋转方向误差）
- 支持两种 3D 点来源：三维扫描仪深度图（`3DS`）与 VGGT pointmap（`VGGT`）

### 工程化
- 统一的 loguru 日志（控制台 + 滚动文件 + 独立错误日志）
- `@timer` 装饰器记录关键函数耗时
- 支持 Colmap 自动重建 / 手动命令调用（预留接口）

## 技术栈

| 类别 | 内容 |
| --- | --- |
| Web 框架 | Django 3.2（项目声明 `Django = "^3.0"`，锁文件为 3.2.13） |
| 数据库 | MySQL（默认配置） + PyMySQL 驱动；`utils/sql.py` 中另有 SQLite 的历史实现 |
| 跨域 | django-cors-headers（允许全部来源） |
| 计算机视觉 | OpenCV、NumPy、SciPy、scikit-learn |
| 深度学习 | PyTorch（CUDA 可用时自动使用 GPU）、LightGlue、DISK、SuperPoint、XFeat |
| 图像检索 | Patch-NetVLAD（外部依赖仓库） |
| 图像/元数据 | Pillow、piexif |
| 日志 | loguru |
| 序列化 | djangorestframework（已安装，但接口以原生 `JsonResponse` 实现） |
| 环境 | Python 3.8+；原开发环境为 Replit（`.replit` / `replit.nix`） |

## 目录结构

```
AR_backend/
├── manage.py                      # Django 管理入口
├── migrate.sh                     # 生成并执行数据库迁移的脚本
├── run_server.sh                  # 启动开发服务器
├── simplify_images_cameras.py     # 命令行工具：抽取文件中的隔行内容
├── pyproject.toml / poetry.lock   # Poetry 依赖声明与锁定
├── .replit / replit.nix           # Replit 运行与 Nix 环境配置
├── .gitignore
├── README.md                      # （当前为 Replit 模板，非项目文档）
├── django_project/                # Django 项目配置
│   ├── settings.py                # 全局配置：数据库、Media 路径、外部工具路径、设备
│   ├── urls.py                    # 根路由：/admin/ 与 /media_app/，并挂载 MEDIA 静态服务
│   ├── asgi.py / wsgi.py
│   └── __init__.py
├── media_app/                     # 核心业务应用
│   ├── models.py                  # Video / Image / Dataset / DatasetFile 数据模型
│   ├── forms.py                   # 对应的 ModelForm
│   ├── urls.py                    # 16 个接口路由
│   ├── views.py                   # 全部视图逻辑（上传、管理、定位流水线）
│   └── migrations/                # 0001 ~ 0005 迁移文件
├── utils/                         # 算法与工具模块
│   ├── nvlad_utils.py             # 定位流水线主体：坐标系转换、特征匹配、PnP、可视化
│   ├── calib3d.py                 # 相机标定与位姿求解算法（PnP、三角化、非线性优化）
│   ├── patchnetvlad_match_two.py  # Patch-NetVLAD 两图匹配脚本（含 MIT 许可头）
│   ├── image_preprocess.py        # 特征友好的图像预处理与数据增强（当前被注释掉未启用）
│   ├── exif_utils.py              # EXIF 方向 → Unity 旋转矩阵
│   ├── vggt_utils.py              # VGGT pointmap 查询与图像方形填充缩放
│   ├── draw.py                    # 匹配结果绘制
│   ├── upload.py                  # 文件名 / 目录名生成（时间戳 + UUID）
│   ├── sql.py                     # 数据集入库与 SQLite 读写辅助
│   ├── logger_config.py           # loguru 日志配置
│   ├── times.py                   # 计时装饰器
│   └── debug_utils.py             # 空文件
├── templates/
│   ├── upload_image.html          # 单图上传测试页
│   └── upload_video.html          # 单视频上传测试页
└── venv/                          # 提交进仓库的虚拟环境目录
```

## 快速开始

### 1. 环境要求

- Python 3.8 或更高版本
- MySQL 服务（若沿用默认配置）
- 可选：NVIDIA GPU（检测到 CUDA 时自动启用 GPU 推理）
- 需要额外准备两个本项目之外的外部组件：
  - **Colmap** 可执行文件（`settings.COLMAP_PATH`，默认为 `colmap` 命令）
  - **Patch-NetVLAD** 仓库，需与本项目目录同级（`settings.NetVLAD_PATH = BASE_DIR.parent / 'Patch-NetVLAD'`），且其中包含 `make_dataset_and_extract.sh` 与 `match_and_cal_pose.sh`

### 2. 安装依赖

```bash
# 方式一：Poetry
poetry install

# 方式二：pip
pip install Django==3.2.13 Pillow==10.1.0 djangorestframework
```

`pyproject.toml` 仅声明了 Django、Pillow、django-rest-framework 等基础依赖。代码中实际还导入了以下包，需要手动安装：

```bash
pip install torch torchvision opencv-python numpy scipy scikit-learn \
            pymysql django-cors-headers loguru piexif matplotlib \
            lightglue
```

> 注意：`lightglue`、`accelerated_features`（XFeat）为本项目引用的外部算法包，不在 PyPI 的标准依赖清单内，需自行获取并按原项目方式安装；`.gitignore` 中已将 `accelerated_features` 列为忽略项。

### 3. 环境变量

| 变量 | 是否必需 | 说明 |
| --- | --- | --- |
| `SECRET_KEY` | 推荐 | Django 密钥。代码在未设置时会回退到一个开发用默认值 |

### 4. 准备数据库

默认使用 MySQL，需先创建库：

```sql
CREATE DATABASE AR_DATASET CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

数据库连接参数位于 `django_project/settings.py` 的 `DATABASES` 配置中，请按本地环境修改。

### 5. 初始化并运行

```bash
# 执行数据库迁移
bash migrate.sh

# 启动开发服务器（监听 0.0.0.0:8000）
bash run_server.sh

# 或直接使用 Django 命令
python manage.py runserver 0.0.0.0:8000
```

服务启动后：

- 管理后台：`http://<host>:8000/admin/`
- 业务接口前缀：`http://<host>:8000/media_app/`
- 上传的媒体文件通过 `MEDIA_URL`（`/media/`）对外提供

## 核心模块与用法说明

### 数据模型（`media_app/models.py`）

| 模型 | 字段 | 说明 |
| --- | --- | --- |
| `Video` | `video`、`uploaded_at` | 上传的视频文件 |
| `Image` | `image`、`uploaded_at` | 上传的图片文件 |
| `Dataset` | `name`（唯一）、`file_path`、`info`、`config`、`old_config`、`updated_at` | 一个场景数据集。`info` 存 `info.json` 内容，`config` 存可编辑的 `config.json`，`old_config` 保留上一版配置 |
| `DatasetFile` | `dataset`(FK)、`name`、`file_path`、`file_type`、`uploaded_at` | 数据集内的单个文件，`file_type` 取值 `object`(`.obj`) / `ply`(`.ply`) / `unknown` |

### 接口一览（`media_app/urls.py`）

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| POST | `/media_app/upload_datasets/` | 批量上传并解压数据集压缩包，入库 |
| POST | `/media_app/upload_video/` | 上传单个视频（表单页 / JSON） |
| POST | `/media_app/upload_image/` | 上传单张图片（表单页 / JSON） |
| POST | `/media_app/upload_multiple_images/` | 上传多张图片 |
| POST | `/media_app/upload_multiple_videos/` | 上传多个视频 |
| GET | `/media_app/request_colmap_auto/` | 调用 Colmap 自动重建 |
| GET | `/media_app/request_colmap/` | 以自定义参数调用 Colmap |
| GET | `/media_app/request_NVLAD/` | 对指定数据集执行 Patch-NetVLAD 特征提取 |
| POST | `/media_app/request_NVLAD_redir/` | **核心定位接口**：上传查询图，返回估算位姿 |
| GET | `/media_app/test_read_image/` | 深度图读取调试接口 |
| GET | `/media_app/get_scence_list/` | 获取场景列表 `{scenceName, scenceKey}` |
| GET | `/media_app/get_config/` | 按 `sceneKey` 读取场景配置 |
| POST | `/media_app/delete_scence/` | 按名称删除场景及其目录 |
| GET | `/media_app/get_single_file/` | 按 `key` 下载单个数据集文件 |
| GET | `/media_app/get_multi_file/` | 按场景 `key` 打包下载全部文件 |
| POST | `/media_app/update_config/` | 更新场景配置（旧配置自动备份到 `old_config`） |

### 定位处理流程（`request_NVLAD_redir`）

1. **参数与目录准备**：解析查询集目录、场景图像目录、内参 / 位姿 / 深度目录；为本次请求创建带时间戳与随机后缀的临时工作目录。
2. **读取数据集信息**：从数据集的 `info.json` 取得内参、图像尺寸、EXIF 方向、坐标系定义与远平面，并预计算数据集坐标系 → OpenCV 坐标系 → 目标坐标系的转换矩阵。
3. **保存查询图**：将上传图片统一转存为 JPG，并写入 `query.txt`；若前端传入 `camera_matrix` 则使用之，否则回退到内置默认内参。
4. **图像检索**：调用外部 Patch-NetVLAD 脚本 `match_and_cal_pose.sh`，输出 `PatchNetVLAD_predictions.txt`，解析出每张查询图对应的候选图像及各自的内参 / 位姿 / 深度文件路径。
5. **逐候选精匹配与位姿估计**（最多取前 20 个候选）：
   - 查询图预处理（EXIF 校正 → 灰度 → 直方图均衡 → 高斯模糊 → 缩放）；
   - 候选图同法预处理，缩放到查询图尺寸，内参按比例缩放；
   - **DISK + LightGlue** 特征匹配，匹配点少于阈值（100）则跳过该候选；
   - 由 2D 像素 + 深度图反投影得到模型坐标系 3D 点，剔除无效深度点；
   - 3D 点数量足够（≥200）时调用 **`cv2.solvePnPRansac`**，以候选图位姿为初值求解；
   - 结果经坐标系转换并左乘 EXIF 旋转矩阵，得到目标坐标系下的相机到世界位姿。
6. **结果筛选**：`update_best_results` 依据内点率（含 0.1 容差窗口）与内点数保存最佳、次佳两组结果；若最佳内点率与内点数同时很高则提前结束。
7. **后处理与返回**：重算内点、输出可视化图片与调试文本、可选计算与 GT 的位姿误差；将位姿写入 `result.txt`，并以 JSON 返回 `saved_path` 与 `positions`（键为查询图文件名，值为 3×4 位姿矩阵）。

### 响应示例

```json
{
  "message": "Folder Found",
  "saved_path": [".../query_folder/xxx.jpg"],
  "positions": {
    "xxx.jpg": [[1.0, 0.0, 0.0, 0.12],
                [0.0, 1.0, 0.0, -0.34],
                [0.0, 0.0, 1.0, 1.05]]
  }
}
```

### 坐标系约定

- 数据集坐标系由 `info.json` 中的 `coordinate` 字段描述，形如 `{'X': 'right', 'Y': 'up', 'Z': 'forward'}`。
- 目标坐标系（代码中 `TARGET`）为 `{'X': 'right', 'Y': 'up', 'Z': 'forward'}`，即 Unity 左手坐标系风格。
- `utils/nvlad_utils.py` 提供 `compute_M_A2B`（构造坐标系转换矩阵）与 `transfer_Pose_from_A2B` / `transfer_Pose_from_B2A` / `transfer_Point_from_A2B` / `invert_Pose_Matrix` 等工具完成位姿与点的坐标系迁移。

### 数据集格式约定

上传的数据集压缩包需满足：

- 解压后根目录下必须存在 `config.json` 与 `info.json`，否则判定失败并回滚；
- `info.json` 至少需包含 `intrinsic`（`fx`/`fy`/`cx`/`cy`）、`image_size`（`height`/`width`）、`exif`、`coordinate`、`Z_Far`、`type`（`3DS` 或 `VGGT`）等字段；
- 图像目录约定为 `<dataset>/color/`，另有 `intrinsic/`、`pose/`、`depth/`，`VGGT` 类型还需 `pointmap/`；
- 单帧相关文件按 `frame-XXXXXX` 前缀对应，例如 `frame-000000.color.jpg`、`frame-000000.pose.txt`、`frame-000000.intrinsic_color.txt`、`frame-000000.depth.jpg`。

### 命令行工具

`simplify_images_cameras.py`：读取一个文本文件，跳过注释行并逐行交替丢弃，输出隔行结果。

```bash
python simplify_images_cameras.py <输入文件> <输出文件>
```

## 说明与已知限制

- **开发阶段配置**：`settings.py` 中 `DEBUG = True`、`ALLOWED_HOSTS = ['*']`、`CORS_ALLOW_ALL_ORIGINS = True`，均为开发环境取值，正式部署前需要收紧。
- **无鉴权**：所有业务接口均带 `@csrf_exempt` 且无登录校验，任意来源均可调用，包括删除场景与上传数据集的接口。
- **依赖外部仓库**：定位链路强依赖同级的 Patch-NetVLAD 仓库及其 shell 脚本，缺少该仓库时 `request_NVLAD_redir` 无法工作。
- **实现细节中的硬编码**：查询图默认内参、`TARGET` 坐标系、EXIF 有效值、调试开关（`is_debug` / `is_write_long_txt` / `is_accelerate` / `is_divide` / `is_K_equal`）等均写死在函数内部，尚未参数化。
- **部分代码未启用**：`utils/image_preprocess.py` 的增强版预处理在 `views.py` 中的导入被注释掉，当前实际使用的是 `nvlad_utils.process_single_image`；`utils/debug_utils.py` 为空文件。
- **遗留代码**：`utils/sql.py` 中保留了基于 SQLite 与 `SCENCE` 表的实现，其中 `insert_sence_batch` / `get_all_scences` 在当前调用链中已不再使用。
- **调试接口指向本地绝对路径**：`test_read_image` 视图内硬编码了开发机上的图片路径，在非原开发环境下会直接报错，仅作调试用途。
- **仓库体积**：仓库包含 `venv/` 虚拟环境目录，属于不应提交的内容。

## 许可证

本仓库未声明许可证文件（无 `LICENSE`）。

需要注意：`utils/patchnetvlad_match_two.py` 文件头部保留了 Patch-NetVLAD 项目的 MIT 许可声明，该文件的使用需遵循其原始许可条款。其余代码的授权状态未作声明。
