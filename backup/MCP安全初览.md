嘻嘻，来自于@ESFJ-MoZhu，之后再塞图，还没研究怎么放超链接
# MCP相关笔记
## 一、MCP概述
### （一）诞生背景
大模型在数学计算等特定任务中非常唐。所以出现了function call，通过调用外部工具，增强大模型的计算能力。但存在显著缺陷：不同模型对接不同工具需适配不同接口，工作量重复繁琐。
MCP（Model Context Protocol）作为大模型与工具之间的中间层应运而生，解决该问题。

### （二）核心优势
大模型与工具仅需各自对接MCP，即可实现互相调用，将原本“模型数量×工具数量”的工作量简化为“模型数量+工具数量”，大幅降低对接成本。（类比的解释起来就是不要依赖重装）

## 二、MCP架构与传输方式
### （一）架构设计（经典C/S架构）
1. **MCP Host**：协调和管理一个或多个MCP Client的AI应用程序，例如VScode、CherryStudio、Cursor等。
2. **MCP Client**：与MCP Server保持一对一连接、获取上下文的组件，可类比为线程或端口。
3. **MCP Server**：工具的具体体现，本质是一段脚本程序（可用Node.js或Python编写），向MCP Client提供上下文，例如计算器工具即对应MCP Server。

### （二）传输方式
| 传输方式 | 底层原理 | 核心特性 |
|----------|----------|----------|
| stdio    | 操作系统底层命令调用，通过标准输入/输出流通信 | 本地运行、低延迟低开销、无需网络配置、一对一关系、本质更安全 |
| sse      | 基于HTTP/HTTPS，建立持久连接 | 支持远程访问、可同时处理多个客户端、通过标准HTTP工作、支持HTTP认证 |
| streamable http | 兼容sse | 功能更优，是更推荐的选择 |

## 三、MCP使用步骤（Python环境为例）
### （一）开发环境安装
1. 安装uv（环境管理工具），参考教程：https://docs.astral.sh/uv/getting-started/installation/
2. 环境配置命令：
   - 列出可用Python版本：`uv python list`
   - 安装3.13版本（示例）：`uv python install 3.13`
   - 初始化项目：`cd 工作目录 && uv init . -p 3.13`（指定Python版本）
   - 安装MCP Python SDK：`uv add "mcp[cli]"`

### （二）编写工具
1. 从官方Python-SDK仓库（https://github.com/modelcontextprotocol/python-sdk）拷贝Demo代码。
2. 使用`@mcp.tool()`注解标记工具，通过参数类型标记语法定义参数，函数内添加注释说明功能。

### （三）配置MCP Host（以Cherry Studio为例）
1. 配置大模型API，安装Cherry Studio内置uv环境。
2. 添加MCP Server，选择传输方式（如stdio），配置启动参数（命令、工作目录、运行脚本等）。
3. 绿灯表示配置正常，可选择工具并向大模型提问，实现智能调用。

### （四）测试调用
示例：向大模型提问“计算114514+1919810”，大模型通过MCP调用加法工具，返回结果2034324。
而它自己是很唐的，算数不是算出来的而是生成的。

## 四、MCP与Function Call的对比及对接方式
### （一）核心差异
| 特性                | Function Call                          | MCP                                  |
|---------------------|----------------------------------------|--------------------------------------|
| 对接逻辑            | 无中间层，模型与工具直接对接           | 中间层架构，模型与工具均对接MCP      |
| 工作量              | 模型数量×工具数量，重复繁琐            | 模型数量+工具数量，大幅简化          |
| 模型要求            | 需经过特殊结构化训练才能支持           | 支持两种对接方式，部分场景无需模型特殊支持 |
| Token消耗           | 相对节约                               | 系统提示词方式消耗较多，function call对接方式节约 |

### （二）大模型对接MCP的两种方式
1. **系统提示词**：
   - 原理：直接通过提示词告知模型MCP工具的调用格式（XML-style标签）、参数要求及规则。
   - 优势：简单易懂，无需模型特殊支持；劣势：Token消耗较多。
2. **Function Call**：
   - 原理：与MCP对接逻辑类似，但仅需连接大模型与MCP，无需重复对接工具。
   - 优势：节约Token；劣势：依赖模型支持（部分模型如Deepseek-R1不支持）。

### （三）对接注意事项
对接效果与MCP Host实现相关：例如Cursor可使不支持function call的Deepseek-R1模型成功调用MCP，而5ire会提示模型不支持。

## 五、MCP安全风险及相关漏洞
### （一）配置文件污染（RCE风险）
- 风险场景：MCP配置文件若被污染，攻击者可注入恶意操作系统命令（如反弹Shell），导致远程代码执行（RCE）。
- 案例：智能体编排Mindspore曾存在该漏洞，修复方式为过滤敏感函数，攻击者可能通过编码绕过防御。
- 参考漏洞：https://github.com/1Panel-dev/MaxKB/security/advisories/GHSA-38q2-4mm7-qf5h

### （二）供应链投毒（CVE-2025-6514）
- 风险场景：任何人可编写MCP Server并上传至Npm/Pip等平台，平台缺乏安全检查，恶意MCP Server可通过OAuth认证响应注入恶意代码。
- 影响范围：污染超437000个主机，波及Cloudflare、Hugging Face等机构的AI开发环境。
- 攻击流程：用户配置远程MCP Server→MCP发起OAuth元数据请求→恶意服务器返回含执行命令的响应→触发代码执行。

### （三）缺乏验证机制（CVE-2025-49596）
- 风险场景：MCP协议设计未充分考虑安全，结合0.0.0.0漏洞（浏览器未正确处理该IP，误判为localhost），攻击者可通过恶意网站向受害者本地MCP Server发送请求，实现命令注入。
- 防御建议：添加CSRF Token防护。
- 参考链接：https://www.oligo.security/blog/critical-rce-vulnerability-in-anthropic-mcp-inspector-cve-2025-49596

### （四）自动批准控制漏洞（CVE-2025-54136）
- 风险场景：Cursor 1.2.4及以下版本中，用户批准MCP工具后可开启“自动批准”，若攻击者修改已信任的MCP配置文件（本地或共享GitHub仓库），替换为恶意命令，不会触发警告，导致持久化RCE。
- 修复情况：Cursor 1.3版本已修复。
- 参考链接：https://research.checkpoint.com/2025/cursor-vulnerability-mcpoison/

### （五）名称冲突风险
- 风险场景：部分MCP Client（如Cherry Studio）支持添加名称相同、功能相似的多个MCP Server。攻击者可向受害者设备添加含恶意代码的同名MCP Server，在用户不知情时触发攻击。

### （六）提示词注入风险
- 风险场景：MCP工具描述中的提示词可能被注入恶意指令，干扰模型正常输出。
- 案例：在天气工具的描述中隐藏提示词，强制模型无论用户提问内容如何，均调用天气工具返回随机天气结果。

## 六、MCP的赋能场景
可基于MCP开发代码审计工具，梳理工具调用链，辅助逆向分析等安全相关应用。