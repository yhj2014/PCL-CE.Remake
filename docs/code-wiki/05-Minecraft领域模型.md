# 05 · Minecraft 领域模型

`PCL.Core/Minecraft` 与 `PCL.Core/Link` 提供 Minecraft 相关的“纯领域能力”：Java 管理、账户认证、存档读写、资源项目模型、服务器地址解析、联机大厅与协议。它们不依赖启动器的安装/启动流程，业务编排位于 `PCL.Launcher.Core`。

## 1. Java 子系统

### 1.1 类型

| 类型 | 职责 |
|---|---|
| `JavaManager` | 管理已发现/已添加的 Java：扫描、解析、启停用、持久化 |
| `JavaService` | 生命周期服务，包装全局 `JavaManager` |
| `JavaInstallation` | 一个 Java 安装（主目录、可执行文件、版本、架构、厂商） |
| `JavaEntry` | 安装 + 启用状态 + 来源 |
| `JavaExecutableLayout` | 可执行文件布局（bin/java、bin/javaw 等） |
| `JavaBrandType` | 厂商类型（Oracle、OpenJDK、Microsoft、Zulu、GraalVM 等） |
| `JavaSource` | 来源（自动扫描、手动添加、全局偏好、相对路径等） |
| `JavaConsts` / `JavaStorageItem` | 常量与持久化模型 |

### 1.2 扫描器（`Java/Scanner`）

| 扫描器 | 平台/来源 |
|---|---|
| `DefaultPathsScanner` | 常见安装目录 |
| `PathEnvironmentScanner` | `PATH` 环境变量 |
| `RegistryJavaScanner` | Windows 注册表 |
| `WindowsWhereCommandScanner` | Windows `where java` |
| `MicrosoftStoreJavaScanner` | Microsoft Store 安装的 JDK |
| `LinuxJavaScanner` | Linux 发行版路径 |
| `MacOsJavaScanner` | macOS `/Library/Java/JavaVirtualMachines` 等 |

### 1.3 解析器（`Java/Parser`）

| 类型 | 职责 |
|---|---|
| `IJavaParser` / `JavaParserBase` | 解析 Java 可执行文件，识别版本/架构/厂商 |
| `WindowsJavaParser` | 解析 PE 头 + 读取 `release` 文件 |
| `PosixJavaParser` | 解析 ELF/Mach-O + `release` 文件 |
| `ElfHeaderReader` / `MachOHeaderReader` | 二进制头读取 |

### 1.4 用户偏好（`Java/UserPreference`）

- `JavaPreference`：基类/接口。
- `AutoSelect`：自动选择满足版本要求的 Java。
- `ExistingJava`：指定已有 Java。
- `UseGlobalPreference`：使用全局默认。
- `UseRelativePath`：使用实例目录内的相对路径。

`JavaManager` 使用 `SemaphoreSlim` 防止并发扫描，并有最小扫描间隔（约 13 秒），避免频繁全盘扫描。

## 2. 账户与认证

### 2.1 模型

| 类型 | 说明 |
|---|---|
| `McProfile` | 一个账户档案（名称、UUID、皮肤、类型、凭据） |
| `SafeProfile` | 可安全展示/持久化的档案视图（脱敏） |
| `ProfileType` | 账户类型（微软、统一通行证、外置登录等） |
| `ProfileManagement<McProfile>` | 档案集合、当前档案、最近使用索引、加载状态 |
| `ProfileService` | 生命周期服务（`Loaded`），负责加载/保存/切换档案 |
| `MicrosoftAccountService` | 微软账户相关流程 |

`ProfileService` 使用配置键 `Profile`（`ConfigSource.SharedEncrypt`）持久化，加载在后台任务中执行，并带锁保护。

### 2.2 认证提供者（`Profile/Authentication`）

| 提供者 | 机制 |
|---|---|
| `MicrosoftProvider` | 微软 OAuth → Xbox Live → XSTS → Minecraft Services → 档案 |
| `AuthlibProvider` | 外置登录（authlib-injector / 统一通行证） |
| `YggdrasilConnectProvider` | Yggdrasil Connect（OIDC 风格） |

微软链路相关模型：`MojangIdentityToken`、`MojangProfile`、`MojangResponse`、`XboxAuthenticate`、`XboxLiveResponse`、`XSTSProperty`、`XboxUserHash` 等；工具类 `MojangUtils`、`XboxUtils`。

### 2.3 IdentityModel

`Minecraft/IdentityModel` 是一个可复用的身份协议实现：

| 子模块 | 内容 |
|---|---|
| `OAuth` | `Client`、`ClientOptions`、`AuthorizeResult`、`DeviceCodeData`、`EndpointMeta`、`IOAuthClient` |
| `Extensions/OpenId` | `OpenIdClient`、`OpenIdMetaData`、`OpenIdOptions` |
| `Extensions/Pkce` | `PkceClient`、`PkceChallengeOptions` |
| `Extensions/JsonWebToken` | `JsonWebToken`、`JsonWebKeys` |
| `Extensions/YggdrasilConnect` | `YggdrasilClient`、元数据与选项 |
| `Yggdrasil` | `Client`、`Options`、`Agent`、`Profile`、`YggdrasilCredential` |
| `Yggdrasil/ApiLocation` | API 地址发现 |

它同时服务于微软认证、外置登录与第三方皮肤站。

## 3. 启动辅助

`Minecraft/Launch/Utils/LaunchEnvUtils.cs` 提供与启动环境相关的通用逻辑（路径、分隔符、编码、参数转义等），被业务层 `PCL.Launcher.Core/Launch` 使用。

## 4. 版本与格式

| 类型 | 职责 |
|---|---|
| `McVersionClassifier` | 识别版本类型：正式版、快照、旧版 Beta/Alpha、愚人节版等 |
| `McFormatter` | 版本号/名称格式化 |
| `MavenArtifact` | Maven 坐标解析（group:artifact:version[:classifier]） |
| `MavenComparableVersion` | Maven 版本比较（用于库冲突解决） |
| `NbtFileHandler` | NBT 文件读写（基于 fNbt） |
| `GameCore` | 向游戏核心 JAR 追加文件（如 Forge 的 `MINECRAFT-JAR`），并删除签名目录 |
| `ServerAddressResolver` | 解析 `host:port`，支持 SRV 记录（`_minecraft._tcp`） |

`GameCore` 的典型用途：安装 Forge 后将补丁内容合并进原版 JAR，并移除 `META-INF` 以避免签名校验失败。

## 5. 存档系统

### 5.1 目录

```text
Saves/
├── SaveManager.cs
├── SaveInfo.cs
├── SaveImportHelper.cs
├── DataVersionBoundaries.cs
├── NBTEnums.cs
├── Parsing/
│   ├── ISaveParser.cs
│   ├── SaveParserFactory.cs
│   └── Internal/
│       ├── NbtReadHelper.cs
│       ├── Pre113SaveParser.cs
│       ├── Version113To1152SaveParser.cs
│       ├── Version116To1211SaveParser.cs
│       ├── Version131To189SaveParser.cs
│       ├── Version19To1122SaveParser.cs
│       └── Version261PlusSaveParser.cs
├── Editing/
│   ├── ISaveEditor.cs
│   ├── Editable.cs
│   ├── SaveChanges.cs
│   └── Internal/
│       ├── Pre261SaveEditor.cs
│       └── Version261PlusSaveEditor.cs
└── Exceptions/
    ├── SaveCorruptedException.cs
    └── SaveNotFoundException.cs
```

### 5.2 解析与编辑

- `SaveParserFactory` 根据 `DataVersion` / 目录结构选择对应解析器。
- `ISaveParser` 读取 `level.dat`、`session.lock`、区域文件等，输出 `SaveInfo`（名称、版本、游戏模式、最后游玩时间、图标等）。
- `ISaveEditor` 修改存档元数据（名称、图标、游戏模式等），不同版本写入字段不同。
- `SaveChanges` 描述待修改项，`Editable<T>` 表示可编辑字段。
- `SaveManager` 负责枚举、导入、导出、复制、删除存档。
- `SaveImportHelper` 支持从压缩包/其他启动器目录导入。

### 5.3 版本边界

`DataVersionBoundaries` 维护 Minecraft 数据版本与游戏版本的映射，是选择解析器/编辑器的依据。

## 6. 资源项目（CurseForge / Modrinth）

### 6.1 模型

| 模型 | 字段/职责 |
|---|---|
| `CurseforgeProject` / `CurseforgeFile` | 项目与文件；含 `CurseforgeAuthors`、`CurseforgeCategories`、`CurseforgeHashes`、`CurseforgeLinks`、`CurseforgePictures`、`CurseforgeScreenshots`、`CurseforgeResponseData` |
| `ModrinthProject` | 项目；含 `ModrinthGallery`、`ModrinthLicense`、`ModrinthDonationUrl`、`ModrinthModeratorMessage` |
| `ModDependencyResolver` | 解析依赖关系（必需/可选、版本约束、加载器约束），生成安装计划 |

这些模型既用于在线搜索，也用于整合包解析（`manifest.json`、`modrinth.index.json`）与依赖补全。

## 7. 联机（`PCL.Core/Link`）

### 7.1 大厅

| 类型 | 职责 |
|---|---|
| `LobbyService` | 生命周期服务（`Loaded`），管理大厅生命周期、玩家列表、世界发现、游戏状态监视 |
| `LobbyController` | 大厅主机/客户端控制逻辑 |
| `LobbyState` | 状态枚举（空闲、创建中、运行、加入中等） |
| `LobbyInfoProvider` / `LobbyTextHandler` | 信息与文本 |
| `LinkAnnounce` | 大厅公告/广播 |

`LobbyService` 每 5 秒检查游戏状态、每 15 秒执行周期任务，并与 `McPing`、`Scaffolding`、`EasyTier` 协作。

### 7.2 局域网广播

- `BroadcastListener`：监听 Minecraft 局域网广播。
- `BroadcastLocal`：发送本地广播。
- `BroadcastRecord`：广播记录（MotD、端口、版本）。

用于“发现本地世界”和“加入局域网房间”。

### 7.3 EasyTier 组网

| 类型 | 职责 |
|---|---|
| `ETController` | 控制 EasyTier 进程/CLI |
| `ETRelay` | 中继选择 |
| `ETInfoProvider` | 网络信息 |
| `ETPeerInfo` | 对端信息 |
| `EasyTierCli` / `EasyTierEntity` / `EasyTierInstaller` / `EasyTierMetadata` / `EasyTierPlatform` | CLI 封装、实体、安装器、元数据 |
| `PublicNodeDto` / `StunInfo` / `EasyPlayerInfo` | 公共节点、STUN 信息、玩家信息 |
| `CliNetTest` | 网络连通性测试 |

用于 P2P 联机：探测 NAT、选择中继、加入虚拟网络。

### 7.4 MC Ping

| 类型 | 职责 |
|---|---|
| `IMcPingService` | Ping 接口 |
| `McPingService` | 现代协议（Handshake + Status） |
| `LegacyMcPingService` | 旧版（1.6 及以前）协议 |
| `McPingServiceFactory` | 根据版本选择实现 |
| `McPingResult` / `McPingPlayerResult` / `McPingVersionResult` / `McPingModInfoResult` | 结果模型 |

### 7.5 Scaffolding 协议

一套自定义的客户端/服务端协议，用于大厅信令：

- `ScaffoldingClient` / `ScaffoldingServer`：连接与请求处理。
- `ProtocolReader` / `ProtocolWriter`：分帧读写。
- 请求：`GetPlayerProfileListRequest`、`GetServerProtRequest`、`PlayerPingRequest` 等。
- 服务端处理器：`GetPlayerProfileListHandler`、`GetProtocolsHandler`、`GetServerPortHandler`、`PingHandler`、`PlayerPingHandler`。
- 模型：`LobbyInfo`、`LobbyType`、`PlayerKind`、`PlayerProfile`。
- `ScaffoldingFactory`、`LobbyCodeGenerator`、`PlayerListHandler`。

### 7.6 Natayark

`NatayarkProfileManager`：Natayark 账户/档案集成（配置项通过 `PCL_NAID_*` 提供）。

## 8. 与其他层的关系

- 业务层 `PCL.Launcher.Core/Download` 使用 `ResourceProject` 模型与 `ModDependencyResolver`。
- 业务层 `PCL.Launcher.Core/Launch` 使用 `JavaManager`、`ProfileService`、`ServerAddressResolver`、`MavenArtifact`。
- 业务层 `PCL.Launcher.Core/Minecraft` 封装 `McInstance`，内部消费本层的版本信息、NBT、存档、Java 能力。
- 界面层的下载页/联机页直接绑定本层的模型与缓存。

## 9. 注意事项

- 认证凭据必须经 `SafeProfile` 脱敏后展示；持久化使用加密配置源。
- Java 扫描较重，必须复用 `JavaManager` 的扫描锁与最短间隔。
- 存档解析按版本分流，新增版本时同时补 `DataVersionBoundaries`、Parser、Editor。
- 网络/联机模块依赖外部进程（EasyTier）与第三方服务，失败时要给出可诊断日志。
