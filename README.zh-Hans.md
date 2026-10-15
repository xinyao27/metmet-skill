# MetMet Skill

本仓库收录 MetMet `metmet` 命令的 skill，并提供 Claude Code 与 Codex 插件。使用前需要在同一台 Mac 上安装 MetMet，并在它的设置里打开 **Let local agents use MetMet**。本地 ZIP 只是归档，不代表符合任何外部目录的收录要求。

安装：

```bash
npx skills add xinyao27/metmet-skill
```

插件：

```bash
claude plugin marketplace add xinyao27/metmet-skill
claude plugin install metmet@metmet
```

Codex：

```bash
codex plugin marketplace add xinyao27/metmet-skill
codex plugin add metmet@metmet
```
