# 回 zc → 本机贡献者：拆分与接口的方向答复（v1）

- 日期：2026-10-10
- 来自：zc（项目推进者 / 宿主侧）
- 对应你那份：`handoff-to-zccored-2026-10-10.md`（两卷合一 v2）
- 结论口径：**我这边从"开发"转向"优化"** —— 结构拆分由我做，功能继续由你往前推

---

## 〇、先说三件你已经可以不用等我做的事

1. **你可以放心改 `dark_theme.py`（仓库那份）** —— 见下面第 8 条，**它不是共享模块**。
2. **`EngineAPI` 我会按你的建议做**，但**先拆包、后定接口**，别为了等我冻契约而停手。
3. **你的四张表我复核过了，结论成立**（引擎寄生在 UI 类里、必须下探到方法级）。抽检了
   `_run_group_loop` 的体量与 `data-coupling` 里的几个属性，与你给的一致。

---

## 一、**第 8 条（README 那句"两边是同一套代码"）—— 已确认：不成立，请按新口径**

你问得对，而且这条迟早会咬人。

**事实**：宿主 `main.py` 走的是 `from img_server import ...`，读的是**它自己同目录**那份；
GitHub 上的 `PortPanel` 是**复制过去的一份**。两边**不是同一个文件**，会各自漂移。

**对你的直接影响（重要）**：

| 你的担心 | 结论 |
|---|---|
| "不单方面改共享模块 `dark_theme.py`（第 8 条未答之前）" | **可以改**。仓库那份 `dark_theme.py` 改了**不会**影响宿主 |
| "改接口会不会连带炸宿主" | 只要不改**宿主目录**里的文件，就不会 |
| 反向风险 | 宿主那边的 `img_server.py` 更新**不会**自动进你这个仓库 —— 需要我手动同步 |

**所以我这边的口径改成**：`PortPanel` 是**独立项目**，宿主只是它的一个"兄弟副本"。
README 里那句我会改成"与主程序的关系：同源的两个副本，各自演进"。

> 建议你那边也据此调整：把 `dark_theme` 当**本仓库自有模块**看待，改之前不用再问我。

---

## 二、**第 2 条（21 个 0 引用符号：宿主会不会调）—— 查完了，11 个不能删**

我把宿主的 `main.py` / `user.py` / `local_server.py` / `screens.py` 全扫了一遍
（共约 10,100 行）：

### ★ 宿主正在用 —— **不能删**（11 个）

| 符号 | 宿主调用处 |
|---|---|
| `secure_store.unlock` | **user.py ×11** |
| `aliyun_client.compare_local_remote_file` | user.py ×4 |
| `aliyun_client.sync_file_from_oss` / `download_bytes_from_oss` / `get_remote_file_info` | user.py ×3 各 |
| `aliyun_client.upload_bytes_to_oss` | user.py ×2 |
| `aliyun_client.send_verification_bytes` | **main.py ×2** |
| `aliyun_client.start_local_sync_server` | main.py ×1 |
| `aliyun_client.sync_file_to_oss` | user.py ×1 |
| `secure_store.ensure_file_encrypted` / `session_remaining_seconds` | user.py ×2 各 |
| `secure_store.get_plaintext` / `set_plaintext` | user.py ×2 各（你已白名单 ✓） |
| **`img_server.open_platform_integrator`** | **main.py ×7**（你白名单里那条 ✓） |

> ⚠️ 你那批 `aliyun_client.*` 的白名单判断只扫了 `img_server` 一侧，所以漏了 ——
> 真正在用的是 **`user.py`（事件查看器）**。这条建议回填进你的 `unused-residue` 表。

### 真正 0 引用（宿主也不用）—— 可进删除候选（10 个）

```
aliyun_client.stop_local_sync_server      aliyun_client.get_global_proxy_client
img_server._make_robust_request           img_server._session_log_close
img_server.manifest_should_skip           img_server.JSONTreeWidget
secure_store.crypto_available             secure_store.clear_cache
secure_store.last_auth_utc                tour_script_image_source._ep_ready
```

**但先别删** —— 等拆包完、跑完回归再统一清（`_make_robust_request` 你我都标过是死代码，
但删它属于"优化"范畴，我想放在拆分之后单独一轮做，好定位问题）。

---

## 三、第 1 条（怎么切）—— 我采纳你的"按职责 + 窄接口"，并已动手

### 已经做完的（本轮）

**把项目从"一堆平铺 .py"拆成了包**，落在**主程序之外**的独立目录：

```
portpanel/
├─ __init__.py
├─ core/            无 Qt 依赖：配置模型、参数解析、算术求值
├─ engine/          执行循环 / 下载 / 落盘 / 清单 / 会话日志 / 交接协议
├─ ui/              flow_editor · image_source · tour_layer · tour_script_*
└─ integration/     theme · logger · secure_store · aliyun_client
```

对应到具体文件：

| 原来 | 现在 |
|---|---|
| `img_server.py`（821 KB / 17,547 行） | `portpanel/ui/flow_editor.py` |
| `api_config_dialog.py` | `portpanel/ui/image_source.py` |
| `dark_theme.py` | `portpanel/integration/theme.py` |
| `logger_manager.py` / `secure_store.py` / `aliyun_client.py` | `portpanel/integration/*` |
| `tour_layer.py` / `tour_script_*.py` | `portpanel/ui/*` |
| `port_panel.py` | 仍在根（入口），只改 import |

**验证**：12 个模块全部可导入、`port_panel.main()` 正常返回 0、`compileall` 通过。
**主程序目录一字未动**，宿主不受影响。

### 还没做的（下一批，按你的方法级刀口）

`flow_editor.py` 现在**还是 821 KB 一整块** —— 这一轮只做了"包边界"，**方法级外迁还没开始**。
我会按你 `method-split` 表的分类往下搬，顺序：

1. `core/`：先搬**零 Qt 依赖**的（`TransferConfig` / `TRANSFER_SPECS` / 参数解析）—— 风险最低
2. **`engine_api.py`**：定窄接口（见下节），**你冻契约就等这个**
3. `engine/`：逐块搬 `_run_group_loop`(561) / `_download_batch` / `_serialize_flow` / 落盘管线
4. `MIXED` 的 8 个方法人工拆
5. `NodeScene` / `NodeView`：按你说的**原样保留**，我不动（等你 Web 化时整体重写）

---

## 四、第 3~6 条 —— 方向（先给判断，细节随拆分落地）

| # | 议题 | 我的方向 |
|---|---|---|
| 3 | `manifest_check` / `manifest_add` 归属 | **采纳你的建议**：`manifest_check`（只读查询）留作接口；`manifest_add` 并入落盘管线，不单独暴露 |
| 4 | `_io_store_data*` 归属 | **采纳你的建议 ①**：留作 `EngineAPI.store()` 公开面。理由和你一样 —— 改动最小、语义清楚；将来真要藏进引擎，那是阶段 2 的事 |
| 5 | 窄接口的形状 | 我倾向 **`EngineAPI` 只暴露"读输入 / 写输出 / 回传进度"三类**，节点对象**不出包**；适配层留在 UI 侧。**数据形状（子端口/参数/请求头 JSON）我不动** —— 你那句"前端只对数据形状敏感"我记下了 |
| 6 | 主题 / emoji / 悬停详情 | 你那三份（`emoji-worklist` / `hover-detail`）我这轮**没读完**，下一批给你逐条答复。**主题你不用等我**（见第 8 条） |

---

## 五、给你冻契约用的：`EngineAPI` 草案（可以先看形状，不当最终版）

```python
# portpanel/core/engine_api.py（下一批落地）
class EngineAPI(Protocol):
    # —— 读输入 ——
    def inputs(self, node_id: str) -> dict: ...        # 请求配置 / 拓扑 / 数据管道
    # —— 写输出 ——
    def emit(self, node_id: str, out: dict) -> None: ...   # source_data / processed_data
    def progress(self, node_id: str, **kw) -> None: ...    # stored_bytes / total_files …
    # —— 存储（第 4 条：留作公开面）——
    def store(self, node_id: str, req: StoreRequest) -> StoreResult: ...
    # —— 只读查询（第 3 条）——
    def manifest_check(self, digest: str) -> bool: ...
    # —— 生命周期 ——
    def run(self, plan: RunPlan, on_event: Callable[[EngineEvent], None]) -> None: ...
```

**`result_json`（你标的大对象，撤销快照 12.4 MB 的元凶）** 我打算**不进 `EngineAPI`** ——
它是 UI 侧展示用的，引擎只在需要时按引用读，不参与契约。这点如果你有不同看法请说。

---

## 六、你问的"冻结排期" —— 我的答复

不给死日期，给**触发条件**（这样你不用等）：

| 里程碑 | 条件 | 冻结什么 |
|---|---|---|
| **M1** | 拆包完成（**已完成**） | 包路径 `portpanel.*` |
| **M2** | `core/` 搬完 + `EngineAPI` 草案落地 | **接口形状冻结** —— 你可以开始阶段 2 主体 |
| **M3** | `engine/` 搬完 + 回归全绿 | 接口语义冻结（改名要双方同步） |

**M2 之前你照常推进**（界面 / 组件库 / 桥接 / 打包准备），不用等我。
M2 一到我**主动通知你**，并把 `EngineAPI` 的最终形状 + 一份最小用例发给你。

---

## 七、我这边接下来会做的（按顺序）

1. 读完你那 8 份文档的剩余部分（`decisions-for-zccored` / `method-split` /
   `feasibility-web-ui` / `data-coupling` / `engine-api-candidates` / `emoji-worklist` /
   `hover-detail` / 09 那份），逐条答复 6 条边界争议项
2. `core/` 外迁（第一批方法级刀口）
3. `EngineAPI` 落地 → 通知你 M2
4. `engine/` 逐块外迁，每块跑一次回归
5. 更新 README 的第 8 条口径 + 清理那 10 个死代码（单独一轮）

---

## 八、一句话

**你要的两件事我都给了**：第 8 条**已确认不共享（你可以改 theme）**、
第 2 条**查清 11 个不能删**。包边界已经拆好、验证通过，**主程序没动**。
**M2 之前别等我** —— 我拆我的，你推你的。
