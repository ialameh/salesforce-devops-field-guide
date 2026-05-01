# Contributing

Thanks for considering a contribution. The Salesforce DevOps and Packaging Field Guide is built and maintained by people who hit these problems in production and wrote down what they learned. Every contribution that helps the next person is welcome.

## What contributions are welcome

- Fixes: typos, broken links, outdated commands, factual errors.
- New case studies: real incidents you have run into, anonymised.
- New cookbook examples: worked-out patterns that fill a gap.
- Diagram improvements: clearer SVGs, additional perspectives.
- Updates for newer Salesforce releases: when something the guide covers changes.

## What we ask you to keep

- **Tone.** Plain prose, conversational, technically precise.
- **No em dashes.** Use commas, parentheses, or sentence breaks.
- **No marketing language.** This is a working document.
- **Concrete over abstract.** Examples beat principles. Code beats prose.
- **Generic, not project-specific.** Anonymise project names, org names, customer names.

## How to contribute

1. Open an issue describing what you want to change. For typos and small fixes, you can skip this and go straight to a PR.
2. Fork the repository.
3. Create a branch.
4. Make your change.
5. Open a pull request.

## Conventions

### Markdown

- Plain markdown. No HTML unless absolutely necessary.
- Tables for reference data. SVGs for conceptual diagrams.
- Code blocks with language tags.
- Sentence-case headings.

### File names

- Chapters use `NN-name-with-dashes.md`.
- Diagrams use `descriptive-name.svg`, lowercase, dashes.

### Diagrams

- SVG, hand-authored or generated cleanly.
- Embed via `![alt text](diagrams/name.svg)`.
- Match the existing style: same fonts, same color palette.
- No embedded fonts. Use system fonts.

### Code samples

- Apex samples should be deployable as-is, with placeholders in `<ANGLE_BRACKETS>`.
- Bash commands should work on macOS or Linux.
- `sf` commands target the latest stable Salesforce CLI.

### Tone

Direct, brief, no padding. Compare:

- "It is generally recommended that practitioners consider the implications..."
- "Validate before you deploy to production."

The second is the tone we are looking for.

## Adding a new case study

Structure:

1. **Context.** What was the team building? What environment?
2. **Symptoms.** What did they see?
3. **Initial hypothesis.** What did they think was wrong before investigating?
4. **What was actually wrong.** With diagnosis.
5. **What they did.** Step by step.
6. **What they learned.** Takeaways.

Anonymise everything. No real org names, customer names, employee names, or project codenames.

## Reviewing pull requests

Maintainers will review:

1. **Accuracy.** Is what you wrote technically correct?
2. **Tone.** Does it match the rest of the guide?
3. **Scope.** Is it generic enough to be useful to other readers?
4. **Sources.** If you reference a Salesforce feature, can someone find it in the official docs?

## Code of Conduct

Be kind. Disagree about content, not about people. Follow the [Code of Conduct](./CODE_OF_CONDUCT.md).

## Licensing

By contributing, you agree your contributions will be licensed under the same Creative Commons Attribution 4.0 license as the rest of the guide.

## Getting help

- Open an issue for questions.
- Tag a maintainer in a PR if you need a faster response.
- For broader Salesforce questions, the Trailblazer Community is a better venue.

Thanks again. This guide gets better with every pair of fresh eyes.
