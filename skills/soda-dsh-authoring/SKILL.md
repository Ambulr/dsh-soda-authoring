---
name: soda-dsh-authoring
description: >
  Build, install, or debug a DeepSeek Harness skill or plugin bundle. Use when creating a skill
  (a SKILL.md the model loads on demand), when a skill does not show up in the catalog, when
  `skill` reports "unknown or no longer available", when converting notes into an installable
  skill, or when building a Host/client plugin bundle that goes into the profile. Triggers:
  做一个 skill / 做一个插件 / 做成 skill / 写 SKILL.md / 装进 profile / 为什么 skill 加载不到 /
  为什么插件没生效 / skill 目录在哪.
metadata:
  version: "3.0.0"
  platform: windows
---

# DSH skill 与插件制作手册

v3 依据：DSH 0.2.2-rc.5 的官方文档（`plugin-development.md`、`host-plugin.md`、`practices.md`、`composition-reference.md`）、社区 50+ 条线上资料，以及亲自把插件装进 profile 踩过的坑。

## 0. 三十秒判断

| | skill | 插件（bundle） |
|---|---|---|
| 形态 | 一份 `SKILL.md`（YAML frontmatter + Markdown） | npm 包：`package.json` + `index.js` + `cordis.patch.yml` |
| 装法 | **把文件夹放进六个根目录之一**，没有安装命令 | `plugin_manager install_bundle`（推荐）或手写 profile |
| 生效 | **热重载**，不用重启 | 重启 DSH / 新会话 |
| 用途 | 告诉模型「这类活怎么干」 | 出工具、监听事件、改界面 |

**只有当需要代码去干预时才做插件。** 绝大多数「让它记住这个流程」的诉求是 skill。

## 1. skill 放哪（第一坑）

**只扫一层**：`<root>/<name>/SKILL.md`，或扁平 `<root>/<name>.md`。
`skills/<name>/SKILL.md`（嵌在 bundle 里）、`a/b/SKILL.md` 全是**刻意不发现**，写多少遍都不会加载。

| rank | 路径 | 范围 |
|---|---|---|
| 100 | `<projectRoot>/.dsh/skills/` | 当前项目 |
| 200 | `<projectRoot>/.agents/skills/` | 当前项目（跨工具共用） |
| 300 | `Config.customSkillDirs` | 自定义 |
| 400 | `~/.dsh/skills/` | **全局，默认放这** |
| 500 | `~/.agents/skills/` | 全局共用 |
| 600 | 内置 / 其他 provider | — |

`projectRoot` = 离工作区最近、含 `.git` 的祖先目录。用户根会跳过自己的 `.system` 子目录。

**推荐做法**（源码留在自己的项目里，skill 根目录挂一条 junction，一处维护）：

```powershell
mklink /J "$env:USERPROFILE\.dsh\skills\<name>" "D:\my-projects\<name>\skills\<name>"
```

junction 会被正常发现（社区实测确认，主路径对子项做 realpath）。根目录不存在时 provider **安静地返回 0 个**，不报错 —— 先 `New-Item -ItemType Directory` 建出来再排查。

## 2. SKILL.md 怎么写

```md
---
name: kebab-case-名字          # 必填，且必须等于父目录名
description: >                  # 必填，模型唯一看到的路由信息
  它解决什么 + 什么时候该用它（触发词中英都写）
metadata:
  version: "1.0.0"
---
正文：命令、判据、步骤。
```

规则：

- **`name` 必须匹配 `^[a-z0-9]+(?:-[a-z0-9]+)*$`**。中文、大写、下划线、空格 → 整条被丢弃，只留一行日志，面板却可能显示「已装」。
- 可选字段：`whenToUse`、`metadata`、`disable-model-invocation`、`user-invocable`。**这两个调用键写成 camelCase 或非布尔值 → 整条 skill 被丢弃**（不是只丢那个字段），这是「明明写了却加载不到」的头号根因。布尔接受 true/false、yes/no、on/off、1/0。
- catalog 里只有 `name` + `description`（**截断 500 字符**，正文/路径/来源都不进目录）—— 所以 description 写触发条件，不写功能清单。
- 资源用相对 skill root 的路径，一层深，别在 references 里再套引用链。
- 预算：metadata 约 100 token 常驻；正文建议 < 5000 token（< 500 行）；scripts/references 按需加载。

## 3. 加载机制（排查时按这个顺序）

1. **Discovery**：只解析 frontmatter，不读正文。
2. **Catalog**：每个 `agent/pre-step` 用会话 cwd 重新 snapshot，有变化就整段替换 `<available_skills>` 注入消息。
3. **Loading**：模型（或用户显式调用）命中 `name` 后，`skill` 工具返回 `<skill_content>` + `<skill_resources>`（含 base directory）+ 正文。

**watcher 只盯这几种变化**：SKILL.md 的增/删/改、skill 目录的增删、扁平 md 的增删。
**改 `references/`、`scripts/`、`assets/` 不会触发目录刷新** —— 文件本身是最新的，只是注入的 description 不会变。

## 4. skill 不出现的排查表

按顺序查，一条命中就停：

| 症状 | 查什么 |
|---|---|
| 完全不在列表 | `<root>/<name>/SKILL.md` 是否**正好一层**（多数是嵌套了） |
| 在列表但加载报 unknown | `name` 是否 kebab-case 且等于目录名 |
| 写了却消失 | frontmatter 是否有 `disableModelInvocation` 这类 camelCase 键或非布尔值 |
| 整个根都没反应 | 那个根目录是否存在（不存在时静默 0 个） |
| 项目根不对 | 当前 cwd 的最近 `.git` 祖先是谁，skill 得在它的 `.dsh/skills` 下 |

## 5. 插件（bundle）

### 最小结构

```json
// package.json
{
  "name": "dsh-my-plugin",
  "version": "1.0.0",
  "private": true,
  "type": "module",
  "exports": { ".": "./index.js", "./locale/*.json": "./locale/*.json", "./icon": "./icon.svg" },
  "dsh": { "bundle": { "patch": "./cordis.patch.yml" } }
}
```

```yaml
# cordis.patch.yml
- insert:
    - id: my-plugin
      name: 'dsh-my-plugin'
      config: {}
```

```js
// index.js —— 二选一，不要混
export function apply(ctx, config) {}   // 可带 export const inject = ['tools'] / export const Config
// 或者：一个 service class 作为 default export
```

一切资源在 `apply` 里用 `ctx.effect` / `ctx.on` 注册并返回 cleanup。

### 装进 profile（官方推荐顺序）

1. **`plugin_manager(action="install_bundle", target="<绝对包目录>")`** —— 它替你写 profile 的 `package.json`、`cordis.patch.yml`、跑 pnpm。
   **不要手写 profile 的 package.json / cordis.patch.yml，也不要在 profile 里跑 pnpm**（那是绕开管理器的做法，每次都要单独审批）。
2. 看返回的 `application` 和 `warnings` 判断是否真生效 —— 不是看日志、进程表或页面。
3. 开关用 `plugin_manager(action="set_plugin", target="include:<id>")`。

手动等价做法（管理器不可用时）：

- profile 在 `~/.dsh/profiles/desktop`；`package.json` 加 `"dsh-my-plugin": "link:D:/my-projects/dsh-my-plugin"`，再往 `dsh.profile.bundles` 末尾加一项（**只加依赖不加列表 = 永不加载**）
- `mklink /J "…\node_modules\dsh-my-plugin" "D:\my-projects\dsh-my-plugin"`

### patch 语法要点

- `insert: [rows]` 追加；带 `id` 且没有 `insert` 则**改现有行**。
- **`config` 是整段替换，不深合并** —— 覆盖时必须重述所有需要的键。
- 匹配不到的行只警告并跳过。
- 行字段：`id`、`name`（包说明符）、`config`、`disabled`、`inject`、`intercept`、`isolate`；`group: true` + `cordis:group` 让 `config` 变成嵌套行列表。
- `!!js` 是 Loader 表达式（不是 `!js`），在 `config` 里按插件上下文求值。
- 最完整的真实语法样本：dsh-base 的 `cordis.patch.yml`。

### 显示元数据（本地不上架也要做）

`locale/en.json` + `locale/zh.json`：

```json
{ "meta": { "title": "标题", "description": "能做什么", "detail": "docs/DETAIL.md" } }
```

- `exports` 要导出 `./locale/*.json` 和 `./icon`；`files` 里列全（链接安装不生效但上架时过滤）。
- icon 支持 SVG / PNG / JPEG / WebP，≤ 256 KiB，必须留在包内（绝对路径、URL、越界符号链接都被拒）。
- 缺字段会回退到 `package.json` 的 name/description，缺图用默认图 —— 不算错，但会在报告里被要求解释。

### 版本兼容

- **`engines.dsh` 宿主目前不读**。真正的闸门是 **peer 依赖**：0.2.0-rc.1 起 boot 时逐个 `semver.satisfies` 校验，任一不满足整包跳过，只留一行日志。
- 市场打包规范（`lib/index.js` + CommonJS + `patches/bundle.yaml`）**不是** 0.2.2-rc.x 的格式（ESM `apply()` + 根目录 `cordis.patch.yml`）—— 别照抄。

## 6. 坑（都踩过或社区实证）

| 坑 | 现象 | 解法 |
|---|---|---|
| skill 藏两层 | `skill` 报 unknown，重启也没用 | 只认 `<root>/<name>/SKILL.md`；bundle 里的要另挂 |
| name 不合法 | 面板显示已装，目录里没有 | 改成 kebab-case，且等于目录名 |
| 调用键 camelCase | 整条 skill 被丢弃 | 用 `disable-model-invocation` / `user-invocable` |
| 只加依赖不加 bundles | 插件永远不加载 | 两个都要，或直接用 `install_bundle` |
| 手写 profile | 每次改动都要审批且易错 | 用 `plugin_manager install_bundle` |
| 改了 references 以为会刷新 | catalog 里还是旧 description | 正常现象，文件已是最新 |
| PS 5.1 不支持 heredoc | `commit -F - <<EOF` 报错 | 多行文本先写成文件 |
| 内联中文被按 GBK 解析 | 报「字符串缺少终止符」 | 中文长命令改用 Python 脚本文件 |
| `Get-Content -Raw` 读 UTF-8 | 中文变「鎻掍欢」 | 用 read 工具或 Python |
| `.ps1` 含中文缺 BOM | PS 5.1 按 GBK 读，静默失败 | 存成 UTF-8 with BOM，或把中文挪进 `.mjs` |
| 改 projcache 标题 | 被前端写回 | 标题归前端，走界面重命名 |
| `dsh plugin remove` | 曾清空 bundles 列表 | 用 `plugin_manager` 的 remove_bundle |

## 7. 卸载

- **skill**：删掉 `~/.dsh/skills/<name>` 那条 junction（源码留在自己的项目目录，不动）
- **插件**：`plugin_manager(action="remove_bundle", target="<包名>")`，或删 bundles 项 + junction + 依赖
- 悬空引用排查：`plugin_manager(action="list_bundles")` 里 `installed:false` 就是 ghosts

## 8. 写 skill 的三条经验

1. **description 写触发条件**，把用户可能说的词（中英）都塞进去 —— 模型只有这 500 字符做路由。
2. **正文给命令和判据**，不写原理科普；「别做的事」单独一节，比「该做什么」省事。
3. **把坑写成表**：以后遇到同样症状，扫一眼就知道查哪。
