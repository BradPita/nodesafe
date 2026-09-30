# nodesafe

> ComfyUI 自定义节点的安全扫描器——也是正在形成的"节点式工作流插件安全"事实标准。

[![PyPI](https://img.shields.io/pypi/v/nodesafe.svg)](https://pypi.org/project/nodesafe/)
[![Python](https://img.shields.io/pypi/pyversions/nodesafe.svg)](https://pypi.org/project/nodesafe/)
[![Downloads](https://img.shields.io/pypi/dm/nodesafe.svg)](https://pypi.org/project/nodesafe/)
[![License: Apache 2.0](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
[![CI](https://github.com/neuregex/nodesafe/actions/workflows/ci.yml/badge.svg)](https://github.com/neuregex/nodesafe/actions/workflows/ci.yml)

`nodesafe` 会在你把第三方插件/节点装进节点式工作流工具之前先扫一遍，通过一条层层升级的流水线——静态分析、特征签名匹配、机器学习、以及可选的大模型语义分析——揪出其中的恶意代码。起点：ComfyUI 生态。

## 为什么有这个东西

**2024 年 6 月**，ComfyUI_LLMVISION 这个节点偷走了几百个用户的浏览器凭据和加密钱包。**2026 年 4 月**，一个挖矿僵尸网络借 Manager 自动装恶意节点，打穿了 1000+ 台公网 ComfyUI。custom_nodes 生态体量大、迭代快，大部分节点基本没人审过。

`nodesafe` 帮你装前扫一遍。

## 快速开始

```bash
pip install nodesafe
nodesafe scan /path/to/custom_node
```

或者不安装，直接用：

```bash
uvx nodesafe scan /path/to/custom_node
```

## 工作原理

一条 9 层级联流水线，一层比一层贵。大多数干净的节点 <100ms 就过完了；只有拿不准的才会往上一层升。

| 层级 | 技术 | 成本 |
|-------|-----------|------|
| 0 | 对恶意软件数据库做哈希比对 | μs |
| 1 | 恶意 URL 布隆过滤器 | μs |
| 2 | Aho-Corasick 多模式危险片段匹配 | ms |
| 3 | AST 分析 + 混淆检测器（chr 链构造关键字、split-concat、香农熵、可疑标识符形态、Unicode 同形字攻击、嵌套解码链、文件级压缩） | ms |
| 4 | 仿冒包(typosquatting) + OSV 漏洞库核查 | ms |
| 5 | 聚合启发式风险分（人工标定；ML 模型等数据集到位再上） | 数十 ms |
| 6 | 异常检测（Isolation Forest + Autoencoder） | 数十 ms |
| 7 | 语义相似度（CodeBERT 向量 + FAISS） | 数百 ms |
| 8 | LLM 复核（可选，默认走 Ollama 本地） | 秒级 |

**当前进度（v0.5.1）：** 0-5 层已跑通并随 PyPI 正式包发行。Python 3.10–3.12 × Linux/macOS/Windows 共 87 个测试全部通过。第 3 层内置 7 个混淆检测器，覆盖按字符码拼关键字、split-concat、高熵字面量、可疑标识符、Unicode 同形字、嵌套解码链、以及整文件被压扁的情况。CLI 支持 `--batch`，一次扫一个装着很多节点的父目录。第 6-8 层排在 M3-M4 路线图里。

## 特性

- ✓ **纯静态分析** —— 全程不执行你被扫的代码
- ✓ **默认零遥测** —— 这条承诺不回头改
- ✓ **离线可用**（首次签名库更新之后）
- ✓ **多种输出格式**：JSON、Markdown（SARIF 排到 v0.6，用于接 GitHub Code Scanning）
- ✓ **可以直接挂 GitHub Action** —— 见示例 workflow
- ✓ **可以直接当 pre-commit hook** —— 适合 custom_nodes 仓库本身的 CI/CD
- ✓ **大模型分析默认本地优先** —— 默认 Ollama；想用云端自己开，自带 key
- ✓ **开源 Apache 2.0** —— 不搞免费试用分层、不藏一个 SaaS、不给钱不进白名单

## 用法

### 扫一个目录

```bash
nodesafe scan /path/to/custom_node
```

### JSON 输出

```bash
nodesafe scan /path/to/custom_node --format json
```

### 只走便宜层（快，不出聚合分）

```bash
nodesafe scan /path/to/custom_node --layers 0,1,2,3
```

### 批量模式（一次扫一整个 `custom_nodes/` 文件夹）

```bash
nodesafe scan ComfyUI/custom_nodes --batch
```

每个节点各出一行自己的判词，末尾追加一行整个目录的"最差判词"汇总。工具化的场景加 `--format json`，能拿到一个逐节点的摘要数组。

### 更新签名库

```bash
nodesafe update
```

### 体检安装环境

```bash
nodesafe doctor
```

## 历史案例复盘

nodesafe 真能抓到这里提到的几起旧事故吗？下面按流程逐层在脑子里跑一遍每个案例：

| 事件 | 命中层级 | 耗时 | 判词 |
|----------|-----------------|------|---------|
| LLMVISION (2024年6月) | 第 2-3 层 | ~30-50ms | malicious 0.98 |
| Pickai (2025年3-6月) | 第 2-3 层 + 5-7 | ~100ms | malicious 0.92 |
| 挖矿僵尸网络 (2026年4月) | 第 2-3 层 + Manager 关卡 | <50ms | malicious 0.95 |

完整分析见 [`docs/retrospective-analysis.md`](docs/retrospective-analysis.md)。

## 诚实的局限

`nodesafe` 是**静态分析**，不是沙箱。它的边界：

- **它挡不住上游供应链攻击**（某个正经供应商自己被攻破那种）。它查的是"当这段恶意代码最终以节点的形式分发出来时能不能发现"，不保证追到最初那次被入侵。
- **它不能替代 Manager** —— 定位上是互补的；最理想的状态是直接集成进去。
- **它不看运行时的行为** —— 那是 IDS/EDR 的活。
- **误报会出现** —— 整体策略偏保守；但每一条被点出来的东西都会精确写出到底触发了哪个检测点，方便你自己判断。

## 配置

`~/.config/nodesafe/config.toml`（可选项 —— 不写也有一套合理默认值）：

```toml
[scanner]
default_layers = "0,1,2,3,4,5,6"   # 第 8 层默认不包含
fail_on = "suspicious"

[llm]
enabled = false                     # 默认关。想开是你的主动选择。
provider = "local"                  # 若开启，默认本地优先

[llm.local]
endpoint = "http://localhost:11434" # Ollama
model = "qwen2.5-coder:7b-instruct"

[telemetry]
enabled = false                     # 永远 false。不回头改的承诺。
```

## 路线图

- **v0.5.x（已发布）：** 0-5 层 + 混淆检测器 + 批量模式。现在 `pip install nodesafe` 就能用。
- **v0.6（下一个版本）：** 运行时安装行为检测器（能抓到节点在运行时才去 pip install 或 git clone 拉代码的行为——正是 2026 年 4 月那次僵尸网络的传播方式）+ SARIF 输出接 GitHub Code Scanning。
- **v0.7（M3 阶段）：** 第 6 层异常检测（对现有特征提取结果跑 Isolation Forest + 自编码器），等标注样本攒够能支撑一个基线之后上。
- **v0.9（M3 阶段）：** 第 7 层语义相似度（CodeBERT 向量 + FAISS），目标是匹配那些换皮变形的多态变体。
- **v1.0：** 第 8 层大模型语境复核（默认 Ollama 本地，云端可选接入）+ 给 ComfyUI-Manager 提第一个 PR，让默认安装前先跑一遍扫描。
- **v1.5：** 公开威胁报告 + 社区签名贡献整合进主签名库。
- **v2+（第二年起）：** `.nodesafe` 标准扩展到其它节点式生态（LangFlow、n8n、Flowise）。

完整规划见 [`ARCHITECTURE.md`](ARCHITECTURE.md)。

## 贡献方式

欢迎提交 PR。详情见 [CONTRIBUTING.md](CONTRIBUTING.md)。

**尤其欢迎这几类：**
- **新的恶意软件签名贡献** —— 参考 [`signatures/README.md`](signatures/README.md)
- **正常节点被误报的反馈**
- **漏报反馈** —— 开 issue 时带上 `[missed-detection]` 标签
- **专门针对 ComfyUI / 扩散模型常见写法的 Semgrep 规则**

## 致谢

灵感来自 HuggingFace 推动的 `safetensors` 标准、[Snyk Labs 关于 ComfyUI 攻击面的一篇研究](https://labs.snyk.io/resources/hacking-comfyui-through-custom-nodes/)，以及 [u/_roblaughter_](https://www.reddit.com/r/StableDiffusion/) 这位老哥——是他自己掏真金白银才发现的 LLMVISION 这个坑。

## 许可证

Apache 2.0。详见 [LICENSE](LICENSE)。

## 长期愿景

ComfyUI 是最急的那一例，但不是唯一。只要节点式工具允许挂"可执行"插件（LangFlow、Flowise、Node-RED、n8n 等），就共享同一个结构性问题。长期来看，`.nodesafe` 的目标是成为一个任何生态都能直接采用的**可移植清单文件**——类比 `.safetensors` 是怎么变成 ML 模型权重事实标准的。

项目 V2-V3 阶段会把这个标准正式化，并去找其它生态的维护者一起对齐。今天，先把全部火力压在 ComfyUI 上。
