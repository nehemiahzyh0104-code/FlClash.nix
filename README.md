# FlClash.nix
这是Flclash在Nixos上的非官方打包


初始打包文件来自 nixpkgs 的 FlClash 0.8.66 配方（NixOS/nixpkgs#351318，合入提交 758d99d）。感谢原打包及后续维护的贡献者。FlClash 应用本身遵循其上游许可证 [旧配方的版权和许可](LICENSES/COPYING) · [初次收录 PR](https://github.com/NixOS/nixpkgs/pull/351318) · [FlClash 上游许可证](https://github.com/chen08209/FlClash/blob/v0.9.5/LICENSE)


## 项目说明

- 请从上游稳定版 `v0.9.5` 的源码构建。旧的 `0.8.94` 配方提供 Nix 依赖、桌面入口和图标的处理的参考 [上游发布](https://github.com/chen08209/FlClash/releases/tag/v0.9.5) · [历史配方](https://github.com/NixOS/nixpkgs/blob/17938e7d40fd7b6862095ce52796a5a3116ca9bd/pkgs/by-name/fl/flclash/package.nix)
- 工具链优先使用 Flutter 3.50、Go 1.27 和 Rust 1.99。请把实际使用的提交锁进 `flake.lock`。 [候选 nixpkgs 提交](https://github.com/NixOS/nixpkgs/commit/60527ba01c99b47c515fad87f16f060e54b12d25)
- 请一起提交 Go 核心和 Rust helper，普通代理模式 [上游 Linux bundle 布局](https://github.com/chen08209/FlClash/blob/v0.9.5/linux/CMakeLists.txt#L110)


# 构建步骤

1. **固定源码和依赖** 在 `pkgs/by-name/fl/flclash/package.nix` 里固定 `v0.9.5`、源码哈希和两个子模块 `core/Clash.Meta`、`plugins/prism_ui`。用该 tag 的 `pubspec.lock` 更新同目录的 `pubspec.lock.json`，让 Nix 管理 Pub、Go 和 Cargo 的依赖获取。主构建在 sandbox 内使用已获取的依赖。 [上游子模块](https://github.com/chen08209/FlClash/blob/v0.9.5/.gitmodules) · [Nixpkgs Dart 打包说明](https://github.com/NixOS/nixpkgs/blob/master/doc/languages-frameworks/dart.section.md)


2. **编译 Go 核心** 用 `buildGoModule` 在 `core/` 下构建 `./cmd/core`，沿用上游的 `with_gvisor`、`CGO_ENABLED=0` 和 `build_config.yaml` 中的链接参数及核心版本设置。输出名为 `FlClashCore`。 [上游 Go 构建逻辑](https://github.com/chen08209/FlClash/blob/v0.9.5/plugins/setup/setup_hooks/lib/src/go_builder.dart) · [上游构建配置](https://github.com/chen08209/FlClash/blob/v0.9.5/build_config.yaml)


3. **编译 Rust helper** 等 Go 核心完成 Nix 的输出处理后再算 SHA-256。用 `services/helper/Cargo.lock` 构建 `FlClashHelperService`，将 `CORE_NAME=FlClashCore` 和 `CORE_SHA256` 传给构建，并用同一个 SHA-256 生成带 `coreSha256` 字段的 `manifest.json`。把这三个文件放入 `libclash/linux/`，由上游 CMake 装到与应用可执行文件同级的目录。最终产物里的核心必须和 manifest 中的哈希一致，请务必避免后续 strip 或 ELF 修改改变哈希。 [上游构建顺序](https://github.com/chen08209/FlClash/blob/v0.9.5/plugins/setup/setup_hooks/lib/src/build.dart) · [helper 构建参数](https://github.com/chen08209/FlClash/blob/v0.9.5/services/helper/build.rs)


4. **验证 Rust 动态库和 Flutter 接口** 用 `plugins/rust_api/rust/Cargo.lock` 单独构建 `librust_api.so`，先验证 Dart 钩子能按原资产标识注册这份库，且程序启动时能加载它。正式组装使用 `flutter350.buildFlutterApplication`，传入 `--dart-define=APP_ENV=stable`。将 `pubspec.yaml` 的 `hooks.user_defines.setup.build_assets` 设为 `false`，让 CMake 使用上一步提供的文件。SQLite 钩子可设为 `source: system`，由 nixpkgs 提供运行时库，并实测配置数据库可读写。 [上游 Rust 钩子](https://github.com/chen08209/FlClash/blob/v0.9.5/plugins/rust_api/hook/build.dart) · [Nixpkgs Flutter builder](https://nixos.org/manual/nixpkgs/unstable/#ssec-dart-flutter) · [SQLite 钩子选项](https://github.com/simolus3/sqlite3.dart/blob/main/sqlite3/doc/hook.md#system-provided-sqlite)


5. **补充本地构建入口和桌面集成** 添加 `flake.nix` 和 `flake.lock`，让 `.#flclash` 调用同一份 `package.nix`。核对桌面启动项的 `Exec`、图标和实际可执行文件名 `FlClash`。 [上游 Linux 可执行文件定义](https://github.com/chen08209/FlClash/blob/v0.9.5/linux/CMakeLists.txt#L7)


# 本地构建

本地入口补齐后，在启用 Nix sandbox 的环境里，从仓库根目录运行以下命令。`sha256sum` 使用 `manifest.json` 中的 `coreSha256`

```sh
nix build -L .#flclash
sha256sum result/app/flclash/FlClashCore
cat result/app/flclash/manifest.json
./result/bin/FlClash
```

# 目标

- 窗口、.desktop、图标和托盘均正常
- 导入的配置能正常被保存
- 系统代理和TUN正常
- 停止代理后网络恢复正常

# 提交和发布测试

编译和打包完成后，请交付打包提交，并使用同一提交构建闭包预发行版


1. **提交 `pkgs/`** 在工作分支提交 `pkgs/by-name/fl/flclash/` 下的 `package.nix`、依赖锁和构建所用的补丁，同时提交根目录的 `flake.nix` 和 `flake.lock`


2. **导出运行时闭包** 提交构建，用下面的 Bash 命令把 `FlClash` 和它依赖的 Nix store 路径一起导出 [Nix 闭包导出说明](https://nix.dev/manual/nix/2.34/command-ref/nix-store/export.html)

   ```bash
   set -euo pipefail
   nix build -L .#flclash
   store_path=$(readlink -f result)
   archive=flclash-0.9.5-x86_64-linux-closure.nar.zst
   closure_paths=$(nix-store --query --requisites "$store_path")
   mapfile -t paths <<< "$closure_paths"
   nix-store --export "${paths[@]}" | zstd -T0 -o "$archive"
   printf '%s\n' "$store_path" > store-path.txt
   sha256sum "$archive" store-path.txt > SHA256SUMS
   ```


3. **预发行版** 发布闭包。上传闭包归档、`store-path.txt` 和 `SHA256SUMS`


4. **测试** 下载release文件，导入store测试 [Nix 闭包导入说明](https://nix.dev/manual/nix/2.34/command-ref/nix-store/import.html)

   ```bash
   set -euo pipefail
   sha256sum -c SHA256SUMS
   zstd -dc flclash-0.9.5-x86_64-linux-closure.nar.zst | sudo nix-store --import
   store_path=$(cat store-path.txt)
   "$store_path/bin/FlClash"
   ```