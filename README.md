# dsh-soda-authoring

一个 **skill 包**：把「怎么给 DeepSeek Harness 写 skill 和插件」写成模型能直接照着做的操作手册。

它不提供工具、不监听事件 —— Host 侧的 `apply()` 是空的，作用全在 `skills/dsh-soda-authoring/SKILL.md` 里。

## 它解决什么

DSH 的插件机制坑很多，而且**多数坑不报错、只静默失败**：

- skill 放深一层（`a/b/SKILL.md`）→ 永远不加载，面板却可能显示「已装」
- frontmatter 的调用键写成 camelCase → **整条 skill 被丢弃**，只留一行日志
- `name` 和目录名不一致 → `skill` 工具报 unknown
- profile 里只加依赖、忘了往 `dsh.profile.bundles` 加一项 → 插件永远不加载
- 手写 profile 的 `package.json` / `cordis.patch.yml` → 每次改动都要审批，还容易写错

这份手册把这些整理成：30 秒判断（skill 还是插件）、六个 skill 根目录的优先级、frontmatter 校验规则、装进 profile 的正确顺序、patch 语法要点、12 个真实坑的「症状 → 查什么」。

## 内容

```
SKILL.md   10 KB，八节：判断 / 放哪 / 怎么写 / 加载机制 / 排查表 / 插件结构 / 坑表 / 三条经验
```

## 安装

```bash
dsh plugin --profile <profile> add dsh-soda-authoring
```

或者直接把 `skills/dsh-soda-authoring/` 拷进 `~/.dsh/skills/`（skill 是热重载的，不用重启）。

> skill 目录必须是 `<root>/<name>/SKILL.md` 正好一层。嵌在 bundle 里的 `skills/<name>/SKILL.md` 会被插件装好，但**skill 根目录只扫一层**，所以从包内安装后仍需确认它出现在 `available_skills` 里。

## License

MIT
