## Framework Notes

This repository collects concise, practical notes and examples for web frameworks. Use these notes as quick references, learning aids, and implementation examples.

Table of Contents
- Overview
- Repository structure
- How to use the notes
- Adding new framework notes
- Naming conventions
- Contributing
- License & contact

Overview
--------
Each framework gets its own folder containing a framework-level README and topic files (guides, examples, references). Files contain short explanations, code snippets, and step-by-step examples.

-Repository structure
---------------------
All notes are single Markdown files (no subfolders). Name files after the concept or method they describe so they're easy to find and scan.

Examples:

- dependency-injection.md       # Concept + short examples
- controllers.md                # Controllers and routing patterns
- testing.md                    # Testing tips and minimal examples
- nest-js-overview.md           # Framework-specific overview (if desired)

How to use the notes
--------------------
- Read the framework README for a high-level orientation.
- Open a guide for step-by-step instructions.
- Use code under `examples/` for runnable snippets and quick experimentation.

Adding new framework notes
--------------------------
1. Create a new folder named after the framework (kebab-case). Example: `express-js/` or `next-js/`.
2. Add a `<concept-name>.md` file with description, examples & links etc.

Naming conventions
------------------
- Folder names: `kebab-case` (e.g., `nest-js`).
- File names: `short-descriptive-title.md` (e.g., `dependency-injection.md`).
- Use `README.md` at the root of each framework folder for quick navigation.

Contributing
------------
Contributions are welcome. When adding or editing notes:
- Keep entries concise and focused.
- Include example code and minimal reproduction steps when applicable.
- Link to relevant external docs or sources.

