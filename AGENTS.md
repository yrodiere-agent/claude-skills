# Writing skills

Brevity is paramount. Use the fewest words that preserve actionable guidance.

- Include only instructions that change an agent's decisions. Remove repetition,
  obvious advice, background essays, and speculative edge cases.
- Keep each skill focused. Its description must say what it does and when to
  load it; avoid broad triggers that pull it into unrelated tasks.
- Put essential rules first. Use direct verbs and concrete commands; include
  examples only when they clarify a non-obvious requirement.
- Keep `SKILL.md` lean. Move substantial, conditional detail to linked references
  and say when to read them. Do not add files merely to hide verbosity.
- Give agents discretion for routine choices. Prescribe exact steps for fragile
  operations; do not turn one incident into a universal restriction or infer
  authorization for external actions.
- Follow the Agent Skills format: YAML frontmatter with `name` and `description`.
  Preserve existing metadata and use relative links to bundled resources.
- Before submitting, cut unnecessary text, check frontmatter and links, and run
  changed scripts. For workflow changes, check behavior on a representative task;
  validating syntax alone does not establish usefulness.

Sources: [Agent Skills specification](https://agentskills.io/specification)
and [Anthropic's skill-authoring guidance](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills).
