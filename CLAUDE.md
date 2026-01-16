# Tolgee Platform Development Conventions

---

## Code Documentation Principle

**Goal:** Save readers brain power and time when trying to understand how things fit together.

All code should be documented if it would not be immediately evident to a first-time reader what it is and how it fits with the rest of the codebase. Assume readers may have no prior knowledge of the domain.

### Core Principle

The purpose of documentation is to help readers build a mental map of how things work together. This means:

- **Guide the reader**: Always make it clear where to start and how to navigate
- **Explain the "why"**: Not just what the code does, but why it exists and why it's structured this way
- **Link liberally**: Connect related pieces so readers can explore the full picture
- **Summarize external concepts**: Don't assume familiarity with external systems, protocols, or libraries

### Documentation Hierarchy

Create a clear hierarchy that guides readers through the codebase:

1. **Entry point documentation**: Each significant module/feature should have a central location (file-level KDoc, package-info, or dedicated doc file) that provides:
   - High-level overview: what this does and why it exists
   - How it fits into the broader system
   - "Start here" guidance for newcomers
   - Links to detailed documentation for sub-components

2. **Component documentation**: Classes, functions, and files should:
   - Reference their entry point ("This is part of the tracing system, see OpenTelemetryConfiguration for overview")
   - Explain their specific role
   - Document non-obvious behavior

3. **Inline documentation**: For specific lines or blocks:
   - Explain complex logic
   - Document magic values, extracting to named constants where appropriate
   - Link to external documentation for library/framework-specific code

4. **External references**: When code uses external concepts (protocols, libraries, APIs):
   - Include 1-2 sentence summary of what the external thing is
   - Link to authoritative external documentation
   - Explain how it applies in this context

### What Specifically to Document

- **Non-obvious code**: Anything where the intent isn't clear from reading the code
- **External integrations**: What the external system is, how we interact with it, relevant docs
- **Configuration**: What each property controls, valid values, why defaults are chosen
- **Magic values**: Extract to named constants with documentation explaining meaning and linking to source
- **Architectural decisions**: Why this approach was chosen over alternatives
- **Error handling**: What can go wrong and how it's handled

### Documentation as a Separate Step

When implementing features:
1. Write the code first
2. **Explicitly plan documentation** as a separate step:
   - Map out what needs to be documented
   - Design the hierarchy (where should readers start?)
   - Identify external concepts that need explanation
3. Implement the documentation
4. Verify a newcomer could follow the trail from entry point to details
