# Contributing

Thanks for helping improve this AppSec hardening checklist.

## What belongs here

- Defensive hardening guidance, secure defaults, and review checklists
- Clear “why it matters” notes tied to real failure modes
- Links to vendor/docs standards (OWASP ASVS, CIS, NIST, vendor hardening guides)

## What does not belong here

- Exploit code, attack payloads, bypass recipes, or offensive tooling
- Instructions that enable unauthorized access or abuse
- Unverified claims without a cited standard or rationale

## How to contribute

1. Fork the repository and create a branch: `git checkout -b docs/improve-apis-checklist`
2. Edit only the relevant checklist under `checklists/` or docs under `docs/` / `examples/`
3. Keep items as Markdown checkboxes: `- [ ] **Title**: short guidance`
4. Include a short **Why it matters** and **Secure default** where practical
5. Run markdown lint locally if you can (`npx markdownlint-cli2 "**/*.md"`)
6. Open a pull request with a clear summary of what changed and why

## Style

- Prefer actionable, reviewable items over vague advice
- Use inclusive, professional language
- Prefer present tense and short sentences
- Do not paste large copyrighted standards verbatim, summarize and link

## License

By contributing, you agree your contributions are licensed under the MIT License.
