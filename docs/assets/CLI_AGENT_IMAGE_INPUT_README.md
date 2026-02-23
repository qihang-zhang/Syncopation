# CLI Agent 图片输入方案

这个目录包含了关于如何在 CLI agent 中处理图片输入的文档和示例代码。

## 文件说明

### 文档
- **`digital_garden/2026-02-17-cli-agent-image-input-guide.md`** - 完整的 CLI agent 图片输入方案指南
  - 涵盖 6 种主流输入方式
  - 包含代码示例
  - 业界最佳实践
  - 主流 CLI agent 案例分析

### 示例代码
- **`scripts/image_agent_example.py`** - 可运行的 Python 示例程序
  - 演示多种图片输入方式
  - 包含完整的错误处理
  - 提供命令行帮助
- **`scripts/vision_api_example.py`** - Vision API 集成示例
  - 展示如何与 GPT-4V 和 Claude 3 集成
  - 生成标准 API 请求格式
  - 支持 dry-run 模式

## 快速开始

### 1. 查看完整文档

```bash
# 查看完整的指南文档
cat digital_garden/2026-02-17-cli-agent-image-input-guide.md
```

### 2. 运行示例程序

#### 基础图片输入示例

```bash
# 查看帮助信息
python3 scripts/image_agent_example.py --help

# 示例1: 处理单个图片
python3 scripts/image_agent_example.py --image /path/to/image.jpg

# 示例2: 批量处理多个图片
python3 scripts/image_agent_example.py --images image1.jpg image2.png

# 示例3: 从标准输入读取
cat image.jpg | python3 scripts/image_agent_example.py --image -

# 示例4: 使用 base64 编码
python3 scripts/image_agent_example.py --image-base64 "$(base64 < image.jpg)"
```

#### Vision API 集成示例

```bash
# 生成 OpenAI GPT-4V API 请求
python3 scripts/vision_api_example.py \
  --image photo.jpg \
  --prompt "这张图片里有什么？" \
  --api openai \
  --pretty

# 生成 Anthropic Claude 3 API 请求
python3 scripts/vision_api_example.py \
  --image diagram.png \
  --prompt "解释这个架构图" \
  --api anthropic \
  --pretty

# 保存请求到 JSON 文件
python3 scripts/vision_api_example.py \
  --image chart.jpg \
  --prompt "分析这个图表" \
  --output request.json
```

## 主要内容

### 六种输入方式

1. **文件路径** (最常用) - 直接传递文件路径
2. **Base64 编码** - 适合 API 集成
3. **URL** - 处理网络图片
4. **交互式** - 用户友好的输入
5. **配置文件** - 批处理和复杂场景
6. **标准输入** - Unix 管道操作

### 最佳实践

- 优先支持文件路径（最简单直接）
- 提供多种输入方式的灵活性
- 完善的错误处理和验证
- 清晰的文档和示例

## 在 Polyrhythm 中的应用

根据 PLAN.md，这些方案可以应用于：

1. **音视频转笔记管线** - 从视频中提取关键帧
2. **灵感采集系统** - 支持图片附件
3. **Browser-use 抓取** - 自动下载和处理网络图片

## 依赖安装（可选）

如果需要完整的图片处理功能：

```bash
pip install Pillow
```

不安装 Pillow 也可以运行示例程序，但功能会受限。

## 主流 CLI Agent 参考

文档中包含了以下工具的实践案例：

- GitHub Copilot CLI
- Aider
- Claude Code / Cursor
- LangChain CLI Tools
- GPT-4V / Claude 3 API

## 相关资源

- [OpenAI Vision API](https://platform.openai.com/docs/guides/vision)
- [Anthropic Claude 3 Vision](https://docs.anthropic.com/claude/docs/vision)
- [Aider Documentation](https://aider.chat/docs/usage.html)

## 问题反馈

如有问题或建议，请在 GitHub Issues 中反馈。
