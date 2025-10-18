# 代理实现逻辑概述

本项目提供本地 AI 代理（Reverse Proxy）与可视化 UI，统一转发并管理 Claude 与 Codex 的 API 调用。核心由一个可复用的基础代理类驱动，按服务定制具体路由与健康检查，并通过缓存式配置、请求过滤、模型路由与负载均衡实现稳定可控的转发闭环。

- 服务端口：Claude 3210、Codex 3211、UI 3300
- 核心类：src/core/base_proxy.py（基础代理与控制器）
- 服务实现：src/claude/proxy.py、src/codex/proxy.py
- 配置管理：src/config/cached_config_manager.py（~/.clp/{service}.json）
- 请求过滤：src/filter/cached_request_filter.py（~/.clp/filter.json）
- 实时事件：src/core/realtime_hub.py（/ws/realtime）
- UI 展示：src/ui/ui_server.py（日志、统计与配置编辑）

## 1. 架构总览

- 单一基础代理 BaseProxyService 负责通用能力：
  - httpx.AsyncClient 转发与流式响应
  - 统一的请求过滤、日志记录与用量统计
  - 模型路由、负载均衡与实时事件推送
- 各服务代理（ClaudeProxy、CodexProxy）仅覆盖服务差异（如 CORS、健康检查请求体等）。
- BaseServiceController 以独立进程方式启动 uvicorn，写入 PID 与日志，便于 CLI 管理。

关键文件：
- src/core/base_proxy.py: BaseProxyService、BaseServiceController
- src/claude/proxy.py: ClaudeProxy 实现
- src/codex/proxy.py: CodexProxy 实现
- src/main.py: CLI 命令编排 start/stop/status/ui/server

## 2. 请求生命周期（入站 → 出站 → 回传）

以 BaseProxyService.proxy 为核心（src/core/base_proxy.py:689）：

1) 接收请求并记录原始头与体
2) 通过 build_target_param 计算目标：
   - 确保路由配置最新（model_router_config.json）
   - 应用模型路由规则（_apply_model_routing）
   - 选择配置（负载均衡 + active 覆盖）并拼接目标 URL、Headers（src/core/base_proxy.py:605）
3) 应用请求过滤器（CachedRequestFilter.apply_filters），避免事件循环阻塞放入线程池
4) 判断是否为流式（SSE/ndjson），构造 httpx 请求并发送
5) 流式回传：边读上游边写给客户端，同时通过 RealTimeRequestHub 广播 started/progress/completed 事件（src/core/realtime_hub.py）
6) 收集部分响应内容（截断至 1MB）用于日志与用量解析，关闭上游响应
7) 按结果更新负载均衡状态（失败计数与临时剔除），写入 jsonl 日志与用量

流式判定依据 Accept/Content-Type 或 x-stainless-helper-method 含 stream（src/core/base_proxy.py:721-732）。

## 3. 模型路由（model_router_config.json）

文件：~/.clp/data/model_router_config.json，BaseProxyService 启动时加载并热感知变更（签名对比）。支持三种模式（src/core/base_proxy.py:443-471）：

- default：不做改写
- model-mapping（_apply_model_mapping）：
  - 模型→模型：匹配 source 模型名，替换为 target 模型名
  - 配置→模型：若当前生效配置等于 mapping.source（source_type=config），则改写 body 中的 model 为 target
- config-mapping（_apply_config_mapping）：
  - 模型→配置：当 body.model 命中 mapping.model，则强制使用 mapping.config 对应的配置通道（不改写模型名）

## 4. 负载均衡（lb_config.json）

文件：~/.clp/data/lb_config.json，按服务维度维护失败阈值、当前失败计数与临时剔除列表（src/core/base_proxy.py:377-420）。

- active-first（默认）：直接使用当前激活配置（config manager 中标记 active）
- weight-based：按配置 weight 降序遍历，跳过达到失败阈值或被临时剔除的通道，择优选择（src/core/base_proxy.py:536-561）
- 每次请求结束，根据状态码更新 currentFailures 与 excludedConfigs（_record_lb_result，src/core/base_proxy.py:568-604）
- 配置热加载：签名变化时自动重新读取（_ensure_lb_config_current）

## 5. 配置管理（缓存）

类：CachedConfigManager（src/config/cached_config_manager.py）

- 读取 ~/.clp/{service}.json
  - 键：配置名；值：{ base_url, auth_token, 可选 api_key, 可选 weight, active }
- 内建 TTL 与文件 mtime 监控，避免频繁 I/O
- 支持 set_active_config 立即落盘并刷新缓存
- CLI 支持列表/切换（src/main.py + src/{service}/ctl.py）

## 6. 请求过滤

类：CachedRequestFilter（src/filter/cached_request_filter.py）

- 读取 ~/.clp/filter.json，支持数组或单对象规则
- 规则字段：op=replace/remove，source，target；自动为含特殊字符的 source 预编译 regex（字节级处理）
- 通过最小间隔与文件 mtime 控制热加载，避免频繁 stat
- BaseProxyService.apply_request_filter 统一调用（src/core/base_proxy.py:680-687）

## 7. 实时事件与 WebSocket

类：RealTimeRequestHub（src/core/realtime_hub.py）

- WebSocket 路由：/ws/realtime（src/core/base_proxy.py:127-146）
- 事件流：
  - started（含已脱敏请求头、目标 URL）
  - progress（流式阶段、响应增量、耗时）
  - completed/failed（最终状态、HTTP 状态码、耗时）
- 自动截断单请求累计响应（默认 2MB）防止内存膨胀
- 脱敏请求头：authorization/x-api-key/cookie（_sanitize_headers）

## 8. 日志与用量统计

- 请求日志：~/.clp/data/proxy_requests.jsonl（src/core/base_proxy.py:147-215, 285-336）
  - 字段：服务、方法、路径/目标 URL、状态码、耗时、脱敏头、过滤后与原始体（base64）、响应头（含剔除标记）、用量、响应截断标记、响应字节数等
  - 日志条数上限：默认 50，可在 ~/.clp/data/system.json 配置 logLimit；裁剪被丢弃的用量会聚合进 history_usage.json
- 历史用量：~/.clp/data/history_usage.json（聚合被裁剪日志的用量，src/core/base_proxy.py:216-284）
- 用量解析：src/utils/usage_parser.py
  - 支持 Claude 与 Codex 响应格式（JSON 与 SSE），标准化为统一 metrics（input/output/reasoning/total…）
- UI 侧：src/ui/ui_server.py 提供读取、聚合、合并历史的 REST 接口并展示

## 9. 服务实现与进程管理

- ClaudeProxy/CodexProxy（src/claude/proxy.py, src/codex/proxy.py）
  - 挂载 CORS 中间件以允许来自 http://localhost:3300 的前端连接
  - 实现 test_endpoint 用于健康检查/连通性测试（示例请求体各不相同）
- 控制器（BaseServiceController，src/core/base_proxy.py:919-1062）
  - 以独立进程启动 uvicorn，并写入 ~/.clp/run/{service}_proxy.pid 与日志
  - CLI（src/main.py）统一编排 start/stop/restart/status/ui/server

## 10. 扩展新服务的步骤

1) 在 src/ 下创建新目录并实现 {Service}Proxy 继承 BaseProxyService：
   - 指定 service_name/port/config_manager
   - 按需添加 CORS 与 test_endpoint
2) 创建 {Service}Controller 继承 BaseServiceController（或复用已有控制器模式）
3) 在 src/main.py 注册 CLI 入口（或 UI 中补充可视化入口）

完成后，即可复用现有的：请求过滤、模型路由、负载均衡、实时事件、日志与用量统计及 UI 展示能力。
