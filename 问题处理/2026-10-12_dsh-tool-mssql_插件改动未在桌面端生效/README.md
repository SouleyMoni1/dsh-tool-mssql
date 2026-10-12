# dsh-tool-mssql 插件改动未在桌面端生效

- 对象：dsh-tool-mssql（DSH 插件，仓库 F:\EdenOS\AI\dsh-tool-mssql）
- 日期：2026-10-12
- 环境：Windows，DSH 桌面端（DeepSeek Harness.exe）+ desktop profile

## 现象

插件改了三次（设置页、排序 + 行内编辑、编辑表单显示明文密码），网页端 GUI 里都看不到新行为，怀疑插件只装到了 web profile、没装到桌面端。

## 原因

插件本来就挂在桌面端，是**桌面端进程一直没重启**：

| 项 | 时间 |
| --- | --- |
| DeepSeek Harness.exe 启动（pid 1716，占用 :19387） | 2026-10-08 14:21:51 |
| 最后一次构建 lib/*.js、client/client.js | 2026-10-10 00:57 |
| 最后一次提交 7c5028c | 2026-10-10 00:58 |

DSH 在**启动时**组装插件：宿主的包元数据（含 client 半区）按 Loader specifier 缓存到本次进程结束，客户端 bundle 也只在启动时加载。所以只要进程没重启，磁盘上再新的代码也不生效。

插件挂载链路本身是好的（三条都验过）：

1. `~/.dsh/profiles/desktop/package.json` → `dsh.profile.bundles` 里有 `dsh-tool-mssql`，deps 是 `link:../.local-links/dsh-tool-mssql`；
2. `~/.dsh/profiles/desktop/node_modules/dsh-tool-mssql` → 符号链接 → `~/.dsh/profiles/.local-links/dsh-tool-mssql` → `/f/EdenOS/AI/dsh-tool-mssql`（就是仓库本身）；
3. 顺着这条链接读到的 `lib/settings-rpc.js`、`client/client.js` 里都有新代码（`dsnEdit`）。

## 解决方案

重启 DSH 桌面端（完全退出再启动，不是刷新页面）。

排查同类问题的固定动作：

```bash
# 进程启动时间
wmic process where "ProcessId=<pid>" get CreationDate /value
# 插件构建时间
ls -l --time-style=full-iso lib/settings-rpc.js client/client.js
# 桌面端实际解析到的文件里有没有新代码
grep -c <新符号> ~/.dsh/profiles/desktop/node_modules/dsh-tool-mssql/lib/*.js
```

进程启动时间早于构建时间 → 没重启，重启即可；晚于构建时间 → 才需要往挂载链路 / profile 配置上查。

## 验证结果

- 桌面端 profile 的 bundles 与软链接指向仓库，链路上读到的文件含新代码：**已确认**
- 桌面端进程启动时间 2026-10-08 早于构建时间 2026-10-10：**已确认** → 结论「需重启」
- 重启后：设置 → SQL Server 数据源 → 编辑，连接字符串里直接显示明文密码；列表行显示 `******`

---
涉及表：无
源码位置：`src/settings-rpc.ts`（宿主 RPC）、`client/client.js`（设置页）、`package.json` 的 `dsh.client`
影响面：仅本机 DSH 桌面端 / web profile 的插件加载时机，无生产影响
遗留问题：无
