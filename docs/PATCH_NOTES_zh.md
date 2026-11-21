# 搜狗输入法 10.32.21 修改说明：删除键上滑清空 / 下滑恢复 / 左滑清空 / 单删连删（r37）

## 成品

- 文件：搜狗输入法_10.32.21_上滑清空下滑恢复左滑清空_r37.apk（同目录另保留 fix37_final.apk）
- 包名 / 版本：com.sohu.inputmethod.sogou.xiaomi / 10.32.21.202608051602.r37（versionCode 20166）
- 大小：131,002,652 字节（原包 91,118,396）
- 签名：v1+v2+v3，自签证书 CN=Test
  - 证书 SHA-256：547443d6cdd6391d74ded1f2c7f25dca448eb242c092e6c5a3c0548bc7bfeaf8
  - 证书 SHA-1：e7f4c0e759d4276a3d71d428c2825d749754638b
- APK SHA-256：6B68955BE50D5B2E9CB2FE13E2E8293FB85C4FD207C3FE20AEDCC6C94A25DDC7

> 自签名与原版/其他修改版签名不同。MI9 真机原包是 /system/app 系统应用，覆盖安装后还要用 root 执行 cmd package install-existing --user 0 com.sohu.inputmethod.sogou.xiaomi 恢复 user 0，再 enable/set 输入法。

## 功能

参考微信输入法、豆包输入法的删除键手势，本版实现：

| 手势 | 行为 |
|---|---|
| 删除键上滑 | 一次性清空输入框里的全部文本 |
| 删除键下滑 | 恢复最近一次上滑清空前的完整文本；即使清空后又输入了新内容，下滑仍恢复旧文本 |
| 删除键单击 | 删除 1 个字符（交给搜狗原生处理，保持原生行为） |
| 删除键长按 | 逐字连续删除（交给搜狗原生处理，保持原生行为） |
| 删除键左滑 | 一次性清空输入框里的全部文本（参考豆包） |

手势判定：

- 只在 ACTION_DOWN 的落点命中删除键（key action -0x5）时才记录状态；普通单击、长按的 DOWN/MOVE/UP 全部放行给搜狗原生，保证单击单删和长按连删不受影响。
- 只有上滑/下滑/左滑超过阈值时才由补丁接管：上/下滑约 50dp，左滑约 100px 级别阈值。
- 下滑恢复触发时会把 UP 放行给原生，让搜狗取消原生长按删除定时器，避免恢复的文本被定时器二次清空。

## 实现方式

改动集中在 com.sohu.inputmethod.input.KeyboardTouchEventHandler（原 classes3.dex 的 method_ids 已达 65535 上限，整类移到 classes5.dex），并新增 com.sohu.inputmethod.input.SgTask。

1. h1(MotionEvent) 入口最前面插入 k2(MotionEvent) 手势状态机：
   - ACTION_DOWN 命中删除键后只记录 k2Armed=1 / k2Restored=0 / k2Consumed=0 并返回 0（放行原生），不吞事件、不预存文本。
   - ACTION_MOVE：上滑超阈值调用 k3() 清空并写 k3Backup；下滑超阈值调用 k3b1() 从 k3Backup 同步恢复；左滑超阈值调用 k8() 清空。接管后返回 true。
   - 未跨阈值的 MOVE 返回 0 放行原生；ACTION_UP 只清理状态标志，不自行删字（单删/连删交给原生）。
2. 清空实现 k3()：MainImeServiceDel.getInstance().getCurrentInputConnection() → finishComposingText() → getTextBeforeCursor/getTextAfterCursor 求长度 → deleteSurroundingText(lenBefore, lenAfter)；取不到文本时回退为连发 KEYCODE_DEL。清空前无条件把当前完整文本写入 k3Saved 与 k3Backup 并更新时间戳。
3. 撤回实现 k3b1() + k3c()：下滑时 k3b1() 校验 5 分钟窗口后从 k3Backup 读取清空前文本，调 k3c(saved) 写回；k3c 用 finishComposingText/deleteSurroundingText/commitText 提交。k9r() 通过 SgTask postDelayed 100ms 做一次幂等兜底恢复。
4. 左滑清空 k8()：直接复用已验证的 k3()。
5. r33–r37 关键修复（对应用户反馈的三个问题）：
   - r33：删除键 DOWN/MOVE/UP 不再从 DOWN 起吞掉，改回放行原生，恢复原生长按逐字删除与单击单删；清空时无条件用本次清空前完整文本覆盖 k3Backup/k3Saved，下滑只读 k3Backup，修复“恢复的文本不是上滑清空的文本”。
   - r34–r36：定位下滑恢复后 UI 仍为空的根因——放行 DOWN 后原生长按定时器在手势期间持续删除，commitText 成功但随即被删空；期间修正了 KeyEvent 包名、postDelayed long 参数寄存器数等构建问题。
   - r37：下滑恢复触发时把 ACTION_UP 放行给原生（k2Restored 为真时令 consumed=0 返回 0），让原生 onDeleteKeyUp 取消长按定时器；k9r 延迟兜底二次写回。MI9 实测四项手势全部通过。

## 已在 MI9 真机实测通过

环境：小米 9 cepheus / Android 11 / MIUI，root；adb -s DEVICE_IP_REDACTED:5555；搜狗目标包 com.sohu.inputmethod.sogou.xiaomi（26 键布局），短信 com.android.mms 作载体；输入基线用讯飞输入法真实键入 digits 后切回搜狗（搜狗 input text 会被候选区吞掉）。

r37 实测（版本号 versionName r37 / versionCode 20166 已确认装机）：

- 单击删除：基线 222222000000 → 单击 → 22222200000（删 1 位）✅
- 长按连删：基线 22222200000 → 长按 2s → 000（连续删除）✅
- 上滑清空：基线 222222000000 → 上滑 → 输入框变占位“短信” ✅
- 下滑恢复：清空后下滑 → 完整恢复 222222000000 ✅
- 边界场景：基线 222222000 → 上滑清空 → 输入新内容 000000 → 下滑 → 恢复的是上次清空的内容 222222000（而非新内容 000000）✅
- 手势期间 IME 进程稳定，无 VerifyError/NoSuchFieldError/崩溃 ✅

## 其他说明

- 只改功能，不删原包组件；广告/权限等保持原样（这是功能修改版，不是精简版）。
- 包体变大是 apktool 重建后 zip 存储方式变化（原包存在共享/重叠数据段，重建后条目独立存储），dex/so/resources 本身变化很小。
- 调试日志 TAG 为 SG_PATCH，保留用于复测；logcat 可过滤 adb logcat -s SG_PATCH。