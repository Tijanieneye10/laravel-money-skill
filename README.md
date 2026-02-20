# PHP(Laravel) Money Skill for Claude

An AI coding rule that enforces safe monetary calculations in Laravel using the [Brick\Math](https://github.com/brick/math) and [Brick\Money](https://github.com/brick/money) packages. Works with [Claude Code](https://docs.anthropic.com), [Cursor](https://cursor.com), and Claude Projects.

## Why?

PHP's native floating point arithmetic causes rounding errors that are unacceptable in financial contexts. This skill instructs Claude to **always** use arbitrary-precision math when writing or editing any Laravel code that deals with money.

## What It Does

When this skill is active, Claude will:

- **Never** use floats or native arithmetic (`+`, `-`, `*`, `/`) for monetary values
- Use `brick/money` for currency-aware calculations or `brick/math` for raw precision
- Apply proper rounding modes and explicit scale on all operations
- Serialize money as strings in JSON responses to preserve precision
- Use Laravel's modern `Attribute::make()` accessor/mutator syntax
- Follow safe patterns for database storage, validation, formatting, and testing

## Installation

### Claude Code

Copy the `SKILL.md` file into your project's `.claude/skills/` directory:

```bash
mkdir -p .claude/skills/laravel-money
cp SKILL.md .claude/skills/laravel-money/SKILL.md
```

### Cursor

Add the `SKILL.md` content to your project's `.cursorrules` file, or copy it into `.cursor/rules/`:

```bash
mkdir -p .cursor/rules
cp SKILL.md .cursor/rules/laravel-money.md
```

### Claude Projects

Paste the contents of `SKILL.md` into your project's custom instructions or knowledge base.

## Customization

The skill is designed to be generic. You may want to adjust:

- **Rounding mode** — currently uses `HALF_DOWN`. Change to `HALF_UP` or `HALF_EVEN` based on your requirements.
- **Storage strategy** — the skill covers string, decimal, and integer (cents) storage. Remove the strategies that don't apply to your project.
- **Crypto section** — included as optional. Remove it if you don't work with cryptocurrency.

## Contributing

PRs welcome! If you find patterns that improve how Claude handles money in Laravel, open an issue or submit a pull request.

## License

MIT
