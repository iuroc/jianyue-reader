# 本地书刊库构建

## 流程

- 18 个分类分片（图书 11 + 期刊 7），4 线程并发抓取。
- 每片先写 `tmp/`；18 片齐全后按顺序提交：`lib_*.json` → `index.json` → `ver.txt`。
- `ix_ready()` = `index.json` 存在且非空 + `ver.txt` 内容匹配当前版本（`"3"`）。
- 中途失败时旧库保持可用，不会出现半新半旧的库。

## 收尾去重与重入

- `finishOnce.lock`：一轮构建只允许一次收尾；重复调用记 `IX finish dup ignored`。
- `finishing.lock`：收尾进行中；超过 120 秒视为进程被杀留下的陈旧锁，自动清理。
- `ix_start` 开始新一轮时清掉两个锁；`running=1` 或 `finishing=1` 时拒绝再起。
- `boot_start()`：`manual = ix_flag("boot.rebuild")`；**非手动重建且 `ix_ready()` 为真时直接关页回首页**，
  避免上一轮已经建好后又被送进 boot 页、再起一轮。
- 只有「我的 → 重建索引」才 force 重建。
- `boot_tick()` 在 `ix_finishing()==1` 时继续轮询，不提前关页。

## 诊断

- 打包态 `i.syso` 不输出；关键路径用 `android.util.Log.e("JY", ...)`。
- 取日志：`adb logcat -d -v time -s JY:V`。
- 失败页 `ix_errText()` 会显示最近一次失败原因。
- 关键日志：`BOOT start manual=... ready=...`、`IX start force=... running=... finishing=...`、
  `IX shard <spec>`、`IX finish n=...`、`IX finish dup ignored`、`abort missing/move fail`。

## 2026-11-20 事故记录（打包版首次建库）

| 现象 | 根因 | 修复 |
|---|---|---|
| 建库成功后页面显示失败，`abort missing b:124 lib=-1` | 收尾被触发两次：第一次已搬走 tmp，第二次判缺失 | `finishOnce.lock` + `finishing.lock` 文件锁去重 |
| 进度停在 0/18，无 `IX start` 日志 | 新加的 `ix_flag(String)` 与原有 `ix_flag(String)` 同名同参数，indexer 模块不可用 | 改名 `ix_lockFile/Has/Make/Clear`；`check_mjava.py` 增加同名同 arity 检查 |
| 第一次建库成功后又被送入 boot，再起一轮后误报 | `boot_start` 无条件 force；成功后首页重入 boot | 非手动且 `ix_ready()` 直接回首页 |
| 工作线程设置的 `i.sss` 标志 UI 线程读不到 | 工作线程解释器的 `i.sss` 表不共享 | 改用文件锁（`createNewFile` 原子） |

✅ 2026-11-20：红米 Note 8 打包版建库成功并进入首页，用户确认。
