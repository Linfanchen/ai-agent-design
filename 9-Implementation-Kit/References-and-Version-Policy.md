# References and Version Policy

## Role in the Agent System

This document records the external sources used to shape the architecture and defines how source versions are tracked so future implementation work does not rely on stale or mixed documentation.

## 1. Reference projects

- AgentScope 2.0.11: <https://docs.agentscope.io/zh/versions/2.0.11>
- AgentScope 2.0.11 quickstart: <https://docs.agentscope.io/zh/versions/2.0.11/quickstart>
- DeerFlow docs: <https://deerflow.tech/zh/docs>
- LangGraph Python overview: <https://docs.langchain.com/oss/python/langgraph/overview>
- LangGraph Python docs index: <https://github.com/langchain-ai/langgraph/blob/main/docs/llms.txt>
- Octop: <https://github.com/TencentCloud/Octop>
- Octop Harness: <https://github.com/TencentCloud/octop-harness>
- Octop Memory: <https://github.com/TencentCloud/octop-memory>
- DeepSeek Harness: <https://github.com/deepseek-ai/deepseek-harness>
- Hermes Agent: <https://github.com/NousResearch/hermes-agent>

## 2. Source classification

Every design statement should be understood as one of:
- **Documented fact**: explicitly stated in the referenced project's docs.
- **Source observation**: verified in a specific source-code version.
- **Engineering inference**: an interpretation derived from the implementation.
- **Handbook recommendation**: the proposed architecture for this project.

Do not present recommendations as upstream behavior.

## 3. Version lock

Before implementation:
1. Record repository URL and commit SHA.
2. Record release/tag and license.
3. Save relevant docs snapshot or links.
4. Verify examples against the exact dependency version.
5. Run relevant tests.
6. Record known limitations and unverified claims.

## 4. Important version cautions

- AgentScope docs are pinned to 2.0.11 in this review. Do not mix 1.x APIs with 2.x.
- DeerFlow docs are a moving documentation set. Record the exact repository commit before copying behavior.
- LangGraph link supplied by the user is the Python documentation. Check JS/TS API parity separately.
- Legacy AgentScope Runtime docs may describe a predecessor whose capabilities were integrated into AgentScope 2.0.

## 5. License and security review

Before adopting source code, verify license compatibility, transitive dependencies, security posture, telemetry behavior, data residency, and third-party service terms.
