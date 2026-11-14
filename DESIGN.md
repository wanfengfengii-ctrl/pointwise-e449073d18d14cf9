# 历史快照到期与载荷回收设计（DESIGN）

本轮只交付设计，不实现回收器。`lake.py`、`catalog.py`、`objects.py`、`checks.py`、
`replay.py`、`samples.py`、`gc_samples.py`、`gc_observe.py` 与 `examples/move.json`
全部保持原样；新增能力以方案端口（facade/包装层）描述，业务回归与观察器继续可跑。
范围限定：单进程、一个 SQLite 目录连接、单回收者、暂停点交错；不做崩溃恢复、
跨进程租约、多回收者、订阅或迁移。

---

## 1. 现状调用链与源码定位

下列行号均为当前仓库真实行号，`gc_check.py verify` 会复核它们落在文件长度内。

### 1.1 目录事务与快照状态

- `catalog.py:22-34` `Catalog.write`：嵌套事务上下文。最外层执行
  `BEGIN IMMEDIATE`（`catalog.py:26`），内层只加入（`catalog.py:24`）；
  最外层异常 `ROLLBACK`（`catalog.py:33`）、正常 `COMMIT`（`catalog.py:30`）。
  这就是“发布和取消随外层事务提交或回滚”的现成机制，方案不另造事务系统。
- `catalog.py:36-37` `head`：单行表读取当前 head。
- `catalog.py:39-46` `state`：未知快照在 `catalog.py:41-42` 抛 `NO_SNAPSHOT`；
  返回 `(sid, 文件集合, 墓碑集合)`。所有“状态取得”都经过这里，是到期排序的插桩点。
- `catalog.py:48-58` `commit`：要求在事务内（`catalog.py:49-50`），按
  `parent+1` 分配快照号（`catalog.py:51-52`），写 snapshots/files/tombstones
  并推进 head（`catalog.py:57`）。新快照保护必须与这些行同事务写入。

### 1.2 不可变对象存储与载荷 I/O 点

- `objects.py:13-28` `put`：名校验（`objects.py:14`）、规范 JSON 字节
  （`objects.py:16`）、文件模式 `xb` 独占创建（`objects.py:20`），同名异值抛
  `IMMUTABLE_CONFLICT`（`objects.py:24,26`）。
- `objects.py:30-32` `get`：读字节并 JSON 解析。对象存储没有 delete 端口；
  观察器的 `ProbeObjects.drop_raw`（`gc_observe.py:21-27`）只是无保护诊断，
  方案端口须另设受目录门控的 `delete(fid)`，不能照抄探针。
- `gc_observe.py:16-19` `inventory`：列对象名（目录 glob 或内存键），
  回收枚举只需要名字，不读载荷。
- 业务侧全部载荷 I/O 位置：
  - `lake.py:22` `read` 逐文件 `objects.get`；
  - `lake.py:29` `delete` 为位置校验 `objects.get(fid)` 取长度——位置删除也读载荷；
  - `lake.py:43` `prepare` 读取全部输入载荷；
  - `lake.py:52` `prepare` 循环 `objects.put` 写部分输出。
  今天这四处均不在 `catalog.write` 内（`prepare` 整体无目录事务，见 `lake.py:39-54`），
  方案必须维持“载荷 I/O 不持目录写锁”，只在它们前后加入短临界区。

### 1.3 业务操作链与冲突语义（不可改变的行为基线）

- `lake.py:10-16` `seed`：先 put 载荷（`lake.py:12`）再目录提交快照 1
  （`lake.py:13-16`）；载荷写不随目录回滚（`GC_CONTRACT.md` 第 20 行所述边界）。
- `lake.py:18-25` `read`：`state` 取快照后排序文件、过滤墓碑。
- `lake.py:27-37` `delete`：先源快照 `state`（`lake.py:28`），再文件/位置校验
  （`lake.py:29-30`，`BAD_POSITION`），进事务后查当前布局
  （`lake.py:32-34`，`STALE_LAYOUT`），重复删除不增快照（`lake.py:35-36`）。
  错误顺序必须保持，到期拒绝插在 `NO_SNAPSHOT` 之后、`BAD_POSITION` 之前。
- `lake.py:39-54` `prepare`：固定当前 head（`lake.py:40`），校验输入
  （`lake.py:41-42`，`BAD_INPUTS`）、覆盖关系（`lake.py:44-47`，`BAD_LAYOUT`），
  输出名 `rewrite-JOB-组号`（`lake.py:50`），返回 base/inputs/outputs（`lake.py:54`）。
- `lake.py:56-66` `publish`：事务内复检 head==base（`lake.py:59-60`，
  `PUBLISH_CONFLICT`）与输入有效（`lake.py:61-63`，`BAD_PLAN`），
  `catalog.commit` 移除输入及其墓碑、加入输出（`lake.py:64-66`）。
- `lake.py:68-71` `rewrite`：今天把 prepare/publish 包在一个目录写事务里，
  与 `lake.py:52` 的载荷写叠加，实际持写锁做 I/O。契约允许方案缩短其锁范围，
  不保证交错期封装必定成功，正常串行业务结果由 `checks.py:10-37` 继续守底。
- 观察器：`gc_observe.py:80-131` `act` 发 begin/end，end 带 head/objects/
  in_transaction（`gc_observe.py:128-129`）；暂停点钩子在
  `gc_observe.py:71-78`；expire/cancel/collect 目前只记 request_only
  （`gc_observe.py:101-103`），所以基线里它们不改变任何状态——本文预测与基线的
  差异全部源自这三个端口被真正实现后的预期行为。

---

## 2. 必须覆盖的保护面

契约把“活动持有者”拆成四类，方案状态必须逐一表达：

1. **删除校验**：`delete` 在 `lake.py:29` 读载荷、在 `lake.py:32-37` 做布局判断，
   从源快照取得至提交/拒绝之间，源对象受保护。
2. **准备输入**：`prepare` 在 `lake.py:43` 读取的输入对象，计划存活期间受保护。
3. **部分输出**：`lake.py:52` 每写完一个输出即可能遇到暂停点（观察器
   `after_put`，见 `gc_samples.py:22-23` 的 prepared 样例），尚未入任何快照的
   输出必须受保护。
4. **冲突后计划**：`PUBLISH_CONFLICT` 只回滚 publish 自己的事务
   （`catalog.py:31-33`），计划仍可显式 cancel 或再次 publish；计划保护在冲突后保留。

另外：

- **读取保护**：`read` 从 `state` 取快照（`lake.py:19`）到全部 get 结束
  （`lake.py:22-24`）。取得快照与到期必须有显式排序点，不能以“文件已经 open”
  为隐含前提。
- **快照保护**：head 与所有未到期快照的文件集合天然可达；到期只拒绝新读/新删，
  不立刻删对象。
- **发布交接**：计划保护必须在同事务内换成新快照保护，外层提交前不得视为已提交，
  外层回滚后计划所需对象仍在。
- **取消语义**：`cancel` 只释放该计划自己的持有；共享同一对象的其他持有者继续保护。
- **共享与空对象**：多个持有者/快照可共享一个对象名（引用计数语义自然退化为集合）；
  零字节行内容（空文件）仍是具名对象，不能因“无行”被当作不存在。
- **名称永不再用**：回收后旧名不得分配给新内容；`gc_observe.py:21-27` 的 raw
  删除允许同名异值重写（`gc_check.py:33-41` 演示），恰恰说明业务端口必须比探针多一层名称台账。

---

## 3. 两种可行保护策略

### 策略 A：目录内保护表（catalog-held protection，推荐并选定）

在 SQLite 目录中新增三张元数据表，全部随 `Catalog.write` 的事务语义提交/回滚：

```sql
CREATE TABLE gc_expiry(sid INTEGER PRIMARY KEY
    REFERENCES snapshots(sid));            -- 到期标记，随外层回滚而消失
CREATE TABLE gc_holds(
    holder TEXT NOT NULL,                  -- 持有者标识
    fid    TEXT NOT NULL,                  -- 被保护对象名
    PRIMARY KEY(holder, fid)
) WITHOUT ROWID;                           -- 持有者→对象名 的多对多集合
CREATE TABLE gc_names(fid TEXT PRIMARY KEY);-- 历史分配名台账，只增不删
```

持有者命名：快照持有者 `snap:<sid>`（覆盖该快照 files 行的全部对象）；
计划持有者 `job:<job>`；在途读取 `read:<seq>`；在途删除 `del:<seq>`。
到期、取消、交接全部是表内行的 INSERT/DELETE，天然满足“随外层事务提交或回滚”。

### 策略 B：进程内保护登记簿 + 目录只存到期

保护集合放在 Python 字典（holder→set(fid)），目录仅加 `gc_expiry` 与名称台账；
通过向 `Catalog.write` 注册提交/回滚回调，用 undo 日志在回滚时补偿登记簿。

### 3.1 比较

| 维度 | 策略 A：目录保护表 | 策略 B：内存登记簿 |
|---|---|---|
| 外层事务接入 | 行级 DML 直接加入 `catalog.py:22-34` 的同一事务，回滚自动恢复 | 需自写 undo 日志与回滚回调，补偿遗漏即泄漏或误删 |
| 到期/读取排序 | 二者在同一个 `BEGIN IMMEDIATE` 短临界区内定序，SQLite 单连接串行化 | 到期行在库里、读取持有在内存，仍需一个短事务做排序，省不掉临界区 |
| collect 判定 | 一条集合查询得到受保护名，与 files 枚举在同一快照视图 | 读字典 + 查库，两份状态要手工对齐 |
| 暂停点交错 | 状态变化只在事务边界可见，与观察器钩子模型（`gc_observe.py:71-78`）一致 | 字典变更即时可见，须自行模仿事务可见性 |
| 目录空间 | 每持有者-对象一行，可被后续 DELETE 回收；台账只增 | 省掉 holds 表，但名称台账仍必须入库，省得有限 |
| 崩溃恢复 | 两策略本轮都不要求；A 的表结构对未来恢复更友好 | 崩溃后内存持有全丢，未来扩展必须重建 |
| 实现复杂度 | 纯 SQL，复用现成嵌套事务 | 回调/补偿/两套真相，审查面更大 |

**选定策略 A**。理由：本题唯一的事务机制就是 `Catalog.write`（`catalog.py:22-34`），
把保护放进同一事务后，发布交接、取消、到期的回滚语义全部由 SQLite 保证，
不需要任何进程内补偿；而 B 既不能省掉读取/到期排序所需的短临界区，
又引入手工 undo，正确性负担明显更大。两策略的 collect 渐近成本相同（第 7 节）。

---

## 4. 选定方案详细设计

### 4.1 状态与不变量

目录新增状态（均为元数据，不含任何行载荷）：

- `gc_expiry(sid)`：sid 已到期。只增于显式 `expire`，可随外层回滚而删除。
- `gc_holds(holder, fid)`：当前全部保护关系。不变量：
  - H1 每个已提交且未到期快照 s，存在持有者 `snap:s`，其 fid 集合等于 s 的 files 行；
  - H2 每个状态为 prepared 的计划 j，存在 `job:j`，覆盖其 inputs 与全部已写 outputs；
  - H3 每个在途 read/delete 持有者覆盖其取得快照/源文件所需对象；
  - H4 任一 fid 只要出现在任一未到期快照 files 行或任一 holder 行中，就不得被回收；
  - H5 `gc_names` 包含历史上所有经业务端口分配的 fid，且只增。
- 对象存储侧新增方案端口 `GuardedObjects`（包装 `Objects`，业务源码不改）：
  - `put(fid, rows)`：先校验 `gc_names` 中无此名或同名同字节，再委托
    `objects.py:13-28` 的原 put；新名在短临界区登记台账。
  - `delete(fid)`：仅供 collect 在门控后调用，语义等同 `gc_observe.py:21-27`
    的删除动作但调用点受 H4 保护；不触碰 `gc_names`（名字保留，载荷释放）。

### 4.2 短临界区与锁纪律

总纪律：**目录写锁（`BEGIN IMMEDIATE`，`catalog.py:26`）内只做 SQL 元数据；
所有 `objects.get/put/delete`（`objects.py:13-32`）都在锁外。** 每个方案操作
被切成“短临界区 → 锁外 I/O → 短临界区”，临界区内无载荷访问，因此不存在持写锁等 I/O。

### 4.3 关键伪代码

伪代码中 `with catalog.write()` 与 `catalog.py:22-34` 完全同义：最外层开
`BEGIN IMMEDIATE`，内层嵌套，异常回滚。`seq` 为进程内单调序号，仅用于生成唯一名。

#### 4.3.1 取得与释放保护的原语

```text
snapshot_holds(tx, sid):                      # 同事务内建立快照保护
    INSERT OR IGNORE INTO gc_holds(holder, fid)
    SELECT 'snap:'||sid, fid FROM files WHERE files.sid = sid;

acquire_read(sid) -> token:                   # 短临界区：到期排序点
    with catalog.write():
        sid, files, dead = catalog.state(sid)     # 未知 -> NO_SNAPSHOT (catalog.py:41)
        if exists(SELECT 1 FROM gc_expiry WHERE sid=sid):
            raise EXPIRED_SNAPSHOT
        token = 'read:' + next_seq()
        INSERT INTO gc_holds SELECT token, fid FROM files WHERE files.sid=sid
    return token, files, dead

release(token):                               # 加入调用者当前外层事务（若有）
    with catalog.write():
        DELETE FROM gc_holds WHERE holder = token
```

#### 4.3.2 read（替代 `lake.py:18-25` 的方案端口，原方法保留给回归）

```text
guarded_read(snapshot=None):
    token, files, dead = acquire_read(snapshot)   # 取得快照即取得保护，排序在锁内
    try:
        rows = []
        for fid in sorted(files):                 # 锁外载荷 I/O（对照 lake.py:22）
            for pos, value in enumerate(objects.get(fid)):
                if (fid, pos) not in dead: rows.append(...)
        return {"snapshot": sid, "rows": rows}
    finally:
        release(token)                            # 读取结束保护才解除
```

排序含义：expire 与 acquire_read 各自需要一个 `BEGIN IMMEDIATE` 临界区，
SQLite 单连接下两者严格串行；先提交 expire 则后到的 acquire 必见 gc_expiry 行而拒绝，
先取得 token 的读取即使中途遇到 expire 也持有 holds 行直到 finally。打开文件句柄
不参与判定，满足“不得以载荷已经打开作为隐含前提”。

#### 4.3.3 delete（保持 `lake.py:27-37` 的错误顺序）

```text
guarded_delete(snapshot, fid, pos):
    with catalog.write():
        sid, source, _ = catalog.state(snapshot)      # NO_SNAPSHOT 优先 (lake.py:28)
        if expired(sid): raise EXPIRED_SNAPSHOT       # 新发起的删除拒绝到期源
        if fid not in source: raise BAD_POSITION
        token = 'del:' + next_seq()
        INSERT INTO gc_holds VALUES (token, fid)      # 源文件先受保护
    length = len(objects.get(fid))                    # 锁外 I/O（对照 lake.py:29）
    try:
        if type(pos) is not int or not 0 <= pos < length:
            raise BAD_POSITION
        with catalog.write():
            sid2, current, dead = catalog.state()     # lake.py:32
            if fid not in current: raise STALE_LAYOUT # lake.py:33-34
            if (fid, pos) in dead: return sid2        # 重复删除不增快照 lake.py:35-36
            new = catalog.commit(current, dead | {(fid, pos)})  # lake.py:37
            snapshot_holds(tx, new)                   # 新快照保护同事务建立
            return new
    finally:
        with catalog.write():
            DELETE FROM gc_holds WHERE holder=token   # 任一路径都释放
```

已在到期前取得 token 的在途 delete：其源 fid 仍在 gc_holds 中，可完成
`lake.py:32-37` 的全部原有判断；expire 之后新发起的 delete 在排序点直接被拒。

#### 4.3.4 prepare（替代 `lake.py:39-54`，锁范围明显缩短）

```text
guarded_prepare(job, inputs, layout):
    base, files, dead = catalog.state()               # 无锁读，沿用 lake.py:40
    if bad_inputs(inputs, files): raise BAD_INPUTS    # lake.py:41-42
    payload = {fid: objects.get(fid) for fid in inputs}   # 锁外输入 I/O（lake.py:43）
    expected  = {(f,p) for f in inputs for p in range(len(payload[f]))}
    requested = [tuple(r) for g in layout for r in g]
    if len(requested)!=len(set(requested)) or set(requested)!=expected:
        raise BAD_LAYOUT                              # lake.py:44-47
    with catalog.write():                             # 短临界区 1：登记计划与输入保护
        if catalog.head() != base: retry/raise        # 锁外期间 head 可能推进
        INSERT INTO plans(job, base, state) VALUES(job, base, 'prepared')
        INSERT INTO gc_holds SELECT 'job:'+job, fid FROM inputs
    outputs = []
    for i, group in enumerate(layout):
        out = "rewrite-" + job + "-" + str(i)         # lake.py:50
        rows = [payload[f][p] for f,p in group if (f,p) not in dead]
        guarded_put(out, rows)                        # 锁外 put（lake.py:52）+台账短临界区
        with catalog.write():
            INSERT INTO gc_holds VALUES ('job:'+job, out)   # 部分输出立即受保护
        outputs.append(out)
    return {"base": base, "inputs": sorted(inputs), "outputs": outputs}
```

每个输出在 put 返回后的同一个短临界区入 holds，因此 `after_put` 暂停点上
（prepared 样例）collect 看到的部分输出已是受保护名。计划行可单设 `plans` 表
（job、base、state），state∈{prepared, published, cancelled}，同样随事务回滚。

#### 4.3.5 publish（在 `lake.py:56-66` 内增加同事务交接）

```text
guarded_publish(plan):
    with catalog.write():
        plan = SELECT ... FOR plans WHERE job=plan.job
        if plan is None: raise NO_JOB
        if plan.state == 'cancelled': raise CANCELLED_JOB
        sid, files, dead = catalog.state()
        if sid != plan.base: raise PUBLISH_CONFLICT   # lake.py:59-60
        inputs = set(plan.inputs)
        if not inputs <= files: raise BAD_PLAN        # lake.py:61-63
        new = catalog.commit((files-inputs)|set(plan.outputs),
                             {(f,p) for f,p in dead if f not in inputs})  # lake.py:64-66
        snapshot_holds(tx, new)                       # 新快照保护先立
        DELETE FROM gc_holds WHERE holder='job:'+job  # 计划保护后撤：同事务原子交接
        UPDATE plans SET state='published' WHERE job=plan.job
```

外层事务回滚时整组动作一起撤销：新快照行、holds 变更、计划状态全部恢复，
计划所需输入/输出对象仍由 `job:` 持有者保护——满足“回滚后仍须保留原计划所需对象”。

#### 4.3.6 cancel 与 expire

```text
cancel(job):
    with catalog.write():
        plan = SELECT state FROM plans WHERE job=job
        if plan is None: raise NO_JOB
        if plan.state in ('published','cancelled'): return   # 无副作用
        UPDATE plans SET state='cancelled' WHERE job=job
        DELETE FROM gc_holds WHERE holder='job:'+job   # 只释放自己的持有
        # 共享对象若另有 snap:/read: 持有者，H4 仍然成立，不会被误删

expire(snapshot):
    with catalog.write():
        if not exists(snapshot): raise NO_SNAPSHOT     # catalog.py:41 语义优先
        if snapshot == catalog.head(): raise HEAD_SNAPSHOT
        INSERT OR IGNORE INTO gc_expiry VALUES(snapshot)   # 重复到期无副作用
```

两者都是纯目录事务：外层提交才生效，外层回滚即恢复到期前/取消前状态
（rollback 样例中 expire_inside 随 ROLLBACK 消失，正是此性质）。

#### 4.3.7 collect（单回收者，可多次调用）

```text
collect() -> list[fid]:                       # 返回按文件名排序的实际删除列表
    if catalog.db.in_transaction:
        return []                             # 活动外层事务：本轮暂缓，不记忆候选
    with catalog.write():                     # 短临界区：只枚举元数据与对象名
        referenced = SELECT DISTINCT fid FROM files
        protected  = SELECT DISTINCT fid FROM gc_holds
        on_store   = objects.inventory()      # gc_observe.py:16-19 的同名能力
        victims = sorted(set(on_store) - set(referenced) - set(protected))
    for fid in victims:                       # 锁外删除，不持目录写锁
        guarded_objects.delete(fid)           # unlink 或内存移除；gc_names 保留
    return victims
```

可达性判定只用到 files/gc_holds 元数据和对象名清单，**不读取任何行载荷**。
“暂缓不遗忘”靠的是无状态重算：候选不存内存，下一轮 collect 从持久化的
inventory 与目录重新求差集，任何一次暂缓都不会让候选永久失踪。

### 4.4 外层事务接入约定

- 所有方案端口内部只通过 `catalog.write()` 访问目录；调用者已持外层事务时
  （契约第 20 行的显式外层事务，观察器 rollback 动作见 `gc_observe.py:104-113`），
  全部 INSERT/DELETE 嵌套加入，COMMIT/ROLLBACK 由最外层统一发出
  （`catalog.py:29-33`）。
- 事务内调用 collect 一律返回暂缓；事务内 read 的 token 在 finally 中释放，
  释放动作同样嵌套进同一事务，回滚后等效于该读取从未发生。
- prepare 按契约不放进调用者尚未提交的外层事务；它自己使用多个独立短临界区，
  与今天 `lake.py:39-54` 无事务的形态兼容。
- 载荷写不回滚的边界不变（对照 `lake.py:12` 与 seed 结构）：若计划登记后、
  输出写完前进程停止（本轮不做崩溃恢复，仅说明），孤儿输出不在任何 files/holds 中，
  下一轮 collect 自然回收，且名称台账阻止其名复用。

---

## 5. 安全性论证与最终回收条件

### 5.1 安全性（不误删）

1. **删除门控**：collect 的 victims 是 inventory 对 `files ∪ gc_holds` 的差集。
   H1–H3 保证删除校验、准备输入、部分输出、冲突后计划、在途读取与全部未到期
   快照所需对象都在并集中，H4 直接推出它们不可能成为 victim。
2. **交接窗口**：publish 在同一事务内先建 `snap:new` 再删 `job:j`（4.3.5），
   任何时刻二者并集都覆盖计划对象；回滚则整体撤销，不存在“两边都没有”的窗口。
3. **到期排序**：到期与取得保护各自经过 `BEGIN IMMEDIATE` 串行临界区（4.3.2），
   持有者身份以 gc_holds 行为准，与文件句柄无关；到期只影响新请求，旧持有者跑完。
4. **共享对象**：holds 是 (holder,fid) 多对多集合，cancel 只删自己那一行；
   只要还有任一持有者或任一未到期快照引用，fid 就留在保护并集中。
5. **空对象**：空文件是 files 表中的具名行（shared_empty 样例的 `empty`），
   判定只看名字不看字节数，零字节载荷与普通对象同等保护。
6. **名称永不复用**：业务端口的 put 先查 `gc_names`（H5），台账只增；
   collect 的 delete 不删台账。raw 探针（`gc_observe.py:21-27,114-117`）
   绕过包装层，不属于协议端口，不能作为回收或复用的依据。

### 5.2 最终回收（不漏删）

定义静止条件：业务操作停止、无打开的外层事务（`catalog.db.in_transaction=False`）、
无在途 read/delete（其 token 由 finally 保证释放）、无 prepared 计划
（每个计划以 published 或 cancelled 终结，二者都删除 `job:` 行）。
静止后：

- 保护并集恰等于 head 与全部未到期快照的 files 并集；
- 此后重复调用 collect，inventory 与保护并集都不再变化，差集中的每个名字在
  第一轮就成为 victim 并在锁外被实际删除；多轮调用返回逐轮缩小的列表，最终为 `[]`；
- 到期链条有限（快照号单调，`catalog.py:51-52`），每个无引用对象至多存活到
  “最后一个引用消失后的下一轮 collect”；暂缓轮次不影响最终性，因为候选由持久状态重算；
- 不允许把“输出丢了之后重算 job”当作保护：重算必须使用新 job/新对象名
  （契约第 14 行），旧名已在台账中永久保留，新写不会顶替旧对象。

---

## 6. 成本核算

| 成本项 | 量级 | 说明 |
|---|---|---|
|  collect 目录扫描 | O(F_all + H) | F_all 为全部快照 files 行总数，H 为 gc_holds 行数；两条 DISTINCT 索引扫描，无载荷读取 |
| collect 对象枚举 | O(N) | N 为现存对象数，一次 listdir/键枚举（`gc_observe.py:16-19`） |
| collect 载荷操作 | O(V) | V 为本轮 victim 数，仅删除（unlink/内存移除），零 get、零内容解析 |
| 业务读载荷 | 与基线相同 | 仅额外两个短临界区（取得/释放），get 次数与 `lake.py:22` 一致 |
| 业务 prepare | 输入 get 与输出 put 次数不变 | 额外 1 个登记临界区 + 每输出 1 个 holds/台账临界区，均为单行 SQL |
| 名称台账空间 | O(K)，K=历史分配过的不同对象名 | 每名一行 TEXT，只增不删；这是“永久名称保留”的唯一永久空间代价，载荷字节仍被释放 |
| holds 表空间 | 随活动持有者伸缩 | 交接/取消/读取结束即 DELETE，长期规模趋近当前快照引用规模 |
| 策略 B 对比 | collect 三项渐近成本相同 | 省下 holds 表却必须保留名称台账，并多出 undo 日志与回调的运行期复杂度 |

---

## 7. 五组观察的契约重放与预测差异

下面每组先给文字解释，结构化的逐操作预测（protected/deleted/reason）在第 8 节
证据块中，顺序严格按真实 end 事件排列，`gc_check.py:56-58` 会核对覆盖与顺序。
基线 actual 中 expire/cancel/collect 全是 request_only（`gc_observe.py:101-103`），
所以“预测差异”集中在这三类端口落地之后。

### 7.1 reader（在途读取跨过到期与回收）

- `reader` 在 `before_get b` 暂停（`gc_samples.py:16-20`）前已通过 acquire 取得
  `read:` 持有，覆盖快照 1 的 a、b。暂停期间 publish 提交快照 2（其 `snap:2`
  覆盖 rewrite-r-0/1），expire(1) 只对其后的新请求生效。
- `during` collect：a、b 被在途 read 持有，rewrite 输出被 snap:2 持有，
  victim 为空——与基线“什么都不删”的表象相同但依据不同（基线是端口未实现）。
- `reader` 结束后 finally 释放 read 持有；`after` collect 立即把 a、b 实际删除
  （快照 1 已到期、快照 2 不含它们），旧载荷不永久占空间。
- `late` 是到期后新发起的 read：acquire 在排序临界区见到 gc_expiry 行，
  预测 `EXPIRED_SNAPSHOT`；基线 actual 中它成功返回 [7,7,9,11]
  （`gc_check.py:20`），这是最明确的基线/方案差异。在途 `reader` 本身仍返回
  [7,7,9,11]，与基线一致。

### 7.2 prepared（部分输出、冲突、取消后的回收）

- `partial` collect 发生在首个输出 rewrite-p-0 的 `after_put` 暂停点：
  prepare 的登记临界区已写入 `job:p` 对该输出的 holds，victim 为空；
  a、b 同时是快照 1 文件。基线 objects 清单停在三个对象
  （`gc_check.py:22`），方案同样不删，依据是部分输出保护而非端口缺席。
- `delete` 提交快照 2（a:0 墓碑），`snap:2` 与 `job:p` 并存；随后 expire(1)
  不影响 a、b——它们仍是 head 快照 2 的文件。
- `conflict` publish 因 head=2≠base=1 得 `PUBLISH_CONFLICT`，与基线相同
  （`gc_check.py:23`）；冲突只回滚 publish 事务，计划仍 prepared、holds 不动。
- `held` collect：输出仍由 `job:p` 持有、a/b 由 snap:2 持有，无删除。
- `cancel` 后 `released` collect：两个 rewrite 输出不在任何快照、持有者已撤，
  成为 victim 被实际删除；a、b 保留。基线 `released` 的 objects 仍含四个对象
  （`gc_check.py:24`），方案预测清单缩减为 a、b。

### 7.3 rollback（外层事务回滚抹掉临时可达性）

- `tx` 外层事务内：publish 建快照 2 并完成 job→snap 交接、expire(1) 写到期行、
  read 看到快照 2（与基线 `read_inside` snapshot=2 一致，`gc_check.py:26`）。
- `collect_inside` 检测到 `in_transaction=True`，**暂缓返回 []**，不做任何删除；
  这是契约允许的“活动事务期间暂缓”，候选不落盘也不遗忘。
- ROLLBACK 后（`gc_observe.py:109-113`）：快照 2、到期行、交接全部撤销，
  head 恢复 1，计划 p 回到 prepared 且重新持有输入与输出；`after_rollback`
  collect 无 victim。基线 objects 仍为四对象（`gc_check.py:28`），方案一致。
- 计划可再次 `publish` 成功（预测结果 2，与基线 `gc_check.py:29` 相同）；
  再 expire(1) 后，`after_commit` collect 把无引用的 a、b 删除，rewrite 输出
  由 snap:2 保留；`current` read 正常返回快照 2 的 [7,9,7,11] 四行。
- 关键预测差异：任何把回滚前临时快照 2 当作最终可达性的设计都会在
  `collect_inside` 误删 a、b；本方案用“事务内暂缓 + 元数据随事务回滚”避开。

### 7.4 cancel_name（取消回收与 raw 探针的边界）

- prepare 产出 rewrite-c-0/1；cancel 撤销 `job:c` 持有；随后 collect 把两个
  无引用输出实际删除，a、b 由 snap:1 保留。
- `raw_remove`（raw_drop）与 `raw_reuse`（raw_put 同名异值）是存储诊断，
  **不是协议操作**：预测中它们的 deleted 为 `[]`、protected 只描述协议状态，
  并在 reason 中单列说明。基线允许删除后同名写回 [999]（`gc_check.py:33-34`），
  方案的 GuardedObjects 以 gc_names 台账在业务端口上封死该复用：collect 只删
  载荷不删台账，任何经业务端口的同名异值 put 都会在台账校验处拒绝，
  合法新对象只能取得新名（新 job → 新 rewrite 名）。

### 7.5 shared_empty（共享引用与零字节对象）

- 快照 1（a、b、empty）、快照 2（a:0 墓碑，三文件仍在）、快照 3
  （empty 输入随 layout=[] 的重写被移除，files 只剩 a、b）。
- expire(1) 后的 `shared` collect：empty 虽不在快照 3，但仍是未到期快照 2
  的 files 行——共享引用使其保留；a、b 同时被快照 2、3 引用。无删除。
- expire(2) 后的 `empty` collect：empty 不再属于任何未到期快照、无持有者，
  零字节不影响判定，作为具名对象被实际删除；a、b 仍由 head 快照 3 保护。
- `current` read 预测与基线一致返回 [7,9,11]（`gc_check.py:32`）；
  差异仅在 collect 的实际删除，基线两轮 collect 都不动对象。

### 7.6 文档核验接通方式

- 结构化预测嵌于下方 `<!-- gc-evidence -->` 代码块，顶层为五个公开样例名；
  `actual` 直接取自 `gc_observe.run(gc_samples.cases()[name])` 的完整事件数组，
  与仓库当前代码逐字节一致，`gc_check.py:53-55` 重放比对。
- `predicted` 按真实 end 事件顺序排列，含 id/outcome/protected/deleted/reason；
  `gc_check.py:56-67` 检查覆盖、顺序、类型及 protected∩deleted=∅。
- 源码行号由 `gc_check.py:68-71` 复核；结构检查通过不等于协议正确，
  人工审查仍须按第 5 节逐步核对读取/到期排序、部分输出、交接窗口、共享与静止回收。

<!-- gc-evidence -->
~~~json
{
  "reader": {
    "actual": [
      {
        "event": "begin",
        "id": "seed",
        "op": "seed"
      },
      {
        "event": "before_put",
        "value": "a",
        "in_transaction": false
      },
      {
        "event": "after_put",
        "value": "a",
        "in_transaction": false
      },
      {
        "event": "before_put",
        "value": "b",
        "in_transaction": false
      },
      {
        "event": "after_put",
        "value": "b",
        "in_transaction": false
      },
      {
        "event": "sql",
        "value": "BEGIN IMMEDIATE",
        "in_transaction": false
      },
      {
        "event": "after_commit",
        "value": 1,
        "in_transaction": true
      },
      {
        "event": "sql",
        "value": "COMMIT",
        "in_transaction": true
      },
      {
        "event": "end",
        "id": "seed",
        "op": "seed",
        "result": 1,
        "head": 1,
        "objects": [
          "a",
          "b"
        ],
        "in_transaction": false
      },
      {
        "event": "begin",
        "id": "reader",
        "op": "read"
      },
      {
        "event": "after_state",
        "value": 1,
        "in_transaction": false
      },
      {
        "event": "before_get",
        "value": "a",
        "in_transaction": false
      },
      {
        "event": "after_get",
        "value": "a",
        "in_transaction": false
      },
      {
        "event": "before_get",
        "value": "b",
        "in_transaction": false
      },
      {
        "event": "begin",
        "id": "p",
        "op": "prepare"
      },
      {
        "event": "after_state",
        "value": 1,
        "in_transaction": false
      },
      {
        "event": "before_get",
        "value": "a",
        "in_transaction": false
      },
      {
        "event": "after_get",
        "value": "a",
        "in_transaction": false
      },
      {
        "event": "before_get",
        "value": "b",
        "in_transaction": false
      },
      {
        "event": "after_get",
        "value": "b",
        "in_transaction": false
      },
      {
        "event": "before_put",
        "value": "rewrite-r-0",
        "in_transaction": false
      },
      {
        "event": "after_put",
        "value": "rewrite-r-0",
        "in_transaction": false
      },
      {
        "event": "before_put",
        "value": "rewrite-r-1",
        "in_transaction": false
      },
      {
        "event": "after_put",
        "value": "rewrite-r-1",
        "in_transaction": false
      },
      {
        "event": "end",
        "id": "p",
        "op": "prepare",
        "result": {
          "base": 1,
          "inputs": [
            "a",
            "b"
          ],
          "outputs": [
            "rewrite-r-0",
            "rewrite-r-1"
          ]
        },
        "head": 1,
        "objects": [
          "a",
          "b",
          "rewrite-r-0",
          "rewrite-r-1"
        ],
        "in_transaction": false
      },
      {
        "event": "begin",
        "id": "publish",
        "op": "publish"
      },
      {
        "event": "sql",
        "value": "BEGIN IMMEDIATE",
        "in_transaction": false
      },
      {
        "event": "after_state",
        "value": 1,
        "in_transaction": true
      },
      {
        "event": "after_commit",
        "value": 2,
        "in_transaction": true
      },
      {
        "event": "sql",
        "value": "COMMIT",
        "in_transaction": true
      },
      {
        "event": "end",
        "id": "publish",
        "op": "publish",
        "result": 2,
        "head": 2,
        "objects": [
          "a",
          "b",
          "rewrite-r-0",
          "rewrite-r-1"
        ],
        "in_transaction": false
      },
      {
        "event": "begin",
        "id": "expire",
        "op": "expire"
      },
      {
        "event": "end",
        "id": "expire",
        "op": "expire",
        "result": {
          "request_only": {
            "id": "expire",
            "op": "expire",
            "snapshot": 1
          }
        },
        "head": 2,
        "objects": [
          "a",
          "b",
          "rewrite-r-0",
          "rewrite-r-1"
        ],
        "in_transaction": false
      },
      {
        "event": "begin",
        "id": "during",
        "op": "collect"
      },
      {
        "event": "end",
        "id": "during",
        "op": "collect",
        "result": {
          "request_only": {
            "id": "during",
            "op": "collect"
          }
        },
        "head": 2,
        "objects": [
          "a",
          "b",
          "rewrite-r-0",
          "rewrite-r-1"
        ],
        "in_transaction": false
      },
      {
        "event": "after_get",
        "value": "b",
        "in_transaction": false
      },
      {
        "event": "end",
        "id": "reader",
        "op": "read",
        "result": {
          "snapshot": 1,
          "rows": [
            {
              "file": "a",
              "pos": 0,
              "value": {
                "reading": 7
              }
            },
            {
              "file": "a",
              "pos": 1,
              "value": {
                "reading": 7
              }
            },
            {
              "file": "b",
              "pos": 0,
              "value": {
                "reading": 9
              }
            },
            {
              "file": "b",
              "pos": 1,
              "value": {
                "reading": 11
              }
            }
          ]
        },
        "head": 2,
        "objects": [
          "a",
          "b",
          "rewrite-r-0",
          "rewrite-r-1"
        ],
        "in_transaction": false
      },
      {
        "event": "begin",
        "id": "after",
        "op": "collect"
      },
      {
        "event": "end",
        "id": "after",
        "op": "collect",
        "result": {
          "request_only": {
            "id": "after",
            "op": "collect"
          }
        },
        "head": 2,
        "objects": [
          "a",
          "b",
          "rewrite-r-0",
          "rewrite-r-1"
        ],
        "in_transaction": false
      },
      {
        "event": "begin",
        "id": "late",
        "op": "read"
      },
      {
        "event": "after_state",
        "value": 1,
        "in_transaction": false
      },
      {
        "event": "before_get",
        "value": "a",
        "in_transaction": false
      },
      {
        "event": "after_get",
        "value": "a",
        "in_transaction": false
      },
      {
        "event": "before_get",
        "value": "b",
        "in_transaction": false
      },
      {
        "event": "after_get",
        "value": "b",
        "in_transaction": false
      },
      {
        "event": "end",
        "id": "late",
        "op": "read",
        "result": {
          "snapshot": 1,
          "rows": [
            {
              "file": "a",
              "pos": 0,
              "value": {
                "reading": 7
              }
            },
            {
              "file": "a",
              "pos": 1,
              "value": {
                "reading": 7
              }
            },
            {
              "file": "b",
              "pos": 0,
              "value": {
                "reading": 9
              }
            },
            {
              "file": "b",
              "pos": 1,
              "value": {
                "reading": 11
              }
            }
          ]
        },
        "head": 2,
        "objects": [
          "a",
          "b",
          "rewrite-r-0",
          "rewrite-r-1"
        ],
        "in_transaction": false
      }
    ],
    "predicted": [
      {
        "id": "seed",
        "outcome": "seed 成功发布快照 1；snap:1 登记并保护 a、b",
        "protected": [
          "a",
          "b"
        ],
        "deleted": [],
        "reason": "受保护依据：该步骤结束时对象仍处于未到期快照 files 行或 gc_holds 持有者并集中"
      },
      {
        "id": "p",
        "outcome": "prepare r 成功（base=1）；job:r 在短临界区登记，输入 a、b 与部分/全部输出 rewrite-r-0、rewrite-r-1 均入 gc_holds",
        "protected": [
          "a",
          "b",
          "rewrite-r-0",
          "rewrite-r-1"
        ],
        "deleted": [],
        "reason": "受保护依据：该步骤结束时对象仍处于未到期快照 files 行或 gc_holds 持有者并集中"
      },
      {
        "id": "publish",
        "outcome": "publish 成功提交快照 2；同事务先立 snap:2 再撤 job:r，a、b 仍由未到期 snap:1 保护，输出由 snap:2 保护",
        "protected": [
          "a",
          "b",
          "rewrite-r-0",
          "rewrite-r-1"
        ],
        "deleted": [],
        "reason": "受保护依据：该步骤结束时对象仍处于未到期快照 files 行或 gc_holds 持有者并集中"
      },
      {
        "id": "expire",
        "outcome": "expire(1) 在独立目录事务中写 gc_expiry；只拒绝其后的新读/新删，不删除任何 holds 行之外的对象，四个对象仍受保护",
        "protected": [
          "a",
          "b",
          "rewrite-r-0",
          "rewrite-r-1"
        ],
        "deleted": [],
        "reason": "受保护依据：该步骤结束时对象仍处于未到期快照 files 行或 gc_holds 持有者并集中"
      },
      {
        "id": "during",
        "outcome": "collect 重算差集：a、b 被在途 read 持有者保护（reader 已在暂停前取得 read token），输出被 snap:2 保护，victim 为空，返回 []",
        "protected": [
          "a",
          "b",
          "rewrite-r-0",
          "rewrite-r-1"
        ],
        "deleted": [],
        "reason": "受保护依据：该步骤结束时对象仍处于未到期快照 files 行或 gc_holds 持有者并集中"
      },
      {
        "id": "reader",
        "outcome": "在途读取在 finally 释放 read token 后结束，返回 [7,7,9,11]，与基线一致；此后 a、b 只剩已到期 snap:1 引用，活动保护仅剩 snap:2 的两个输出",
        "protected": [
          "rewrite-r-0",
          "rewrite-r-1"
        ],
        "deleted": [],
        "reason": "受保护依据：该步骤结束时对象仍处于未到期快照 files 行或 gc_holds 持有者并集中"
      },
      {
        "id": "after",
        "outcome": "静止后 collect：a、b 不属于任何未到期快照且无持有者，锁外实际删除；名称台账保留旧名，输出由 snap:2 继续保护",
        "protected": [
          "rewrite-r-0",
          "rewrite-r-1"
        ],
        "deleted": [
          "a",
          "b"
        ],
        "reason": "删除依据：对象不在任何未到期快照 files 行、不在 gc_holds，且无活动外层事务，collect 在锁外执行实际删除；gc_names 台账保留旧名"
      },
      {
        "id": "late",
        "outcome": "到期后新发起 read(1)：acquire 临界区见到 gc_expiry 行，报 EXPIRED_SNAPSHOT（基线 actual 中该读取成功，这是核心预测差异）；无删除",
        "protected": [
          "rewrite-r-0",
          "rewrite-r-1"
        ],
        "deleted": [],
        "reason": "受保护依据：该步骤结束时对象仍处于未到期快照 files 行或 gc_holds 持有者并集中"
      }
    ]
  },
  "prepared": {
    "actual": [
      {
        "event": "begin",
        "id": "seed",
        "op": "seed"
      },
      {
        "event": "before_put",
        "value": "a",
        "in_transaction": false
      },
      {
        "event": "after_put",
        "value": "a",
        "in_transaction": false
      },
      {
        "event": "before_put",
        "value": "b",
        "in_transaction": false
      },
      {
        "event": "after_put",
        "value": "b",
        "in_transaction": false
      },
      {
        "event": "sql",
        "value": "BEGIN IMMEDIATE",
        "in_transaction": false
      },
      {
        "event": "after_commit",
        "value": 1,
        "in_transaction": true
      },
      {
        "event": "sql",
        "value": "COMMIT",
        "in_transaction": true
      },
      {
        "event": "end",
        "id": "seed",
        "op": "seed",
        "result": 1,
        "head": 1,
        "objects": [
          "a",
          "b"
        ],
        "in_transaction": false
      },
      {
        "event": "begin",
        "id": "p",
        "op": "prepare"
      },
      {
        "event": "after_state",
        "value": 1,
        "in_transaction": false
      },
      {
        "event": "before_get",
        "value": "a",
        "in_transaction": false
      },
      {
        "event": "after_get",
        "value": "a",
        "in_transaction": false
      },
      {
        "event": "before_get",
        "value": "b",
        "in_transaction": false
      },
      {
        "event": "after_get",
        "value": "b",
        "in_transaction": false
      },
      {
        "event": "before_put",
        "value": "rewrite-p-0",
        "in_transaction": false
      },
      {
        "event": "after_put",
        "value": "rewrite-p-0",
        "in_transaction": false
      },
      {
        "event": "begin",
        "id": "partial",
        "op": "collect"
      },
      {
        "event": "end",
        "id": "partial",
        "op": "collect",
        "result": {
          "request_only": {
            "id": "partial",
            "op": "collect"
          }
        },
        "head": 1,
        "objects": [
          "a",
          "b",
          "rewrite-p-0"
        ],
        "in_transaction": false
      },
      {
        "event": "before_put",
        "value": "rewrite-p-1",
        "in_transaction": false
      },
      {
        "event": "after_put",
        "value": "rewrite-p-1",
        "in_transaction": false
      },
      {
        "event": "end",
        "id": "p",
        "op": "prepare",
        "result": {
          "base": 1,
          "inputs": [
            "a",
            "b"
          ],
          "outputs": [
            "rewrite-p-0",
            "rewrite-p-1"
          ]
        },
        "head": 1,
        "objects": [
          "a",
          "b",
          "rewrite-p-0",
          "rewrite-p-1"
        ],
        "in_transaction": false
      },
      {
        "event": "begin",
        "id": "delete",
        "op": "delete"
      },
      {
        "event": "after_state",
        "value": 1,
        "in_transaction": false
      },
      {
        "event": "before_get",
        "value": "a",
        "in_transaction": false
      },
      {
        "event": "after_get",
        "value": "a",
        "in_transaction": false
      },
      {
        "event": "sql",
        "value": "BEGIN IMMEDIATE",
        "in_transaction": false
      },
      {
        "event": "after_state",
        "value": 1,
        "in_transaction": true
      },
      {
        "event": "after_commit",
        "value": 2,
        "in_transaction": true
      },
      {
        "event": "sql",
        "value": "COMMIT",
        "in_transaction": true
      },
      {
        "event": "end",
        "id": "delete",
        "op": "delete",
        "result": 2,
        "head": 2,
        "objects": [
          "a",
          "b",
          "rewrite-p-0",
          "rewrite-p-1"
        ],
        "in_transaction": false
      },
      {
        "event": "begin",
        "id": "expire",
        "op": "expire"
      },
      {
        "event": "end",
        "id": "expire",
        "op": "expire",
        "result": {
          "request_only": {
            "id": "expire",
            "op": "expire",
            "snapshot": 1
          }
        },
        "head": 2,
        "objects": [
          "a",
          "b",
          "rewrite-p-0",
          "rewrite-p-1"
        ],
        "in_transaction": false
      },
      {
        "event": "begin",
        "id": "conflict",
        "op": "publish"
      },
      {
        "event": "sql",
        "value": "BEGIN IMMEDIATE",
        "in_transaction": false
      },
      {
        "event": "after_state",
        "value": 2,
        "in_transaction": true
      },
      {
        "event": "sql",
        "value": "ROLLBACK",
        "in_transaction": true
      },
      {
        "event": "end",
        "id": "conflict",
        "op": "publish",
        "error": "PUBLISH_CONFLICT",
        "head": 2,
        "objects": [
          "a",
          "b",
          "rewrite-p-0",
          "rewrite-p-1"
        ],
        "in_transaction": false
      },
      {
        "event": "begin",
        "id": "held",
        "op": "collect"
      },
      {
        "event": "end",
        "id": "held",
        "op": "collect",
        "result": {
          "request_only": {
            "id": "held",
            "op": "collect"
          }
        },
        "head": 2,
        "objects": [
          "a",
          "b",
          "rewrite-p-0",
          "rewrite-p-1"
        ],
        "in_transaction": false
      },
      {
        "event": "begin",
        "id": "cancel",
        "op": "cancel"
      },
      {
        "event": "end",
        "id": "cancel",
        "op": "cancel",
        "result": {
          "request_only": {
            "id": "cancel",
            "op": "cancel",
            "job": "p"
          }
        },
        "head": 2,
        "objects": [
          "a",
          "b",
          "rewrite-p-0",
          "rewrite-p-1"
        ],
        "in_transaction": false
      },
      {
        "event": "begin",
        "id": "released",
        "op": "collect"
      },
      {
        "event": "end",
        "id": "released",
        "op": "collect",
        "result": {
          "request_only": {
            "id": "released",
            "op": "collect"
          }
        },
        "head": 2,
        "objects": [
          "a",
          "b",
          "rewrite-p-0",
          "rewrite-p-1"
        ],
        "in_transaction": false
      }
    ],
    "predicted": [
      {
        "id": "seed",
        "outcome": "seed 成功发布快照 1；snap:1 保护 a、b",
        "protected": [
          "a",
          "b"
        ],
        "deleted": [],
        "reason": "受保护依据：该步骤结束时对象仍处于未到期快照 files 行或 gc_holds 持有者并集中"
      },
      {
        "id": "partial",
        "outcome": "after_put(rewrite-p-0) 暂停点上的 collect：prepare 登记临界区已让 job:p 持有 a、b 与首个部分输出，victim 为空，返回 []",
        "protected": [
          "a",
          "b",
          "rewrite-p-0"
        ],
        "deleted": [],
        "reason": "受保护依据：该步骤结束时对象仍处于未到期快照 files 行或 gc_holds 持有者并集中"
      },
      {
        "id": "p",
        "outcome": "prepare p 正常结束；job:p 覆盖输入 a、b 与输出 rewrite-p-0/1，snap:1 同时保护 a、b",
        "protected": [
          "a",
          "b",
          "rewrite-p-0",
          "rewrite-p-1"
        ],
        "deleted": [],
        "reason": "受保护依据：该步骤结束时对象仍处于未到期快照 files 行或 gc_holds 持有者并集中"
      },
      {
        "id": "delete",
        "outcome": "delete 成功提交快照 2（a:0 墓碑），同事务建立 snap:2；a、b 仍是 head 文件，计划输出由 job:p 持有，四对象全保护",
        "protected": [
          "a",
          "b",
          "rewrite-p-0",
          "rewrite-p-1"
        ],
        "deleted": [],
        "reason": "受保护依据：该步骤结束时对象仍处于未到期快照 files 行或 gc_holds 持有者并集中"
      },
      {
        "id": "expire",
        "outcome": "expire(1) 生效但 a、b 仍由未到期 snap:2 引用，计划对象由 job:p 持有，无对象失去保护",
        "protected": [
          "a",
          "b",
          "rewrite-p-0",
          "rewrite-p-1"
        ],
        "deleted": [],
        "reason": "受保护依据：该步骤结束时对象仍处于未到期快照 files 行或 gc_holds 持有者并集中"
      },
      {
        "id": "conflict",
        "outcome": "publish 因 head=2 不等于 plan.base=1 报 PUBLISH_CONFLICT，与基线一致；冲突只回滚 publish 自身事务，计划保持 prepared、holds 不动",
        "protected": [
          "a",
          "b",
          "rewrite-p-0",
          "rewrite-p-1"
        ],
        "deleted": [],
        "reason": "受保护依据：该步骤结束时对象仍处于未到期快照 files 行或 gc_holds 持有者并集中"
      },
      {
        "id": "held",
        "outcome": "collect 重算：输出由 job:p 持有、a、b 由 snap:2 持有，无 victim，返回 []；冲突后计划保护被保留",
        "protected": [
          "a",
          "b",
          "rewrite-p-0",
          "rewrite-p-1"
        ],
        "deleted": [],
        "reason": "受保护依据：该步骤结束时对象仍处于未到期快照 files 行或 gc_holds 持有者并集中"
      },
      {
        "id": "cancel",
        "outcome": "cancel(p) 在目录事务内把计划置 cancelled 并删除 job:p 持有行；a、b 仍由 snap:2 保护，cancel 本身不做载荷删除",
        "protected": [
          "a",
          "b"
        ],
        "deleted": [],
        "reason": "受保护依据：该步骤结束时对象仍处于未到期快照 files 行或 gc_holds 持有者并集中"
      },
      {
        "id": "released",
        "outcome": "collect：两个 rewrite 输出不在任何快照且已无持有者，锁外实际删除；a、b 为 head 所用而保留（基线 objects 仍含四对象，此处为预测差异）",
        "protected": [
          "a",
          "b"
        ],
        "deleted": [
          "rewrite-p-0",
          "rewrite-p-1"
        ],
        "reason": "删除依据：对象不在任何未到期快照 files 行、不在 gc_holds，且无活动外层事务，collect 在锁外执行实际删除；gc_names 台账保留旧名"
      }
    ]
  },
  "rollback": {
    "actual": [
      {
        "event": "begin",
        "id": "seed",
        "op": "seed"
      },
      {
        "event": "before_put",
        "value": "a",
        "in_transaction": false
      },
      {
        "event": "after_put",
        "value": "a",
        "in_transaction": false
      },
      {
        "event": "before_put",
        "value": "b",
        "in_transaction": false
      },
      {
        "event": "after_put",
        "value": "b",
        "in_transaction": false
      },
      {
        "event": "sql",
        "value": "BEGIN IMMEDIATE",
        "in_transaction": false
      },
      {
        "event": "after_commit",
        "value": 1,
        "in_transaction": true
      },
      {
        "event": "sql",
        "value": "COMMIT",
        "in_transaction": true
      },
      {
        "event": "end",
        "id": "seed",
        "op": "seed",
        "result": 1,
        "head": 1,
        "objects": [
          "a",
          "b"
        ],
        "in_transaction": false
      },
      {
        "event": "begin",
        "id": "p",
        "op": "prepare"
      },
      {
        "event": "after_state",
        "value": 1,
        "in_transaction": false
      },
      {
        "event": "before_get",
        "value": "a",
        "in_transaction": false
      },
      {
        "event": "after_get",
        "value": "a",
        "in_transaction": false
      },
      {
        "event": "before_get",
        "value": "b",
        "in_transaction": false
      },
      {
        "event": "after_get",
        "value": "b",
        "in_transaction": false
      },
      {
        "event": "before_put",
        "value": "rewrite-p-0",
        "in_transaction": false
      },
      {
        "event": "after_put",
        "value": "rewrite-p-0",
        "in_transaction": false
      },
      {
        "event": "before_put",
        "value": "rewrite-p-1",
        "in_transaction": false
      },
      {
        "event": "after_put",
        "value": "rewrite-p-1",
        "in_transaction": false
      },
      {
        "event": "end",
        "id": "p",
        "op": "prepare",
        "result": {
          "base": 1,
          "inputs": [
            "a",
            "b"
          ],
          "outputs": [
            "rewrite-p-0",
            "rewrite-p-1"
          ]
        },
        "head": 1,
        "objects": [
          "a",
          "b",
          "rewrite-p-0",
          "rewrite-p-1"
        ],
        "in_transaction": false
      },
      {
        "event": "begin",
        "id": "tx",
        "op": "rollback"
      },
      {
        "event": "sql",
        "value": "BEGIN IMMEDIATE",
        "in_transaction": false
      },
      {
        "event": "begin",
        "id": "publish_inside",
        "op": "publish"
      },
      {
        "event": "after_state",
        "value": 1,
        "in_transaction": true
      },
      {
        "event": "after_commit",
        "value": 2,
        "in_transaction": true
      },
      {
        "event": "end",
        "id": "publish_inside",
        "op": "publish",
        "result": 2,
        "head": 2,
        "objects": [
          "a",
          "b",
          "rewrite-p-0",
          "rewrite-p-1"
        ],
        "in_transaction": true
      },
      {
        "event": "begin",
        "id": "expire_inside",
        "op": "expire"
      },
      {
        "event": "end",
        "id": "expire_inside",
        "op": "expire",
        "result": {
          "request_only": {
            "id": "expire_inside",
            "op": "expire",
            "snapshot": 1
          }
        },
        "head": 2,
        "objects": [
          "a",
          "b",
          "rewrite-p-0",
          "rewrite-p-1"
        ],
        "in_transaction": true
      },
      {
        "event": "begin",
        "id": "collect_inside",
        "op": "collect"
      },
      {
        "event": "end",
        "id": "collect_inside",
        "op": "collect",
        "result": {
          "request_only": {
            "id": "collect_inside",
            "op": "collect"
          }
        },
        "head": 2,
        "objects": [
          "a",
          "b",
          "rewrite-p-0",
          "rewrite-p-1"
        ],
        "in_transaction": true
      },
      {
        "event": "begin",
        "id": "read_inside",
        "op": "read"
      },
      {
        "event": "after_state",
        "value": 2,
        "in_transaction": true
      },
      {
        "event": "before_get",
        "value": "rewrite-p-0",
        "in_transaction": true
      },
      {
        "event": "after_get",
        "value": "rewrite-p-0",
        "in_transaction": true
      },
      {
        "event": "before_get",
        "value": "rewrite-p-1",
        "in_transaction": true
      },
      {
        "event": "after_get",
        "value": "rewrite-p-1",
        "in_transaction": true
      },
      {
        "event": "end",
        "id": "read_inside",
        "op": "read",
        "result": {
          "snapshot": 2,
          "rows": [
            {
              "file": "rewrite-p-0",
              "pos": 0,
              "value": {
                "reading": 7
              }
            },
            {
              "file": "rewrite-p-0",
              "pos": 1,
              "value": {
                "reading": 9
              }
            },
            {
              "file": "rewrite-p-1",
              "pos": 0,
              "value": {
                "reading": 7
              }
            },
            {
              "file": "rewrite-p-1",
              "pos": 1,
              "value": {
                "reading": 11
              }
            }
          ]
        },
        "head": 2,
        "objects": [
          "a",
          "b",
          "rewrite-p-0",
          "rewrite-p-1"
        ],
        "in_transaction": true
      },
      {
        "event": "sql",
        "value": "ROLLBACK",
        "in_transaction": true
      },
      {
        "event": "end",
        "id": "tx",
        "op": "rollback",
        "result": "ROLLED_BACK",
        "head": 1,
        "objects": [
          "a",
          "b",
          "rewrite-p-0",
          "rewrite-p-1"
        ],
        "in_transaction": false
      },
      {
        "event": "begin",
        "id": "after_rollback",
        "op": "collect"
      },
      {
        "event": "end",
        "id": "after_rollback",
        "op": "collect",
        "result": {
          "request_only": {
            "id": "after_rollback",
            "op": "collect"
          }
        },
        "head": 1,
        "objects": [
          "a",
          "b",
          "rewrite-p-0",
          "rewrite-p-1"
        ],
        "in_transaction": false
      },
      {
        "event": "begin",
        "id": "publish",
        "op": "publish"
      },
      {
        "event": "sql",
        "value": "BEGIN IMMEDIATE",
        "in_transaction": false
      },
      {
        "event": "after_state",
        "value": 1,
        "in_transaction": true
      },
      {
        "event": "after_commit",
        "value": 2,
        "in_transaction": true
      },
      {
        "event": "sql",
        "value": "COMMIT",
        "in_transaction": true
      },
      {
        "event": "end",
        "id": "publish",
        "op": "publish",
        "result": 2,
        "head": 2,
        "objects": [
          "a",
          "b",
          "rewrite-p-0",
          "rewrite-p-1"
        ],
        "in_transaction": false
      },
      {
        "event": "begin",
        "id": "expire",
        "op": "expire"
      },
      {
        "event": "end",
        "id": "expire",
        "op": "expire",
        "result": {
          "request_only": {
            "id": "expire",
            "op": "expire",
            "snapshot": 1
          }
        },
        "head": 2,
        "objects": [
          "a",
          "b",
          "rewrite-p-0",
          "rewrite-p-1"
        ],
        "in_transaction": false
      },
      {
        "event": "begin",
        "id": "after_commit",
        "op": "collect"
      },
      {
        "event": "end",
        "id": "after_commit",
        "op": "collect",
        "result": {
          "request_only": {
            "id": "after_commit",
            "op": "collect"
          }
        },
        "head": 2,
        "objects": [
          "a",
          "b",
          "rewrite-p-0",
          "rewrite-p-1"
        ],
        "in_transaction": false
      },
      {
        "event": "begin",
        "id": "current",
        "op": "read"
      },
      {
        "event": "after_state",
        "value": 2,
        "in_transaction": false
      },
      {
        "event": "before_get",
        "value": "rewrite-p-0",
        "in_transaction": false
      },
      {
        "event": "after_get",
        "value": "rewrite-p-0",
        "in_transaction": false
      },
      {
        "event": "before_get",
        "value": "rewrite-p-1",
        "in_transaction": false
      },
      {
        "event": "after_get",
        "value": "rewrite-p-1",
        "in_transaction": false
      },
      {
        "event": "end",
        "id": "current",
        "op": "read",
        "result": {
          "snapshot": 2,
          "rows": [
            {
              "file": "rewrite-p-0",
              "pos": 0,
              "value": {
                "reading": 7
              }
            },
            {
              "file": "rewrite-p-0",
              "pos": 1,
              "value": {
                "reading": 9
              }
            },
            {
              "file": "rewrite-p-1",
              "pos": 0,
              "value": {
                "reading": 7
              }
            },
            {
              "file": "rewrite-p-1",
              "pos": 1,
              "value": {
                "reading": 11
              }
            }
          ]
        },
        "head": 2,
        "objects": [
          "a",
          "b",
          "rewrite-p-0",
          "rewrite-p-1"
        ],
        "in_transaction": false
      }
    ],
    "predicted": [
      {
        "id": "seed",
        "outcome": "seed 成功发布快照 1；snap:1 保护 a、b",
        "protected": [
          "a",
          "b"
        ],
        "deleted": [],
        "reason": "受保护依据：该步骤结束时对象仍处于未到期快照 files 行或 gc_holds 持有者并集中"
      },
      {
        "id": "p",
        "outcome": "prepare p 成功；job:p 持有输入 a、b 与输出 rewrite-p-0/1，与 snap:1 并集为四对象",
        "protected": [
          "a",
          "b",
          "rewrite-p-0",
          "rewrite-p-1"
        ],
        "deleted": [],
        "reason": "受保护依据：该步骤结束时对象仍处于未到期快照 files 行或 gc_holds 持有者并集中"
      },
      {
        "id": "publish_inside",
        "outcome": "外层事务内 publish 成功建快照 2，同事务完成 job:p→snap:2 交接；变更尚未提交，snap:1 仍活动，四对象全保护",
        "protected": [
          "a",
          "b",
          "rewrite-p-0",
          "rewrite-p-1"
        ],
        "deleted": [],
        "reason": "受保护依据：该步骤结束时对象仍处于未到期快照 files 行或 gc_holds 持有者并集中"
      },
      {
        "id": "expire_inside",
        "outcome": "外层事务内 expire(1) 写入到期行；仅事务内可见，四对象仍被事务内状态保护",
        "protected": [
          "a",
          "b",
          "rewrite-p-0",
          "rewrite-p-1"
        ],
        "deleted": [],
        "reason": "受保护依据：该步骤结束时对象仍处于未到期快照 files 行或 gc_holds 持有者并集中"
      },
      {
        "id": "collect_inside",
        "outcome": "检测到 in_transaction=True，collect 暂缓并返回 []，不枚举删除、不记忆候选；候选可在事务后由持久状态无状态重算，不会被遗忘",
        "protected": [
          "a",
          "b",
          "rewrite-p-0",
          "rewrite-p-1"
        ],
        "deleted": [],
        "reason": "受保护依据：该步骤结束时对象仍处于未到期快照 files 行或 gc_holds 持有者并集中"
      },
      {
        "id": "read_inside",
        "outcome": "事务内 read 看到快照 2 并正常完成，与基线 read_inside snapshot=2 一致；读取 token 在同事务内取得与释放",
        "protected": [
          "a",
          "b",
          "rewrite-p-0",
          "rewrite-p-1"
        ],
        "deleted": [],
        "reason": "受保护依据：该步骤结束时对象仍处于未到期快照 files 行或 gc_holds 持有者并集中"
      },
      {
        "id": "tx",
        "outcome": "外层 ROLLBACK：快照 2、到期行、交接全部撤销，head 恢复 1，计划 p 恢复 prepared 并重新持有输入与输出，四对象全保护",
        "protected": [
          "a",
          "b",
          "rewrite-p-0",
          "rewrite-p-1"
        ],
        "deleted": [],
        "reason": "受保护依据：该步骤结束时对象仍处于未到期快照 files 行或 gc_holds 持有者并集中"
      },
      {
        "id": "after_rollback",
        "outcome": "回滚后 collect：snap:1 与 job:p 覆盖全部四对象，无 victim，返回 []；回滚前的临时可达性未被当成最终状态",
        "protected": [
          "a",
          "b",
          "rewrite-p-0",
          "rewrite-p-1"
        ],
        "deleted": [],
        "reason": "受保护依据：该步骤结束时对象仍处于未到期快照 files 行或 gc_holds 持有者并集中"
      },
      {
        "id": "publish",
        "outcome": "计划重新 publish 成功，提交快照 2（结果 2，与基线一致）；snap:1 未到期仍护 a、b，snap:2 护输出",
        "protected": [
          "a",
          "b",
          "rewrite-p-0",
          "rewrite-p-1"
        ],
        "deleted": [],
        "reason": "受保护依据：该步骤结束时对象仍处于未到期快照 files 行或 gc_holds 持有者并集中"
      },
      {
        "id": "expire",
        "outcome": "expire(1) 提交生效；a、b 不再属于任何未到期快照且无其他持有者，保护并集仅剩 snap:2 的两个输出",
        "protected": [
          "rewrite-p-0",
          "rewrite-p-1"
        ],
        "deleted": [],
        "reason": "受保护依据：该步骤结束时对象仍处于未到期快照 files 行或 gc_holds 持有者并集中"
      },
      {
        "id": "after_commit",
        "outcome": "静止后 collect：a、b 成为 victim 并锁外实际删除，输出由 snap:2 保留；名称台账保留 a、b 旧名",
        "protected": [
          "rewrite-p-0",
          "rewrite-p-1"
        ],
        "deleted": [
          "a",
          "b"
        ],
        "reason": "删除依据：对象不在任何未到期快照 files 行、不在 gc_holds，且无活动外层事务，collect 在锁外执行实际删除；gc_names 台账保留旧名"
      },
      {
        "id": "current",
        "outcome": "head read 正常返回快照 2 的 [7,9,7,11] 四行，与基线行为一致；无新删除",
        "protected": [
          "rewrite-p-0",
          "rewrite-p-1"
        ],
        "deleted": [],
        "reason": "受保护依据：该步骤结束时对象仍处于未到期快照 files 行或 gc_holds 持有者并集中"
      }
    ]
  },
  "cancel_name": {
    "actual": [
      {
        "event": "begin",
        "id": "seed",
        "op": "seed"
      },
      {
        "event": "before_put",
        "value": "a",
        "in_transaction": false
      },
      {
        "event": "after_put",
        "value": "a",
        "in_transaction": false
      },
      {
        "event": "before_put",
        "value": "b",
        "in_transaction": false
      },
      {
        "event": "after_put",
        "value": "b",
        "in_transaction": false
      },
      {
        "event": "sql",
        "value": "BEGIN IMMEDIATE",
        "in_transaction": false
      },
      {
        "event": "after_commit",
        "value": 1,
        "in_transaction": true
      },
      {
        "event": "sql",
        "value": "COMMIT",
        "in_transaction": true
      },
      {
        "event": "end",
        "id": "seed",
        "op": "seed",
        "result": 1,
        "head": 1,
        "objects": [
          "a",
          "b"
        ],
        "in_transaction": false
      },
      {
        "event": "begin",
        "id": "p",
        "op": "prepare"
      },
      {
        "event": "after_state",
        "value": 1,
        "in_transaction": false
      },
      {
        "event": "before_get",
        "value": "a",
        "in_transaction": false
      },
      {
        "event": "after_get",
        "value": "a",
        "in_transaction": false
      },
      {
        "event": "before_get",
        "value": "b",
        "in_transaction": false
      },
      {
        "event": "after_get",
        "value": "b",
        "in_transaction": false
      },
      {
        "event": "before_put",
        "value": "rewrite-c-0",
        "in_transaction": false
      },
      {
        "event": "after_put",
        "value": "rewrite-c-0",
        "in_transaction": false
      },
      {
        "event": "before_put",
        "value": "rewrite-c-1",
        "in_transaction": false
      },
      {
        "event": "after_put",
        "value": "rewrite-c-1",
        "in_transaction": false
      },
      {
        "event": "end",
        "id": "p",
        "op": "prepare",
        "result": {
          "base": 1,
          "inputs": [
            "a",
            "b"
          ],
          "outputs": [
            "rewrite-c-0",
            "rewrite-c-1"
          ]
        },
        "head": 1,
        "objects": [
          "a",
          "b",
          "rewrite-c-0",
          "rewrite-c-1"
        ],
        "in_transaction": false
      },
      {
        "event": "begin",
        "id": "cancel",
        "op": "cancel"
      },
      {
        "event": "end",
        "id": "cancel",
        "op": "cancel",
        "result": {
          "request_only": {
            "id": "cancel",
            "op": "cancel",
            "job": "c"
          }
        },
        "head": 1,
        "objects": [
          "a",
          "b",
          "rewrite-c-0",
          "rewrite-c-1"
        ],
        "in_transaction": false
      },
      {
        "event": "begin",
        "id": "collect",
        "op": "collect"
      },
      {
        "event": "end",
        "id": "collect",
        "op": "collect",
        "result": {
          "request_only": {
            "id": "collect",
            "op": "collect"
          }
        },
        "head": 1,
        "objects": [
          "a",
          "b",
          "rewrite-c-0",
          "rewrite-c-1"
        ],
        "in_transaction": false
      },
      {
        "event": "begin",
        "id": "raw_remove",
        "op": "raw_drop"
      },
      {
        "event": "drop",
        "value": "rewrite-c-0",
        "in_transaction": false
      },
      {
        "event": "end",
        "id": "raw_remove",
        "op": "raw_drop",
        "result": null,
        "head": 1,
        "objects": [
          "a",
          "b",
          "rewrite-c-1"
        ],
        "in_transaction": false
      },
      {
        "event": "begin",
        "id": "raw_reuse",
        "op": "raw_put"
      },
      {
        "event": "before_put",
        "value": "rewrite-c-0",
        "in_transaction": false
      },
      {
        "event": "after_put",
        "value": "rewrite-c-0",
        "in_transaction": false
      },
      {
        "event": "end",
        "id": "raw_reuse",
        "op": "raw_put",
        "result": null,
        "head": 1,
        "objects": [
          "a",
          "b",
          "rewrite-c-0",
          "rewrite-c-1"
        ],
        "in_transaction": false
      }
    ],
    "predicted": [
      {
        "id": "seed",
        "outcome": "seed 成功发布快照 1；snap:1 保护 a、b",
        "protected": [
          "a",
          "b"
        ],
        "deleted": [],
        "reason": "受保护依据：该步骤结束时对象仍处于未到期快照 files 行或 gc_holds 持有者并集中"
      },
      {
        "id": "p",
        "outcome": "prepare c 成功；job:c 持有 a、b 与 rewrite-c-0/1，名称台账登记四个历史名",
        "protected": [
          "a",
          "b",
          "rewrite-c-0",
          "rewrite-c-1"
        ],
        "deleted": [],
        "reason": "受保护依据：该步骤结束时对象仍处于未到期快照 files 行或 gc_holds 持有者并集中"
      },
      {
        "id": "cancel",
        "outcome": "cancel(c) 删除 job:c 持有行；a、b 由 snap:1 继续保护，协议不做载荷删除",
        "protected": [
          "a",
          "b"
        ],
        "deleted": [],
        "reason": "受保护依据：该步骤结束时对象仍处于未到期快照 files 行或 gc_holds 持有者并集中"
      },
      {
        "id": "collect",
        "outcome": "两个 rewrite 输出无任何快照引用且已无持有者，collect 锁外实际删除；a、b 保留；gc_names 中旧名永久保留",
        "protected": [
          "a",
          "b"
        ],
        "deleted": [
          "rewrite-c-0",
          "rewrite-c-1"
        ],
        "reason": "删除依据：对象不在任何未到期快照 files 行、不在 gc_holds，且无活动外层事务，collect 在锁外执行实际删除；gc_names 台账保留旧名"
      },
      {
        "id": "raw_remove",
        "outcome": "raw_drop 是绕过目录的存储诊断（gc_observe.py:21-27），不是 collect 的合法删除：本步骤协议 deleted 为空；对象物理消失不改变 gc_names 台账",
        "protected": [
          "a",
          "b"
        ],
        "deleted": [],
        "reason": "受保护依据：该步骤结束时对象仍处于未到期快照 files 行或 gc_holds 持有者并集中"
      },
      {
        "id": "raw_reuse",
        "outcome": "raw_put 同名异值同样是诊断操作，不属于回收协议；方案的 GuardedObjects 在业务端口以名称台账拒绝同名异值复用，合法新对象只能取新名，故协议 deleted 为空",
        "protected": [
          "a",
          "b"
        ],
        "deleted": [],
        "reason": "受保护依据：该步骤结束时对象仍处于未到期快照 files 行或 gc_holds 持有者并集中"
      }
    ]
  },
  "shared_empty": {
    "actual": [
      {
        "event": "begin",
        "id": "seed",
        "op": "seed"
      },
      {
        "event": "before_put",
        "value": "a",
        "in_transaction": false
      },
      {
        "event": "after_put",
        "value": "a",
        "in_transaction": false
      },
      {
        "event": "before_put",
        "value": "b",
        "in_transaction": false
      },
      {
        "event": "after_put",
        "value": "b",
        "in_transaction": false
      },
      {
        "event": "before_put",
        "value": "empty",
        "in_transaction": false
      },
      {
        "event": "after_put",
        "value": "empty",
        "in_transaction": false
      },
      {
        "event": "sql",
        "value": "BEGIN IMMEDIATE",
        "in_transaction": false
      },
      {
        "event": "after_commit",
        "value": 1,
        "in_transaction": true
      },
      {
        "event": "sql",
        "value": "COMMIT",
        "in_transaction": true
      },
      {
        "event": "end",
        "id": "seed",
        "op": "seed",
        "result": 1,
        "head": 1,
        "objects": [
          "a",
          "b",
          "empty"
        ],
        "in_transaction": false
      },
      {
        "event": "begin",
        "id": "delete",
        "op": "delete"
      },
      {
        "event": "after_state",
        "value": 1,
        "in_transaction": false
      },
      {
        "event": "before_get",
        "value": "a",
        "in_transaction": false
      },
      {
        "event": "after_get",
        "value": "a",
        "in_transaction": false
      },
      {
        "event": "sql",
        "value": "BEGIN IMMEDIATE",
        "in_transaction": false
      },
      {
        "event": "after_state",
        "value": 1,
        "in_transaction": true
      },
      {
        "event": "after_commit",
        "value": 2,
        "in_transaction": true
      },
      {
        "event": "sql",
        "value": "COMMIT",
        "in_transaction": true
      },
      {
        "event": "end",
        "id": "delete",
        "op": "delete",
        "result": 2,
        "head": 2,
        "objects": [
          "a",
          "b",
          "empty"
        ],
        "in_transaction": false
      },
      {
        "event": "begin",
        "id": "prepare",
        "op": "prepare"
      },
      {
        "event": "after_state",
        "value": 2,
        "in_transaction": false
      },
      {
        "event": "before_get",
        "value": "empty",
        "in_transaction": false
      },
      {
        "event": "after_get",
        "value": "empty",
        "in_transaction": false
      },
      {
        "event": "end",
        "id": "prepare",
        "op": "prepare",
        "result": {
          "base": 2,
          "inputs": [
            "empty"
          ],
          "outputs": []
        },
        "head": 2,
        "objects": [
          "a",
          "b",
          "empty"
        ],
        "in_transaction": false
      },
      {
        "event": "begin",
        "id": "publish",
        "op": "publish"
      },
      {
        "event": "sql",
        "value": "BEGIN IMMEDIATE",
        "in_transaction": false
      },
      {
        "event": "after_state",
        "value": 2,
        "in_transaction": true
      },
      {
        "event": "after_commit",
        "value": 3,
        "in_transaction": true
      },
      {
        "event": "sql",
        "value": "COMMIT",
        "in_transaction": true
      },
      {
        "event": "end",
        "id": "publish",
        "op": "publish",
        "result": 3,
        "head": 3,
        "objects": [
          "a",
          "b",
          "empty"
        ],
        "in_transaction": false
      },
      {
        "event": "begin",
        "id": "expire_old",
        "op": "expire"
      },
      {
        "event": "end",
        "id": "expire_old",
        "op": "expire",
        "result": {
          "request_only": {
            "id": "expire_old",
            "op": "expire",
            "snapshot": 1
          }
        },
        "head": 3,
        "objects": [
          "a",
          "b",
          "empty"
        ],
        "in_transaction": false
      },
      {
        "event": "begin",
        "id": "shared",
        "op": "collect"
      },
      {
        "event": "end",
        "id": "shared",
        "op": "collect",
        "result": {
          "request_only": {
            "id": "shared",
            "op": "collect"
          }
        },
        "head": 3,
        "objects": [
          "a",
          "b",
          "empty"
        ],
        "in_transaction": false
      },
      {
        "event": "begin",
        "id": "expire_middle",
        "op": "expire"
      },
      {
        "event": "end",
        "id": "expire_middle",
        "op": "expire",
        "result": {
          "request_only": {
            "id": "expire_middle",
            "op": "expire",
            "snapshot": 2
          }
        },
        "head": 3,
        "objects": [
          "a",
          "b",
          "empty"
        ],
        "in_transaction": false
      },
      {
        "event": "begin",
        "id": "empty",
        "op": "collect"
      },
      {
        "event": "end",
        "id": "empty",
        "op": "collect",
        "result": {
          "request_only": {
            "id": "empty",
            "op": "collect"
          }
        },
        "head": 3,
        "objects": [
          "a",
          "b",
          "empty"
        ],
        "in_transaction": false
      },
      {
        "event": "begin",
        "id": "current",
        "op": "read"
      },
      {
        "event": "after_state",
        "value": 3,
        "in_transaction": false
      },
      {
        "event": "before_get",
        "value": "a",
        "in_transaction": false
      },
      {
        "event": "after_get",
        "value": "a",
        "in_transaction": false
      },
      {
        "event": "before_get",
        "value": "b",
        "in_transaction": false
      },
      {
        "event": "after_get",
        "value": "b",
        "in_transaction": false
      },
      {
        "event": "end",
        "id": "current",
        "op": "read",
        "result": {
          "snapshot": 3,
          "rows": [
            {
              "file": "a",
              "pos": 1,
              "value": {
                "reading": 7
              }
            },
            {
              "file": "b",
              "pos": 0,
              "value": {
                "reading": 9
              }
            },
            {
              "file": "b",
              "pos": 1,
              "value": {
                "reading": 11
              }
            }
          ]
        },
        "head": 3,
        "objects": [
          "a",
          "b",
          "empty"
        ],
        "in_transaction": false
      }
    ],
    "predicted": [
      {
        "id": "seed",
        "outcome": "seed 发布快照 1，含零字节对象 empty；空文件是具名对象，snap:1 同等保护 a、b、empty",
        "protected": [
          "a",
          "b",
          "empty"
        ],
        "deleted": [],
        "reason": "受保护依据：该步骤结束时对象仍处于未到期快照 files 行或 gc_holds 持有者并集中"
      },
      {
        "id": "delete",
        "outcome": "delete 提交快照 2（a:0 墓碑）；三文件仍在 files 行，snap:2 与 snap:1 并集保护 a、b、empty",
        "protected": [
          "a",
          "b",
          "empty"
        ],
        "deleted": [],
        "reason": "受保护依据：该步骤结束时对象仍处于未到期快照 files 行或 gc_holds 持有者并集中"
      },
      {
        "id": "prepare",
        "outcome": "prepare e 以 empty 为输入、layout=[] 合法且无输出；job:e 持有输入 empty，与两个快照并集仍为三对象",
        "protected": [
          "a",
          "b",
          "empty"
        ],
        "deleted": [],
        "reason": "受保护依据：该步骤结束时对象仍处于未到期快照 files 行或 gc_holds 持有者并集中"
      },
      {
        "id": "publish",
        "outcome": "publish 提交快照 3，empty 随输入移除；job:e 同事务撤销，但 empty 仍由未到期 snap:2、snap:1 引用，三对象全保护",
        "protected": [
          "a",
          "b",
          "empty"
        ],
        "deleted": [],
        "reason": "受保护依据：该步骤结束时对象仍处于未到期快照 files 行或 gc_holds 持有者并集中"
      },
      {
        "id": "expire_old",
        "outcome": "expire(1) 生效；empty 与 a、b 仍被未到期 snap:2（及 head snap:3）引用，保护不缩减",
        "protected": [
          "a",
          "b",
          "empty"
        ],
        "deleted": [],
        "reason": "受保护依据：该步骤结束时对象仍处于未到期快照 files 行或 gc_holds 持有者并集中"
      },
      {
        "id": "shared",
        "outcome": "collect：empty 被 snap:2 共享引用而保留，零字节不等于对象不存在；无 victim，返回 []",
        "protected": [
          "a",
          "b",
          "empty"
        ],
        "deleted": [],
        "reason": "受保护依据：该步骤结束时对象仍处于未到期快照 files 行或 gc_holds 持有者并集中"
      },
      {
        "id": "expire_middle",
        "outcome": "expire(2) 生效；empty 不再属于任何未到期快照且无持有者，a、b 仍由 head snap:3 保护",
        "protected": [
          "a",
          "b"
        ],
        "deleted": [],
        "reason": "受保护依据：该步骤结束时对象仍处于未到期快照 files 行或 gc_holds 持有者并集中"
      },
      {
        "id": "empty",
        "outcome": "collect：empty 成为 victim 并实际删除（判定只看具名引用，不看字节数）；a、b 保留",
        "protected": [
          "a",
          "b"
        ],
        "deleted": [
          "empty"
        ],
        "reason": "删除依据：对象不在任何未到期快照 files 行、不在 gc_holds，且无活动外层事务，collect 在锁外执行实际删除；gc_names 台账保留旧名"
      },
      {
        "id": "current",
        "outcome": "head read 返回快照 3 的 [7,9,11]，与基线一致；无新删除",
        "protected": [
          "a",
          "b"
        ],
        "deleted": [],
        "reason": "受保护依据：该步骤结束时对象仍处于未到期快照 files 行或 gc_holds 持有者并集中"
      }
    ]
  }
}
~~~
