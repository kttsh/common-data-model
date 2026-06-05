# Task Completion

No automated gates (no linter/tests/type-checker). For a doc-editing task, "done" means:

1. **Owner principle respected** (see `mem:conventions`): the fact/decision body lives only in its owner doc; other docs carry summary + relative link, not duplicated substance.
2. **Glossary updated** (`docs/03-authorization/glossary.md`) if new terms/abbreviations were introduced.
3. **Links valid**: relative paths resolve; no new dangling references (a known one already exists — don't add more).
4. **改訂履歴 / revision-history table updated** in decision docs (02-architecture/, 03-authorization/) when a decision changed.
5. If memories were edited, run `serena memories check` from project root.