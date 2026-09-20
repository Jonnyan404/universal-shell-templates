# 三方源示例（example-third-party）

这是一个**自建模板源的示例**，展示如何托管自己的模板仓库，供客户端拉取。

> 官方源（本仓库 `registry/`）就是按同样的两级结构组织的。照抄结构、换成你自己的模板即可。
> 只想加/改官方模板请直接改本仓库 `templates/`（见仓库根 README），不必新建源。

## 目录结构（两级惰性拉取）

```
example-third-party/            # 本示例源（可整体丢到任意静态托管上）
├── manifests.json              # ① 索引：分类 + 每个模板的元信息
└── templates/
    └── fd.json                 # ② 具体模板：id 为 fd 的 Program 定义
```

- 客户端只拉 `manifests.json` 索引（轻量），用户点某个模板时才惰性拉取 `templates/<id>.json`。
- `manifests.json` 里 `templates[].id` 必须与 `templates/<id>.json` 的 `id` 一致。
- **`revision` 每次更新内容后要改**（建议 `rev-<unix时间戳>`），否则本地缓存可能不刷新。

## 如何自建

1. 把本目录结构上传到任意静态托管（如 GitHub Pages、对象存储、`python -m http.server`）。
2. 保守起见，模板内 `bind` 等监听字段默认 `127.0.0.1`，端口用高位端口。
3. 在客户端的「模板库 → 源」设置里，把 `https://你的域名/路径/` 加为源。
   - 可选：配置源对应的 Ed25519 公钥；命中公钥的源会强制验签 `manifests.sig`。

## 模板字段说明（Program）

`templates/<id>.json` 各字段：

| 字段 | 说明 |
| --- | --- |
| `id` / `name` / `description` | 标识与展示文本 |
| `repo` | GitHub `owner/repo`，用于解析最新版本与下载链接；留空 + 配置 `source` 时走 HTTP 直链源 |
| `source` | 可选下载源描述（见下方「HTTP 直链源」）：`kind="http"` 时不再依赖 GitHub API |
| `binary` | 解压后可执行文件内部名 |
| `assets.<os>.candidates` | 按顺序尝试的下载文件名模板（支持 `{name}` `{version}` `{arch}`）|
| `assets.<os>.urls` | HTTP 直链源的下载 URL 模板列表（每项一个，按序尝试，支持 `{version}` `{arch}` `{os}` `{name}` `{ext}`）|
| `assets.<os>.mode` | `single`（单裸二进制）/ `whole`（整包解压）|
| `assets.<os>.member` | 压缩包内可执行文件成员名（支持 `{version}` `{arch}` 等占位符）|
| `arch_map` / `os_map` | 把客户端的架构/系统名映射成 release 里的命名 |
| `fields` | 用户填写字段（`string` / `file` / `directory` / `boolean` / `autostart`）|
| `args` | 启动参数模板，`{字段key}` 会被替换为用户填写的值 |

> **自启动**：自启动由客户端统一管理（`program-autostart.json`），模板**不必**再声明 `autostart` 字段；
> 客户端会对每个程序（含内置/用户/该源导入的）独立提供开机自启开关。

## HTTP 直链源（方案 A：`source.kind="http"`）

适合自建文件站、厂商直链、Electron `latest.yml`、以及想绕过 GitHub release API（被墙/限流）的场景。

```json
"source": {
  "kind": "http",
  "version_url": "https://…/latest.json",
  "version_json_path": "version",
  "version_regex": "tag=v([0-9.]+)",
  "sha256_url": "https://…/app-{version}.sha256"
}
```

- 版本探测三选一：`version_json_path`（JSON 点路径）> `version_regex` > `version_url` 响应整段文本。
- 下载走 `assets.<os>.urls` 直链模板，**不查 GitHub API**；`member` 支持 `{version}`/`{arch}` 等占位。
- 校验：优先模板钉住的 `check_sha256`，其次 `sha256_url`（支持 `{version}` 等占位）；两者都无 = **显式免检**（不会静默降级）。
- `repo` 留空、`source.kind=http` 时，下载/更新/版本检查全部走该直链源（不再查 GitHub API）；
  `repo` 留空且无 `source` = 本地程序（见下）。

**本地程序（不进模板源）**：`repo` 留空 + `binary` 直接填本机可执行文件路径即「本地程序」，客户端不下载、不更新。
路径是单机私有的，分发给别的机器没有意义，因此不要放进模板源——在客户端的编辑器里自建即可。

## 示例模板说明

- `templates/fd.json` 基于真实项目 `sharkdp/fd`（Rust 写的 find 替代品），其 GitHub
  release 命名形如 `fd-v10.2.0-aarch64-apple-darwin.tar.gz`，与 `candidates` 一一对应。
  导入后不联网也能按字段填 `{pattern}`、`{dir}` 启动。
- `templates/uv.json` 基于 `astral-sh/uv`，演示 HTTP 直链源：`version_url` 用
  raw.githubusercontent 上的 `Cargo.toml` 正则探版本，下载走 `releases/latest/download/…` 直链
  （绕过 GitHub release API），并用同名的 `.sha256` 资产做校验。导入即用，更新/安装均不依赖 GitHub token。

## 参考

- 官方源（本仓库根目录的 `templates/` + `registry/`）就是同样的两级结构，可直接对照。
- 本示例的 `manifests.json` 只含 `fd` / `uv` 两个模板，仅用于演示；格式若与官方索引不一致，
  以官方 `registry/manifests.json` 为准。
