# MetMet skill

This repository packages the skill for MetMet's `metmet` command as a plugin for Claude Code and Codex. It needs the MetMet app on the same Mac with **Let local agents use MetMet** turned on in its Settings. The local ZIP is only an archive and does not imply eligibility for any external directory.

## Install

The installer lets you choose agents and scope:

```bash
npx skills add xinyao27/metmet-skill
```

Claude Code plugin:

```bash
claude plugin marketplace add xinyao27/metmet-skill
claude plugin install metmet@metmet
```

Codex plugin:

```bash
codex plugin marketplace add xinyao27/metmet-skill
codex plugin add metmet@metmet
```

Source: maintained in the MetMet app. Generated with pnpm publish:metmet-skill.
