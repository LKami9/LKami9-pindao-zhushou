# LKami9-cn-channel-guard

Omni Toolbox **在线模块**：国服频道私聊守卫。

监听 Omni 国服频道的私聊，正文命中关键词就自动 `/logout`；也能只提醒不登出。

## 这是什么形态

| 项 | 值 |
| --- | --- |
| 形态 | 单个 `.cs`，由 Omni 运行时用 Roslyn 编译（不是 Dalamud 插件，没有 dll/json 清单） |
| 依赖 | 只能用 Omni 环境自带的程序集；本模块只用了 Dalamud + `OmniToolbox.Common`，不引第三方 |
| 配置 | 宿主自动存到 `pluginConfigs\OmniToolbox\TreeHouse\LKami9ChannelWhisperGuard.json`，模块里没有 `config.Save()` |
| 安装 | Omni「插件设置 → 在线模块」填仓库地址 |

## 安装

```text
https://raw.githubusercontent.com/<你的账号>/LKami9-cn-channel-guard/main/TreeHouseModules.json
```

也可以填仓库主页地址（Omni 会自己找默认分支根目录的 `TreeHouseModules.json`）。分支只写 `main`。

装完默认关闭：到「树树妙妙屋 → 在线」启用，再进模块设置填关键词。

## 四步流程（代码也按这四段分）

```text
1 定位客户端    反射认出「含嵌套类型 ChannelWhisper」的客户端类
                → 从插件实例 / 宿主回调出发做广度搜索拿活动实例（找不到再扫静态字段）
2 订阅事件      client.WhisperReceived（Action<ChannelWhisper>）→ 泛型处理器 + CreateDelegate
3 入队 + 判定   回调可能来自后台线程：只把私聊塞进 ConcurrentQueue
                Framework.Update（主线程）里取出来做关键词 / 发送者 / 冷却判定
4 登出          ICommandManager.ProcessCommand("/logout")
```

为什么不直接调接口：在线模块只拿到 `OmniModuleBase` + Dalamud 服务，Omni 没有公开国服频道 API，所以第 1 步只能用反射。

为什么按嵌套类型名认：Omni 的 `OmniToolbox.Tree.dll` 做过混淆，顶层类名是随机串（当前是 `eSs8PyYWSfhkmesgIyu`），
但嵌套的 `ChannelWhisper / ChannelMessage / ChannelEvent` 名字没变，认这个名字比认随机类名稳。

为什么不用 OmenTools 的 `ChatManager.Instance()`：本地插件开发笔记里记着服务定位器在在线模块里可能拿不到实例（静默无效），
所以登出和提示都走 Dalamud 注入的服务（`ICommandManager` / `IChatGui`）。

## 配置项

| 配置 | 默认 | 说明 |
| --- | --- | --- |
| 关键词 | 空 | 每行一条，忽略大小写，空行忽略 |
| 匹配方式 | 包含 | 包含 / 完全相等 / 通配符（`*` `?`） |
| 发送者过滤 | 空 | 显示名片段、服务器名或化名编号，逗号分隔；留空不限 |
| 命中后自动登出 | 开 | 关闭 = 只提醒（先这样跑一遍验证规则） |
| 登出延迟 | 0 ms | 0 表示立即 |
| 触发冷却 | 10 s | 冷却内的重复命中只写日志 |
| 动作前在聊天框提示 | 开 | 登出前 `IChatGui.Print` 一条 |
| 同时检测游戏内 /tell | 关 | 备用通道，反射失效时不至于全盲 |

## 排错

- 日志前缀 `[频道守卫]`，用 `/xllog` 或 DalamudMCP 的 `xllog` 看（关键日志都是 `Information`，`Debug` 默认被过滤）。
- 状态「已认出类型，等待客户端创建」：类型认出来了但实例还没建。先在 Omni 里启用「聊天框优化」并连上国服频道，模块每 5 秒自动重试。
- 状态「未找到国服频道类型」：Omni 版本低于 1.1.8.0，或频道模块没加载。
- 规则对不对：点「校验关键词规则」，用第一条关键词做自检。
- 定位细节：展开「诊断 → 定位链」，会显示 `Plugin.xxx -> ... -> 频道客户端`，反射路径一眼可见。

## 发布 / 更新

1. 改代码。
2. 编译验证（0 警告 0 错误）。
3. 算 SHA256 并写回清单：

   ```powershell
   (Get-FileHash -LiteralPath '.\Modules\LKami9ChannelWhisperGuard.cs' -Algorithm SHA256).Hash
   ```

4. **提高 `Version`**（只改 SHA 不提版本不会触发更新）+ 换 `Sha256`。
5. `git add -A; git commit -m "vX.Y.Z：说明"; git push origin main`。
6. 抓 raw 校验哈希（别用 curl，本机解析不了 raw 域名）：

   ```powershell
   $bytes = (Invoke-WebRequest -Uri 'https://raw.githubusercontent.com/<你的账号>/LKami9-cn-channel-guard/main/Modules/LKami9ChannelWhisperGuard.cs').Content
   ```

7. 生效流程：**停用模块 → 重载 OmniToolbox（或重启游戏）→ 刷新仓库 → 更新 → 启用**。
   Omni 只替换 `module.cs` 文件，运行中的编译产物仍是旧的。

硬规则：`.cs` 必须是 **LF 行尾**（仓库 `.gitattributes` 已固定 `*.cs eol=lf`），否则下载后哈希对不上；
清单里的 `Sha256` 用大写十六进制。

## 目录

```text
LKami9-cn-channel-guard/
  Modules/LKami9ChannelWhisperGuard.cs
  README.md
  TreeHouseModules.json
  .gitattributes
```

## 实测记录（2026-10-24，Omni 1.1.8.0 / Dalamud CN API 15）

定位链（设置页「诊断」原样）：

```text
root -> Plugin.host -> OmniHost.featureLifetime -> FeatureLifetime.disposers
     -> [1].Target -> Rq5quyOoq826sxWrhFc.chatFrameOptimization
     -> KoeOjqoUDkJMSImtjyR.omniWorldChat
```

- 首次扫描即命中：`三路根：插件实例=有；图A=572(OmniHost/控模块/模块管理器)`，`扫描次数：1`。
- 实收私聊：`08:28:14 冒险者213786#213786：测试` → 关键词「测试」命中 → `已发送 /logout`。
- 客户端藏在 `FeatureLifetime.disposers` 的委托 Target 里，所以对象图很小，问题从来不是「遍历不够深」，
  而是 1.0.1 之前只认单个 `Type`（Omni 重载后旧 Assembly 的同名类型仍在 AppDomain 里）。

## 更新记录

### 2.0.0

- 迁移到 Omni 在线模块 **API 1**（宿主 1.1.8.2 起强制）：
  `ModuleBase` → `OmniModuleBase` + `[ModuleApiVersion(1)]`，
  `OnEnable/OnDisable/OnDispose` → `OnStart(FeatureLifetime)/OnStop/OnDestroy`（事件订阅改由 `activation.OnStopping` 解绑），
  界面文字改由 `ModuleLocalization` 词条 + `OmniLoc.Get` 提供，状态串改成「键 + 参数 + 是否异常」。
- 反射定位逻辑（类型识别、对象图遍历、后台扫描）未改动；配置 JSON 键与 InternalName 不变，升级不丢设置。

### 1.0.3

- **修登出**：`/logout` 是游戏命令，Dalamud 的 `CommandManager.ProcessCommand` 只在它自己的命令表里查
  （插件命令 + Dalamud 自带命令），查不到直接返回 `false`——1.0.2 的「已发送 /logout」就是这里假成功。
  现在主路改走 `OmenTools.OmenService.ChatManager.SendCommand("/logout")`（反射调用），
  它内部直接调 `ExecuteCommandInner`，是游戏真正解析命令的地方；`ProcessCommand` 只当无害的附加尝试。
- 设置页新增「发送 /logout 测试」按钮，不用等私聊就能验证登出通道。

### 1.0.2

- 客户端类型改成**多候选匹配**：Omni 每次重载都会在同一个 AppDomain 里留下旧的 Assembly，
  同名 `Type` 对象可能有多个，只认第一个会永远匹配不上活动实例。
- 遍历深度 12→16、节点上限 8 万→15 万。
- 诊断多一段「关键类型」：`图A=节点数(OmniHost/控模块/模块管理器)`，
  用来区分「客户端没创建」和「遍历没走到」。

### 1.0.1

- 定位过程整体挪到**后台线程**：1.0.0 在主线程做对象图搜索，实测 `OnUpdate` 卡到 1707ms、
  每次重扫 80~95ms，日志里是一串 HITCH。现在主线程只读一个标志位并消费结果。
- 取插件实例改走 `Dalamud.Service<PluginManager>.GetNullable()`：插件作用域里
  `PluginInterface.GetService(typeof(PluginManager))` 返回 null，1.0.0 的「插件对象根」其实从未生效，
  一直靠静态字段兜底。`Get()` 会在服务未构造时挂住线程，所以只用 `GetNullable()` 并加 2 秒看门狗。
- 深度 10→12、节点上限 4 万→8 万；第一次扫描就带上静态字段那一轮。
- 诊断区新增「三路根」（插件实例 / 宿主根 / 静态根，以及各自展开的节点数），
  能一眼看出是根没拿到还是遍历没走通。

