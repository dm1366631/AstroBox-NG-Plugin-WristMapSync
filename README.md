# FetchD 本地地图包

一个 AstroBox NG 手机侧插件，只做一件事：把手环地图快应用发出的 `local://`
取包请求，映射到手机插件数据目录里的离线地图包，并保留 `/packs/favs.json`
收藏点链路。

- 插件**不代手环访问网络**，也不从云端下载任何地图数据；
- 地图包由电脑侧浏览器工具 `2026-11-13/web-pack-builder` 生成 `.obm` 文件，
  在插件页面导入到手机；
- 本地包服务默认开启，安装后无需在界面里做任何授权设置。

## 来源与许可

本仓库基于 [AstroBox-NG-Plugin-MiWear-InterconnectFetch](https://github.com/AstralSightStudios/AstroBox-NG-Plugin-MiWear-InterconnectFetch)
（作者 Searchstars / AstralSight Studios，MIT License）二次开发：

- `src/` 下除 `proto_v5.rs`、`proto_v5_core.rs`、`localpack.rs`、`obm_pack.rs`
  等新增文件外，基础实现沿用上游，版权与 MIT 声明保留在 [LICENSE](./LICENSE)；
  新增文件头同样标注来源。
- 新增内容：协议 v5（`__hs5__` / `fd` / `fd-h` / `fd-c` / `fd-e` / `fd-x` /
  `fd-a`）、`local://` 取包、OBM 导入、收藏点接口。
- 原样分发时请同时保留本 README 的「来源与许可」一节。

## 当前状态（如实）

| 项目 | 状态 |
| --- | --- |
| 插件编译 / 打包 | 已通过，产物见 `dist/` |
| v5 帧格式（宿主测试） | 已通过：2 MiB → 256 帧、CRC、奇数长度、错误帧 |
| `local://` 取包（宿主测试） | 已通过：清单、元数据、瓦片、道路包 |
| OBM 导入（宿主测试） | 已通过：STORE / deflate、大包、路径穿越拒绝、清单合并 |
| 真机联调 | 未在本次改动中重新验证，自测步骤见 [docs/REPLACE.md](./docs/REPLACE.md) |

协议细节见 [docs/PROTOCOL-v5.md](./docs/PROTOCOL-v5.md)，收藏点格式见
[docs/FAVORITES.md](./docs/FAVORITES.md)。

## 环境准备

### 1. 安装 Rust

👉 https://www.rust-lang.org/learn/get-started

### 2. 安装 Python 3

构建脚本使用 Python 编写，需要安装 Python 3。

👉 https://www.python.org/downloads/

### 3. 安装 wasm32-wasip2 编译目标

```bash
rustup target add wasm32-wasip2
```

## 构建

```bash
# Debug 构建到 dist 文件夹
python scripts/build_dist.py

# Release 构建到 dist 文件夹
python scripts/build_dist.py --release

# Release 构建并打包为 .abp 插件包
python scripts/build_dist.py --release --package
```

构建产物会输出到 `dist/` 目录，包含编译后的 wasm 文件、`manifest.json` 和图标。

使用 `--package` 时会额外生成一个 `.abp` 文件，可直接通过 AstroBox 安装。

## 地图包（`local://`）

地图包的解析、出瓦片、`road.bin` 与 `.obm` 打包由电脑侧浏览器工具
`2026-11-13/web-pack-builder` 完成。插件不内置构建能力，也不从远端服务器
拉取或缓存地图包归档。

插件只负责把手环发来的 `local://` 请求映射到插件数据目录下已有的包：
根部的 `index.json` 提供清单，`packs/` 下是包文件，按清单里的相对路径直接读取。
地址规则与错误码见 [docs/PROTOCOL-v5.md](./docs/PROTOCOL-v5.md)。

### 导入 `.obm`

插件页面的「导入地图包（.obm）」可以选一个 `web-pack-builder` 产出的 `.obm`
（根 `index.json` + `packs/`），直接解到插件数据根目录：

- 选包时只让宿主把文件 `copy_to` 到 `imports/`，不读进 wasm 内存（大包
  远超插件 8 MiB 单文件限额）。
- 中央目录流式解析，支持大包与超过 65535 个条目；路径按 `[A-Za-z0-9._-]`
  校验并拒绝 `..`，只接受根 `index.json` 和 `packs/**`。
- 每片解析 8192 条目 / 解压 512 KiB / 256 个文件后返回，由 Timer 事件续跑，
  不阻塞宿主事件循环。
- 先解到 `.import/tmp`，完成后合并 `index.json`（同名 `dir` 覆盖、其余保留）
  再把 `packs/<dir>` 整目录换入；导入结束会删掉 `imports/` 里那份副本。

### 测试

纯逻辑（协议帧、`local://` 清单与文件读取、收藏点、OBM 导入）有独立测试包：

```bash
cd tools/packcache-tests
cargo test
```
