#钢云纪元：核心 API 文档
 
文档版本：v0.2
上一版本：v0.1
目标平台：Minecraft 1.20.1 / Fabric Loader + Fabric API
维护者：拉面、乌龙
更新规则：任何接口签名变更必须先改本文档，再改代码；PR 中引用本文档章节号。
 
v0.2 变更摘要（详细见第 6 节变更日志）：
• spawnTrain 改为返回 Optional<UUID>；resetDailyLimitsIfNeeded、requireOrWarn 改为返回 boolean
• getBalance 对未注册 UUID 抛 NoSuchAccountException；getSlotItem 明确返回副本
• 新增 2.12–2.24 共 13 个补齐类型定义
• 新增 3.9 服务注册表（SteelCloudServices）
• 补强 3.2 NbtKeys、3.5 NetworkChannels、3.8 GameTest 接入
• 修订 3 处文档未明确语义（创造模式健康、ItemOrigin 打标时机、radius 单位） 
 
0. 设计原则
1. 核心只放契约，不放玩法实现。 接口在这里定义，实现由各模组提供。
2. 核心不依赖 Create、MTR、TACZ、Aether 等外部模组。 外部模组只出现在 compat 或对应功能模块。
3. 服务端权威。 经济、死亡、限额、列车状态等关键数据以服务端为准，客户端只做展示。
4. 所有接口通过服务注册表获取，不直接 new 实现类。 
 
1. 包结构
com.steelcloud.core.api
├── SteelCloudServices              // 服务注册表入口 [v0.2]
├── SteelCloudRegistry              // 注册辅助
├── NbtKeys                         // NBT 键常量 [v0.2]
├── train
│   ├── TrainService
│   ├── TrainSpec                   // [v0.2]
│   ├── TrainSnapshot               // [v0.2]
│   ├── TrainLifecycleListener      // [v0.2]
│   ├── TrainStorageService
│   └── StoredTrain                 // [v0.2]
├── economy
│   ├── EconomyService
│   ├── TransferResult              // [v0.2]
│   ├── DailyLimit                  // [v0.2]
│   └── PriceTableService
├── health
│   ├── HealthService
│   ├── HealthThreshold             // [v0.2]
│   └── HealthChangeListener        // [v0.2]
├── death
│   ├── DeathPipeline
│   ├── DeathHandler
│   └── DeathResult                 // [v0.2]
├── item
│   ├── ItemOriginService
│   └── ItemOrigin
├── slot
│   ├── SlotService
│   └── SlotDefinition              // [v0.2]
├── dimension
│   ├── DimensionTravelService
│   └── DimensionTravelRule         // [v0.2]
├── compat
│   └── ModCompatService
├── data
│   ├── DataTableService
│   ├── DataTableCodec              // [v0.2]
│   └── DataTableException          // [v0.2]
├── persistence
│   └── SteelCloudPersistentState
├── network
│   └── NetworkChannels             // [v0.2]
└── util
    └── ServerThreadAssert          // [v0.2]

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
解决什么问题：玩法、经济、交通内容模块不应该直接 import Create 或 MTR 的类。它们只通过这个接口说"给我生成一列车""这节车现在在哪"。
由谁实现：train-dispatch 或 train-api 的具体实现层。
由谁调用：掠夺者专车、收缩球、交通内容、经济运输。

    public interface TrainService {
    // 在指定位置生成一列车；失败（无轨道、位置非法、实现层未加载）返回 empty
    Optional<UUID> spawnTrain(ServerWorld world, BlockPos pos, TrainSpec spec); // [v0.2] 原 UUID

    boolean despawnTrain(UUID trainId);

    Optional<TrainSnapshot> getTrain(UUID trainId);

    // radius 单位为方块（blocks）
    List<TrainSnapshot> getTrainsNear(ServerWorld world, BlockPos pos, double radius);

    void addListener(TrainLifecycleListener listener);
    void removeListener(TrainLifecycleListener listener); // [v0.2] 模块卸载时必须调用
    }
 
边界：
• 不定义调度算法、时刻表、信号系统。
• 不直接暴露 Create/MTR 的实体类型。
• TrainSpec 是核心定义的数据对象，包含车体类型、车厢数、初始速度等抽象字段。 
Fabric 1.20.1 提示：实体生成走 world.spawnEntity，具体实体类型由实现层决定；跨维度查询需遍历 MinecraftServer.getWorlds()。 
 
2.2 TrainStorageService
 
定位：列车收纳/放出的抽象接口。
解决什么问题：收缩球需要"把一节列车存进球里，再在铁轨上放出来"，但不应该知道列车是 Create 的还是 MTR 的。
由谁实现：train-api 或 transport_content。
由谁调用：收缩球物品、未来的列车复制/迁移工具。

    public interface TrainStorageService {
    // 收纳一节车厢/车头。传入整列编组 UUID 时返回 empty（不支持整列收纳）
    Optional<StoredTrain> capture(ServerWorld world, UUID trainId);

    boolean release(ServerWorld world, BlockPos railPos, StoredTrain stored);

    // 校验快照是否仍可放出（格式版本兼容、车体存在）
    boolean validate(StoredTrain stored);
    }
 
边界：
• StoredTrain 必须可序列化为 NBT，支持跨维度携带。
• 一次只收纳一节车厢或一个车头，不处理整列编组。
• 不负责收缩球的物品逻辑和 GUI，只负责列车快照。 
Fabric 1.20.1 提示：StoredTrain 建议用 NbtCompound 实现 writeNbt/readNbt，不要依赖 1.20.5+ 的 Data Components。 
 
2.3 EconomyService
 
定位：玩家账户与转账的统一入口。
解决什么问题：政府交易、转账税、限额查询、成就奖励都可能改余额，必须走同一套服务端权威逻辑。
由谁实现：economy 模块。
由谁调用：政府 NPC、转账命令、成就系统、调试命令。

    public interface EconomyService {
    // 未注册 UUID 抛 NoSuchAccountException（附带消息 "Error: selected player doesn't exist"）
    long getBalance(UUID playerId);

    boolean withdraw(UUID playerId, long amount, String reason);
    boolean deposit(UUID playerId, long amount, String reason);

    // 语义：从发送方扣 amount，收款方收 (amount - tax)，tax = 5% * amount（向下取整）
    TransferResult transfer(UUID from, UUID to, long amount);

    DailyLimit getDailyLimit(UUID playerId);

    // 返回是否实际发生了重置
    boolean resetDailyLimitsIfNeeded(); // [v0.2] 原 void
    }
 
边界：
• 不实现税率规则、禁售表、物价计算，这些属于 economy 实现层。
• 所有方法必须在服务端线程调用。
• reason 用于审计日志，不参与业务判断。
• amount 为负值时视为非法参数，withdraw/deposit 返回 false，transfer 返回 Rejected。 
Fabric 1.20.1 提示：数据持久化用 PersistentState 子类，存在主世界 overworld 的 PersistentStateManager 中。 
 
2.4 PriceTableService
 
定位：物价表读取接口。
解决什么问题：政府购买/售出、成就"我全都要"估值、调试命令都需要查价，但物价数据不应硬编码在多个模块。
由谁实现：economy 模块，数据来自 data/steelcloud/prices.json。
由谁调用：政府 GUI、交易逻辑、测试。

    public interface PriceTableService {
    OptionalLong getBuyPrice(ItemStack stack);   // 政府卖给玩家的价格
    OptionalLong getSellPrice(ItemStack stack);  // 政府从玩家收购的价格
    boolean isTradeable(ItemStack stack);        // 价格 > 0 且不在禁售状态
    String getTableVersion();                    // 语义版本号，如 "1.0.3"，与 xlsx 源文件对应
    }
 
边界：
• 只读，不提供修改价格的方法。
• 禁售状态由 economy 实现层结合 ItemOriginService 判断，接口只暴露结果。
• 物价表版本必须与 xlsx 源文件版本对应。
• 对空栈（stack.isEmpty()）的三个方法：getBuyPrice/getSellPrice 返回 OptionalLong.empty()，isTradeable 返回 false。 
 
2.5 HealthService
 
定位：健康值（0%–100%）的读写与阈值事件。
解决什么问题：雨衣、细雪、饥饿、金苹果、幸运药水、死亡优先级都涉及健康值，需要统一入口。
由谁实现：health 模块（或 gameplay 内的健康子模块）。
由谁调用：环境伤害逻辑、食物效果、死亡管线、HUD。

    public interface HealthService {
    float getHealth(ServerPlayerEntity player);          // 0.0–100.0
    void setHealth(ServerPlayerEntity player, float value);
    void addHealth(ServerPlayerEntity player, float delta, String reason);
    HealthThreshold getThreshold(ServerPlayerEntity player); // NORMAL / WEAK / CRITICAL / DEAD
    void addListener(HealthChangeListener listener);
    void removeListener(HealthChangeListener listener);      // [v0.2]
    }
 
边界：
• 不处理饥饿、细雪、淋雨的具体伤害来源，由调用方计算 delta。
• =0% 时触发死亡由 DeathPipeline 统一处理，不在这里直接 kill。
• 仅生存/冒险模式生效，创造模式 getHealth 返回 100。
• 写操作约定（v0.2 明确）：创造模式 / 已死亡玩家调用 setHealth、addHealth 静默忽略，不抛异常；数值钳制到 [0, 100]；setHealth 和 addHealth 均触发监听器（setHealth 在值实际变化时触发）。 
Fabric 1.20.1 提示：玩家附加数据用 Cardinal Components API 的 Component，或自己用 PersistentState + UUID 映射。前者更方便同步到客户端。 
 
2.6 DeathPipeline
 
定位：统一死亡触发管线。
解决什么问题：灵魂绑定、死亡不掉落图腾、原版不死图腾、keepInventory、/kill、健康值归零，这些触发顺序如果分散在各模块，必然冲突。
由谁实现：core.impl.death。
由谁调用：ServerPlayerEntity 死亡事件、健康值归零、/kill 命令钩子。

    public interface DeathPipeline {
    void registerHandler(DeathHandler handler);
    DeathResult process(ServerPlayerEntity player, DamageSource source);
    }

    public interface DeathHandler {
    int priority(); // 数值越小越先执行
    Optional<DeathResult> handle(ServerPlayerEntity player, DamageSource source);
    }
 
process 返回值消费方式（v0.2 明确）：
• 若管线结果或任意 handler 返回 DeathResult.Cancelled(hp)：取消死亡事件，将玩家血量置为 hp。
• 若最终为 DeathResult.Proceed()：走正常死亡掉落。
• 管线会按 priority 升序调用每个 handler，遇到第一个返回 Optional.of(...) 的 handler 就终止并返回其结果；若全部返回 empty，返回 Proceed()。 
建议优先级顺序（文档锁定，代码实现可调）：
1. keepInventory 检查（直接取消掉落，灵魂绑定休眠）。
2. 原版不死图腾（副手优先）。
3. 死亡不掉落图腾（取消死亡事件，血量置 1）。
4. 灵魂绑定（对每件物品做耐久三层判定）。
5. 正常死亡掉落。 
边界：
• 不实现具体物品逻辑，只编排顺序。
• 每个 handler 返回 Optional.empty() 表示"我不处理，交给下一个"。
• 所有判定必须在服务端完成。 
 
2.7 ItemOriginService
 
定位：物品来源追踪。
解决什么问题：经济规则要求"村民交易获得的物品不得卖给政府"，但物品栈本身不带来源信息，需要核心统一记录。
由谁实现：core.impl.item。
由谁调用：economy 的售卖逻辑、调试命令、审计日志。

    public interface ItemOriginService {
    ItemOrigin getOrigin(ItemStack stack);           // 未知时返回 UNKNOWN
    void markOrigin(ItemStack stack, ItemOrigin origin);
    boolean isSellableToGovernment(ItemStack stack); // 综合来源与禁售表
    }

    public enum ItemOrigin {
    CRAFTED, VILLAGER_TRADE, WANDERING_TRADE, LOOT, MOB_DROP, UNKNOWN
    }
 
打标时机（v0.2 明确，实现层必须覆盖以下事件）：
• 合成完成事件 → CRAFTED
• 村民交易完成事件 → VILLAGER_TRADE
• 流浪商人交易完成事件 → WANDERING_TRADE
• 战利品箱打开、怪物掉落拾取 → LOOT / MOB_DROP 
边界：
• 只记录来源，不判断价格。
• 来源标记不通过工作台复制传播，合成产物强制 CRAFTED，无论原料来源。
• 对空栈：getOrigin 返回 UNKNOWN，markOrigin 为 no-op，isSellableToGovernment 返回 false。 
Fabric 1.20.1 提示：1.20.1 没有原生 Data Components，用 Cardinal Components 的 ItemStack 组件，或 NBT 子标签 steelcloud:origin。前者更干净，后者更省依赖。 
 
2.8 SlotService
 
定位：扩展槽位注册与查询。
解决什么问题：短刃专属槽、旗帜槽、雨衣槽都需要"额外槽位"，但每个槽的规则不同，核心只提供注册和查询框架。
由谁实现：core.impl.slot。
由谁调用：战斗模块（短刃槽）、装饰模块（旗帜/雨衣槽）、GUI。

    public interface SlotService {
    void registerSlot(SlotDefinition definition);
    Optional<SlotDefinition> getSlot(Identifier id);
    Collection<SlotDefinition> getAllSlots();

    // 返回副本，调用方修改不影响内部存储
    ItemStack getSlotItem(ServerPlayerEntity player, Identifier slotId); // [v0.2] 明确副本

    boolean setSlotItem(ServerPlayerEntity player, Identifier slotId, ItemStack stack);

    // v0.2 新增：可选重载，返回详细结果；原 setSlotItem 等价于 trySetSlotItem(...).isSuccess()
    SlotOperationResult trySetSlotItem(ServerPlayerEntity player, Identifier slotId, ItemStack stack);
    }

    public sealed interface SlotOperationResult {
    record Success() implements SlotOperationResult {}
    record UnknownSlot(Identifier slotId) implements SlotOperationResult {}
    record InvalidItem(ItemStack stack) implements SlotOperationResult {}
    record Conflict(Identifier conflictingSlot) implements SlotOperationResult {}

    default boolean isSuccess() { return this instanceof Success; }
    }
 
边界：
• 核心只定义"槽位存在、能存什么、是否可见"，不定义短刃缩回动画、旗帜渲染。
• 槽位数据必须随玩家 NBT 持久化。
• 互斥规则（短刃槽 vs 主手、旗帜 vs 雨衣）由对应模块监听 setSlotItem 实现。 
Fabric 1.20.1 提示：Fabric 没有 Forge 的 IItemHandler 槽位系统，需要自己维护玩家附加数据；GUI 同步用 ServerPlayNetworking 发包。 
 
2.9 DimensionTravelService
 
定位：跨维度传送统一入口。
解决什么问题：海王星 Y=500、冰溪雪原蓝冰门、未来更多维度入口，都需要"传送到某维度某坐标"，但不应各写各的 player.teleport。
由谁实现：core.impl.dimension。
由谁调用：维度模块、传送门方块、调试命令。

    public interface DimensionTravelService {
    boolean teleport(ServerPlayerEntity player, RegistryKey<World> dimension, Vec3d targetPos);

    // 玩家无坐骑时退化为普通 teleport
    boolean teleportWithMount(ServerPlayerEntity player, RegistryKey<World> dimension, Vec3d targetPos);

    void registerRule(DimensionTravelRule rule); // 例如 Y=500 触发
    }
 
边界：
• 不生成维度，不注册传送门方块。
• 只负责"安全地把玩家送过去"，包括加载目标区块、避免卡墙。
• 规则（何时触发）由调用方注册。 
Fabric 1.20.1 提示：自定义维度需要 fabric-dimensions 或数据包 dimension_type + dimension JSON；传送用 player.teleport 或 FabricDimensions.teleport。 
 
2.10 ModCompatService
 
定位：外部模组存在性与版本检查。
解决什么问题：所有模块都可能需要"Create 在不在""MTR 版本够不够"，但不应各自写 FabricLoader.getInstance().isModLoaded。
由谁实现：core.impl.compat。
由谁调用：所有需要软依赖外部模组的模块。

    public interface ModCompatService {
    boolean isLoaded(String modId);
    Optional<String> getVersion(String modId);
    boolean isVersionAtLeast(String modId, String minimum);

    // 未满足时记 WARN 日志，返回是否满足；不抛异常
    boolean requireOrWarn(String modId, String minimumVersion, String featureName); // [v0.2] 原 void
    }
 
边界：
• 只检查，不加载类。真正调用外部 API 的代码必须放在独立的 compat 类中，且用 isLoaded 保护。
• 不负责解决依赖冲突，只报告。 
 
2.11 DataTableService
 
定位：JSON/CSV 数据表的统一加载框架。
解决什么问题：物价表、禁售表、成就条件、列车路线、纹饰效果都可以是数据驱动，但不应每个模块写一套加载器。
由谁实现：core.impl.data。
由谁调用：economy、gameplay、train-dispatch 等。

    public interface DataTableService {
    // 重复注册同一 id 抛 IllegalStateException；注册时记录 type 泛型签名用于后续校验
    <T> void registerTable(Identifier id, Class<T> type, DataTableCodec<T> codec);

    // 类型不符抛 ClassCastException，附带注册 type 与实际请求 type 的诊断信息
    <T> Optional<T> getTable(Identifier id, Class<T> type);

    void reloadAll(MinecraftServer server);

    Set<Identifier> getLoadedTables();

    // v0.2 新增：表版本号，每次成功 reload 递增；调用方可据此判断缓存是否陈旧
    long getRevision(Identifier id);

    // v0.2 新增：reload 后通知参数为本次实际发生变化的表 id 集合
    void addReloadListener(Consumer<Set<Identifier>> listener);
    void removeReloadListener(Consumer<Set<Identifier>> listener);
    }
 
边界：
• 只提供加载、校验、重载框架，不定义具体表结构。
• 数据文件放 data/steelcloud/tables/<id>.json。
• 加载失败必须记录日志并阻止服务器启动（或按配置降级）。
• reload 语义（v0.2 明确）：旧 getTable 返回值是不可变快照，reload 后仍可继续使用（内容为旧数据），不会失效；调用方如需最新数据应重新调用 getTable 或通过 addReloadListener 收到通知后刷新。 
Fabric 1.20.1 提示：Fabric 的 ResourceManagerHelper 可以监听数据包重载；服务端启动时手动加载一次。 
 
2.12 TrainSpec

    public record TrainSpec(
        Identifier bodyType,
        int carCount,
        double initialSpeed,
        Direction facing,
        NbtCompound customData) {

    public TrainSpec {
        if (carCount < 1) throw new IllegalArgumentException("carCount must be >= 1");
        if (initialSpeed < 0) throw new IllegalArgumentException("initialSpeed must be >= 0");
        // 构造时复制，保证不可变性
        if (customData == null) customData = new NbtCompound();
        else customData = customData.copy();
    }

    public static TrainSpec of(Identifier bodyType, int carCount);
    }
 
 
2.13 TrainSnapshot

    public record TrainSnapshot(
        UUID trainId,
        Identifier bodyType,
        RegistryKey<World> dimension,
        Vec3d position,
        float yaw,
        double speed,
        TrainState state) {

    public enum TrainState { IDLE, MOVING, ARRIVED, DESPAWNED }
    }
 
 
2.14 TrainLifecycleListener

    public interface TrainLifecycleListener {
    default void onSpawn(TrainSnapshot snapshot) {}
    default void onArrive(TrainSnapshot snapshot, String stationName) {}
    default void onDespawn(TrainSnapshot snapshot) {}
    }
 
约定：任何 addListener 必须配对 removeListener。模块卸载时清理，防泄漏。 
 
2.15 StoredTrain

    public interface StoredTrain {
    NbtCompound writeNbt(NbtCompound nbt);
    void readNbt(NbtCompound nbt);
    Identifier getTrainType();

    // 格式版本，单调递增；validate 时与实现层支持的 [MIN_SUPPORTED, CURRENT] 范围比对
    int getFormatVersion();
    }
 
 
2.16 TransferResult

    public sealed interface TransferResult {
    record Success(long sent, long tax, long received) implements TransferResult {}
    record InsufficientFunds(long required, long available) implements TransferResult {}
    record Rejected(String reason) implements TransferResult {}

    static TransferResult success(long sent, long tax, long received);
    }
 
不变式：sent = received + tax。 
 
2.17 DailyLimit

    public record DailyLimit(
        long purchaseLimit,
        long purchaseUsed,
        long saleLimit,
        long saleUsed) {

    public long purchaseRemaining();
    public long saleRemaining();
    public boolean purchaseExhausted();
    public boolean saleExhausted();
    }
 
 
2.18 HealthThreshold

    public enum HealthThreshold { NORMAL, WEAK, CRITICAL, DEAD }
 
具体阈值（如 WEAK < 30%、CRITICAL < 10%）由 health 实现层定义，文档不锁定。 
 
2.19 HealthChangeListener

    public interface HealthChangeListener {
    void onHealthChanged(ServerPlayerEntity player, float oldValue, float newValue, String reason);
    }
 
约定：任何 addListener 必须配对 removeListener。 
 
2.20 DeathResult

    public sealed interface DeathResult {
    record Cancelled(float remainingHealth) implements DeathResult {} // 取消死亡事件
    record Proceed() implements DeathResult {}                         // 放行走正常掉落

    static DeathResult cancel(float hp);
    static DeathResult proceed();
    }
 
 
2.21 SlotDefinition

    public record SlotDefinition(
        Identifier id,
        Predicate<ItemStack> validator,
        boolean visible,
        int order,
        Text displayName) {

    public boolean accepts(ItemStack stack);
    }
 
 
2.22 DimensionTravelRule

    public interface DimensionTravelRule {
    Identifier getId();
    boolean shouldApply(ServerPlayerEntity player, ServerWorld from, Vec3d pos);
    Target resolve(ServerPlayerEntity player, ServerWorld from, Vec3d pos);

    record Target(RegistryKey<World> dimension, Vec3d position) {}
    }
 
 
2.23 DataTableCodec

    public interface DataTableCodec<T> {
    T decode(JsonObject json) throws DataTableException;
    JsonObject encode(T value);
    }
 
 
2.24 DataTableException

    public class DataTableException extends RuntimeException {
    public DataTableException(String message);
    public DataTableException(String message, Throwable cause);
    }
 
 
3. 非接口但属于核心的共享内容
 
3.1 共享材料注册
 
只放真正跨三个以上模块、且无明确玩法归属的基础材料：
• steel_ingot（钢锭）：钢刀、钢盾、液压机、水车、转子都依赖。
• steel_plate（钢板）：液压机、风桨、防护服依赖。
• steel_bearing（钢制轴承）：水车、转子依赖。 
其余材料归各自模块：铜缆归 power，焦煤电池归 power，云合金归 gameplay，烧钢归 industry。
 
3.2 共享数据组件 / NBT 键
 
Fabric 1.20.1 下统一 NBT 命名空间。以下键是 NbtCompound 的字符串键，不是 Identifier：
public final class NbtKeys {
    public static final String ORIGIN     = "steelcloud:origin";     // ItemOrigin
    public static final String TRAIN_LINK = "steelcloud:train_link"; // 收缩球内列车快照
    public static final String HEALTH     = "steelcloud:health";     // 玩家健康值（若不用 Cardinal Components）
    public static final String SLOTS      = "steelcloud:slots";      // 扩展槽位数据

    private NbtKeys() {}
}
 
3.3 共享标签
#steelcloud:steel_ingots
#steelcloud:train_rails
#steelcloud:government_blacklist
 
3.4 注册辅助
 
Fabric 没有 DeferredRegister，核心提供：
public final class SteelCloudRegistry {
    public static final String MOD_ID = "steelcloud";

    public static <T> T register(Registry<T> registry, String path, T value);
    public static Identifier id(String path);
}
 
register 对重复 ID 抛异常，这是期望行为（与 Fabric Registry.register 一致）。
 
3.5 网络通道
 
统一通道：steelcloud:core。子通道由各模块注册。
public final class NetworkChannels {
    public static final String CORE    = "steelcloud:core";
    public static final String POWER   = "steelcloud:power";
    public static final String ECONOMY = "steelcloud:economy";
    public static final String TRAIN   = "steelcloud:train";
    public static final String HEALTH  = "steelcloud:health";

    private NetworkChannels() {}
}
 
核心提供 PacketCodec 工具和 ServerPlayNetworking 包装，不定义具体包。
 
3.6 持久化
 
核心提供 PersistentState 基类：
public abstract class SteelCloudPersistentState extends PersistentState {
    public abstract Identifier getId();
}
 
经济账户、怪物死亡计数、每日限额刷新时间、列车收纳记录都继承它。
 
getId() 建议与 PersistentStateManager.getOrCreate(...) 的 name 参数保持一致，便于核心注册表与调试命令定位。
 
3.7 调试命令
/steelcloud debug economy <player>
/steelcloud debug train list
/steelcloud debug health <player>
/steelcloud reload tables
/steelcloud compat
 
3.8 测试工具
• GameTest 入口：com.steelcloud.core.impl.test.SteelCloudGameTest，在 fabric.mod.json 中以 fabric-gametest entrypoint 注册。
• 服务端集成测试脚本放 run/ 下，CI 用 ./gradlew runServer + 超时检查。
• 单元测试覆盖：税收、限额、耐久三层、功率计算、食物溢出。‌注：耐久三层、功率计算、食物溢出属于各功能模块（combat / power / gameplay）的测试项，核心只提供 GameTest 入口与断言工具。 
3.9 服务注册表
 
整个核心的入口。所有接口通过它获取实现，不直接 new。
public final class SteelCloudServices {
    // 必需依赖：fail-fast。服务未注册时抛 ServiceUnavailableException
    public static <T> T get(Class<T> type);

    // 可选依赖：调用方自行降级
    public static <T> Optional<T> find(Class<T> type);

    public static boolean isRegistered(Class<?> type);

    // 显式注册 stub（仅限 dev/测试环境，核心不自动调用）
    public static <T> void registerStub(Class<T> type, T stub);

    // 整合包启动自检：缺任一服务则 abort 并输出缺失列表
    public static void requireAll(Class<?>... types);

    // 仅测试/关机清理用
    public static void clear();
}

public class ServiceUnavailableException extends IllegalStateException { /* ... */ }
 
调用方规范：
• 必需依赖 → get()，fail-fast，错误清晰可诊断。
• 可选依赖 → find()，自行降级。
• 任何 addListener 必须配对 removeListener，模块卸载时清理。 
stub 策略：核心不自动注册任何 stub。开发环境如需 stub，在 impl 模块显式调用 registerStub；生产环境由整合包启动自检 requireAll 保证所有必需服务就位。
 
3.10 线程断言
public final class ServerThreadAssert {
    public static void check(MinecraftServer server); // 非服务端线程抛 IllegalStateException
}
 
文档中所有标注"服务端线程调用"的接口方法，实现层应在入口调用 ServerThreadAssert.check(server)。 
 
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
 
把本文档作为 AI_CONTEXT.md 的一部分喂给 AI 时，附上以下约束：
• 本项目使用 Minecraft 1.20.1 + Fabric Loader + Fabric API。
• 核心接口定义在 com.steelcloud.core.api，实现不得直接依赖 Create/MTR/TACZ/Aether。
• 不要使用 NeoForge 的 DeferredRegister、IEnergyStorage、SavedData、Data Components。
• 物品来源用 NBT 键 steelcloud:origin，玩家数据用 PersistentState 或 Cardinal Components。
• 服务通过 SteelCloudServices.get()（必需）或 find()（可选）获取，不直接 new 实现类。
• 生成的代码必须编译通过，并通过对应模块的 GameTest。 
 
6. 变更流程与版本
 
6.1 变更流程
1. 新增/修改接口：先改本文档，PR 标题带 [core-api]。
2. 实现模块负责人 review 签名兼容性。
3. 版本号在 gradle.properties 中 core_api_version 递增。
4. 破坏性变更必须提供迁移说明，并在 CHANGELOG.md 标注。 
6.2 版本号规则
 
gradle.properties：
core_api_version=0.2.0
 
格式：语义版本 MAJOR.MINOR.PATCH。
• PATCH：javadoc、注释、文档措辞修正，不改签名。
• MINOR：新增接口方法（提供 default 实现）、新增类型、新增可选方法。0.x 阶段允许 MINOR 含破坏性变更。
• MAJOR：破坏性签名变更、删除接口、语义不兼容变更。 
当前版本 0.2.0。
 
6.3 变更日志
 
v0.2.0
破坏性变更：
• TrainService.spawnTrain 返回 UUID → Optional<UUID>。
• EconomyService.resetDailyLimitsIfNeeded 返回 void → boolean。
• ModCompatService.requireOrWarn 返回 void → boolean。
• EconomyService.getBalance 对未注册 UUID 抛 NoSuchAccountException。 
新增：
• SteelCloudServices（服务注册表）、ServiceUnavailableException。
• 类型：TrainSpec、TrainSnapshot、TrainLifecycleListener、StoredTrain、TransferResult、DailyLimit、HealthThreshold、HealthChangeListener、DeathResult、SlotDefinition、SlotOperationResult、DimensionTravelRule、DataTableCodec、DataTableException。
• NbtKeys、NetworkChannels、ServerThreadAssert。
• TrainService.removeListener、HealthService.removeListener。
• DataTableService.getRevision、addReloadListener、removeReloadListener。
• SlotService.trySetSlotItem。 
语义明确：
• EconomyService.transfer 税额从 amount 中扣除，收款方得 amount - tax。
• HealthService 写操作在创造模式/已死亡时静默忽略，值钳制 [0, 100]。
• SlotService.getSlotItem 返回副本。
• ItemOrigin 打标时机与"合成产物强制 CRAFTED"。
• DataTableService.reloadAll 后旧表快照继续有效，不失效。
• getTrainsNear 的 radius 单位为方块。

v0.1.0
初始版本。
