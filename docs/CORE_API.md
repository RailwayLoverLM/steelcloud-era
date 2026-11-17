#钢云纪元：核心 API 文档
 
文档版本：v0.1
目标平台：Minecraft 1.20.1 / Fabric Loader + Fabric API
维护者：拉面、乌龙
更新规则：任何接口签名变更必须先改本文档，再改代码；PR 中引用本文档章节号。 
 
0. 设计原则
1. 核心只放契约，不放玩法实现。 接口在这里定义，实现由各模组提供。
2. 核心不依赖 Create、MTR、TACZ、Aether 等外部模组。 外部模组只出现在 compat 或对应功能模块。
3. 服务端权威。 经济、死亡、限额、列车状态等关键数据以服务端为准，客户端只做展示。
4. 所有接口通过服务注册表获取，不直接 new 实现类。 
 
1. 包结构
com.steelcloud.core.api
├── train
│   ├── TrainService
│   └── TrainStorageService
├── economy
│   ├── EconomyService
│   └── PriceTableService
├── health
│   └── HealthService
├── death
│   └── DeathPipeline
├── item
│   └── ItemOriginService
├── slot
│   └── SlotService
├── dimension
│   └── DimensionTravelService
├── compat
│   └── ModCompatService
└── data
    └── DataTableService

com.steelcloud.core.impl
├── registry
├── network
├── config
├── persistence
├── command
├── test
└── content.materials
 
 
2. 接口清单
 
2.1 TrainService
 
定位：列车实体的统一查询与生成入口。
解决什么问题：玩法、经济、交通内容模块不应该直接 import Create 或 MTR 的类。它们只通过这个接口说“给我生成一列车”“这节车现在在哪”。
由谁实现：train-dispatch 或 train-api 的具体实现层。
由谁调用：掠夺者专车、收缩球、交通内容、经济运输。
 
核心方法（示意）：

    public interface TrainService {
    
    // 在指定位置生成一列车，返回列车唯一 ID
    UUID spawnTrain(ServerWorld world, BlockPos pos, TrainSpec spec);

    // 销毁列车
    boolean despawnTrain(UUID trainId);

    // 查询列车当前状态
    Optional<TrainSnapshot> getTrain(UUID trainId);

    // 查询某位置附近的列车
    List<TrainSnapshot> getTrainsNear(ServerWorld world, BlockPos pos, double radius);

    // 注册列车生命周期监听器（生成、到站、销毁）
    void addListener(TrainLifecycleListener listener);
    }
 
边界：
• 不定义调度算法、时刻表、信号系统。
• 不直接暴露 Create/MTR 的实体类型。
• TrainSpec 是核心定义的数据对象，包含车体类型、车厢数、初始速度等抽象字段。 
Fabric 1.20.1 提示：实体生成走 world.spawnEntity，但具体实体类型由实现层决定；跨维度查询需要遍历 MinecraftServer.getWorlds()。 
 
2.2 TrainStorageService
 
定位：列车收纳/放出的抽象接口。
解决什么问题：收缩球需要“把一节列车存进球里，再在铁轨上放出来”，但不应该知道列车是 Create 的还是 MTR 的。
由谁实现：train-api 或 transport_content。
由谁调用：收缩球物品、未来的列车复制/迁移工具。
 
核心方法：

    public interface TrainStorageService {
    
    // 尝试收纳一节列车，成功返回存储快照
    Optional<StoredTrain> capture(ServerWorld world, UUID trainId);

    // 在指定铁轨位置放出
    boolean release(ServerWorld world, BlockPos railPos, StoredTrain stored);

    // 校验快照是否仍可放出（版本兼容、车体存在）
    boolean validate(StoredTrain stored);
    }
 
边界：
• StoredTrain 必须可序列化为 NBT，支持跨维度携带。
• 一次只收纳一节车厢或一个车头，不处理整列编组。
• 不负责收缩球的物品逻辑和 GUI，只负责列车快照。 
Fabric 1.20.1 提示：StoredTrain 建议用 NbtCompound 实现 writeNbt / readNbt，不要依赖 1.20.5+ 的 Data Components。 
 
2.3 EconomyService
 
定位：玩家账户与转账的统一入口。
解决什么问题：政府交易、转账税、限额查询、成就奖励都可能改余额，必须走同一套服务端权威逻辑。
由谁实现：economy 模块。
由谁调用：政府 NPC、转账命令、成就系统、调试命令。
 
核心方法：

    public interface EconomyService {
    long getBalance(UUID playerId);
    boolean withdraw(UUID playerId, long amount, String reason);
    boolean deposit(UUID playerId, long amount, String reason);
    TransferResult transfer(UUID from, UUID to, long amount); // 含 5% 转账税
    DailyLimit getDailyLimit(UUID playerId); // 购买/售卖已用额度
    void resetDailyLimitsIfNeeded();
    }
 
边界：
• 不实现税率规则、禁售表、物价计算，这些属于 economy 实现层。
• 所有方法必须在服务端线程调用。
• reason 用于审计日志，不参与业务判断。 
Fabric 1.20.1 提示：数据持久化用 PersistentState 子类，存在主世界 overworld 的 PersistentStateManager 中。 
 
2.4 PriceTableService
 
定位：物价表读取接口。
解决什么问题：政府购买/售出、成就“我全都要”估值、调试命令都需要查价，但物价数据不应硬编码在多个模块。
由谁实现：economy 模块，数据来自 data/steelcloud/prices.json。
由谁调用：政府 GUI、交易逻辑、测试。
 
核心方法：

    public interface PriceTableService {
    OptionalLong getBuyPrice(ItemStack stack);   // 政府卖给玩家的价格
    OptionalLong getSellPrice(ItemStack stack);  // 政府从玩家收购的价格
    boolean isTradeable(ItemStack stack);        // 价格 > 0 且不在禁售状态
    String getTableVersion();                    // 用于审计和兼容检查
    }
 
边界：
• 只读，不提供修改价格的方法。
• 禁售状态由 economy 实现层结合 ItemOriginService 判断，接口只暴露结果。
• 物价表版本必须与 xlsx 源文件版本对应。 
 
2.5 HealthService
 
定位：健康值（0%–100%）的读写与阈值事件。
解决什么问题：雨衣、细雪、饥饿、金苹果、幸运药水、死亡优先级都涉及健康值，需要统一入口。
由谁实现：health 模块（或 gameplay 内的健康子模块）。
由谁调用：环境伤害逻辑、食物效果、死亡管线、HUD。
 
核心方法：
    
    public interface HealthService {
    float getHealth(ServerPlayerEntity player);          // 0.0–100.0
    void setHealth(ServerPlayerEntity player, float value);
    void addHealth(ServerPlayerEntity player, float delta, String reason);
    HealthThreshold getThreshold(ServerPlayerEntity player); // NORMAL / WEAK / CRITICAL / DEAD
    void addListener(HealthChangeListener listener);
    }
 
边界：
• 不处理饥饿、细雪、淋雨的具体伤害来源，由调用方计算 delta。
• =0% 时触发死亡由 DeathPipeline 统一处理，不在这里直接 kill。
• 仅生存/冒险模式生效，创造模式返回 100。 
Fabric 1.20.1 提示：玩家附加数据用 Cardinal Components API 的 Component，或自己用 PersistentState + UUID 映射。前者更方便同步到客户端。 
 
2.6 DeathPipeline
 
定位：统一死亡触发管线。
解决什么问题：灵魂绑定、死亡不掉落图腾、原版不死图腾、keepInventory、/kill、健康值归零，这些触发顺序如果分散在各模块，必然冲突。
由谁实现：core.impl.death。
由谁调用：ServerPlayerEntity 死亡事件、健康值归零、/kill 命令钩子。
 
核心方法：
    
    public interface DeathPipeline {
    // 注册处理器，按 priority 排序
    void registerHandler(DeathHandler handler);

    // 处理一次致命伤，返回最终结果
    DeathResult process(ServerPlayerEntity player, DamageSource source);
    }

    public interface DeathHandler {
    int priority(); // 数值越小越先执行
    Optional<DeathResult> handle(ServerPlayerEntity player, DamageSource source);
    }
 
建议优先级顺序（文档锁定，代码实现可调）：
1. keepInventory 检查（直接取消掉落，灵魂绑定休眠）。
2. 原版不死图腾（副手优先）。
3. 死亡不掉落图腾（取消死亡事件，血量置 1）。
4. 灵魂绑定（对每件物品做耐久三层判定）。
5. 正常死亡掉落。 
边界：
• 不实现具体物品逻辑，只编排顺序。
• 每个 handler 返回 Optional.empty() 表示“我不处理，交给下一个”。
• 所有判定必须在服务端完成。 
 
2.7 ItemOriginService
 
定位：物品来源追踪。
解决什么问题：经济规则要求“村民交易获得的物品不得卖给政府”，但物品栈本身不带来源信息，需要核心统一记录。
由谁实现：core.impl.item。
由谁调用：economy 的售卖逻辑、调试命令、审计日志。
 
核心方法：

    public interface ItemOriginService {
    ItemOrigin getOrigin(ItemStack stack);           // 未知时返回 UNKNOWN
    void markOrigin(ItemStack stack, ItemOrigin origin);
    boolean isSellableToGovernment(ItemStack stack); // 综合来源与禁售表
    }

    public enum ItemOrigin {
    CRAFTED, VILLAGER_TRADE, WANDERING_TRADE, LOOT, MOB_DROP, UNKNOWN
    }
 
边界：
• 只记录来源，不判断价格。
• 村民交易来源需要在交易完成事件中打标；流浪商人同理。
• 来源标记不能通过工作台复制传播，合成产物默认 CRAFTED。 
Fabric 1.20.1 提示：1.20.1 没有原生 Data Components，用 Cardinal Components 的 ItemStack 组件，或 NBT 子标签 steelcloud:origin。前者更干净，后者更省依赖。 
 
2.8 SlotService
 
定位：扩展槽位注册与查询。
解决什么问题：短刃专属槽、旗帜槽、雨衣槽都需要“额外槽位”，但每个槽的规则不同，核心只提供注册和查询框架。
由谁实现：core.impl.slot。
由谁调用：战斗模块（短刃槽）、装饰模块（旗帜/雨衣槽）、GUI。
 
核心方法：

    public interface SlotService {
    void registerSlot(SlotDefinition definition);
    Optional<SlotDefinition> getSlot(Identifier id);
    Collection<SlotDefinition> getAllSlots();

    // 查询某玩家某槽位的物品
    ItemStack getSlotItem(ServerPlayerEntity player, Identifier slotId);
    boolean setSlotItem(ServerPlayerEntity player, Identifier slotId, ItemStack stack);
    }
 
边界：
• 核心只定义“槽位存在、能存什么、是否可见”，不定义短刃缩回动画、旗帜渲染。
• 槽位数据必须随玩家 NBT 持久化。
• 互斥规则（短刃槽 vs 主手、旗帜 vs 雨衣）由对应模块监听 setSlotItem 实现。 
Fabric 1.20.1 提示：Fabric 没有 Forge 的 IItemHandler 槽位系统，需要自己维护玩家附加数据；GUI 同步用 ServerPlayNetworking 发包。 
 
2.9 DimensionTravelService
 
定位：跨维度传送统一入口。
解决什么问题：海王星 Y=500、冰溪雪原蓝冰门、未来更多维度入口，都需要“传送到某维度某坐标”，但不应各写各的 player.teleport。
由谁实现：core.impl.dimension。
由谁调用：维度模块、传送门方块、调试命令。
 
核心方法：
    
    public interface DimensionTravelService {
    boolean teleport(ServerPlayerEntity player, RegistryKey<World> dimension, Vec3d targetPos);
    boolean teleportWithMount(ServerPlayerEntity player, RegistryKey<World> dimension, Vec3d targetPos);
    void registerRule(DimensionTravelRule rule); // 例如 Y=500 触发
    }
 
边界：
• 不生成维度，不注册传送门方块。
• 只负责“安全地把玩家送过去”，包括加载目标区块、避免卡墙。
• 规则（何时触发）由调用方注册。 
Fabric 1.20.1 提示：自定义维度需要 fabric-dimensions 或数据包 dimension_type + dimension JSON；传送用 player.teleport 或 FabricDimensions.teleport。 
 
2.10 ModCompatService
 
定位：外部模组存在性与版本检查。
解决什么问题：所有模块都可能需要“Create 在不在”“MTR 版本够不够”，但不应各自写 FabricLoader.getInstance().isModLoaded。
由谁实现：core.impl.compat。
由谁调用：所有需要软依赖外部模组的模块。
 
核心方法：
    
    public interface ModCompatService {
    boolean isLoaded(String modId);
    Optional<String> getVersion(String modId);
    boolean isVersionAtLeast(String modId, String minimum);
    void requireOrWarn(String modId, String minimumVersion, String featureName);
    }
 
边界：
• 只检查，不加载类。真正调用外部 API 的代码必须放在独立的 compat 类中，且用 isLoaded 保护。
• 不负责解决依赖冲突，只报告。 
 
2.11 DataTableService
 
定位：JSON/CSV 数据表的统一加载框架。
解决什么问题：物价表、禁售表、成就条件、列车路线、纹饰效果都可以是数据驱动，但不应每个模块写一套加载器。
由谁实现：core.impl.data。
由谁调用：economy、gameplay、train-dispatch 等。
 
核心方法：

    public interface DataTableService {
    <T> void registerTable(Identifier id, Class<T> type, DataTableCodec<T> codec);
    <T> Optional<T> getTable(Identifier id, Class<T> type);
    void reloadAll(MinecraftServer server);
    Set<Identifier> getLoadedTables();
    }
 
边界：
• 只提供加载、校验、重载框架，不定义具体表结构。
• 数据文件放 data/steelcloud/tables/<id>.json。
• 加载失败必须记录日志并阻止服务器启动（或按配置降级）。 
Fabric 1.20.1 提示：Fabric 的 ResourceManagerHelper 可以监听数据包重载；服务端启动时手动加载一次。 
 
3. 非接口但属于核心的共享内容
 
3.1 共享材料注册
 
只放真正跨三个以上模块、且无明确玩法归属的基础材料：
• steel_ingot（钢锭）：钢刀、钢盾、液压机、水车、转子都依赖。
• steel_plate（钢板）：液压机、风桨、防护服依赖。
• steel_bearing（钢制轴承）：水车、转子依赖。 
其余材料归各自模块：铜缆归 power，焦煤电池归 power，云合金归 gameplay，烧钢归 industry。
 
3.2 共享数据组件 / NBT 键
 
在 Fabric 1.20.1 下，建议统一 NBT 命名空间：
steelcloud:origin        // ItemOrigin
steelcloud:train_link    // 收缩球内列车快照
steelcloud:health        // 玩家健康值（若不用 Cardinal Components）
steelcloud:slots         // 扩展槽位数据
 
3.3 共享标签
#steelcloud:steel_ingots
#steelcloud:train_rails
#steelcloud:government_blacklist
 
3.4 注册辅助
 
Fabric 没有 DeferredRegister，核心提供：
public final class SteelCloudRegistry {
    public static <T> T register(Registry<T> registry, String path, T value);
    public static Identifier id(String path);
}
 
3.5 网络通道
 
统一通道：steelcloud:core。
子通道由各模块注册：steelcloud:power、steelcloud:economy、steelcloud:train、steelcloud:health。
核心提供 PacketCodec 工具和 ServerPlayNetworking 包装，不定义具体包。
 
3.6 持久化
 
核心提供 PersistentState 基类和注册工具：
public abstract class SteelCloudPersistentState extends PersistentState {
    public abstract Identifier getId();
}
 
经济账户、怪物死亡计数、每日限额刷新时间、列车收纳记录都继承它。
 
3.7 调试命令
/steelcloud debug economy <player>
/steelcloud debug train list
/steelcloud debug health <player>
/steelcloud reload tables
/steelcloud compat
 
3.8 测试工具
• GameTest 入口：SteelCloudGameTest.register()。
• 服务端集成测试脚本放 run/ 下，CI 用 ./gradlew runServer + 超时检查。
• 单元测试覆盖：税收、限额、耐久三层、功率计算、食物溢出。 
 
4. Fabric 1.20.1 特别说明
需求 Fabric 1.20.1 方案 注意
数据组件 Cardinal Components API 或 NBT 1.20.1 无原生 Data Components
网络 ServerPlayNetworking / ClientPlayNetworking 需要 fabric-networking-api-v1
持久化 自写 PersistentState 子类 没有 Forge SavedData
配置 JSON + FabricLoader.getConfigDir()，或 Cloth Config 核心不强制依赖 Cloth
维度 fabric-dimensions + 数据包 JSON 自定义维度注册较繁琐
事件 ServerLifecycleEvents、ServerTickEvents、ServerPlayerEvents 优先用 Fabric API 事件
测试 fabric-gametest-api-v1 需要单独 source set
 
与之前 NeoForge 讨论的差异：DeferredRegister、IEnergyStorage、SavedData、Data Components 这些在 Fabric 1.20.1 都不直接可用，实现层需要按上表替换。核心接口本身不变。 
 
5. AI 使用说明
 
把本文档作为 AI_CONTEXT.md 的一部分喂给 AI 时，附上以下约束：本项目使用 Minecraft 1.20.1 + Fabric Loader + Fabric API。
核心接口定义在 com.steelcloud.core.api，实现不得直接依赖 Create/MTR/TACZ/Aether。
不要使用 NeoForge 的 DeferredRegister、IEnergyStorage、SavedData、Data Components。
物品来源用 NBT 键 steelcloud:origin，玩家数据用 PersistentState 或 Cardinal Components。
生成的代码必须编译通过，并通过对应模块的 GameTest。 
 
6. 变更流程
1. 新增/修改接口：先改本文档，PR 标题带 [core-api]。
2. 实现模块负责人 review 签名兼容性。
3. 版本号在 gradle.properties 中 core_api_version 递增。
4. 破坏性变更必须提供迁移说明，并在 CHANGELOG.md 标注。
