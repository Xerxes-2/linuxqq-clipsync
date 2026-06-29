# 重构 TODO

目标：消除两条同步循环的重复，去掉 stringly-typed 状态机，收紧锁粒度。**行为不变**。

## 1. 类型化状态机
- [ ] 定义 `enum Direction { X2W, W2X }`（替换 `last_dir: String` 的 `"W2X"/"X2W"`），`last_dir: Option<Direction>`。
- [ ] 定义 `enum ProcessMode { UriList, Text, Raw }`（替换 `process_mode: &str`），`calc_hash` 改成 `match` 枚举。
- [ ] 用 `Option<u128>` 或专门标记替换 `EMPTY_HASH = 0` 哨兵，避免真实内容 hash 出 0 被误判为空。

## 2. 抽公共逻辑（核心）
- [ ] 提取 MIME 探测表：单一函数 `pick_mime(types: &str, dir: Direction) -> Option<(source_mime, sync_mime, ProcessMode)>`，统一两处 if/else 阶梯（注意对齐 X2W 独有的 `gnome-copied-files` / `x-qt-image` / `UTF8_STRING` 分支，确认是有意差异还是漏写）。
- [ ] 提取 uri-list 重写：`fn rewrite_uri_list(data: &[u8]) -> Vec<u8>`（替换 L271-289 与 L361-379 两段逐字重复）。
- [ ] 合并两条循环为一个泛型 `sync_loop`，由参数区分方向：触发源（clipnotify / wl-paste --watch）、reader/writer 命令、MIME 表、target 转换（W2X 的 `text/plain → UTF8_STRING`）。

## 3. 收紧锁粒度
- [ ] 临界区只保护 `last_dir` / `last_time` / `last_sync_hash` 的读写，**不要**把 `read_clipboard`（spawn + wait 阻塞）圈进 `MutexGuard`。
- [ ] 回音抑制（时间窗 + hash）逻辑提成小函数，两方向共用。

## 4. 错误处理 / 可观测性
- [ ] `read_clipboard` / `write_clipboard` 的静默失败至少 `log("WARN", ...)`，便于排查。
- [ ] 评估 `let _ = child.wait()` 等是否需要保留返回值。

## 5. 收尾
- [ ] `cargo clippy --all-targets -- -D warnings` 通过。
- [ ] 手动验证四类内容仍能双向同步：纯文本、图片(png/jpeg)、文件(uri-list)、html。
- [ ] 重构前后 diff review，确认无行为回归。
