# Contributing

This repository is intended to stay simple and easy to install.

## Add a skill

1. Create a new directory using lowercase kebab-case:

   ```txt
   skill-name/
   ```

2. Add the required entrypoint:

   ```txt
   skill-name/SKILL.md
   ```

3. Use optional folders only when they add value:

   ```txt
   skill-name/references/
   skill-name/scripts/
   skill-name/assets/
   ```

4. Update the root `README.md` skill table.
5. Update `skills.sh.json` if the skill should be grouped for discovery.
6. Verify discovery before publishing:

   ```bash
   npx skills add IdrisMalik/skills --list
   ```

## Skill standards

- Keep `SKILL.md` focused on when to use the skill and how the agent should behave.
- Keep long reference material out of `SKILL.md` and inside `references/`.
- Avoid broad, vague, or promotional descriptions.
- Do not include secrets, live credentials, tokens, private URLs, or customer data.
- Prefer examples and checklists over long conceptual prose.
- Keep scripts deterministic and document their inputs.
- Avoid adding workflows, build systems, or dependencies unless multiple skills need them.
