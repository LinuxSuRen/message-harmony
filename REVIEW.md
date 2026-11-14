## Review 结论：Request Changes（1 个 🔴 需修复后合并）

评审方式：独立差分评审 + 维护者复核（🔴 逻辑链已亲自核实）。整体质量不错：SSE 字节级分帧跨 chunk 处理正确、资源释放路径完整、JSON 全守卫不崩溃、权限最小化（仅 INTERNET）、密码不落盘。以下按 ai-code-review-gate 三级分类。

**构建证据**：fork 内 CI（Build HAP debug）在本 PR head 提交 `60f3094` 上全部 success；Release v0.0.1 HAP 产物存在（95,651 字节）。

### 🔴 CRITICAL（阻止合并）

**C1. SSE 看门狗饿死重连定时器 → 断网恢复后实时流永久失效**
- 位置：`entry/src/main/ets/Net.ets` — `scheduleReconnect`（会 `clearTimeout` 未到期的重连定时器再重设全量退避时长）+ `startWatchdog`（每 15s 检查一次 75s 无数据即触发 `scheduleReconnect`）+ `lastDataAt` 仅在 `dataReceive` 回调刷新。
- 推演：网络分区 ≥75s（无 FIN，`dataEnd` 不触发）→ 看门狗每 15s 触发一次 `scheduleReconnect`；当退避 ≥16s（attempts ≥4）时，每次触发都在定时器到期前将其取消并重设 60s——`connect()` 永远不会被调用；断连状态下 `lastDataAt` 永不刷新，死循环成立。即使网络恢复，SSE 实时订阅也永久失效，只能重启应用。这正是自动重连机制要覆盖的核心场景。
- **修复（一行，二选一）**：① `scheduleReconnect` 开头加 `this.lastDataAt = Date.now();`（让看门狗在退避等待期静默）；② 看门狗回调改为 `if (this.reconnectTimer >= 0) return;`。建议 ①，顺带消除退避等待期重复的掉线状态通知。

### 🟡 IMPORTANT（修完或书面登记理由后可过）

1. **`normalizeBaseUrl` 缺省协议默认 `http://` → 会话 Cookie 明文传输** — `Net.ets:75-77`。裸域名输入被补成 http。服务端主打内置 HTTPS/mTLS，客户端默认值与之相悖。建议缺省补 `https://`（显式输入 `http://` 仍放行，便于内网）。
2. **`loadMore` 与 scope 切换竞态** — `Messages.ets:316-322` 只守卫 `loading` 未守卫 `loadingMore`。在途加载期间切 scope，旧响应会混入新列表且覆盖新 scope 游标。建议请求代数（generation counter）守卫。

### 🟢 SUGGESTION（不阻塞，记录备查）

mh_session 明文存 Preferences（沙箱内可接受，如需加固迁 @ohos.security.asset）；`appendChunk` 全量拷贝 O(n²) 且无上限（当前契约下风险低）；reload 整页替换会短暂覆盖 SSE 实时消息；RFC3339Nano 个别运行时解析失败的显示回退；会话静默校验建议改用 `/api/v1/auth/me`；`flush()` 未 await；release 混淆可开启；SSE 裸 `\r` 行终止未处理（契约内无影响）。

### 已检查无问题的类别

SSE 分帧与多行 data 合并（字节缓冲、半行/半帧、混合换行）✅；流资源释放（closeClient off×3 + destroy，页面销毁 destroy）✅；JSON 解析健壮性 ✅；401 状态清理链路 ✅；权限最小化 ✅。

---

**结论**：C1 一行修复后即可合并，建议顺带 I1（同为一行）。I2 可登记跟进。
