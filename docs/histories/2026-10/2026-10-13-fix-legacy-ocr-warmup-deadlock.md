## 2026-10-13 | 任务：修复 legacy Vision OCR 预热死锁

**Links:** None

### 执行上下文

- **Agent Name:** `Claude Code`
- **Model:** `claude-opus-5-5`
- **Environment:** `macOS 15.8.1 / Xcode 26.3 (17C529)`

### 用户请求

Debug 构建中截图 OCR 一直显示加载动画、没有结果；排查原因并将 legacy 预热请求移到 dispatch
队列执行。

### 变更

- `performLegacyWarmUpOCR(on:)` 改为通过 `withCheckedContinuation` 在
  `DispatchQueue.global(qos: .utility)` 上同步执行 `VNImageRequestHandler.perform`，与真实 legacy
  OCR 路径一致，不再阻塞 Swift Concurrency 线程。
- 未新增或修改测试。

### 设计意图

`VaporServer` 将 SwiftNIO 安装为 Swift Concurrency 全局 executor，预热的 `Task.detached` 因此运行在
NIO event loop 线程上。macOS 15 走 legacy 路径，同步 `perform` 阻塞该线程后 `TextRecognition`
在内部信号量上永久等待（`sample` 显示预热卡在 `NIO-SGLTN-1-#4`），之后的真实 OCR 请求排在卡住的
预热之后，加载动画不再结束。日志中 debug 构建的预热从未输出 `warm-up finished`；同机独立运行
相同 Vision 请求 0.34 秒完成。macOS 26+ 的 modern 路径使用 async `perform`，不受影响，保持不变。

### 验证

- `BuildTools/.build/release/swiftformat --lint Easydict/Swift/Service/Apple/AppleOCREngine/AppleOCREngine+WarmUp.swift --config .swiftformat`：通过，0/1 文件需要格式化。
- `xcodebuild build -workspace Easydict.xcworkspace -scheme Easydict -derivedDataPath <checkout-derived Easydict-Agent>`：`BUILD SUCCEEDED`。
- `xcodebuild test ... -only-testing:EasydictTests/OCRQRCodeTests`：4/4 通过。
- 手动检查：启动修复后的 `Easydict-debug.app`（日志确认 SwiftNIO executor），预热输出
  `Vision OCR warm-up finished, cost time: 0.325 seconds`；修复前同一构建预热持续数分钟未完成。
- 未手动执行截图 OCR 端到端流程。
- `git diff --check`：通过。

### 受影响文件

- `Easydict/Swift/Service/Apple/AppleOCREngine/AppleOCREngine+WarmUp.swift`
- `docs/histories/2026-10/2026-10-13-fix-legacy-ocr-warmup-deadlock.md`

### 后续事项

- None
