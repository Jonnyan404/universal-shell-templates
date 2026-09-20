# universal-shell-templates

模板库仓库：**模板源（`templates/`）+ 生成产物（`registry/`）**。
`registry/` 是纯静态 JSON，可直接用 raw 地址拉取，不需要任何服务端或构建步骤。

源地址：

```
https://raw.githubusercontent.com/Jonnyan404/universal-shell-templates/main/registry/
```

## 目录结构

```
.
├── templates/            # ① 单一事实来源：一个文件一个模板（<id>.json）
├── registry/             # ② 生成产物（提交进仓库，供客户端 raw 直连）
│   ├── manifests.json    #    索引：revision + categories + 模板轻量元信息
│   ├── manifests.sig     #    manifests.json 的 Ed25519 签名
│   └── templates/<id>.json
└── example-third-party/  # 三方源示例（自建源照抄这个结构即可）
    ├── manifests.json
    └── templates/<id>.json
```

## 两级结构（惰性拉取）

- 客户端先拉 `registry/manifests.json`：轻量索引，只有 `id/name/category/description/repo`。
- 用户选中某个模板时才拉 `registry/templates/<id>.json`：完整定义。
- `manifests.json` 的 `revision` 每次生成都会变，客户端据此判断缓存是否过期。
- `registry/templates/<id>.json` 是 `templates/<id>.json` 的逐字节快照，两者必须一致；
  `manifests.json` 里 `templates[].id` 也必须与 `<id>.json` 一一对应。

## 加 / 改一个模板

1. 在 `templates/` 新增或修改 `<id>.json`：**文件名必须等于文件里的 `id` 字段**。
   - 字段语义见 [example-third-party/README.md](example-third-party/README.md)。
   - 安全默认值：网络监听类字段一律 `127.0.0.1` + 高位端口。
   - 不要放 `repo` 为空的本地程序模板：那是单机私有路径，没有分发意义。
2. 同批次更新 `registry/`（三部分缺一不可，否则客户端会对不上或验签失败）：
   - `registry/manifests.json`：重算索引并更新 `revision`；
   - `registry/templates/<id>.json`：同步快照（模板被删除时同步删掉对应快照）；
   - `registry/manifests.sig`：对**新的 `manifests.json` 字节**重新做 Ed25519 签名。
3. `templates/` 与 `registry/` 一起提交、一起 push。

> 生成与签名工具**不在本仓库内**：本仓库刻意保持零代码、零构建依赖，只有模板与产物。
> 生成/签名请在本地用模板库工具完成（索引字段就是上面那几个轻量字段，签名对象是
> `manifests.json` 的原始文件字节）。

## 客户端地址的用法

- 完整源地址 = `<base>/manifests.json` 与 `<base>/templates/<id>.json`，因此 `<base>` 必须以 `/` 结尾。
- raw 直连示例：`https://raw.githubusercontent.com/Jonnyan404/universal-shell-templates/main/registry/`
- 若改用 GitHub Pages，把 Pages 根设为仓库根目录即可，源地址形如 `https://<user>.github.io/universal-shell-templates/registry/`。
- 客户端缓存用 ETag/If-None-Match 增量校验；`revision` 变化即视为内容更新。

## 自建源

想让自己的模板库被客户端拉取，只需把 `example-third-party/` 那种两级结构丢到任意静态托管上，
再在客户端的「模板库 → 源」里把 base 地址加上（可选：为该 base 配置 Ed25519 公钥以强制验签）。

## 许可

MIT，见 [LICENSE](LICENSE)。

