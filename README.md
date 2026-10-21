# security-audit

一个把编码代理变成安全审计员的 skill。它调度相互隔离的代理，依次完成侦察、按覆盖率狩猎、候选验证、结构化输出、独立记录核验，以及与目标无关的报告。

这个 skill 是 Cloudflare 漏洞发现框架的起点，详见 [Build your own vulnerability harness](https://blog.cloudflare.com/build-your-own-vulnerability-harness)。该框架后来发展成多阶段、覆盖整个舰队的系统；本 skill 就是它由之演化的单仓库起点。

## 它做什么

本 skill 按六个阶段执行结构化审计：

1. **侦察** — 映射架构、信任边界、输入面、既有证据，以及 `architecture.md` 与 `coverage-ledger.json` 里的确定性覆盖。
2. **按覆盖率狩猎** — 按账本单元指派相互隔离的 hunter，记录它们的检查项，并用 coverage critic 找出缺口。
3. **候选验证** — 把每一条不重复的候选交给一个全新的 verifier，由它尝试证伪。
4. **结构化输出** — 把 `confirmed`、`needs_validation`、`rejected` 记录写入 `findings.json`，并对照 `report-schema.json` 校验。
5. **独立记录核验** — 由新的代理核验最终的源码断言。实质性替换会再交给另一个独立 verifier。
6. **与目标无关的报告** — 从已核验记录和覆盖账本生成 `REPORT.md`、`FINDINGS-DETAIL.md` 和 `NEEDS-VALIDATION.md`。

主流程在创建账本之后，以及此后每次更新账本时，运行 `validate-coverage-ledger.cjs`。它在阶段 4 运行 `validate-findings.cjs`，并在阶段 5 的每一次替换之后再运行一次。

三种结论彼此不同：`confirmed` 有完整的源码追踪和有边界的观察结果；`needs_validation` 有一条确切的未决事实，且不标严重级别；`rejected` 记录一条已被证伪的候选。

对同一仓库的多次运行是累加的。本 skill 使用先前的账本和发现，针对缺口、复验已变更的源码，并把仍然对应当前源码的证据带下去，不把过期或未决的工作当成已覆盖。

## 文件

| 文件 | 作用 |
|------|------|
| `SKILL.md` | 安装、核心原则、平台术语、工作流概览，以及审计反模式 |
| `RECONNAISSANCE.md` | 阶段 1 的侦察提示词与综合说明 |
| `HUNTING.md` | 阶段 2 的编排、狩猎方法与验证规则 |
| `ATTACK-CLASSES.md` | 核心、通配（wildcard）与 obvious-things 攻击提示词 |
| `MEMORY-SAFETY-AND-BINARY.md` | 面向原生目标的内存安全、二进制与内核狩猎类别 |
| `AI-AND-LLM.md` | 面向 LLM 后端目标的提示注入、代理/工具与输出处理狩猎类别 |
| `WEB-PROTOCOL-AND-AUTH.md` | 面向 HTTP 协议与认证目标的请求分帧、缓存与认证协议狩猎类别 |
| `CLIENT-SIDE.md` | 面向客户端/浏览器目标的 DOM 注入、消息信任、UI redress 与原型污染狩猎类别 |
| `SUPPLY-CHAIN-AND-RELEASE.md` | 依赖、CI、发布、签名、更新、插件与扩展的狩猎类别 |
| `CLOUD-AND-DEPLOYMENT.md` | IAM、基础设施即代码、容器、无服务器、入口与运行时配置的狩猎类别 |
| `PROTOCOLS-RPC-AND-MESSAGING.md` | RPC、序列化、队列、broker、webhook 与流式协议的狩猎类别 |
| `RESOURCE-EXHAUSTION-AND-AVAILABILITY.md` | 共享资源、配额、队列、worker 与操作者支出的狩猎类别 |
| `DATA-ISOLATION-AND-LIFECYCLE.md` | 租户隔离、缓存、搜索、导出、备份、迁移、删除与恢复的狩猎类别 |
| `DESKTOP-MOBILE-AND-LOCAL-IPC.md` | 原生应用、深链、webview、导出组件、helper、daemon 与本地 IPC 的狩猎类别 |
| `VALIDATION-AND-REPORTING.md` | 阶段 3–6：候选验证、结构化输出、记录核验与报告 |
| `report-schema.json` | 覆盖三种 `findings.json` 结论的 JSON Schema |
| `validate-findings.cjs` | 阶段 4 与阶段 5 使用的、零依赖的 `findings.json` 校验器 |
| `validate-findings.test.cjs` | findings 校验器测试，以及与生产者兼容的 fixture 检查 |
| `validate-coverage-ledger.cjs` | 阶段 1–5 使用的、零依赖的 `coverage-ledger.json` 校验器 |
| `validate-coverage-ledger.test.cjs` | coverage-ledger 校验器测试 |

## 安装

使用 [Skills CLI](https://skills.sh) 安装本 skill：

```bash
npx skills add https://github.com/cloudflare/security-audit-skill \
  --skill security-audit
```

用户级安装加上 `--global`：

```bash
npx skills add https://github.com/cloudflare/security-audit-skill \
  --skill security-audit \
  --global
```

运行 `npx skills --help` 查看代理选择与非交互选项。

## 用法

在要审计的代码库里启动编码代理，或让代理指向该代码库，然后要求它做安全审计：

```
security audit this codebase
```

```
find security vulnerabilities in ./src
```

```
do a security review, output to ~/audits/my-project
```

请求命中触发词时，skill 会自动启用（例如 security audit、find vulnerabilities、pen-test the code）。上面三条是原文触发说法，保持英文。直接针对代码库的审计或渗透测试请求走完整审计模式。安全问答和范围收窄的漏洞工作走指导模式，除非你要求产出报告。完整审计模式下，如果没有指定输出目录，默认写到 `~/security-audit-skill/<repo-name>/run-<N>`。只有当你明确指定一个被版本控制忽略的目录时，工作流才会把产物写进目标仓库里。

## 要求

- 编码代理所用模型需支持工具调用和并行子代理
- Node.js，用于零依赖的 findings 与 coverage-ledger 校验器
- 对受目标控制的构建、测试、进程、浏览器、模拟器、fuzzer 和 fixture，需要由操作系统强制执行的沙箱。沙箱必须关闭外部网络、使用经过清洗的白名单环境、施加资源限制，并且只允许写入分配好的临时路径。没有这些控制时，工作流将线索保留为 `needs_validation`，不执行目标代码。

## 设计原则

- **只确认已经坐实的边界失效。** 有源码依据、但攻击仍被挡住的线索保持为 `needs_validation`，并写明确切的未决事实。
- **对抗式验证。** 检查某条发现的代理，绝不是发现它的那个代理。
- **严重级别取决于影响。** 用可能性 × 影响，而不是偏离检查清单的程度。
- **纵深防御的缺口不是漏洞。** 如果第 A 层已经挡住攻击，缺少第 B 层只记为加固说明。
- **多次运行能提高覆盖。** 在我们的测试里，单次运行找到的漏洞，大约只有反复运行合计数量的一半。

## 联系

问题、反馈，或想交流 AI 驱动的安全工具：security-ai-research@cloudflare.com

## 许可证

MIT — 见 [LICENSE](LICENSE)。
