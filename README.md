# schematic-reviewer — 原理图检查 skill（WorkBuddy / Codex / Claude Code）

仓库里只维护**一份** skill 源，渲染出 WorkBuddy、Codex 与 Claude Code 三套可直接安装的产物。

构建、安装与规范符合性说明见 [docs/build-and-install.md](docs/build-and-install.md)。

```bash
python build.py --install claude     # Claude Code：装到 ~/.claude/skills
python build.py --install workbuddy  # WorkBuddy：装到 ~/.workbuddy/skills
python build.py --install codex      # Codex：装到 ~/.codex/skills
```

## 下载安装包

不想跑构建就用现成的 zip。**从 [Releases](https://github.com/chen66663/schematic-reviewer/releases/latest) 下载**，
解压后得到 `schematic-reviewer/` 目录，整个放进对应平台的 skills 目录即可。

| 平台 | 安装包 | 解压到 |
|---|---|---|
| Claude Code | [schematic-reviewer-claude.zip](https://github.com/chen66663/schematic-reviewer/releases/latest/download/schematic-reviewer-claude.zip) | `~/.claude/skills/` |
| Codex | [schematic-reviewer-codex.zip](https://github.com/chen66663/schematic-reviewer/releases/latest/download/schematic-reviewer-codex.zip) | `~/.codex/skills/` |
| WorkBuddy | [schematic-reviewer-workbuddy.zip](https://github.com/chen66663/schematic-reviewer/releases/latest/download/schematic-reviewer-workbuddy.zip) | `~/.workbuddy/skills/` |

同样的文件也放在仓库的 [`packages/`](packages/) 目录（`/packages` 这个 URL 是 GitHub 包注册表，
不是这个目录，别点错）。校验和见
[Releases 里的 SHA256SUMS](https://github.com/chen66663/schematic-reviewer/releases/latest/download/SHA256SUMS)：

```bash
sha256sum -c SHA256SUMS                   # Linux / Git Bash（在下载目录执行）
Get-FileHash .\schematic-reviewer-claude.zip -Algorithm SHA256   # PowerShell
```

安装包由 `python build.py --pack` 从同一份源生成；改了 `skills/` 之后要重新打包再提交。

## 包含的 skill

| skill | 作用 |
|---|---|
| [schematic-reviewer](docs/schematic-reviewer.md) | 原理图检查与审核。读芯片数据手册做逐引脚分析，检查去耦、上拉、匹配等外围电路是否缺失或参数错误，识别大电流路径并按 IPC-2221 计算走线宽度，通过立创/JLC 接口核实元器件规格与现货库存。 |

## 仓库结构

```
.
|-- README.md                    本文件：总览与使用步骤
|-- build.py                     构建器：单一源 -> dist/<平台>/
|-- packages/                    三平台 zip 安装包（由 build.py --pack 生成）
|-- platforms/                   平台适配定义
|   |-- README.md                平台标记与占位符语法
|   |-- workbuddy.json           WorkBuddy：变量、安装目录
|   |-- codex.json               Codex：变量、安装目录
|   |-- claude.json              Claude Code：变量、安装目录
|   `-- codex/overlay/           Codex 专属文件（agents/openai.yaml）
|-- skills/                      唯一需要维护的源码
|   `-- schematic-reviewer/
|       |-- SKILL.md             主流程、输出原则、Gotchas 与约束（含平台标记）
|       |-- references/          外部 checklist 索引、立创接口用法、外围电路规则、载流速查表
|       |-- scripts/             5 个纯标准库脚本
|       `-- tests/               4 个回归测试，共 23 项
|-- docs/
|   |-- build-and-install.md     构建、安装与规范符合性
|   |-- schematic-reviewer.md    skill 使用说明（不随 skill 安装）
|   `-- images/                  嘉立创 EDA 导出操作截图
`-- dist/                        构建产物，不入库（见 .gitignore）
```

## 使用步骤（嘉立创 EDA 导出网表 → 让 AI 检查）

### 第 1 步：在原理图界面点击导出

打开嘉立创 EDA 的原理图，点击顶部菜单「导出(R)」→「网表文件...」。

![第1步：导出菜单中选择网表文件](docs/images/step1-export-netlist.png)

### 第 2 步：选择需要的信息

在弹出的「导出网表」窗口中：

- 网表类型：Allegro(.tel)
- 范围：选择要检查的原理图（如 `电源板 : Schematic2`）
- 器件：建议**只保留「器件 {Device}」「描述 {Description}」「数据手册 {Datasheet}」三项**，
  即 `{Device}{Description}{Datasheet}`；封装、附加属性（Value/Tolerance）等无需勾选

在「器件」输入框中选择这三个字段：

![第2步：器件字段只选器件、描述、数据手册](docs/images/step2-select-fields.png)

确认器件栏显示为「(器件)(描述)(数据手册)」后，点击「导出」，保存生成的 .tel 网表文件：

![第2步：确认选项后点击导出](docs/images/step3-export.png)

### 第 3 步：把文件地址发给 AI

将导出的网表文件完整路径（文件地址）发送给 WorkBuddy、Codex 或 Claude Code。

### 第 4 步：使用 skill 检查原理图

让 AI 调用 `schematic-reviewer` skill 检查这份网表，即可得到外围电路、载流走线加宽、
元器件选型与库存等方面的问题清单。

## 验证

```bash
cd skills/schematic-reviewer
python tests/test_review_export.py
python tests/test_lcsc_lookup.py
python tests/test_parse_schematic_export.py
python tests/test_parse_tel_netlist.py
```

23 项回归测试全部通过即脚本正常。`test_lcsc_lookup.py` 与 `review_export.py` 的批量核验需要访问
`jlcpcb.com`；离线环境下这部分用例会跳过或报网络错误，其余用例不受影响。

## 环境要求

- Python 3.8+，脚本只依赖标准库
- 可访问 `jlcpcb.com`（元器件接口）与厂商数据手册站点
- WorkBuddy、Codex 或 Claude Code 任一环境；三套产物互不影响

## 已知限制

- 无浏览器自动化：立创商城网页是 JS 单页应用，抓 HTML 拿不到商品数据，必须走接口。
- 嘉立创 EDA/BOM 导出通常只有器件参数，没有网络连接；不能据此判断具体引脚接线。
  带 `$NETS` 的 `.tel` 可以核对 `位号.引脚号`，但仍没有坐标和封装引脚名。
- 元器件接口为第三方公开接口，无鉴权但可能变更；失效时脚本会明确报错，需重新探查接口。
- 立创手册链接是在线阅读页而非 PDF 直链，数据手册优先从厂商官网取。
