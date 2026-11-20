# 搜狗输入法 10.32.21 修改说明：删除键上滑清空 / 下滑恢复 / 左滑清空（r32）

## 成品

- 文件：搜狗输入法_10.32.21_上滑清空下滑恢复左滑清空_r32.apk（同目录另保留 fix32_final.apk）
- 包名 / 版本：com.sohu.inputmethod.sogou.xiaomi / 10.32.21.202608051602.r32（versionCode 20161）
- 大小：131,002,652 字节（原包 91,118,396）
- 签名：v1+v2+v3，自签证书 CN=Test
  - 证书 SHA-256：547443d6cdd6391d74ded1f2c7f25dca448eb242c092e6c5a3c0548bc7bfeaf8
  - 证书 SHA-1：e7f4c0e759d4276a3d71d428c2825d749754638b
- APK SHA-256：AE2A5F46555A0F0BD5F0E35EC03EAC3FB2D62305716DB221EC0D0822BA060E19

> 自签名与原版/其他修改版签名不同。MI9 真机原包是 /system/app 系统应用，覆盖安装后还要用 root 执行 cmd package install-existing --user 0 com.sohu.inputmethod.sogou.xiaomi 恢复 user 0，再 enable/set 输入法。

## 功能

参考微信输入法、豆包输入法的删除键手势，本版实现：

| 手势 | 行为 |
|---|---|
| 删除键上滑 | 一次性清空输入框里的全部文本 |
| 删除键下滑 | 撤回最近一次清空，把清空前文本原样写回；清空后即使用户又输入了新内容，下滑仍恢复旧文本 |
| 删除键单击 | 删除 1 个字符（本补丁自实现，绕过原生长按连删接管） |
| 删除键左滑 | 一次性清空输入框里的全部文本（参考豆包） |

手势判定：

- 只在 ACTION_DOWN 的落点命中删除键（key action -0x5）时才接管；其他键上/下/左滑不受影响。
- 上滑/下滑/左滑由触摸位移判定触发（上/下滑约 50dp，左滑约 100px 级别阈值）。
- 从 ACTION_DOWN 起吞掉删除键事件，阻止搜狗原生长按连删与后续 UP 删除干扰恢复结果；单击由补丁自行执行单删。

## 实现方式

改动集中在 com.sohu.inputmethod.input.KeyboardTouchEventHandler（原 classes3.dex 的 method_ids 已达 65535 上限，整类移到 classes5.dex），并新增 com.sohu.inputmethod.input.SgTask。

1. h1(MotionEvent) 入口最前面插入 k2(MotionEvent) 手势状态机：
   - ACTION_DOWN 命中删除键后，先 k3s() 保存完整文本作为撤回点，设置 k2Armed=1/k2Restored=0/k2Consumed=0，并返回 true 消费 DOWN。
   - ACTION_MOVE：上滑超过阈值调用 k3() 清空；下滑超过阈值调用 k3b1() 同步恢复；左滑超过阈值调用 k8() 清空。所有已处理手势返回 true，避免原生继续处理。
   - 未跨阈值的 MOVE 也返回 true 保持删除键接管；ACTION_UP 只在未消费时调用 k9() 实现单击删 1 字符；已消费的手势直接吞掉。
2. 清空实现 k3()：MainImeServiceDel.getInstance().getCurrentInputConnection() → finishComposingText() → getTextBeforeCursor/getTextAfterCursor 求长度 → deleteSurroundingText(lenBefore, lenAfter)；取不到文本时回退为连发 KEYCODE_DEL。
3. 撤回实现 k3s() + k3b1() + k3c()：清空前保存光标前后完整文本和时间戳；下滑时 k3b1() 校验 5 分钟窗口后调用 k3c(saved) 写回；k3c 用 beginBatchEdit/finishComposingText/commitText/endBatchEdit 提交。
4. 左滑清空 k8()：直接复用已验证的 k3()，避免单独实现时 null/text 处理不一致。
5. r29–r32 关键修复：
   - r29：删除键 DOWN 即消费，根治搜狗原生长按连删把恢复文本清空的问题；单击改由 k9() 自实现。
   - r30：修正 UP 分支 consumed 判断，已消费手势不再误调 k9 多删 1 位。
   - r31：左滑清空复用 k3()，解决日志触发但文本未清的问题。
   - r32：加入内容日志确认保存/提交值；连续 5 轮下滑恢复验证稳定。

## 已在 MI9 真机实测通过

环境：小米 9 cepheus / Android 11 / MIUI，root；adb -s DEVICE_IP_REDACTED:5555；搜狗目标包 com.sohu.inputmethod.sogou.xiaomi（9 键布局），短信 com.android.mms 作载体；输入基线用讯飞输入法真实键入 digits 后切回搜狗。

- 上滑清空：基线 222222222000 → 上滑 → 输入框变占位“短信” ✅
- 下滑恢复：上滑清空后下滑 → 恢复 222222222000 ✅
- 单击删除：基线 000000 → 单击删除键 → 00000 ✅
- 左滑清空：基线 2222220000 → 左滑 → 输入框变占位“短信” ✅
- r32 连续 5 轮“输入 222222222000 → 上滑清空 → 下滑恢复”全部完整恢复，5/5 ✅
- 手势期间 IME 进程稳定，无 VerifyError/NoSuchFieldError/崩溃 ✅

## 其他说明

- 只改功能，不删原包组件；广告/权限等保持原样（这是功能修改版，不是精简版）。
- 包体变大是 apktool 重建后 zip 存储方式变化（原包存在共享/重叠数据段，重建后条目独立存储），dex/so/resources 本身变化很小。
- 调试日志 TAG 为 SG_PATCH，保留用于复测；logcat 可过滤 adb logcat -s SG_PATCH。

