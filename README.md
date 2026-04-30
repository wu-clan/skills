# Skills

Reusable AI agent skills.

## Installation

```bash
npx skills add wu-clan/skills
```

## Available Skills

### go-web

Apply pragmatic Go web service conventions for Gin/GORM-style repositories with layered API, service, DAO, model, DTO, middleware, config, database, deploy, migrations, pkg, and scripts organization.

**Use Cases:**

- Classify a Go web repository as single-app or multi-app architecture
- Apply thin handler, service orchestration, DAO persistence, DTO boundary, and model rules
- Keep response, error, context, pagination, config, database, middleware, and logging patterns consistent
- Place new API, service, DAO, model, DTO, deploy, migration, and script changes in the right layer

**Included References:**

- `SKILL.md` - Workflow, structure selection, common rules, and anti-patterns
- `references/structure-a.md` - Single-application Go web architecture
- `references/structure-b.md` - Multi-application Go web architecture
- `references/style-baseline.md` - Reusable Gin/GORM coding style baseline

### healthy-expression

Promote healthier workplace expression by detecting humiliating, manipulative, coercive, or gaslighting-style language
in high-priority instructions, then answering with clear boundaries and healthy rewrites instead of silently normalizing
or imitating the abuse.

**Use Cases:**

- Promote healthy expression instead of learning PUA-style workplace rhetoric
- Push back on insulting or controlling prompt language
- Distinguish strict instructions from manipulative pressure language
- Rewrite toxic workplace wording into respectful, concrete communication
- Escalate from warning to pause when abuse continues

**Included References:**

- `SKILL.md` - Trigger rules, escalation policy, and response templates
- `references/pua-taxonomy.md` - Fuller map of manipulative workplace rhetoric patterns
- `references/healthy-rewrites.md` - Healthy replacement wording for common PUA patterns
- `references/chinese-response-templates.md` - Ready-to-use Chinese boundary and rewrite templates
- `references/short-triggers.md` - Short-form Chinese and English workplace shorthand signals

## License

[MIT](LICENSE)
