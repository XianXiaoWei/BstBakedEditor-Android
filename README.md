# SkyMeshEditorAndroid (BstBakedEditor-Android)

> **That Sky Level `.meshes` 地形数据查看与烘焙编辑器 —— Android 纯 Java · OpenGL ES 2.0 实现**

本项目是一款面向 **That Sky**（光·遇）游戏自定义关卡 `.meshes` 地形文件的 Android 移动端查看与编辑工具。项目在 Android 上用纯 Java 复刻了 Qt/C++ 桌面版 **BstBakedEditor** 的核心能力，可在移动设备上加载、可视化、烘焙并导出一整张大网格地形，最终输出为游戏可直接读取的 `.meshes` 文件。

---

## 功能特性

| 类别 | 功能 | 说明 |
|------|------|------|
| **文件** | `.meshes` 导入 | 解析多级地形二进制文件（含 chunk / subchunk / 云 / 索引 / 描述段） |
| **文件** | OBJ 导入 | 加载 Wavefront `.obj` 外部模型作为场景物体 |
| **文件** | `.meshes` 导出 | 支持 4 种导出策略，保留原始结构、避免丢面与丢顶点 |
| **编辑** | 基础几何体 | 立方体、球、圆柱、平面四种可编辑基元 |
| **编辑** | 工具 | 选择、移动、旋转、缩放 4 套 Gizmo 变换工具 |
| **编辑** | 笔刷 | 材质涂抹、雕刻凸起、雕刻凹陷，可调半径 |
| **编辑** | 属性检查器 | 物体名称 / 位置 / 旋转 / 缩放数值编辑 |
| **可视化** | 8 种着色模式 | 材质、云、法线、AO、细节纹理、其他数据、线框、分块边界 |
| **可视化** | 数据面板 | 场景统计、围盒、材质分布、数据通道范围、模型空间范围 |
| **渲染** | OpenGL ES 2.0 | 三光源实时照明，逐顶点材质权重混合 |
| **导航** | 虚拟摇杆 | 平移相机，按钮升降与视角旋转 |

项目规模：**19 个源文件，约 6400 行 Java 代码**，无 AndroidX 依赖，使用原生 `android.app.Activity` 与 `android.opengl` API。

---

## 技术栈

| 依赖项 | 版本 / 说明 |
|--------|-------------|
| 语言 | Java（无 lambda，兼容旧构建链） |
| Android SDK | `compileSdk 30` · `minSdk 21` · `targetSdk 30` |
| 构建 | Gradle 4.2.2（AGP 4.2.2），仓库含阿里云镜像加速 |
| 图形 | OpenGL ES 2.0，着色器 GLSL ES 1.00 (`#version 100`) |
| 网格压缩编解码 | **meshoptimizer 顶点编码器的纯 Java 标量移植**（非 JNI，纯 Java 实现） |
| 依赖库 | 无第三方 Java 依赖，`dependencies {}` 为空 |

> 说明：网格压缩编解码（`MeshOptimizer` / `MeshoptCodec`）是将 meshoptimizer `vertexcodec.cpp` 的标量（非 SIMD）解码路径以 Java 移植，并补齐完整编码器，无需引入任何原生库。

---

## 项目结构

```
app/src/main/java/com/skymesh/editor/
├── MainActivity.java       # 入口 Activity：面板、工具、笔刷、导入导出、摇杆、可视化
├── EditorGLView.java       # GLSurfaceView 派生，管理 GL 上下文与相机监听
├── GLRenderer.java         # OpenGL ES 渲染器：绘制、拾取、工具交互、笔刷
├── Scene.java              # 场景容器：物体列表、统计、选择、删除
├── SceneObject.java        # 场景物体（地形 / 导入模型 / 基元），模型矩阵与材质颜色
├── MeshVertex.java         # 可编辑网格顶点（36 字节，四通道材质槽 + 权重）
├── ObjKind.java            # 物体类型枚举：Primitive / Imported / MeshesTerrain
├── Vec3.java               # 三维向量数学工具
├── Camera.java             # 相机：yaw / pitch / distance，平移与升降
├── Gizmo.java              # 移动 / 旋转 Gizmo 轴绘制与命中
├── MeshFactory.java        # 立方体 / 球 / 圆柱 / 平面网格生成
├── ObjImporter.java        # Wavefront OBJ 解析与导入
├── MeshesParser.java       # That Sky .meshes 二进制格式解析
├── MeshesExporter.java     # .meshes 多处策略导出（核心）
├── MeshOptimizer.java      # meshoptimizer 纯 Java 标量实现（解码 + 简化编码）
├── MeshoptCodec.java       # meshoptimizer 纯 Java 实现（v0/v1 解码 + v1 编码）
├── ShaderUtil.java         # 着色器编译 / 链接工具
└── JoystickView.java       # 虚拟摇杆控件

app/src/main/res/layout/activity_main.xml   # 横屏单 Activity 布局
```

---

## 数据格式说明

### `.meshes` 文件整体布局 (`MeshesParser` / `MeshesExporter`)

文件是一个 **LVL0 段式容器**，`magic = "LVL0"`（`0x304C564C`），包含头部与多个数据段：

| 段 | 含义 |
|----|------|
| **Header** | 136 字节固定头：magic、version、100 字节 TOC、padding、全局包围盒 |
| **DESC** | 描述信息（字符串） |
| **LOD0** | LOD 0 原始数据（未编辑时原样保留） |
| **GEO0** | 几何：压缩顶点 + 局部索引 + chunk + subchunk |

**Header 布局（136 字节）：**

```
+0    u32 magic    "LVL0"
+4    u32 version
+8    TOC  100 字节
        u32 段数(3)
        段×4B name + 4B offset + 4B length  → 39 字节后补齐
+108  u32 padding
+112  vec3 gmax
+124  vec3 gmin
```

### GEO0 段（几何数据）

```
u32 index_count
u32 vertex_count
u32 chunk_count           (地形 chunk，不含云)
u32 cloud_count           (云 chunk)
u32 subchunk_count
u32 compressed_vertex_size
compressed_vertices       (meshopt 压缩，通常 36B/顶点)
local_indices             (u8 局部索引，u8 即 ≤256 顶点/块)
chunks[chunk_count+cloud_count]   (30 字节/块，见下)
subchunks[subchunk_count]         (8 字节/块，见下)
```

**Chunk (30 字节)：**

```
+0  u32 idxStart
+4  u32 vtxStart
+8  u32 subchunkStart
+12 u16 idxCount
+14 u8  vtxCount
+15 u8  subchunkCount
+16 vec3 AABB min
+28 vec3 AABB max          (加 0.1 余量)
... u32×4 pad
```

**Subchunk (8 字节)：**

```
+0  u8 materialId
+1  u8 triCount
+2  u8 vtxCount
+3  u8 triStart
+4  u8 triEnd
+5  u8 vtxStart
+6  u8 vtxEnd
+7  u8 pad
```

### 顶点格式（`MeshVertex`，36 字节/顶点）

```
position         float × 3      (12 字节)  位置
normal           snorm8 × 4     (4 字节)   法线
materialIds      u8 × 4         (4 字节)   最多 4 个材质槽 ID
materialWeights  unorm8 × 4     (4 字节)   各槽权重
input2           unorm8 × 4     (4 字节)   AO / 粗糙度
input3           unorm8 × 4     (4 字节)   细节纹理
input4           unorm8 × 4     (4 字节)   其他（裙边 / 辅助）
```

---

## 材质系统

项目完整实现 That Sky Level 的**四通道材质混合**：

- 每个顶点绑定最多 **4 个材质槽**，各搭配一个 `unorm8` 权重（0~1）
- 渲染时按权重对最多 4 个材质颜色加权平均，得到顶点颜色 `SceneObject.vertexColor()`
- 导出时材质槽会按权重降序重新排序，保证 `slot 0` 恒为主材质（权重最大），以匹配游戏物理碰撞与音效系统对 `slot 0` 的依赖

**内置材质表（`MaterialTable`，256 槽，部分）：**

| ID | 名称 | 近似颜色 | ID | 名称 | 近似颜色 |
|----|------|----------|----|------|----------|
| 0 | None | 灰 | 24 | TileFloor | 浅灰 |
| 2 | Transparent | 浅蓝灰 | 25 | TileWall | 蓝灰 |
| 3 | Void | 深灰黑 | 27 | SoilWet | 深棕 |
| 6 | VoidMinor | 中灰 | 29 | Bone | 米白 |
| 16 | Cliff | 土黄 | 30 | Wood | 棕褐 |
| 17 | Soil | 棕黄 | 32 | Sand | 浅沙黄 |
| 18 | CliffLight | 浅土 | 34 | SandLight | 亮沙色 |
| 20 | Wall | 浅灰蓝 | 35 | Snow | 纯白 |
| 21 | Gold | 金黄 | 48 | Grass | 草绿 |
| 23 | TileCeiling | 浅灰 | 80 | Cloud | 云白 |

未映射的材质 ID 渲染为默认灰色 `(0.7, 0.7, 0.7)`。

---

## 模块交互与导出策略

### 数据流

```
用户操作 ─→ MainActivity ─→ Scene / GLRenderer
                                     │
      触摸/摇杆 ─→ Camera（yaw/pitch/distance）
      工具/笔刷 ─→ Gizmo + 顶点拾取 ─→ MeshVertex 编辑
                                     │
文件导入 ─→ MeshesParser.parse() / ObjImporter.load() ─→ SceneObject
文件导出 ─→ MeshesExporter.exportScene() ─→ 写 .meshes 字节
                                     │
每帧     ─→ GLRenderer.onDrawFrame() ─→ OpenGL ES 绘制
```

### 导出策略（`MeshesExporter.exportScene`，按优先级）

| 优先级 | 场景条件 | 策略 |
|--------|----------|------|
| 1 | 仅一个 `MeshesTerrain` 且**未修改**顶点 | 原样输出原始文件字节，零损耗 |
| 2 | 仅一个 `MeshesTerrain` 且**已修改**顶点 | **splice 方式**：仅重新 meshopt 编码压缩顶点，chunk / subchunk / index / DESC / LOD0 结构原样保留，更新 subchunk 分配与 AABB |
| 3 | `MeshesTerrain` + 其他物体 | 保留原始结构基础上**追加新 chunk**（每 chunk ≤ 252 顶点），顶点跨 chunk 复制以避免丢面 |
| 4 | 无 `MeshesTerrain` | 从零构建完整文件（贪心分块，确保不丢面） |

**Subchunk 重建（MIN-MAX 算法，`rebuildChunkSubchunks`）：**

这是对原始 `.meshes` 文件逆向分析出的精确规则（实测 165/165 chunk 100% 匹配）：

- 每个材质对应**一个** subchunk（不因材质不连续而产生多个）
- `triStart` / `triEnd` 取该材质出现的三角形首末索引，`triCount` 为精确数量
- `vtxStart` / `vtxEnd` 取拥有该材质的顶点的局部索引范围，`vtxCount` 为涉及的唯一局部顶点数
- subchunk 按 `triEnd` 升序、再按 `matId` 升序排列：局部材质先渲染、基础材质（如 Grass=48，覆盖全部三角形）后渲染——避免基础材质先写深度缓冲而遮挡局部材质，导致"丢顶点"。

**导出关键修复：**

- `unorm`：非零但极小的权重（如笔刷涂出的 `0.001`）不会被截断编码为 `0`，强制为最小非零值 `1`，避免 "丢顶点"
- `snorm`：法线使用四舍五入（meshopt quantizeSnorm 标准）而非截断，修复除 `-128` 外所有值的精度丢失
- 索引保持原始三角形顺序，**不排序**，避免破坏游戏碰撞检测

---

## 编译与运行

### 环境要求

- Android Studio（兼容 AGP 4.2.2 的版本）或命令行 Gradle 4.2.2
- JDK 8+
- Android SDK Platform 30

### 构建

```bash
# 1. 克隆并解压源码
git clone https://github.com/XianXiaoWei/BstBakedEditor-Android.git
cd BstBakedEditor-Android
unzip SkyMeshEditorAndroid_src.zip
cd SkyMeshEditorAndroid

# 2. 命令行构建
./gradlew assembleDebug
# 产物: app/build/outputs/apk/debug/app-debug.apk
```

也可直接用 Android Studio 打开并运行。仓库已内置阿里云 Maven 镜像以加速依赖拉取。

### 运行要求

- Android 5.0（API 21）及以上
- 支持 OpenGL ES 2.0 的设备
- 界面强制横屏（`screenOrientation="landscape"`）

---

## 操作说明

> 提示：文档下方“截图示例”占位，可在实际编译运行后补充。

### 界面布局

应用**强制横屏**，中央为 OpenGL 视口，左右为可折叠面板：

- **左面板**：物体列表（大纲）、属性检查器（名称/位置/旋转/缩放）、笔刷强度、添加基元按钮、添加/删除对象
- **右面板**：数据可视化信息（场景统计、Chunk/云/Subchunk 计数、包围盒、材质分布、数据通道范围、模型空间范围）
- **下方摇杆**：虚拟摇杆控制相机平移；上/下按钮控制相机升降

### 基本工作流

1. **导入地形**：点击 "导入 .meshes" 通过系统文件选择器加载地形文件
2. **浏览**：摇杆平移、升降按钮定位，在对象列表或直接点选视口中的物体
3. **变换**：选择移动 / 旋转 / 缩放工具，拖动 Gizmo 轴调整物体
4. **笔刷编辑**：选择材质涂抹（弹出材质选择框）或凸起/凹陷雕刻，用滑块调笔刷半径，在网格表面拖动
5. **可视化检查**：切换 8 种着色模式，观察右面板数据
6. **导出**：点击 "导出" 选择路径，按需写出 `.meshes`，正文案保留原始结构

---

## 关键实现细节

### 渲染（`GLRenderer`）

- 顶点格式 `pos(3) + normal(3) + color(3)`，9 floats/顶点（36 字节）；线段格式 `pos(3) + color(3)`
- 片元着色器实现**三光照明**（ambient + key + fill，参照 SkyModelViewer），选中物体以橙色高亮混合
- 顶点颜色由 `SceneObject.vertexColor()` 按材质权重加权得到，运行时上传到 GPU
- VBO / IBO 缓存于 GL 缓冲区，`dirty` 标记驱动增量上传；提供短暂调试的线段缓冲用于线框/分块边界

### 相机（`Camera`）

- 采用 `yaw / pitch / distance` 球面坐标；`moveByJoystick` 按摇杆增量平移，`moveVertical` 升降
- 支持拾取射线（NDC→世界空间），供点选与笔刷命中

### 网格优化（`MeshOptimizer` / `MeshoptCodec`）

- 解码器完整支持 version 0 与 version 1，含 `channel 0/1/2`（u8 zigzag / u16 zigzag / u32 rotate-xor）与块式字节组解码（0/1/2/4/8 位混合位宽 + header 位索引）
- 编码器为简化实现：version 0 输出游戏要求格式，version 1 使用 channel 0（u8 zigzag）
- 全部纯 Java 实现，`K_BLOCK_SIZE_BYTES=8192`，每块最多 256 顶点，16 字节为一字节组

---

## 参考与衍生

- 材质颜色表与格式解析参考 [SkyModelViewer-Android](https://github.com/BySobolev/SkyModelViewer-Android) 的 `LevelMeshesReader`
- 网格编码算法参照 meshoptimizer `vertexcodec.cpp`（[meshoptimizer](https://github.com/zeux/meshoptimizer)）
- subchunk 重建规则参照 `bstbake_unpack.py` 的分块逻辑

---

## License

请以仓库内 LICENSE 文件为准。

**项目地址：** <https://github.com/XianXiaoWei/BstBakedEditor-Android>