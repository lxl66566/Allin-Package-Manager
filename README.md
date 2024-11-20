# AllIn Package Manager

因为不满 Nix/NixOS 的包管理，因此尝试设计（开发：有生之年）一款属于我自己的 package manager。参考我的[最初的文章](https://absx.pages.dev/coding/package_manager.html)

以下是该包管理器的设计部分。

## 基本信息

- 不限制打包脚本语言。一阶段支持：Bash，Python, Fish, lua, js/ts（bun）.
- 托管于 Github，使用 CI 进行验证；
- 使用类似 scoop 的 buckets 进行分级。官方的仓库为 main，用户可以简单地维护第三方 bucket。
- 两阶段安装，类似 pacman：
  1. 从 manifest 构建包，打成单 zstd，并验证 hash。
  2. 解压，并根据内部的 info 进行安装。该 info 需要尽可能简短。
  - 两阶段安装的目的：1. 解耦 2. 方便后期的去中心化
- 编写测试函数，由 CI 自动更新

## 结构

### buckets

每一个 bucket 所包含的字段：

- name：名称，可以用户自定义
- repo：git 仓库地址，必须唯一
- branch：分支名称，默认为 github default branch
- rev（Optional）: commit hash，设置后可以锁死 bucket 版本
- description（Optional）：描述，可以用户自定义
- download_link（Optional）：仓库下载链接。原因：直接下载 zip 存档要比 git clone 更快。

### package

每个 package 所包含的字段：

- name：名称，必须唯一，不允许使用特殊字符
- marker：安装的包的标识。如果把 package 比作 repo，那么标识就是 branch。计划支持（下载时若未指定，按照以下顺序查找可用标识）：
  - bin：直接下载二进制
  - src：从源码编译
  - 其他自定义标识，（例如 libreoffice 的 fresh 与 still 就算是标识）
- version：包的版本号
- pkgrel：与 archlinux 相同，标识包的单调递增的发行版本号，和 version 共同区分包版本。用于漏洞修复。
- 其他与 marker 相同的字段。若 marker 的字段不存在，则自动 fallback 到 package 中的字段。

#### marker

我们通过 `package@marker` 来表示一个唯一的 marker。

每一个 marker 需要包含的字段：

- marker_name：marker 的名称
- hash：marker 打出的 zstd 压缩包的 hash
- dependencies
  - runtime_deps：`list[str]`, marker 的运行时依赖项。
  - build_deps：marker 的构建时依赖项。
- scripts：打包脚本
  - pre_build：构建前脚本
  - build：构建脚本
  - post_build：构建后脚本
  - install_info：安装信息
  - 每一个脚本除了内容，还额外包含：
    - language：脚本的语言
    - interpreter_version（Optional）：脚本的解释器版本
    - 这些可以通过 shellbang 和头部注释获取。
- test：测试脚本。如果测试脚本 exit code 不为 0，则认为测试失败。

### info

info 是打包到 zstd 压缩包中的信息。包含的字段：

- bucket, name, marker, version，与包一致
- paths，zstd 压缩包中所有文件/文件夹的去向。
