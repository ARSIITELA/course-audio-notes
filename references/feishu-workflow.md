# 飞书/Lark 同步工作流（可选发布环节）

本地笔记写好后，如用户要求发布到飞书，按本文件创建云文档、挂知识库、验证、交付。

## 配置（首次使用必填）

分享/复制本 Skill 后，先向用户确认以下信息，在对话中直接使用即可；也可把值填回本文件方便长期复用：

| 变量 | 说明 | 如何获取 |
|---|---|---|
| `<YOUR_TENANT_DOMAIN>` | 飞书租户域名，如 `xxx.feishu.cn` | 打开任意飞书文档，URL 中的域名 |
| `<YOUR_SPACE_ID>` | 目标知识库 space_id | `lark-cli wiki +node-get --node-token <知识库URL>` 返回的 `data.space_id` |
| `<YOUR_PARENT_NODE_TOKEN>` | 笔记要挂的父节点 node_token | 父节点 wiki URL 中 `/wiki/` 后的那串 |
| `<YOUR_PROJECT_DIR>` | 笔记 .md 所在的本地目录 | 本地产物的项目目录 |

用户没有提供 space_id / 父节点时，先让用户给出目标知识库或父节点的飞书链接，用 `wiki +node-get` 解析，不要凭空猜。

文档 URL 格式：`https://<YOUR_TENANT_DOMAIN>/docx/<doc_token>`
知识库 URL 格式：`https://<YOUR_TENANT_DOMAIN>/wiki/<node_token>`

## 前置：命令运行环境

- 所有命令统一带 `--as user` 和 `--format json`。
- `--content @./文件名.md` 只接受**当前工作目录的相对路径**，传绝对路径会报 unsafe file path。
- 每条命令都先 `cd` 到笔记所在目录，形如：

```bash
cd "<YOUR_PROJECT_DIR>" && lark-cli ...
```

## 第 1 步：创建飞书文档

```bash
cd "<YOUR_PROJECT_DIR>" && lark-cli docs +create \
  --doc-format markdown \
  --content @./<笔记文件名>.md \
  --title "<文档标题，如：AI产品经理第二十三天>" \
  --as user --format json
```

返回 JSON 中取 `data.document.document_id`（即 doc_token）和 `data.document.url`。

注意：

- 标题命名遵循用户习惯（如本地文件名用阿拉伯数字"第23天"，飞书标题用中文数字"第二十三天"）。
- 创建前先确认同名文档不存在，一篇笔记只创建一次。
- 服务端返回的 `warnings` 要逐条看，有降级内容要修。

## 第 2 步：挂到知识库父节点

```bash
cd "<YOUR_PROJECT_DIR>" && lark-cli wiki +move \
  --obj-type docx \
  --obj-token <第1步doc_token> \
  --target-space-id <YOUR_SPACE_ID> \
  --target-parent-token <YOUR_PARENT_NODE_TOKEN> \
  --as user --format json
```

- 正常同步返回 `ready: true`、`node_token`（即 wiki token）。
- 若返回 `ready: false` + `task_id`（异步），按返回的 `next_command` 用 `lark-cli drive +task_result --scenario wiki_move --task-id <TASK_ID>` 轮询。

## 第 3 步：验证（两步都要过）

板块大纲：

```bash
lark-cli docs +fetch --doc <doc_token> \
  --scope outline --max-depth 2 \
  --as user --format json
```

核对：标题正确、每个 `##` 板块都在（数量与本地文件一致）、无乱码。

挂载位置：

```bash
lark-cli wiki +node-get --node-token <wiki_node_token> \
  --as user --format json
```

核对：`data.parent_node_token` = 目标父节点 token，`data.space_id` = 目标 space_id，`data.title` 正确。

## 第 4 步：交付

- 用 present_files 一次性交付两个 artifact：
  1. 本地文件绝对路径
  2. wiki 链接（docx 链接也可一并附上）
- 最终回复保持简短：覆盖了哪几段录音、板块数、模糊处理项。

## 异常处理

| 情况 | 处理 |
|---|---|
| 文档已创建但内容有错/要补 | 用 `lark-cli docs +update`（局部 str_replace / block_replace），**绝不重建文档** |
| wiki +move 返回 rate_limit | 退避后重试；其他错误（invalid_parameters/not_found/permission_denied）按 hint 改参数，不重复调用 |
| create 成功但 move 失败 | 文档已在云空间，修好参数后单独重跑 move，不要重建文档 |
| 导入后 Markdown 符号异常 | 检查 `<`、`[`、`|` 等字符是否需转义；代码块和行内代码内不用转义 |
| Markdown 里写了 XML 尖括号 | 字面量 `<` 写成 `\<`，否则会被当成标签解析 |

## 依赖的 Skill

本流程依赖 `lark-doc`（docs +create/+fetch/+update）和 `lark-wiki`（wiki +move/+node-get）两个内置 Skill。若当前环境 CLI 命令语法有变化，先重读这两个 Skill 的 SKILL.md 和 references 再执行，不要凭记忆拼参数。

## 适配其他发布目标

不使用飞书时，保留笔记撰写流程不变，把第 1-3 步替换为目标平台的发布动作（如 Notion 用 notion-operations、本地 Git 仓库直接 commit 推送），交付物仍是本地 Markdown + 平台链接。
