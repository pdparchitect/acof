# Skills

A skill is a `SKILL.md` playbook - methodology, or a vulnerability-class hunting
guide - that the agent reads when it decides the skill is relevant. Acof does
**not** compile a skill library into the binary. The only embedded skill is a
**catalog**: an index that tells the agent where external skill collections live
and how to install them. Everything else is fetched on demand.

**Where skills live.** Acof loads skills from `~/.config/acof/skills`
(`$XDG_CONFIG_HOME/acof/skills`), each as `<skill-name>/SKILL.md`. It rescans
that directory every turn, so a skill added while a run is going - including one
the agent clones itself - is available on the next step, no restart. A skill on
disk overrides an embedded one of the same name.

**How the agent gets them.** The catalog names a public collection and the
agent, using its `shell` tool, clones it into the skills directory - for
example [Claude-BugHunter](https://github.com/elementalsouls/Claude-BugHunter),
a broad library of bug-bounty methodology and per-vulnerability-class playbooks
(SQLi, XSS, SSRF, IDOR, OAuth, SAML, GraphQL, cloud, business logic, and more).
Nothing is baked into the binary, so the collection can grow and stay current
without a acof release, and Acof carries no third-party skill content or license
of its own.

For an air-gapped or locked-down box, drop a skills directory into
`~/.config/acof/skills` ahead of time and Acof runs with no fetch at all - the
skills are just files it reads.

**Adding your own skill.** Drop `<name>/SKILL.md` into `~/.config/acof/skills`:

```markdown
---
name: My Skill
description: One sentence the model uses to decide when to apply this skill.
---

# My Skill

Step-by-step guidance...
```

No rebuild - it is picked up on the next turn. A private methodology skill, or
an override of a collection's shipped one, is just a file in that directory.

## Skill collections

Acof bundles no third-party skill content. The catalog points at external
collections you fetch on demand; the default one is **claude-bughunter** by
**[Sachin Sharma](https://www.linkedin.com/in/sachinsharma8080/)**:

> https://github.com/elementalsouls/Claude-BugHunter

That collection is MIT-licensed and keeps its own license when you clone it -
Acof neither redistributes nor relicenses it. Our thanks to the author and the
bug-bounty community whose disclosed reports informed it.
