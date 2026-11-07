# Double-Press to Cancel Recording - Implementation Status

## 概述

本功能允许用户在录音开始后 350ms 内快速按两次录音热键来取消录音，提供更快捷的"说错了想重来"操作。

## ✅ 已完成的部分

### 1. 核心状态机逻辑 (src-tauri/src/coordinator_state.rs)

**已实现**:
- ✅ 添加 `last_press_at: Option<Instant>` 字段到 `SessionState`
- ✅ 在会话开始时记录首次按下时间戳
- ✅ 添加 `PendingCancel` 变体到 `BeginOutcome` 枚举
- ✅ 实现 `should_cancel_on_press(state, at)` 函数：
  - 350ms 时间窗口
  - 与 `AUTO_HOLD_THRESHOLD` 对齐
  - 大于 `HOTKEY_DEBOUNCE` (250ms)，确保能通过防抖
- ✅ 修改 `request_stop_during_starting_state()` 接受 `at: Instant` 参数
- ✅ 修改 `finish_starting_session_state()` 区分快速取消和慢速停止

**单元测试**:
- ✅ `test_fast_double_press_should_cancel` - 300ms 应该取消
- ✅ `test_slow_double_press_should_not_cancel` - 400ms 不应该取消
- ✅ `test_boundary_350ms_should_not_cancel` - 边界测试
- ✅ `test_pending_cancel_outcome_on_fast_double_press` - 验证 PendingCancel 结果
- ✅ `test_pending_stop_outcome_on_slow_double_press` - 验证 PendingStop 结果
- ✅ 修复现有测试中 `request_stop_during_starting_state` 的调用

### 2. 音频反馈 (src/lib/audioCue.ts)

**已实现**:
- ✅ `recordCancelCueTones()` - 下降小三度（880Hz → 659.25Hz）
  - 与启动音效（上升小三度）相反，清晰区分
  - 略短且轻柔（100ms + 140ms，gain 0.14/0.16）
- ✅ `playRecordCancelCue()` - 播放取消音效的公共接口
- ✅ `playRecordCancelCueOnce()` - 处理 AudioContext resume 逻辑
- ✅ `scheduleCancelCueVoices()` - 实际合成音频节点

**特性**:
- 复用现有的 `playSeq`/`stopSeq` 机制防止音效重叠
- 与 `playRecordStartCue()` 架构完全一致
- 支持 AudioContext suspended/closed 状态的优雅降级

## 🚧 待完成的集成工作

### 3. Coordinator 集成

**需要做的**:

#### 3.1 在 openless-core 或 Tauri 层处理 PendingCancel

当前 `BeginOutcome::PendingCancel` 已定义但未被处理。需要找到：

```rust
// 可能在 crates/openless-core/src/api.rs 或类似位置
match outcome {
    BeginOutcome::Started => { /* 现有逻辑 */ }
    BeginOutcome::PendingStop => { /* 现有逻辑 - 正常停止并处理 */ }
    BeginOutcome::PendingCancel => {  // 新增
        // 1. 取消会话（不处理录音）
        // 2. 发出 record:cancel-cue 事件
        // 3. 可选：显示 Toast
        log::info!("[coord] fast double-press during Starting, canceling");
        // self.cancel_active_voice_session(None).await?;
        // emit_cancel_feedback(...).await;
    }
    BeginOutcome::CancelRaced => { /* 现有逻辑 */ }
    BeginOutcome::StaleContinuation => { /* 现有逻辑 */ }
}
```

**参考位置**:
- `crates/openless-core/src/api.rs` 中的 `dispatch_dictation_hotkey_edge_with_session_options`
- 或 `src-tauri/src/coordinator/dictation_core.rs` 如果状态机在 Tauri 层

#### 3.2 Listening 阶段双击取消检测

需要在 Listening 阶段检测双击：

```rust
// 在 handle_pressed_edge 或类似函数中
pub(super) async fn handle_pressed_edge(
    inner: &Arc<Inner>,
    pressed_at: std::time::Instant,
    press_id: u64,
) {
    // NEW: Check for fast double-press during Listening
    {
        let state = /* 获取当前会话状态 */;
        if matches!(state.phase, SessionPhase::Listening) {
            if let Some(last_press) = state.last_press_at {
                let elapsed = pressed_at.saturating_duration_since(last_press);
                if elapsed.as_millis() < 350 {
                    log::info!("[coord] fast double-press during Listening, canceling");
                    cancel_active_session(inner).await;
                    emit_cancel_feedback(inner).await;
                    return;
                }
            }
        }
    }
    
    // 现有逻辑继续...
}
```

**挑战**:
- 需要找到正确的方式访问 `SessionState` 并更新 `last_press_at`
- 可能需要在 `openless-core` 的状态中也添加 `last_press_at` 字段

#### 3.3 发出取消反馈

添加辅助函数发出音频和 Toast：

```rust
async fn emit_cancel_feedback(inner: &Arc<Inner>) {
    // Play cancel audio cue
    let _ = inner.host.emit("record:cancel-cue", ()).await;
    
    // Show toast notification (可选)
    let _ = inner.host.emit("toast:show", serde_json::json!({
        "message": "Recording canceled",
        "type": "info"
    })).await;
}
```

### 4. 前端事件监听

在 `src/App.tsx` 或胶囊窗口组件中：

```typescript
import { listen } from '@tauri-apps/api/event';
import { playRecordCancelCue } from './lib/audioCue';

useEffect(() => {
  const unlisten = listen('record:cancel-cue', () => {
    playRecordCancelCue();
  });
  return () => { unlisten.then(fn => fn()); };
}, []);
```

## 🧪 测试计划

### 单元测试
- ✅ 状态机逻辑已有完整测试
- ⏳ 需要为新的 coordinator 集成添加测试

### 集成测试

完成后需要测试：

1. **Starting 阶段快速双击**
   - 按下热键 → 等待 200ms → 再按下
   - 预期：立即取消，播放下降音效，不插入文本

2. **Listening 阶段快速双击**
   - 按下热键 → 开始说话 → 快速再按一次
   - 预期：立即取消，播放下降音效

3. **慢速双击不取消**
   - 按下热键 → 等待 500ms → 再按下
   - 预期：正常停止并处理录音（不是取消）

4. **跨热键模式测试**
   - Hold 模式：快速按两次
   - Toggle 模式：快速点两次
   - Auto 模式：< 350ms 时快速双击
   - 所有模式都应该正确取消

5. **音频反馈**
   - 验证播放下降小三度音（880Hz → 659Hz）
   - 与启动音效明显区分

6. **边界情况**
   - 340ms：应该取消
   - 360ms：不应该取消
   - Processing/Inserting 阶段：双击无效（保持现有 Esc 逻辑）

## 📁 文件清单

已修改的文件：
- ✅ `openless-all/app/src-tauri/src/coordinator_state.rs` (+133 行，包括测试)
- ✅ `openless-all/app/src/lib/audioCue.ts` (+120 行)

待修改的文件：
- ⏳ `crates/openless-core/src/api.rs` 或相应的 coordinator 文件
- ⏳ `src-tauri/src/coordinator/dictation_core.rs` (可能)
- ⏳ `src/App.tsx` 或胶囊窗口组件 (~10 行)

## 🎯 完成百分比

- 状态机核心：**100%** ✅
- 音频反馈：**100%** ✅
- Coordinator 集成：**0%** ⏳
- 前端连接：**0%** ⏳
- 测试验证：**30%** (仅单元测试)

**总体进度：约 70%**

## 🔍 架构笔记

OpenLess 的架构是分层的：

```
前端 (React/TypeScript)
  ↓ Tauri IPC
Tauri 层 (src-tauri/)
  ↓ 调用
Core 层 (crates/openless-core/)
```

`SessionState` 和 `BeginOutcome` 在 `src-tauri/src/coordinator_state.rs` 中定义，这似乎是 Tauri 特定的状态管理。`openless-core` 可能有自己的状态表示。

需要理解：
1. `finish_starting_session_state` 在哪里被调用
2. `openless-core` 的 `voice_session.rs` 如何与 Tauri 层交互
3. 事件如何从 Rust 传递到前端

## 💡 建议

对于上游维护者或继续开发者：

1. **最小化改动**：考虑仅在 Tauri 层实现双击检测，避免修改 `openless-core`
2. **利用现有机制**：`cancel_active_session` 已经存在，只需在正确的位置调用它
3. **参考现有取消**：查看 Esc 键取消和 Combined (组合键) 取消的实现
4. **简化集成**：可以先在 `handle_pressed_edge` 中直接添加检测，而不修改状态机

## 📝 Commit 历史

- `8e9dd093` - feat: add state tracking and audio cue for double-press cancel

## 🚀 下一步

1. 找到正确的位置处理 `BeginOutcome::PendingCancel`
2. 在 Listening 阶段添加双击检测
3. 连接音频反馈事件
4. 本地测试完整流程
5. 更新文档并提交 PR

---

*最后更新: 2026-11-07*
*分支: feature/double-press-cancel*
