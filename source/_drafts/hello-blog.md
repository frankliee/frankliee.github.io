---
title: 博客开篇：AI Infra 与 Agent
date: 2026-10-06 12:00:00
tags:
  - AI Infra
  - Agent
---

记录 AI Infra 与 Agent 的学习和实践。

这是一篇用于预览主题排版的示例草稿，不作为正式博客发布。

## 内容方向

- **AI Infra**：模型推理、服务调度、缓存与性能分析。
- **Agent 工程**：工具调用、状态管理、评估与可观测性。
- **实验与源码笔记**：通过代码和实验理解系统的工作方式。

## 代码排版示例

```python
def summarize_latency(model_ms, tool_ms):
    return {
        "model_ms": model_ms,
        "tool_ms": tool_ms,
        "total_ms": model_ms + tool_ms,
    }
```

## 表格排版示例

| 方向 | 关注的问题 |
| --- | --- |
| 推理服务 | 延迟、吞吐与资源使用 |
| Agent 执行 | 工具调用、状态与故障恢复 |

> 每篇文章从一个具体问题出发，记录过程、证据与结论。
