# C# Bridge Notes

This bridge follows the repo-wide [CONTRIBUTING.md](../../CONTRIBUTING.md).

Bridge-specific work should keep the generated C# project idiomatic and split by concern:

- `src/Domain` for POCOs, shared enums, and the Domain csproj.
- `src/Application` for application services and permission helpers.
- `src/Infrastructure` for `DbContext`, EF Core configuration, and infrastructure-specific data.
- `docs` and seed files for developer-facing output.

## Bridge goals

- Keep entity classes lean and free of persistence-heavy logic.
- Move EF Core mapping into fluent configuration classes.
- Prefer standard EF Core conventions over custom migration templates.
- Keep bridge YAML aligned with the actual generated file tree.
- Make it easy to add future language-specific features without rewriting the core.

## Bridge checklist

- Verify the generated layout follows the target language’s conventions.
- Update templates when new YAML features are added.
- Ensure relationship and index output stays deterministic.
- Keep permission and seed output useful but lightweight.
- Avoid adding migration scaffolding; EF Core owns migrations.
