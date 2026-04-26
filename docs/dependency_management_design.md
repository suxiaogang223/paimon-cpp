# Paimon C++ 灵活依赖管理实现设计

关联 issue: [#103 Flexible Third-Party Dependency Management System](https://github.com/alibaba/paimon-cpp/issues/103)

## 1. 背景

paimon-cpp 当前所有第三方依赖都通过 `cmake_modules/ThirdpartyToolchain.cmake` 中的 `ExternalProject_Add` 从源码构建。该方案保证了构建可复现性，但带来若干问题：

- **构建慢**：Arrow / ORC / Avro 等大型依赖每次 clean build 都要重新编译，CI 与本地迭代成本高。
- **无法复用系统库**：开发机或 CI 容器中即使已有合适版本的 Arrow / zstd / glog，仍需重建一份。
- **环境适配差**：conda / vcpkg / 发行版包管理器无法直接接入。
- **粒度不可控**：无法对单个依赖选择 SYSTEM，对其他依赖保持 BUNDLED。

本设计在保留当前 BUNDLED 行为为可选默认的前提下，引入与 Apache Arrow C++ 同款的 `*_SOURCE` / `*_ROOT` 机制，并定义一套统一的 imported target 约定，让上层 CMake 代码完全与依赖来源解耦。

## 2. 当前状态评估

### 2.1 文件结构

| 路径 | 行数 | 职责 |
| --- | --- | --- |
| [CMakeLists.txt](../CMakeLists.txt) | 453 | 顶层项目配置、子目录组织 |
| [cmake_modules/DefineOptions.cmake](../cmake_modules/DefineOptions.cmake) | 211 | 选项注册框架（借自 Arrow） |
| [cmake_modules/ThirdpartyToolchain.cmake](../cmake_modules/ThirdpartyToolchain.cmake) | 1467 | 15 个 `build_xxx()` 宏 + ExternalProject_Add |
| [cmake_modules/BuildUtils.cmake](../cmake_modules/BuildUtils.cmake) | 351 | `add_paimon_lib` 等高层封装 |
| [cmake_modules/arrow.diff](../cmake_modules/arrow.diff) | 213 | Arrow 源码 patch |
| [cmake_modules/orc.diff](../cmake_modules/orc.diff) | 437 | ORC 源码 patch |
| [cmake_modules/jieba.diff](../cmake_modules/jieba.diff) | 16 | Jieba 源码 patch |
| [third_party/versions.txt](../third_party/versions.txt) | — | 所有依赖版本号 + checksum |

### 2.2 当前依赖清单

**始终构建**（[ThirdpartyToolchain.cmake:1441-1450](../cmake_modules/ThirdpartyToolchain.cmake#L1441-L1450)）：

`fmt` / `rapidjson` / `re2` / `snappy` / `zstd` / `zlib` / `lz4` / `arrow` / `tbb` / `glog`

**条件构建**（[ThirdpartyToolchain.cmake:1452-1467](../cmake_modules/ThirdpartyToolchain.cmake#L1452-L1467)）：

| 选项 | 触发依赖 |
| --- | --- |
| `PAIMON_ENABLE_AVRO` | `avro` |
| `PAIMON_ENABLE_ORC` | `protobuf` + `orc` |
| `PAIMON_ENABLE_JINDO` | `jindosdk_c` + `jindosdk_nextarch` |
| `PAIMON_ENABLE_LUCENE` | `boost` + `lucene` + `jieba` |
| `PAIMON_BUILD_TESTS` | `gtest` |

**仓库内自带**（不在本设计范围）：`roaring_bitmap`、`xxhash`、`lance`（预编译 .so）、`lumina`（预编译 .so）。

### 2.3 当前 imported target 约定

每个 `build_xxx()` 宏完成后会创建一个 IMPORTED 静态库目标，使用**裸名**（无命名空间），例如 [ThirdpartyToolchain.cmake:554-557](../cmake_modules/ThirdpartyToolchain.cmake#L554-L557)：

```cmake
add_library(fmt STATIC IMPORTED)
set_target_properties(fmt PROPERTIES IMPORTED_LOCATION ${FMT_STATIC_LIB})
target_include_directories(fmt INTERFACE ${FMT_INCLUDE_DIR})
```

上层使用方式（[src/paimon/CMakeLists.txt:323-345](../src/paimon/CMakeLists.txt#L323-L345)）：

```cmake
add_paimon_lib(paimon
               SOURCES ...
               DEPENDENCIES arrow tbb glog fmt ...
               STATIC_LINK_LIBS arrow tbb glog dl fmt ...)
```

裸名与 `find_package()` 在 SYSTEM 模式下导出的命名空间目标（如 `Arrow::arrow_static`、`fmt::fmt`）冲突，是本设计需要解决的核心兼容性问题。

### 2.4 已有的有利条件

- 顶层已设置 `set(CMAKE_CXX_VISIBILITY_PRESET hidden)`（[CMakeLists.txt:329-330](../CMakeLists.txt#L329-L330)）。
- 所有 BUNDLED 依赖均强制静态构建（`ARROW_BUILD_STATIC=ON`、`ARROW_BUILD_SHARED=OFF` 等）。
- `DefineOptions.cmake` 框架本身就来自 Arrow，扩展成本低。
- 仅 3 处使用了老式 `${PKG_INCLUDE_DIR}` 变量引用（[CMakeLists.txt:344-347](../CMakeLists.txt#L344-L347)），迁移工作量小。
- `versions.txt` + `set_urls` 机制本身已对 SYSTEM 模式无害，可直接保留。

### 2.5 当前问题

1. `build_arrow()` 宏内部 `get_target_property(... INTERFACE_INCLUDE_DIRECTORIES)` 取出已构建 BUNDLED 依赖（snappy/lz4/zstd/zlib/re2）的 `_ROOT` 喂给 Arrow 子构建（[ThirdpartyToolchain.cmake:1108-1192](../cmake_modules/ThirdpartyToolchain.cmake#L1108-L1192)）。这条路径在 Arrow 切换为 SYSTEM 时会失效，需要重新设计。
2. patch 文件（`arrow.diff`、`orc.diff`、`jieba.diff`）仅在 BUNDLED 模式下应用，SYSTEM 模式必须跳过。
3. `gtest` 与测试目标耦合（[CMakeLists.txt:368](../CMakeLists.txt#L368)），SYSTEM 路径需保证 `GTEST_INCLUDE_DIR` / `GTEST_LINK_TOOLCHAIN` 等变量仍可被消费。

## 3. 设计目标与非目标

### 目标

1. 引入 `PAIMON_DEPENDENCY_SOURCE=<AUTO|BUNDLED|SYSTEM|CONDA>` 全局选项。
2. 引入 `<Pkg>_SOURCE` / `<Pkg>_ROOT` 单依赖覆盖。
3. 提供 `resolve_dependency()` 宏作为统一入口，原 `build_xxx()` 改为内部实现。
4. 为每个支持 SYSTEM 的依赖提供 `FindXxxAlt.cmake`，统一暴露 `Xxx::xxx` 形式的 imported target。
5. 上层代码（`src/`、`test/`）完全不感知依赖来源，仅消费命名空间目标。
6. 默认行为保持 **BUNDLED**（issue 中讨论后选 `AUTO` 也可，见 §9），现有 `cmake -B build` 命令零改动。

### 非目标

1. 不实现传递依赖版本一致性的硬校验（属 Phase 4，可选 strict mode）。
2. 不解决跨 SYSTEM/BUNDLED 树的 ABI 兼容性（用户责任，文档化）。
3. 不重写 `versions.txt` / `download_dependencies.sh` 机制。
4. 仓库内自带依赖（`roaring_bitmap` / `xxhash` / `lance` / `lumina`）不纳入。

## 4. 整体架构

```
                         ┌─────────────────────────────┐
                         │  PAIMON_DEPENDENCY_SOURCE   │   全局策略
                         │  (AUTO / BUNDLED / SYSTEM   │
                         │   / CONDA)                  │
                         └──────────────┬──────────────┘
                                        │
                                        ▼
   ┌────────────────────────────────────────────────────────────────┐
   │                    resolve_dependency(Foo)                     │
   │                                                                │
   │   1. 读取 Foo_SOURCE（用户覆盖）→ 否则继承全局策略              │
   │   2. 读取 Foo_ROOT / PAIMON_PACKAGE_PREFIX                      │
   │   3. 分派：                                                     │
   │       SYSTEM/AUTO → find_package(FooAlt)                        │
   │       BUNDLED     → build_foo()                                 │
   │       AUTO 失败   → 回退到 BUNDLED + 警告                       │
   │   4. 校验：所有路径完成后必须存在 imported target Foo::foo      │
   └────────────────────────────────────────────────────────────────┘
                                        │
                ┌───────────────────────┼───────────────────────┐
                ▼                       ▼                       ▼
    ┌───────────────────┐   ┌───────────────────┐   ┌───────────────────┐
    │ FindFooAlt.cmake  │   │   build_foo()     │   │  CONDA: 走 SYSTEM │
    │                   │   │ (现有实现保留，   │   │   + $CONDA_PREFIX │
    │ 1) find_package   │   │  仅作为内部宏)    │   │   作为 PREFIX     │
    │    (Foo CONFIG)   │   └───────────────────┘   └───────────────────┘
    │ 2) fallback:      │
    │    pkg-config /   │
    │    find_library   │
    │ 3) wrap →         │
    │    Foo::foo       │
    │    (IMPORTED)     │
    └───────────────────┘
```

## 5. 关键设计

### 5.1 全局选项注册

在 `DefineOptions.cmake` 顶层 cmake 区块新增 `Thirdparty` 类别：

```cmake
set_option_category("Thirdparty")

define_option_string(PAIMON_DEPENDENCY_SOURCE
                     "Method to use for finding third-party dependencies"
                     "BUNDLED"
                     "AUTO" "BUNDLED" "SYSTEM" "CONDA")

define_option_string(PAIMON_PACKAGE_PREFIX
                     "Common installation prefix for unspecified dependencies"
                     "")

define_option(PAIMON_DEPENDENCY_USE_SHARED
              "Prefer shared libraries when consuming SYSTEM dependencies"
              OFF)
```

每个具体依赖的 `<Pkg>_SOURCE` 不在 `define_option` 注册（避免列表炸开），仅在 `resolve_dependency()` 中按需读取，未设置时继承全局值。

### 5.2 `resolve_dependency` 宏

```cmake
macro(resolve_dependency DEPENDENCY_NAME)
    set(options)
    set(one_value_args REQUIRED_VERSION FORCE_BUNDLED)
    set(multi_value_args COMPONENTS)
    cmake_parse_arguments(ARG
        "${options}" "${one_value_args}" "${multi_value_args}" ${ARGN})

    # 1. 决定 source
    if(DEFINED ${DEPENDENCY_NAME}_SOURCE)
        set(_SOURCE "${${DEPENDENCY_NAME}_SOURCE}")
    elseif(ARG_FORCE_BUNDLED)
        set(_SOURCE "BUNDLED")  # 例如 jindosdk: 无 system 来源，固定 BUNDLED
    elseif(PAIMON_DEPENDENCY_SOURCE STREQUAL "CONDA")
        set(_SOURCE "SYSTEM")
        if(NOT DEFINED ENV{CONDA_PREFIX})
            message(FATAL_ERROR "PAIMON_DEPENDENCY_SOURCE=CONDA but $CONDA_PREFIX not set")
        endif()
        list(APPEND CMAKE_PREFIX_PATH "$ENV{CONDA_PREFIX}")
    else()
        set(_SOURCE "${PAIMON_DEPENDENCY_SOURCE}")
    endif()

    # 2. 处理 _ROOT / PACKAGE_PREFIX
    if(NOT DEFINED ${DEPENDENCY_NAME}_ROOT
       AND NOT "${PAIMON_PACKAGE_PREFIX}" STREQUAL "")
        set(${DEPENDENCY_NAME}_ROOT "${PAIMON_PACKAGE_PREFIX}")
    endif()

    # 3. 分派
    set(_FOUND FALSE)
    if(_SOURCE STREQUAL "SYSTEM" OR _SOURCE STREQUAL "AUTO")
        find_package(${DEPENDENCY_NAME}Alt
                     ${ARG_REQUIRED_VERSION}
                     COMPONENTS ${ARG_COMPONENTS})
        if(${DEPENDENCY_NAME}Alt_FOUND)
            set(_FOUND TRUE)
            message(STATUS "Resolved ${DEPENDENCY_NAME} from SYSTEM")
        endif()
    endif()

    if(NOT _FOUND)
        if(_SOURCE STREQUAL "SYSTEM")
            message(FATAL_ERROR
                "${DEPENDENCY_NAME}_SOURCE=SYSTEM but find_package failed. "
                "Set ${DEPENDENCY_NAME}_ROOT or switch to AUTO/BUNDLED.")
        endif()
        # AUTO 回退 / 显式 BUNDLED
        message(STATUS "Building ${DEPENDENCY_NAME} from source")
        build_${DEPENDENCY_NAME}_lower()  # 内部宏，下面 5.3 解释
    endif()

    # 4. 后置校验：确保有 namespaced target
    string(TOLOWER "${DEPENDENCY_NAME}" _lower)
    if(NOT TARGET ${DEPENDENCY_NAME}::${_lower})
        message(FATAL_ERROR
            "${DEPENDENCY_NAME}::${_lower} target not created after resolve")
    endif()
endmacro()
```

### 5.3 命名空间 Imported Target 规范

为避免 SYSTEM/BUNDLED 在目标名上冲突，统一使用命名空间形式：

| 旧裸名 | 新规范名 |
| --- | --- |
| `fmt` | `fmt::fmt`（与官方 config 一致） |
| `arrow` | `Arrow::arrow_static` / `Arrow::arrow_shared`（与 Arrow 官方一致） |
| `glog` | `glog::glog` |
| `tbb` | `TBB::tbb` |
| `snappy` | `Snappy::snappy` |
| `zstd` | `zstd::libzstd_static`（与 zstd 官方 config 一致） |
| `lz4` | `LZ4::lz4` |
| `zlib` | `ZLIB::ZLIB`（CMake 标准） |
| `re2` | `re2::re2` |
| `RapidJSON` | `RapidJSON::RapidJSON`（仅头文件） |
| `protobuf` | `protobuf::libprotobuf` |
| `orc` | `orc::orc` |
| `avro` | `Avro::avrocpp_s` |
| `gtest` | `GTest::gtest` / `GTest::gmock` |

迁移要求：

1. `build_xxx()` 内部已 `add_library(xxx STATIC IMPORTED)` 处改为 `Xxx::xxx`，并对老裸名同时建立 `add_library(<old> ALIAS Xxx::xxx)`，**Phase 1 内**不破坏现有 `src/paimon/CMakeLists.txt` 的依赖列表。
2. Phase 2 完成后，逐个子目录将依赖列表从裸名改为命名空间名，删除 ALIAS。

### 5.4 Find module 设计

`cmake_modules/` 下新增 `FindXxxAlt.cmake`，每个文件遵循统一模板：

```cmake
# FindArrowAlt.cmake
include(FindPackageHandleStandardArgs)

# 1. 优先信任官方 Config 模式
find_package(Arrow CONFIG QUIET)
if(Arrow_FOUND AND TARGET Arrow::arrow_static)
    set(ArrowAlt_FOUND TRUE)
    set(ArrowAlt_VERSION ${Arrow_VERSION})
    return()
endif()

# 2. fallback: 手工查找 + 包装为 imported target
find_path(ArrowAlt_INCLUDE_DIR
    NAMES arrow/api.h
    HINTS ${Arrow_ROOT} $ENV{Arrow_ROOT}
    PATH_SUFFIXES include)
find_library(ArrowAlt_LIBRARY
    NAMES arrow
    HINTS ${Arrow_ROOT} $ENV{Arrow_ROOT}
    PATH_SUFFIXES lib lib64)

find_package_handle_standard_args(ArrowAlt
    REQUIRED_VARS ArrowAlt_INCLUDE_DIR ArrowAlt_LIBRARY)

if(ArrowAlt_FOUND AND NOT TARGET Arrow::arrow_static)
    add_library(Arrow::arrow_static UNKNOWN IMPORTED)
    set_target_properties(Arrow::arrow_static PROPERTIES
        IMPORTED_LOCATION "${ArrowAlt_LIBRARY}"
        INTERFACE_INCLUDE_DIRECTORIES "${ArrowAlt_INCLUDE_DIR}")
endif()
```

**为什么用 `XxxAlt` 后缀**：避免与上游可能存在的 `FindXxx.cmake` 重名导致行为不可控；这是 Arrow 自身的命名约定。

### 5.5 Patch 文件处理

`arrow.diff` / `orc.diff` 仅在 BUNDLED 路径需要应用：

```cmake
# build_arrow() 内部
if(_PAIMON_ARROW_PATCH_REQUIRED)
    set(ARROW_PATCH_COMMAND ${CMAKE_COMMAND} -E chdir <SOURCE_DIR> bash -c
        "[ -f .patched ] && echo 'patched, ignore' || patch -s -N -p1 -i '${PATCH_FILE}'")
endif()
```

SYSTEM 模式不会进入 `build_arrow`，patch 自然不会执行。需在 `FindArrowAlt.cmake` 文档中说明：用户使用 SYSTEM Arrow 时，需自行确保 Arrow 已包含等价于 `arrow.diff` 的修复（或不需要这些修复，例如已升级到包含相应修复的版本）。当前的修复内容包括 `-include cstdint`（GCC 15）相关的 build flag 已通过 CMake 注入而非 patch（[ThirdpartyToolchain.cmake:1131](../cmake_modules/ThirdpartyToolchain.cmake#L1131)），不影响 SYSTEM 路径。

### 5.6 传递依赖一致性

实现 issue #103 评论中达成的方向（BUNDLED 全部静态 + hidden visibility 已具备），在 `resolve_dependency` 内增加**软约束**：

```cmake
# Arrow 的传递依赖
if(DEPENDENCY_NAME STREQUAL "Arrow"
   AND NOT DEFINED zstd_SOURCE
   AND _SOURCE STREQUAL "SYSTEM")
    message(STATUS "Defaulting zstd_SOURCE=SYSTEM (transitive of Arrow=SYSTEM)")
    set(zstd_SOURCE "SYSTEM")
endif()
```

这套对齐规则集中维护在 `resolve_dependency` 顶部一张表里：

```cmake
set(_PAIMON_TRANSITIVE_DEPS_Arrow zstd snappy lz4 zlib re2)
set(_PAIMON_TRANSITIVE_DEPS_orc protobuf zstd snappy lz4 zlib)
set(_PAIMON_TRANSITIVE_DEPS_avro snappy)
```

不强制，只设默认；用户显式指定 `<Dep>_SOURCE` 时不被覆盖。

### 5.7 build_arrow 改造

`build_arrow()` 当前依赖 `get_target_property(snappy INTERFACE_INCLUDE_DIRECTORIES)` 反推 `Snappy_ROOT`（[ThirdpartyToolchain.cmake:1111-1124](../cmake_modules/ThirdpartyToolchain.cmake#L1111-L1124)），在传递依赖混合模式下不可靠。改为：

- BUNDLED 路径：保留现状（取自 BUNDLED 子构建的 install prefix）。
- SYSTEM 路径：不进入 `build_arrow`，无需考虑。
- 校验：进入 `build_arrow` 前断言其传递依赖均为 BUNDLED 或被显式标注为 SYSTEM 且 `Snappy::snappy` 等目标已存在 `IMPORTED_LOCATION`，从中提取 prefix。

## 6. 顶层文件改造

### 6.1 `CMakeLists.txt`

删除 [CMakeLists.txt:344-347](../CMakeLists.txt#L344-L347) 的：

```cmake
include_directories(SYSTEM ${ARROW_INCLUDE_DIR})
include_directories(SYSTEM ${TBB_INCLUDE_DIR})
include_directories(SYSTEM ${GLOG_INCLUDE_DIR})
```

`Xxx::xxx` 目标的 `INTERFACE_INCLUDE_DIRECTORIES` 已自动通过 `target_link_libraries` 传播，不需要全局 `include_directories`。

### 6.2 `ThirdpartyToolchain.cmake` 末尾改造

```cmake
# 旧
build_fmt()
build_rapidjson()
...

# 新
resolve_dependency(fmt)
resolve_dependency(RapidJSON)
resolve_dependency(re2)
resolve_dependency(Snappy)
resolve_dependency(zstd)
resolve_dependency(ZLIB)
resolve_dependency(LZ4)
resolve_dependency(Arrow)         # Arrow 必须在其传递依赖之后
resolve_dependency(TBB)
resolve_dependency(glog)

if(PAIMON_ENABLE_AVRO)
    resolve_dependency(Avro)
endif()
if(PAIMON_ENABLE_ORC)
    resolve_dependency(Protobuf)
    resolve_dependency(orc)
endif()
if(PAIMON_ENABLE_JINDO)
    resolve_dependency(jindosdk_c FORCE_BUNDLED)
    resolve_dependency(jindosdk_nextarch FORCE_BUNDLED)
endif()
if(PAIMON_ENABLE_LUCENE)
    resolve_dependency(Boost FORCE_BUNDLED)  # 当前 boost 来自仓库内 tarball
    resolve_dependency(lucene FORCE_BUNDLED)
    resolve_dependency(jieba FORCE_BUNDLED)
endif()
```

`FORCE_BUNDLED` 用于无 system 替代品的依赖（jindosdk / lucene / jieba 自带 patch，且无标准发行版包）。

### 6.3 子目录 CMake

[src/paimon/CMakeLists.txt](../src/paimon/CMakeLists.txt) 等使用 `arrow tbb glog fmt` 处保持不变（Phase 1 通过 ALIAS 兼容）；Phase 2 批量替换为 `Arrow::arrow_static fmt::fmt glog::glog TBB::tbb`。

## 7. 分阶段实施

### Phase 1 — 基础设施（本设计核心交付）

**目标**：在不改变默认行为的前提下，落地选项 + `resolve_dependency` + 一个示范依赖打通完整链路。

具体工作：

1. `DefineOptions.cmake` 新增 `Thirdparty` 类别与 3 个新选项。
2. 新增 `cmake_modules/ResolveDependency.cmake`，实现 `resolve_dependency` 宏。
3. 选 `fmt` 作为示范（小、独立、无传递依赖）：
   - 编写 `cmake_modules/FindfmtAlt.cmake`。
   - 修改 `build_fmt()`：完成后建 `add_library(fmt::fmt ALIAS fmt)`，保留旧裸名。
   - `ThirdpartyToolchain.cmake` 中 `build_fmt()` → `resolve_dependency(fmt)`。
4. 在 `build_and_package.sh` 与 CI 增加一组 `-DPAIMON_DEPENDENCY_SOURCE=BUNDLED` 显式构建（验证向后兼容）以及一组 `-Dfmt_SOURCE=SYSTEM` 构建（验证 SYSTEM 路径）。
5. README 增加"依赖来源"小节。

**验收**：
- 所有现有 CI 通过（默认行为零回归）。
- `cmake -B build -Dfmt_SOURCE=SYSTEM -Dfmt_ROOT=/path/to/fmt` 能成功构建并运行测试。

### Phase 2 — 核心依赖

按 issue 的优先级，先压缩库再 Arrow：

1. `zstd` / `zlib` / `lz4` / `snappy` / `re2` / `RapidJSON`（独立、被 Arrow 依赖）。
2. `Arrow`（含 Parquet）—— 同时实现传递依赖软约束。
3. `glog` / `TBB`。

每个依赖的工作量约等于：1 个 Find module（~50 行）+ `build_xxx` 内 ALIAS（~3 行）+ CI 一组 SYSTEM build。

### Phase 3 — 可选依赖

`orc` + `protobuf`、`gtest`。`avro` 因 `versions.txt` 中固定为 git commit hash（`PAIMON_AVRO_BUILD_VERSION=c499eefb...`），SYSTEM 路径需在文档中注明仅支持 release 版本。

### Phase 4 — 高级特性

1. `PAIMON_DEPENDENCY_USE_SHARED` 真正生效（决定 `find_library` 时偏好 .so 还是 .a）。
2. `PAIMON_STRICT_DEPENDENCY_VERSIONS` —— 启用时校验 SYSTEM 版本与 BUNDLED 期望版本不一致就 fail。
3. CONDA prefix 自动接入 `CMAKE_PREFIX_PATH`（实际逻辑已在 `resolve_dependency` 草案中）。
4. 子目录 CMakeLists 完成裸名 → 命名空间名迁移，删除 ALIAS。

## 8. 兼容性策略

| 场景 | 行为 |
| --- | --- |
| 不传任何新选项 | 与今天完全一致：所有依赖 BUNDLED。 |
| 仅传 `-DPAIMON_DEPENDENCY_SOURCE=AUTO` | 新行为：找到则用，找不到回退 BUNDLED。 |
| 仅传 `-Dfmt_SOURCE=SYSTEM` | 仅 fmt 走 SYSTEM，其他保持默认。 |
| `cmake -B build` 现有命令 | 输出末尾增加 "Thirdparty options:" 区块；其他不变。 |

不破坏：`versions.txt`、`download_dependencies.sh`、`build_and_package.sh`、所有 `add_paimon_lib(... DEPENDENCIES arrow ...)` 调用。

## 9. 默认值的选择

issue #103 评论中 @zjw1111 倾向 `AUTO`，但本设计建议 Phase 1 仍保持 `BUNDLED`：

| 方案 | 优点 | 风险 |
| --- | --- | --- |
| 默认 `BUNDLED` | 行为零回归，CI 完全可控 | 用户需显式开 AUTO 才能享受加速 |
| 默认 `AUTO` | 与 Arrow 一致，开箱即用 | 不同环境下结果不一样，SYSTEM 库版本若不兼容会让默认构建失败 |

建议：Phase 1 ~ Phase 3 期间默认 `BUNDLED`，待全部依赖支持完毕、CI 在两种模式下都稳定后，Phase 4 再切换为 `AUTO`。

## 10. 风险与开放问题

1. **Arrow 静态库链接顺序**：Arrow + arrow_bundled_dependencies + parquet 的静态归档在 SYSTEM 模式下不存在（Arrow 系统包通常仅装 .so + 共享版传递依赖），可能导致符号解析顺序与 BUNDLED 不一致。需要在 `FindArrowAlt.cmake` 中显式 link parquet/arrow_dataset。
2. **`avro` git hash 版本**：`versions.txt` 中固定的 commit 与 SYSTEM 包版本无法对齐，SYSTEM 路径建议 `_REQUIRED_VERSION` 留空，由用户负责。
3. **Boost SYSTEM 支持**：当前仅 `LUCENE` 启用 boost，Lucene 实现紧密依赖 `boost::regex` 等，SYSTEM Boost 风险较高。本设计 Phase 1-3 全部 `FORCE_BUNDLED`。
4. **patch 与 SYSTEM 的协议**：用户使用 SYSTEM Arrow / ORC 时若遇到 BUNDLED 中已 patch 修复的问题，需自行升级到包含修复的发行版本，文档需明确列出当前 patch 解决的问题清单。
5. **macOS / Linux 行为差异**：`FindXxxAlt.cmake` 的 `find_library` 在不同平台下的搜索路径需测试覆盖（`PATH_SUFFIXES lib lib64`）。

## 11. 验证计划

每个 Phase 至少覆盖以下 CI 矩阵：

| 配置 | 期望 |
| --- | --- |
| `-DPAIMON_DEPENDENCY_SOURCE=BUNDLED`（默认） | 全套 unit + integration test 通过 |
| `-DPAIMON_DEPENDENCY_SOURCE=AUTO` | 全通过（SYSTEM 库满足版本时使用 SYSTEM） |
| `-DPAIMON_DEPENDENCY_SOURCE=SYSTEM`（容器内预装） | 全通过 |
| 单依赖 `-D<Pkg>_SOURCE=SYSTEM`，其他 BUNDLED | 全通过 |
| `PAIMON_ENABLE_ORC=ON` + Arrow=SYSTEM + ORC=BUNDLED | 全通过，无 zstd 双份警告 |

CI 中容器镜像可基于 `apt-get install libarrow-dev libzstd-dev ...` 提前预装系统库。

## 12. 参考实现

- Apache Arrow 的 `resolve_dependency`：[apache/arrow cpp/cmake_modules/ThirdpartyToolchain.cmake](https://github.com/apache/arrow/blob/main/cpp/cmake_modules/ThirdpartyToolchain.cmake#L252-L366)
- Apache Arrow 的 `*_SOURCE` 选项：[apache/arrow cpp/cmake_modules/DefineOptions.cmake](https://github.com/apache/arrow/blob/main/cpp/cmake_modules/DefineOptions.cmake#L456-L464)
- Apache Arrow 的 Find modules：[apache/arrow cpp/cmake_modules/](https://github.com/apache/arrow/tree/main/cpp/cmake_modules)

## 13. 下游受益场景：Apache Doris 集成

Doris 已经集成 paimon-cpp，但目前存在 **Arrow 双份** 问题：

- Doris 自身依赖 Arrow（一份）。
- Doris 链入的 paimon-cpp 内部又静态打包了 paimon-cpp 自己的 Arrow 17.0.0（第二份）。

虽然 paimon-cpp 已经做了 `-fvisibility=hidden` + 静态归档，运行时动态符号表不会冲突，但仍存在：

1. **二进制膨胀**：同一份 Arrow 代码在最终 Doris 二进制中存在两份。
2. **API 边界 ODR 风险**：paimon-cpp 公共头文件目前会暴露 `arrow::RecordBatch` / `arrow::Array` 等类型；Doris 与 paimon-cpp 若分别针对不同 Arrow 版本编译这些类型，类型布局可能不一致，即便链接通过也存在 ODR 违规。

本设计交付后，Doris 可以通过：

```bash
cmake -B build \
  -DArrow_SOURCE=SYSTEM \
  -DArrow_ROOT=<doris 的 arrow 安装路径>
```

让 paimon-cpp 复用 Doris 已有的 Arrow，最终二进制只保留一份 Arrow，并消除 API 边界的 ODR 风险。这也是本 issue 优化最直接的下游收益场景之一。

### 13.1 Doris 接入需要 paimon-cpp 这边解决的问题

**这些是落地 Doris SYSTEM Arrow 的 hard prerequisites，必须在 Phase 2（Arrow 接入）专门验证一次**：

1. **Arrow 版本对齐**。paimon-cpp 当前锁定 Arrow 17.0.0（[third_party/versions.txt](../third_party/versions.txt)）。需在 `FindArrowAlt.cmake` 中声明可接受的版本范围，并验证 Doris 当前使用的 Arrow 版本落在区间内。跨大版本时要做一致性测试。

2. **patch 对齐**。paimon-cpp 给 Arrow 打了 213 行的 [`cmake_modules/arrow.diff`](../cmake_modules/arrow.diff)，Doris 的 Arrow 不会带这些 patch。Phase 2 时需要逐条 audit：

   - 哪些 patch 已被 Arrow 上游合入 → 提升 `_REQUIRED_VERSION` 下限即可
   - 哪些可以改为 build flag 注入（如 `-include cstdint`，[ThirdpartyToolchain.cmake:1131](../cmake_modules/ThirdpartyToolchain.cmake#L1131) 已是这种形式）
   - 哪些是 paimon-cpp 必需且无法替代的私有改动 → 成为 SYSTEM 路径的 hard blocker，需要推动上游合入或重新设计 paimon-cpp 对 Arrow 的使用方式

3. **传递依赖对齐**。Arrow 依赖 zstd / snappy / lz4 / zlib，本设计 §5.6 的传递依赖软约束会在 `Arrow_SOURCE=SYSTEM` 时自动把这些设为 SYSTEM，确保 Doris 二进制中也只保留一份压缩库。

4. **公共 API 中的 Arrow 类型**。这一项不需要本 issue 修改，但需要在 Doris 接入文档中明确说明：使用 SYSTEM Arrow 时，Doris 与 paimon-cpp 必须用同一份 Arrow 头文件编译，否则公共 API 中的 Arrow 类型布局不一致会导致 UB。

### 13.2 工作清单（Phase 2 完成后跟进）

- [ ] 完成 §13.1 中 1-3 项验证，确认 Doris 主线 Arrow 版本可作为 SYSTEM 输入
- [ ] 在 Doris 侧切换 paimon-cpp 构建参数为 `-DArrow_SOURCE=SYSTEM -DArrow_ROOT=...`
- [ ] 测量切换前后 Doris 二进制大小、加载时间、Arrow 相关代码路径行为
- [ ] 在 paimon-cpp README 增加 "Integrating with Doris" 段落，沉淀经验
