# OpenClaw Medical Skills — Architecture Decision Records (ADR)

> **Document Purpose**: Record significant architectural decisions and their rationale, following the practice of Architecture Decision Records (ADRs).

---

## ADR-001: Markdown-Based Skill Format

**Status**: ✅ Accepted
**Date**: Project Inception
**Context**: Need a format for AI agent skills that is human-readable, version-controllable, and easy to author.

### Decision
Use Markdown files with YAML frontmatter as the skill format.

### Consequences

**Positive**:
- Human-readable and writable
- Git-friendly (diffable, mergeable)
- Rich formatting support (code blocks, tables, lists)
- Familiar to developers and researchers
- Supports embedded code examples

**Negative**:
- Requires YAML parsing
- No built-in type safety (enforced via validation)
- Larger file size than JSON for machine consumption

### Alternatives Considered
- JSON: Rejected — not human-friendly for large content
- Python modules: Rejected — too complex for simple skills
- XML: Rejected — verbose and dated
- Custom format: Rejected — unnecessary invention

---

## ADR-002: Kebab-Case Naming Convention

**Status**: ✅ Accepted
**Date**: Project Inception
**Context**: Need consistent naming for 869+ skills.

### Decision
Enforce kebab-case (lowercase with hyphens) for all skill names.

### Specification
```regex
^[a-z0-9]+(-[a-z0-9]+)*$
```

### Consequences

**Positive**:
- URL-safe (no encoding needed)
- Command-line friendly (no escaping needed)
- Consistent visual appearance
- Easy to validate

**Negative**:
- Long names can be verbose
- No camelCase for multi-word concepts

### Alternatives Considered
- snake_case: Rejected — less URL-friendly
- camelCase: Rejected — harder to read in URLs
- PascalCase: Rejected — same issues as camelCase

---

## ADR-003: Validation via Python Script

**Status**: ✅ Accepted
**Date**: Project Inception
**Context**: Need to ensure 869+ skills follow the format specification.

### Decision
Create a standalone Python validation script (`validate_skill.py`).

### Design
- Single file, minimal dependencies (PyYAML)
- Validates YAML frontmatter structure
- Checks naming conventions
- Returns exit codes for CI/CD integration

### Consequences

**Positive**:
- Fast execution
- Clear error messages
- CI/CD friendly
- Easy to extend

**Negative**:
- Requires Python runtime
- Manual dependency management (PyYAML)

### Alternatives Considered
- JSON Schema: Rejected — less flexible for custom validations
- TypeScript: Rejected — adds Node.js dependency
- Built-in to OpenClaw: Rejected — skills should be self-validating

---

## ADR-004: GitHub Actions for CI/CD

**Status**: ✅ Accepted
**Date**: Project Inception
**Context**: Need automated validation on commits and PRs.

### Decision
Use GitHub Actions with diff-based validation.

### Design
- Trigger on changes to `skills/*/SKILL.md`
- Use git diff to find changed skills only
- Validate only modified skills (not all 869)
- Support partial deletion detection

### Consequences

**Positive**:
- Fast validation (only changed skills)
- Free for open source
- Integrated with GitHub workflow
- Supports PR reviews

**Negative**:
- GitHub-specific (vendor lock-in)
- Requires understanding of git diff output

### Alternatives Considered
- Pre-commit hooks: Rejected — client-side only
- Jenkins: Rejected — requires infrastructure
- GitLab CI: Rejected — GitHub is primary platform

---

## ADR-005: No Centralized Skill Registry

**Status**: ✅ Accepted
**Date**: Project Inception
**Context**: Need to discover and load skills.

### Decision
Use filesystem-based discovery (directory traversal) instead of a centralized registry.

### Design
- Skills loaded from `skills/{name}/SKILL.md`
- Name derived from directory name
- No central index file to maintain

### Consequences

**Positive**:
- No coordination needed for adding skills
- Simple to understand
- Works with standard file operations
- Version control friendly

**Negative**:
- Slower discovery (filesystem traversal)
- No fast lookup by capability
- Potential naming collisions

### Alternatives Considered
- Central JSON registry: Rejected — requires updates on every skill addition
- Database: Rejected — overkill for static content
- Generated index: Rejected — adds build step

---

## ADR-006: Optional Extended Frontmatter Fields

**Status**: ✅ Accepted
**Date**: During growth to 869 skills
**Context**: Different skills need different metadata (license, category, tags).

### Decision
Allow optional frontmatter fields beyond required `name` and `description`.

### Design
```yaml
---
name: required-string
description: required-string
tool_type: optional-string
license: optional-string
category: optional-string
tags: optional-list
allowed-tools: optional-string
biomodals_script: optional-string
---
```

### Consequences

**Positive**:
- Flexible for different skill types
- Backward compatible
- Skills can self-declare capabilities

**Negative**:
- Less strict schema
- Potential for inconsistent metadata
- Requires documentation of optional fields

### Alternatives Considered
- Strict schema: Rejected — too limiting
- Separate metadata files: Rejected — fragmentation

---

## ADR-007: Co-located Examples and References

**Status**: ✅ Accepted
**Date**: During bioinformatics skill expansion
**Context**: Skills need code examples and extended documentation.

### Decision
Allow skills to include `examples/` and `references/` subdirectories.

### Design
```
skills/{name}/
├── SKILL.md              # Interface
├── examples/             # Implementation code
│   ├── example1.py
│   └── example2.sh
└── references/           # Extended docs
    └── detailed_guide.md
```

### Consequences

**Positive**:
- Self-contained skills
- Examples versioned with skill
- Clear separation of interface and implementation

**Negative**:
- Larger repository size
- More files to review
- Potential for skill bloat

### Alternatives Considered
- Separate examples repo: Rejected — harder to keep in sync
- Inline examples only: Rejected — insufficient for complex workflows

---

## ADR-008: Skill Independence (No Dependencies)

**Status**: ✅ Accepted
**Date**: Project Inception
**Context**: Need to prevent complex dependency chains.

### Decision
Skills are independent — no explicit dependency declarations between skills.

### Design
- Each skill is self-contained
- No "import" or "require" mechanism between skills
- Agent combines skills at runtime

### Consequences

**Positive**:
- No dependency resolution complexity
- Skills can be added/removed freely
- No version conflicts between skills
- Easier to reason about individual skills

**Negative**:
- Cannot build composite skills from smaller ones
- Potential for duplicated patterns
- Agent must manage skill combinations

### Alternatives Considered
- Skill dependencies: Rejected — adds complexity
- Skill inheritance: Rejected — complicates simple format

---

## ADR-009: Validation Before Runtime

**Status**: ✅ Accepted
**Date**: Project Inception
**Context**: Need to catch skill errors early.

### Decision
Validate skills at commit time (CI/CD), not at runtime.

### Design
- CI/CD validates changed skills
- Runtime assumes skills are valid
- No runtime validation overhead

### Consequences

**Positive**:
- Fast runtime performance
- Errors caught before deployment
- Clear separation of concerns

**Negative**:
- Requires CI/CD setup
- Local changes may pass but fail in CI
- No runtime safety net

### Alternatives Considered
- Runtime validation: Rejected — adds overhead
- Both: Rejected — redundant if CI/CD is reliable

---

## ADR-010: Minimal Required Frontmatter

**Status**: ✅ Accepted
**Date**: After reviewing 869 skills
**Context**: Need to balance flexibility with consistency.

### Decision
Only `name` and `description` are required in frontmatter.

### Rationale
- Every skill must have an identifier (`name`)
- Every skill must explain its purpose (`description`)
- Everything else is optional context

### Consequences

**Positive**:
- Low barrier to entry for new skills
- Skills can evolve to add metadata
- Simple validation logic

**Negative**:
- Less structured information
- Agent must handle missing optional fields

### Alternatives Considered
- Require more fields: Rejected — discourages contribution
- Require category/tags: Rejected — hard to categorize some skills

---

## ADR-011: Documentation-First Design

**Status**: ✅ Accepted
**Date**: Project Inception
**Context**: Skills must be understandable by AI agents and humans.

### Decision
The SKILL.md IS the implementation — documentation is not separate.

### Design
- Frontmatter declares capabilities
- Markdown explains usage
- Examples demonstrate correct patterns
- No separate "implementation" to maintain

### Consequences

**Positive**:
- Single source of truth
- Documentation always up to date
- Natural place for examples

**Negative**:
- Large files for complex skills
- Mix of metadata and documentation
- Parsing required to extract frontmatter

### Alternatives Considered
- Separate doc files: Rejected — duplication risk
- Code-only: Rejected — hard for AI to understand

---

## ADR-012: Category-Based Organization

**Status**: ✅ Accepted
**Date**: After reaching 500+ skills
**Context**: Need to organize 869 skills for discoverability.

### Decision
Use 9 major categories for documentation organization (not filesystem).

### Categories
1. General & Core (10 skills)
2. Medical & Clinical (119 skills)
3. Scientific Databases (43 skills)
4. Bioinformatics (239 skills)
5. Omics & Computational Biology (59 skills)
6. ClawBio Pipelines (21 skills)
7. BioOS Extended Suite (285 skills)
8. Data Science & Tools (93 skills)

### Design
- Categories are documentation-only
- Filesystem is flat (`skills/{name}/`)
- Categories listed in README.md

### Consequences

**Positive**:
- Skills can belong to multiple conceptual categories
- No filesystem restructuring needed
- Flexible categorization

**Negative**:
- Categories may become outdated
- No filesystem organization
- Manual categorization effort

### Alternatives Considered
- Filesystem categories: Rejected — complicates paths
- Tag-based: Rejected — less structured than categories

---

## ADR-013: No Versioning in Skill Names

**Status**: ✅ Accepted
**Date**: Project Inception
**Context**: Need to handle skill evolution.

### Decision
Skill names are versionless; versions managed via Git history.

### Design
- `alphafold` not `alphafold-v2`
- Updates modify existing skill
- Breaking changes handled via agent compatibility

### Consequences

**Positive**:
- Simple naming
- No version conflicts
- Git provides history

**Negative**:
- Breaking changes affect all users
- No parallel version support
- Agent must handle compatibility

### Alternatives Considered
- Version in name: Rejected — complicates discovery
- Semantic versioning: Rejected — overkill for documentation

---

## ADR-014: Allow Multiple Implementation Languages

**Status**: ✅ Accepted
**Date**: During bioinformatics expansion
**Context**: Different domains prefer different languages.

### Decision
Skills can include examples in Python, Bash, JavaScript, or any language.

### Design
- `tool_type` frontmatter hints at primary language
- Examples directory can contain mixed languages
- No restriction on implementation language

### Consequences

**Positive**:
- Domain-appropriate languages (Python for bioinformatics)
- Flexibility for different use cases
- No artificial constraints

**Negative**:
- Inconsistent language usage
- Harder to provide IDE support
- Validation harder across languages

### Alternatives Considered
- Python only: Rejected — excludes bioinformatics tools
- JavaScript only: Rejected — excludes scientific computing

---

## Summary Table

| ADR | Decision | Status | Impact |
|-----|----------|--------|--------|
| 001 | Markdown + YAML format | ✅ Accepted | Core format |
| 002 | Kebab-case naming | ✅ Accepted | Consistency |
| 003 | Python validation | ✅ Accepted | Quality |
| 004 | GitHub Actions CI | ✅ Accepted | Automation |
| 005 | Filesystem discovery | ✅ Accepted | Simplicity |
| 006 | Optional frontmatter | ✅ Accepted | Flexibility |
| 007 | Co-located examples | ✅ Accepted | Completeness |
| 008 | Skill independence | ✅ Accepted | Decoupling |
| 009 | Validation before runtime | ✅ Accepted | Performance |
| 010 | Minimal frontmatter | ✅ Accepted | Accessibility |
| 011 | Documentation-first | ✅ Accepted | Maintainability |
| 012 | Category organization | ✅ Accepted | Discoverability |
| 013 | Versionless names | ✅ Accepted | Simplicity |
| 014 | Multi-language examples | ✅ Accepted | Flexibility |

---

*Architecture decisions recorded per project standards.*
*Generated: 2026-03-17*
