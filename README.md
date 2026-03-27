# Talon Documentation

**AI-Native Multi-Model Data Engine** — Official documentation site.

SQL + KV + TimeSeries + MessageQueue + Vector + Full-Text Search + GEO + Graph
— 8 open-source engines in a single binary with zero external dependencies.
— AI Engine available as commercial extension via [talon-ai](https://github.com/darkmice/talon-ai).

## Links

- **Documentation**: [https://darkmice.github.io/talon-docs/](https://darkmice.github.io/talon-docs/)
- **Source Code**: [github.com/darkmice/talon-core](https://github.com/darkmice/talon-core)
- **SDK**: [github.com/darkmice/talon-sdk](https://github.com/darkmice/talon-sdk)

## Talon Ecosystem

Talon 包含以下核心扩展生态：

| Crate | 许可 | 描述 |
|-------|------|------|
| **[talon-core](https://github.com/darkmice/talon-core)** | 开源 (MIT) | 8 合 1 基础数据引擎（SQL/KV/向量/时序/图等），零外部依赖。 |
| **[talon-ai](https://github.com/darkmice/talon-ai)** | 商业扩展 | 提供 Session、Memory、上下文、RAG 检索等 AI 原生数据模型支持。 |
| **[talon-llm](https://github.com/darkmice/talon-llm)** | 开源 (MIT) | 大模型统一网关，支持 20+ Provider，SSE 安全解析及 token 计量。 |
| **[talon-trace](https://github.com/darkmice/talon-trace)** | 开源 (MIT) | 追踪引擎，记录执行日志与完整上下文，支持 Team → Agent → Span 三级追踪。 |
| **[talon-sandbox](https://github.com/darkmice/talon-sandbox)** | 开源 (MIT) | 瀑布式高可用安全沙箱引擎（WASM → Deno → Docker），防 SSRF 与 OOM。 |
| **[talon-evo-core](https://github.com/darkmice/talon-evocore)** | 商业扩展 | Soul 记忆库与模型自进化核心（主动沉淀经验并自我迭代）。 |
| **[talon-agent](https://github.com/darkmice/talon-agent)** | 开源 (MIT) | 多 Agent 编排调度器，提供 Tool calling、MCP 支持及 Guardrails 双向校验。 |

## AI Coding Skills

This repository includes Talon usage skills for major AI coding tools. Clone this repo alongside your project and your AI assistant will automatically learn how to use Talon.

| Tool | Path |
|------|------|
| Windsurf | `.windsurf/skills/talon/` |
| Claude Code | `.claude/skills/talon/` |
| Cursor | `.cursor/skills/talon/` |
| Gemini | `.gemini/skills/talon/` |
| Kiro | `.kiro/skills/talon/` |
| Cline | `.cline/skills/talon/` |
| Codex | `.codex/skills/talon/` |
| Adal | `.adal/skills/talon/` |
| Agent | `.agent/skills/talon/` |
| npx-compatible | `skills/talon/` |

Each skill covers: SQL, KV, Vector, AI engine (via talon-ai), SDK usage, and more.

## Build

Documentation is built from [packages/docs](https://github.com/darkmice/talon-core/tree/main/packages/docs) in the main Talon repo and auto-published via GitHub Actions.

## License

MIT
