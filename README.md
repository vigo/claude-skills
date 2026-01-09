![Version](https://img.shields.io/badge/version-0.0.0-orange.svg?style=for-the-badge)

# Claude-Code Skills

What Skills are available?

1. **explaining-code** - ~104 tokens - (grabbed from https://code.claude.com/docs/en/skills)
1. **code-like-gopher** - ~5.0k tokens
1. **code-like-djangonout** - ~5.9k tokens

---

## Installation

Clone the repo:

```bash
cd ~/.claude/
git clone git@github.com:vigo/claude-skills.git skills
```

Restart your claude; `/skills`

How to invoke:

1. Slash command: `/code-like-gopher`
1. Ask directly: "Write go code" or "refactor this go code"
1. Ask explicitly: "Write this function using a code-like-gopher skill"

---

## Contributor(s)

* [Uğur "vigo" Özyılmazel](https://github.com/vigo) - Creator, maintainer

---

## Contribute

All PR’s are welcome!

1. `fork` (https://github.com/vigo/claude-skills/fork)
1. Create your `branch` (`git checkout -b my-branch`)
1. `commit` yours (`git commit -am 'add XXX skill'`)
1. `push` your `branch` (`git push origin my-branch`)
1. Than create a new **Pull Request**!

This project is intended to be a safe, welcoming space for collaboration, and
contributors are expected to adhere to the [code of conduct][coc].

---

## License

This project is licensed under MIT

---

[coc]: https://github.com/vigo/claude-skills/blob/main/CODE_OF_CONDUCT.md
