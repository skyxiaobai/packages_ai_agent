# Auto Research — AI Agent 自主优化

你是一个嵌入式 AI Agent 的自主研究员。你的任务是对 ai_agent 项目进行自主实验优化。

## 环境

- 项目：Vela/NuttX 上的 AI Agent（C 语言）
- 编译：`cmake --build /home/justinz/vela/vela_update/build_qemu -j$(nproc)`
- 模拟器：QEMU ARM virt（端口 28789 转发）
- 测试：`curl http://localhost:28789/a2a/health` + `curl -X POST http://localhost:28789/a2a/invoke -d '{"message":"hello"}'`
- 指标：编译成功（0 error）、功能测试通过、RAM 占用（`struct mallinfo`）、Flash 占用（链接输出 ROM Used）

## 实验循环

每次实验严格按以下步骤执行：

### 1. 记录基线
- 编译当前代码，记录 ROM/RAM 占用
- 如果模拟器可用，运行功能测试记录结果

### 2. 提出假设
- 用一句话描述你要尝试什么
- 说明预期效果（减少 RAM？提高响应速度？简化代码？）

### 3. 实施改动
- 只改一个变量（单变量原则）
- 改动尽量小（<50 行）
- 不要同时改多个模块

### 4. 验证
- 编译：`cmake --build build_qemu -j$(nproc) 2>&1 | tail -5`
- 检查 ROM/RAM 变化
- 检查编译 warning 数量（不能增加）
- 如果是代码优化类改动，用 `cppcheck` 或 `gcc -fsyntax-only` 验证
- 如果编译失败，立即回滚，记录失败原因

### 5. 判定
- **keep**：指标改善或持平，代码更简洁，warning 不增加
- **rollback**：指标退化、编译失败、或引入新 warning
- **inconclusive**：变化在噪声范围内（<1%），记录但不保留

### 6. 记录
每次实验记录到 `logs/autoresearch.jsonl`，格式：
```json
{
  "id": 1,
  "type": "ram|rom|code-quality|speed|security",
  "hypothesis": "...",
  "files_changed": ["..."],
  "lines_added": 0,
  "lines_removed": 0,
  "result": "keep|rollback|inconclusive",
  "rom_before": 0,
  "rom_after": 0,
  "ram_before": 0,
  "ram_after": 0,
  "warnings_before": 0,
  "warnings_after": 0,
  "notes": "..."
}
```

## 优化方向（按优先级）

### P0: 减少 RAM 占用
- 静态 buffer 改为按需 malloc
- 减少全局变量
- 字符串常量去重
- 栈 buffer 大小合理化

### P1: 减少 Flash/ROM 占用
- 删除死代码
- 合并重复逻辑（如 URL 解析已完成）
- 条件编译未使用的模块

### P2: 代码质量优化
- 消除重复代码（DRY）：相似的 HTTP 请求构建、JSON 解析、错误处理
- 简化过度抽象：如果一个函数只被调用一次且很短，考虑内联
- 统一错误处理模式：goto cleanup vs 多 return vs early return
- 删除未使用的函数/变量/include
- 减少嵌套层级（提前 return 替代深层 if-else）
- 常量提取：magic number 改为 #define
- 函数拆分：超过 100 行的函数考虑拆分

### P3: 提高响应速度
- 减少 LLM API 调用的 context 大小
- 优化 JSON 构建/解析
- TLS 连接池复用率
- 减少不必要的 malloc/free 次数

### P4: 安全加固
- 检查所有 malloc/strdup 返回值
- 检查 snprintf 截断
- 检查 strncpy 后是否补 '\0'
- 锁的对称性（lock/unlock 配对）
- 指针释放后置 NULL

## 约束

- **不要改 agent_compat.h 和 agent_config.h 的公共接口**
- **不要改 message_bus 的消息格式**
- **不要删除任何 Kconfig 选项**
- **每次实验只改一个方向**
- **3 次失败后停下来，重新审视假设**

## 可修改的文件

优先级从高到低：
1. `src/core/` — agent_loop, context_builder, memory_store
2. `src/llm/` — llm_proxy, llm_parse, llm_router
3. `src/infra/` — vela_tls, http_proxy, config_store
4. `src/channels/` — nsh_commands, ws_server
5. `src/tools/` — tool_registry, 各 tool 实现

## 不可修改的文件

- `prepare.py` 等价物：CMakeLists.txt, Kconfig, Makefile（构建系统）
- 第三方代码：cJSON, mbedTLS, MQTT-C
- NuttX 内核代码
