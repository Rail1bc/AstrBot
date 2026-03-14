# 硬编码提示词/消息注入审计（聚焦不可配置项）

本文档记录 AstrBot 运行时的提示词/消息注入行为，重点关注**无法直接通过配置覆盖的硬编码文本**。
当前快照已完成 NC-01~NC-10 的可配置化治理，`NC-HC` 清单为空。

本次快照审计范围：
- 主 Agent 请求构建与装饰流程
- 工具循环执行器（Tool Loop Runner）
- Cron/后台任务唤醒流程
- 上下文压缩路径

不在本次范围：
- 第三方 Agent Runner 以及不在核心运行路径中的插件私有提示逻辑

## 分类说明

- `NC-HC`（Non-configurable hardcoded）：文本为硬编码，且没有直接配置项可覆盖该文本。
- `TG-HC`（Toggle-gated hardcoded）：文本为硬编码，但是否注入由配置开关或运行模式决定。
- `CFG`（Config-driven）：文本本身来自配置/人格/路由，不纳入硬编码风险清单。

---

## A. NC-HC：不可配置的硬编码注入

当前版本未发现 `NC-HC` 项（空表）。

| ID | 位置 | 注入目标 | 硬编码内容（摘要） | 触发条件 |
|---|---|---|---|---|
| - | - | - | - | - |

### A.1 已治理项（NC -> CFG 对照）

以下历史 NC 项已迁移为配置驱动，统一从 `provider_settings` 读取，且保留默认值兜底：

| ID | 现状态 | 配置键 | 占位符 | 回退策略 |
|---|---|---|---|---|
| NC-01 | CFG | `provider_settings.tool_call_prompts.follow_up_notice_prompt` | 无 | 缺失/空字符串回退内部默认 notice |
| NC-02 | CFG | `provider_settings.tool_call_prompts.max_step_reached_prompt` | 无 | 缺失/空字符串回退 `ToolLoopAgentRunner` 默认文案 |
| NC-03 | CFG | `provider_settings.tool_call_prompts.requery_instruction_prompt` | `{tool_names}` | 缺失/空字符串回退默认模板；模板 `.format` 异常时也回退默认模板 |
| NC-04 | CFG | `provider_settings.proactive_capability.cron_prompts.history_wrap_prompt` | `{history}` | 缺失/空字符串回退默认模板；模板 `.format` 异常时回退默认模板 |
| NC-05 | CFG | `provider_settings.proactive_capability.cron_prompts.execution_prompt` | 无 | 缺失/空字符串回退默认文案 |
| NC-06 | CFG | `provider_settings.proactive_capability.background_prompts.history_wrap_prompt` | `{history}` | 缺失/空字符串回退默认模板；模板 `.format` 异常时回退默认模板 |
| NC-07 | CFG | `provider_settings.proactive_capability.background_prompts.execution_prompt` | 无 | 缺失/空字符串回退默认文案 |
| NC-08 | CFG | `provider_settings.context_summary_prompts.user_prompt` | `{summary}` | 缺失/空字符串回退默认模板；模板 `.format` 异常时回退默认模板 |
| NC-09 | CFG | `provider_settings.context_summary_prompts.ack_prompt` | 无 | 缺失/空字符串回退默认文案 |
| NC-10 | CFG | `provider_settings.kb_repair_user_prompt_template` | `{chunk}` | 缺失/空字符串/非字符串回退默认模板；模板 `.format` 异常时回退默认模板 |

---

## B. TG-HC：受开关/模式控制的硬编码文本

这些文本同样是硬编码，但注入行为可通过配置或运行模式间接启停。

| ID | 位置 | 注入目标 | 硬编码内容（摘要） | 触发控制 |
|---|---|---|---|---|
| TG-01 | `astrbot/core/astr_main_agent.py:1187` + `astrbot/core/astr_main_agent_resources.py:63` | `req.system_prompt` | `TOOL_CALL_PROMPT` / `TOOL_CALL_PROMPT_SKILLS_LIKE_MODE` | `req.func_tool` 存在时触发；文本本体无配置项 |
| TG-02 | `astrbot/core/astr_main_agent.py:836` + `astrbot/core/astr_main_agent_resources.py:41` | `req.system_prompt` | `LLM_SAFETY_MODE_SYSTEM_PROMPT` | 由 `provider_settings.llm_safety_mode` 控制 |
| TG-03 | `astrbot/core/astr_main_agent.py:1191` + `astrbot/core/astr_main_agent_resources.py:100` | `req.system_prompt` | `LIVE_MODE_SYSTEM_PROMPT` | `action_type == live` 时触发 |
| TG-04 | `astrbot/core/astr_main_agent.py:917` + `astrbot/core/astr_main_agent_resources.py:52` | `req.system_prompt` | `SANDBOX_MODE_PROMPT` | 由 `computer_use_runtime == sandbox` 控制 |
| TG-05 | `astrbot/core/astr_main_agent.py:868` | `req.system_prompt` | `[Shipyard Neo File Path Rule] ...` | `computer_use_runtime == sandbox` 且 booter 为 `shipyard_neo` |
| TG-06 | `astrbot/core/astr_main_agent.py:875` | `req.system_prompt` | `[Neo Skill Lifecycle Workflow] ...` | `computer_use_runtime == sandbox` 且 booter 为 `shipyard_neo` |
| TG-07 | `astrbot/core/astr_main_agent.py:283` | `req.system_prompt` | 本地运行能力声明（`You have access to the host local environment...`） | `computer_use_runtime == local` |
| TG-08 | `astrbot/core/astr_main_agent.py:261` | `req.contexts`（`role=system`） | `File Extract Results of user uploaded files...` | `file_extract.enable` 且存在上传文件 |
| TG-09 | `astrbot/core/astr_main_agent.py:577` | `req.extra_user_content_parts` | `<Quoted Message>...</Quoted Message>` 包装 | 引用消息解析开启且成功解析内容 |
| TG-10 | `astrbot/core/astr_main_agent.py:620` | `req.extra_user_content_parts` | 带固定标签的 `<system_reminder>...</system_reminder>` | 由 `identifier/group_name_display/datetime_system_prompt` 开关控制 |

---

## C. 未纳入硬编码风险清单（CFG 示例）

以下项有意不计入 NC-HC，因为文本来自配置或用户数据：
- 人格提示正文（`persona.prompt`），位置：`astrbot/core/astr_main_agent.py:330`
- 路由提示词（`subagent_orchestrator.router_system_prompt`），位置：`astrbot/core/astr_main_agent.py:441`
- 前缀提示（`provider_settings.prompt_prefix`），位置：`astrbot/core/astr_main_agent.py:635`
- 图像描述提示（`provider_settings.image_caption_prompt`），在 `_request_img_caption` 中使用
- 历史 NC 注入项（`tool_call_prompts.*`、`cron_*`、`background_*`、`context_summary_*`、`kb_repair_user_prompt_template`）

---

## 建议加固顺序

1. 在配置文档中明确所有模板占位符（`{tool_names}`、`{history}`、`{summary}`、`{chunk}`）的语义与示例。
2. 为注入类配置增加可选校验器（保存配置时静态检查占位符格式），降低运行时格式化回退概率。
3. 为合成消息增加显式来源标记（例如 `source=system_injection`），降低隐式上下文耦合。
4. 在回归测试中覆盖“自定义模板 + 异常模板回退默认值”两类路径。

---

## 备注

- 本文件是审计快照，不是行为规范文档。
- 行号基于创建文档时的工作区状态，后续代码变更可能导致偏移。
