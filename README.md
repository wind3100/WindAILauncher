# WindAIlauncher

**零第三方依赖**的 Minecraft: Java 版启动器：内置多下载源（官方 / BMCLAPI 镜像 / 自定义）、
Microsoft 登录与离线模式、Java 自动搜索与自动下载、模组与整合包安装、常用文件夹一键打开，
并且所有功能都暴露为本地 HTTP JSON 接口。

> 设计目标就是"官方源连不上时也能把游戏装下来并启动"：所有下载地址都经过**源解析层**，
> 官方失败自动回退镜像，镜像规则可在界面上随意改，也可以整体换成你自己的源。

---

## 一、快速开始

1. 需要 **Node.js 18+**（本项目只用 Node 内置模块，不需要 `npm install`）。
2. 双击 **`McOpen.cmd`**（会启动本地服务并打开一个独立应用窗口）。
   也可以在终端里运行：

   ```bat
   node server\index.mjs              :: 默认 http://127.0.0.1:25564
   node server\index.mjs --port 26000 --no-open
   ```

3. 首次使用建议顺序：**账号**（加离线号或登录 Microsoft）→ **游戏版本**（装原版 + 加载器）→
   **实例** 页新建实例并启动。

数据都放在 `data/` 下（见下文目录说明），可整体备份/搬移。

---

## 二、目录结构

```
McOpen/
├─ McOpen.cmd              双击启动（自动找 node、启动服务、开窗口）
├─ package.json
├─ server/                 后端（全部 Node 内置模块）
│  ├─ index.mjs            HTTP 服务 / 路由 / SSE / 任务编排
│  ├─ config.mjs           配置读写（data/config.json）与默认值
│  ├─ net.mjs              源解析（SourceResolver）+ 并发下载器（重试/校验/进度）
│  ├─ mcmeta.mjs           版本清单、版本 JSON、Fabric/Quilt/Forge 元数据
│  ├─ install.mjs          版本安装：客户端 jar / 依赖库 / 资源 / natives / 日志配置
│  ├─ gameinstall.mjs      原版 + 加载器编排（Forge/NeoForge 走官方 installer）
│  ├─ java.mjs             Java 扫描 / 探测 / Adoptium 自动下载
│  ├─ auth.mjs             离线账号 + Microsoft OAuth2（XBL → XSTS → Minecraft）
│  ├─ launch.mjs           启动参数构建、classpath、natives、进程管理
│  ├─ folders.mjs          打开模组/存档/资源包等目录
│  ├─ mods.mjs             Modrinth / CurseForge / 自定义源搜索与安装、整合包解析
│  └─ zip.mjs              零依赖 ZIP 读/解压/打包
├─ web/                    前端（原生 HTML/CSS/JS，无构建步骤）
└─ data/                   运行时数据（自动创建）
   ├─ config.json          全部配置（含源规则、账号令牌、Java 设置）
   ├─ shared/              共享游戏文件：libraries / assets / versions / natives
   ├─ instances/<id>/      每个实例的独立游戏目录（mods/saves/resourcepacks/…）
   ├─ runtime/             自动下载的 Java
   ├─ packs/               上传的整合包
   ├─ logs/                每次启动的游戏日志
   └─ cache/               版本清单等缓存（断网时仍能显示版本列表）
```

---

## 三、下载源（本项目最重要的部分）

### 3.1 实测结论（决定默认策略）

下表来自本项目开发时的**实测**（同一台机器、同一天不同时段）；注意这类网络是**动态变化**的，
所以启动器的策略是"多候选源 + 动态健康度排序"，而不是写死某一个源：

| 地址 | 结果 |
|---|---|
| `launchermeta.mojang.com`（官方版本清单） | ❌ 超时不可达 |
| `piston-meta.mojang.com`（版本 JSON） | ✅ 可直连 |
| `piston-data.mojang.com`（客户端 jar） | ⚠️ 可直连但很慢（41MB 约 10s），有时不可达 |
| `libraries.minecraft.net`（依赖库） | ⚠️ 前期可用，后期**完全不可达** → 默认走 BMCLAPI |
| `bmclapi2.bangbang93.com/maven/<path>`（依赖库镜像） | ✅ 对真实库路径全部可用（实测 4 个抽样全 200） |
| `resources.download.minecraft.net`（资源文件） | ✅ 可直连 |
| `bmclapi2.bangbang93.com`（BMCLAPI 镜像） | ✅ 可用，但 `/maven`、`/assets/<id>.json` 路径**不存在**（404），正确路径见下 |
| `bmclapi2.bangbang93.com/v1/packages/<sha>/<id>.json` | ✅ 资源索引的正确镜像路径 |
| `bmclapi2.bangbang93.com/assets/<xx>/<hash>` | ✅ 资源对象的正确镜像路径 |
| `bmclapi2.bangbang93.com/version/<id>/client` | ✅ 客户端 jar 的镜像快捷路径 |
| `maven.minecraftforge.net` / `maven.fabricmc.net` / `maven.quiltmc.org` | ✅ 可直连 |
| `maven.neoforged.net` | ❌ 超时（NeoForge 需自备镜像） |
| `meta.fabricmc.net` / `meta.quiltmc.org` | ✅ 可直连 |
| `api.modrinth.com`（搜索/元数据） | ✅ 可直连（免 Key） |
| `cdn.modrinth.com`（模组文件） | ⚠️ **时通时断**：能通时 1MB 秒下，不通时 20s+ 超时 |
| `api.curseforge.com` | 🔑 需 API Key（搜索无 Key 返回 403） |
| `edge.forgecdn.net`（CurseForge 文件） | ✅ 主机可达（403/404 是路径问题的正常响应） |
| `api.adoptium.net` | ✅ 可直连（自动下载 Java） |
| `login.microsoftonline.com` / `login.live.com` | ✅ 可直连 |
| `user.auth.xboxlive.com` / `xsts.auth.xboxlive.com` | ✅ 可直连 |
| `api.minecraftservices.com` | ✅ 可直连（Microsoft 登录可用） |
| `api.mojang.com` / `sessionserver.mojang.com` / `textures.minecraft.net` | ❌ 不可达（旧版 Yggdrasil 登录不可用；正版皮肤可能显示不出来） |

因此默认 `auto` 模式：**版本清单 / 版本 JSON / 依赖库 / 客户端 jar 走镜像，其余走官方，失败自动换下一个候选源**。

### 3.2 源解析原理

`server/net.mjs` 的 `SourceResolver` 把任意官方 URL 映射成**一组候选 URL**：

```
官方 URL ──按规则前缀替换──> 镜像 URL₁, 镜像 URL₂, …  ──下载器依次尝试──> 成功为止
```

内置规则（可在「下载源」页直接改）：

| 规则 | from（官方） | to（镜像） | 默认 |
|---|---|---|---|
| BMCLAPI 清单 | `https://launchermeta.mojang.com` | `https://bmclapi2.bangbang93.com` | 开启（**优先**，因为官方被墙） |
| BMCLAPI 版本 JSON | `https://piston-meta.mojang.com` | `https://bmclapi2.bangbang93.com` | 开启（优先） |
| BMCLAPI 资源 | `https://resources.download.minecraft.net` | `https://bmclapi2.bangbang93.com/assets` | 开启（兜底） |
| BMCLAPI 依赖库 | `https://libraries.minecraft.net` | `https://bmclapi2.bangbang93.com/maven` | 开启（**优先**：官方库主机实测会完全不可达） |
| BMCLAPI Forge maven | `https://maven.minecraftforge.net` | `https://bmclapi2.bangbang93.com/maven` | 开启（兜底） |
| BMCLAPI Fabric | `https://meta.fabricmc.net` | `https://bmclapi2.bangbang93.com/fabric-meta` | 开启（兜底） |
| Modrinth CDN 镜像 | `https://cdn.modrinth.com` | *（示例，默认关闭）* | 关闭：自己填镜像前缀后启用 |
| CurseForge CDN 镜像 | `https://edge.forgecdn.net` | *（示例，默认关闭）* | 关闭：自己填镜像前缀后启用 |

> 早期版本曾把依赖库镜像关掉（当时对某个 log4j-core 路径返回 404 被误判为镜像不可用），
> 后来用真实库路径复测确认 **镜像完全可用**，因此改回镜像优先。这类判断都以实测为准。

四种模式：

- `auto`：官方优先、镜像兜底（被标记 `forceFirst` 的规则排最前）
- `official`：只用官方
- `mirror`：镜像优先、官方兜底
- `custom`：只用你自己添加的规则（完全自建源）

另外，下载失败会被**记住**（10 分钟内降权），所以排在前面的坏源只会拖慢一次，后续文件会自动换源。
下载器对每个文件都有 sha1 校验和重试，失败的只是那一个文件，重试安装会跳过已校验通过的部分。

### 3.3 换成你自己的源

在「下载源 → 自定义镜像规则」里加一条：

```
from: https://piston-meta.mojang.com
to:   https://your-mirror.example.com/mc
```

规则按**前缀替换**：`https://piston-meta.mojang.com/v1/packages/x/1.20.1.json`
→ `https://your-mirror.example.com/mc/v1/packages/x/1.20.1.json`。

也可以用 API 直接改配置：

```bash
curl -X POST http://127.0.0.1:25564/api/sources/save \
  -H "content-type: application/json" \
  -d "{\"custom\":[{\"id\":\"mine\",\"name\":\"我的源\",\"enabled\":true,\"from\":\"https://piston-meta.mojang.com\",\"to\":\"https://your-mirror.example.com/mc\"}]}"
```

### 3.4 自定义内容源（模组 / 整合包 / 地图 / 资源包）

除了内置的 Modrinth、CurseForge，你可以在「设置 → 自定义内容源」粘贴任意 JSON API 描述，
用模板变量 `{query} {mcVersion} {loader} {limit} {offset} {id}` 拼接请求：

```json
[
  {
    "id": "my-source",
    "name": "我的内容源",
    "enabled": true,
    "search": {
      "url": "https://example.com/api/search?q={query}&mc={mcVersion}&loader={loader}&offset={offset}&limit={limit}",
      "itemsPath": "data.list",
      "fields": { "id": "id", "name": "title", "description": "desc", "author": "author", "downloads": "dl", "icon": "icon" }
    },
    "files": {
      "url": "https://example.com/api/project/{id}/files?mc={mcVersion}&loader={loader}",
      "itemsPath": "data",
      "fields": { "fileId": "id", "name": "name", "downloadUrl": "url", "fileName": "filename", "size": "size", "sha1": "sha1" }
    }
  }
]
```

`itemsPath` / `fields` 支持点号路径（如 `a.b.c`），留空则用默认字段名。

### 3.5 当模组 CDN 被墙时怎么办

`cdn.modrinth.com`（Modrinth 文件）和 `edge.forgecdn.net`（CurseForge 文件）可能在你这边打不开。
本项目给了三条路，任选：

1. **自定义 CDN 镜像规则**：「下载源 → 自定义镜像规则」里加一条
   `from: https://cdn.modrinth.com` → `to: https://你的镜像/前缀`（路径保持一致即可）。
   配置里已经预置了这两条**默认关闭**的示例规则，填上 `to` 再启用就行。
2. **从链接安装**：「模组 / 资源」页底部有输入框，粘贴任意能打开的 `.jar` / `.zip` 地址
   （比如你浏览器里能下的镜像链接、代理后的直链），选择放到模组/资源包/光影/存档，点安装即可。
   对应接口是 `POST /api/content/install-url`。
3. **离线拷贝**：直接把文件放进实例的 `mods/` 目录（实例页有「模组」文件夹按钮）。

内容下载失败时不会长时间挂起：单文件 45 秒超时、重试 2 次，并在错误里给出具体主机名与建议。

---

## 四、账号

### 4.1 离线账号

用户名即游戏内 ID，UUID 按官方离线规则生成（`OfflinePlayer:<name>` 的 MD5 v3），
与离线服务器兼容。适合单机与离线服务器。

### 4.2 Microsoft 登录

采用 **OAuth2 授权码 + 本机回环回调**：

```
浏览器授权 → http://localhost:<port>/auth/ms/callback?code=…
   → Microsoft 令牌 → Xbox Live (user.auth.xboxlive.com)
   → XSTS (xsts.auth.xboxlive.com, RelyingParty=rp://api.minecraftservices.com/)
   → Minecraft 令牌 (authentication/login_with_xbox)
   → 档案 (minecraft/profile)
```

- 回调地址：`http://localhost:<端口>/auth/ms/callback`
- 若浏览器没自动跳回，可复制地址栏完整 URL 粘贴到登录弹窗完成登录
- 令牌会缓存并在启动前自动续期（refresh_token）
- 常见错误已翻译：`2148916233` 未创建 Xbox 档案 / `2148916238` 未成年需加入家庭组 等
- 登录信息保存在 `data/config.json` 的 `auth.accounts`（明文，请注意本机安全）

> 注意：旧版 Mojang（Yggdrasil）登录在本机网络下不可用（`api.mojang.com`、`sessionserver` 均不可达），
> 所以本项目只提供 Microsoft 登录 + 离线模式。

---

## 五、Java

- **自动搜索**：`PATH`、`JAVA_HOME`/`JDK_HOME`、注册表 `JavaSoft`、
  `Program Files\{Java,Eclipse Adoptium,Microsoft,Zulu,BellSoft,Amazon Corretto,…}`、
  `%APPDATA%\.minecraft\runtime`（官方启动器的运行时）、`data/runtime`（本启动器下载的）、
  以及界面里手动添加的额外目录。
- **手动配置**：在「Java」页填 `java.exe` 绝对路径或用文件选择器挑，立即生效。
- **自动下载**：走 [Adoptium](https://adoptium.net) API（`api.adoptium.net`，实测可直连），
  下载对应大版本 JDK 解压到 `data/runtime/jdk-<major>` 并自动选中。
- **自动挑选**：按版本 JSON 的 `javaVersion.majorVersion` 选；缺失时按启动参数自动下载。
  常见对应：1.16 及更早 → 8、1.18–1.20.4 → 17、1.20.5+ → 21、最新版 → 25。

---

## 六、游戏与加载器安装

1. **原版**：读取版本 JSON → 下载客户端 jar、全部依赖库（含 `natives-*`）、资源索引与资源对象
   （`sha1` 校验、已存在且校验通过即跳过）→ 解压 natives → 写入 `shared/versions/<id>/<id>.json`。
2. **Fabric / Quilt**：取官方 meta 的 `profile/json`，与基础版本合并（`inheritsFrom`），只补加载器依赖库。
3. **Forge / NeoForge**：下载官方 `installer.jar`，用 Java 以 `--installClient` headless 运行，
   由官方 installer 自己写 `versions/`，然后自动识别生成的版本 ID。

安装过程全程有进度与日志（SSE 实时推送）。**下载失败只是该文件失败**，重试安装会跳过已校验通过的文件。

---

## 七、启动

- classpath 由依赖库 + 客户端 jar 组成（同名库去重；支持 `natives-*` 分类 jar）
- 变量替换：`${auth_player_name}` `${auth_uuid}` `${auth_access_token}` `${version_name}`
  `${game_directory}` `${assets_root}` `${assets_index_name}` `${natives_directory}`
  `${classpath}` `${classpath_separator}` `${library_directory}` `${launcher_name}` 等
- 规则（`rules`: os / arch / features）完整求值，Windows 上自动带 `-Djava.library.path`
- 支持老版本 `minecraftArguments` 与新版本 `arguments.game/jvm`
- 进程输出实时推到界面；若系统限制了管道 stdio（某些受限环境会 `spawn EPERM`），
  自动**退回日志文件模式**继续采日志

---

## 八、开放接口

启动器本身就是本地 HTTP 服务（默认 `http://127.0.0.1:25564`），全部功能都有 JSON 接口。
若在设置里配置了 `apiToken`，请求需带 `x-mcopen-token` 头或 `?token=`。

| 方法 | 路径 | 说明 |
|---|---|---|
| GET | `/api/ping` | 存活检测 |
| GET | `/api/state` | 完整状态（配置/实例/账号/Java/任务） |
| GET | `/api/versions` | 版本清单（`?force=1` 强制刷新） |
| GET | `/api/versions/installed` | 本地已安装版本 |
| GET | `/api/versions/{id}/detail` | 版本详情 |
| POST | `/api/versions/install` | 安装 `{baseVersion, loader, loaderVersion}` |
| GET | `/api/loaders?kind=fabric&mc=1.20.1` | 加载器版本列表 |
| GET | `/api/jobs` / `/api/jobs/{id}` | 任务列表 / 任务进度 |
| POST | `/api/instances` | 新建实例 `{name, baseVersion, loader, memoryMb}` |
| POST | `/api/instances/{id}/install` | 安装 / 修复实例 |
| POST | `/api/instances/{id}/launch` | 启动实例 |
| POST | `/api/instances/{id}/open` | 打开实例文件夹 `{kind}` |
| POST | `/api/folders/open` | 打开启动器文件夹 `{kind}` |
| GET | `/api/java` / `/api/java/scan` | Java 列表 / 重新扫描 |
| POST | `/api/java/select` | 手动指定 `{path}` |
| POST | `/api/java/download` | 自动下载 `{major}` |
| GET | `/api/accounts` / POST `/api/accounts/offline` | 账号列表 / 添加离线 |
| POST | `/api/auth/ms/start` / `complete` | 发起 / 完成 Microsoft 登录 |
| GET | `/api/auth/diagnose` | 登录服务可达性诊断 |
| GET | `/api/content/sources` | 内容源列表 |
| GET | `/api/content/search?source=modrinth&q=sodium&type=mod&mc=1.20.1&loader=fabric` | 搜索 |
| GET | `/api/content/versions?source=modrinth&projectId=…` | 项目文件列表 |
| POST | `/api/content/install` | 安装内容 `{instanceId, type, url, fileName}` |
| POST | `/api/content/install-url` | 从任意链接安装（CDN 被墙时的兜底） |
| POST | `/api/packs/upload` | 上传整合包（multipart） |
| POST | `/api/packs/install-local` | 安装本地整合包 `{instanceId, path}` |
| GET | `/api/sources` | 源规则与健康度 |
| POST | `/api/sources/mode` | 切换模式 `{mode}` |
| POST | `/api/sources/save` | 保存规则 `{rules, custom}` |
| POST | `/api/sources/test` | 测试源连通性 `{url}` |
| GET | `/api/stream` | SSE：`job` `progress` `log` `game-log` `game-exit` `auth` |

示例：

```bash
# 装 Fabric 1.20.1
curl -X POST http://127.0.0.1:25564/api/versions/install ^
  -H "content-type: application/json" -d "{\"baseVersion\":\"1.20.1\",\"loader\":\"fabric\"}"

# 搜索并安装一个模组（先用 search 拿 downloadUrl）
curl "http://127.0.0.1:25564/api/content/search?source=modrinth&q=sodium&type=mod&mc=1.20.1&loader=fabric"
```

---

## 九、常用文件夹

实例页每个实例都有：**模组 / 存档 / 资源包 / 光影 / 配置 / 日志 / 截图 / 崩溃报告**一键打开。
设置页可打开：数据目录、共享目录（libraries/assets/versions）、日志目录、Java 目录。
实例页与「模组列表」弹窗里可以直接删除模组文件（仅限 `data/` 目录内，防止误删）。

---

## 十、常见问题

**Q：CurseForge 搜不到东西 / 整合包装到一半报错？**
A：CurseForge 搜索与下载都需要 API Key。到 <https://console.curseforge.com> 免费申请，
填到「设置 → 内容源」。没有 Key 时请用 Modrinth（免 Key，整包 `.mrpack` 也不需要 Key）。

**Q：CurseForge API Key 怎么获取？**

1. 用浏览器打开 <https://console.curseforge.com>（CurseForge for Studios 控制台），用 CurseForge/Overwolf 账号登录（免费注册）。
2. 首次进入需同意 **CurseForge 3rd Party API 使用条款**（明确只做个人/非商业用途，不得转售或再分发 Key）。
3. 创建一个应用（Create API Key / New Application），填名称与用途，例如 `McOpen personal launcher`。
4. 生成后得到形如 `$2a$10$...` 的 Key，**只在创建时显示一次**，请立刻复制保存。
5. 粘到「设置 → 内容源 → CurseForge API Key」保存，然后到「模组 / 资源」把源切到 CurseForge 搜一次即验证是否可用。

补充说明：

- 本机实测 `console.curseforge.com` 与 `docs.curseforge.com` **可直接访问**（无需代理），`api.curseforge.com` 不带 Key 返回 403（符合预期）。
- 网上流传的 `$2a$10$bL4bIL...` 之类的 Key 来自公开示例/他人内置，随时会失效或因滥用被封，不建议长期使用，也请不要把它打包进分发的程序里。
- 不想申请 Key 的三条替代路线：① 用 **Modrinth**（免 Key，本启动器内置，整合包 `.mrpack` 也不需要 Key）；
  ② 用「模组 / 资源」页底部的**从链接安装**，直接粘贴能打开的 CF CDN 直链（`https://edge.forgecdn.net/files/<id/1000>/<id%1000>/<文件名>`）；
  ③ 浏览器手动下载后放进实例的 `mods/` 目录。
- 只有 **CurseForge 整合包**（zip 内 `manifest.json` 只有 projectID/fileID、没有文件名）必须靠 API 解析，这也是它硬依赖 Key 的原因。
**Q：NeoForge 装不上？**
A：`maven.neoforged.net` 在本机网络下超时。解决：在「下载源 → 自定义镜像规则」里把
`https://maven.neoforged.net/releases` 映射到你的镜像（或全局代理），再装。

**Q：资源文件下载很久？**
A：1.20.1 的资源约 3600 个文件、600MB+。可以调大「设置 → 下载并发数」（默认 24），
或在 `data/config.json` 里设置 `downloads.assetFilter` 跳过某个前缀的目录（例如声音）。

**Q：启动报 `java.library.path` / LWJGL 找不到 dll？**
A：natives 没解压成功。到「游戏版本」页对该版本重跑安装（会补齐并重新解压 natives），
或直接把 `data/shared/natives/<版本>/` 删掉后重装。

**Q：端口被占用 / 不小心双击了两次 McOpen.cmd？**

如果占用者就是 McOpen 自己，第二次启动会**直接打开已有窗口并静默退出**（退出码 0，不报错）。
若是别的程序占用端口：`node server\index.mjs --port 26000`，或改 `data/config.json` 的 `port`。

**Q：想让别的设备访问接口？**
A：改 `data/config.json`：`"allowLan": true`（建议同时设置 `apiToken`）。

---

## 十一、实测验证记录

下面每一条都是在本机真实跑过并看到输出后才写下的（不是"应该可以"）：

| 验证项 | 结果 |
|---|---|
| 版本清单（920 个版本） | ✅ 走镜像拿到，断网回退本地缓存 |
| 原版 1.20.1 完整安装 | ✅ 客户端 jar + 88 个依赖库 + 3598 个资源文件（651MB）共 87 秒，全部 sha1 校验通过 |
| natives 解压 | ✅ 21 个文件（lwjgl.dll / glfw.dll / OpenAL.dll / jemalloc.dll 等） |
| 离线账号 + 原版启动 | ✅ 日志出现 `Setting user: McOpenTest`、`Backend library: LWJGL 3.3.1`、`Sound engine started`、纹理图集创建（= 进入主菜单） |
| Fabric 加载器安装 | ✅ 合并 profile，72 个加载器依赖库从镜像下载 |
| Fabric + Sodium 启动 | ✅ `Loading Minecraft 1.20.1 with Fabric Loader 0.20.1`，模组树里能看到 sodium，Sodium 检测到 GPU 并加载 42 项设置 |
| Modrinth 搜索/取文件 | ✅ 39 条结果，Sodium 1.20.1 文件正确匹配 |
| 整合包（.mrpack）安装 | ✅ 自动识别依赖版本与 fabric-loader、下载文件、释放 overrides（2 个覆盖文件落盘） |
| 打开文件夹 | ✅ 资源管理器真实弹出（实例 logs / mods） |
| Java 自动搜索 | ✅ 找到 3 个 JVM（17 / 21 / 25，含官方启动器 runtime） |
| Java 自动下载（Adoptium） | ✅ 元数据接口返回 Java 21 安装包（196MB 链接） |
| Microsoft 登录服务可达性 | ✅ 5 个端点全部可达（token / XBL / XSTS / mc-services），假 code 会按预期报错 |
| 前端界面 | ✅ 无头 Edge 无报错，8 个标签页全部渲染；文本对比度 ≥ 4.5；窄窗口侧栏收成图标栏 |

**未能在本机完成验证的部分（如实说明）：**

- **Microsoft 真实登录**：需要你的正版账号在浏览器里授权，我无法代替。代码路径、端点可达性、错误提示都已验证；
  实际拿到令牌那一步需要你自己点一次登录。
- **CurseForge 搜索/下载**：没有 API Key 就无法调用，接口已按"无 Key 时给出友好提示"实现。
- **Forge / NeoForge 安装**：installer 下载与 headless 执行代码已写好，但 Forge installer 体积大、
  NeoForge 的 maven 在本机被墙，未做端到端跑通。
- **旧版本（1.7 及更早）的 native（.lzma）**：Node 内置 zlib 不支持 lzma，这类版本需要另外解包。



## 十二、免责声明

本项目仅用于学习与个人管理自己购买的游戏副本；请通过 Microsoft 账号合法拥有 Minecraft: Java 版。
所有下载内容版权归各自所有者（Mojang / 模组作者 / 整合包作者）。

