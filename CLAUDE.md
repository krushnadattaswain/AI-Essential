# CLAUDE.md - AI Assistant Guide for AI-Essential

This document provides guidance for AI assistants (Claude, Copilot, etc.) working on the AI-Essential project.

## Project Overview

**Name:** AI-Essential
**License:** GNU General Public License v3 (GPL-3.0)
**Status:** Initial development phase - skeleton project

AI-Essential is a new open-source project. The codebase is currently in its initial setup phase with no source code implemented yet.

## Current Repository Structure

```
AI-Essential/
├── .git/              # Git version control
├── LICENSE            # GNU GPL v3 license
├── README.md          # Project description (minimal)
└── CLAUDE.md          # This file - AI assistant guidance
```

## Development Status

This project is at the very beginning of its development lifecycle:

- **No source code** has been written yet
- **No technology stack** has been chosen
- **No build system** is configured
- **No tests** are present
- **No dependencies** are defined

## Getting Started with Development

When beginning development on this project, the following steps should be considered:

### 1. Technology Stack Selection

Before writing code, determine:
- Primary programming language (Python, TypeScript, Rust, etc.)
- Framework requirements (if any)
- Target platform (CLI, web, library, etc.)

### 2. Project Initialization

Based on the chosen stack, initialize with:
- Package manager configuration (`package.json`, `pyproject.toml`, `Cargo.toml`, etc.)
- TypeScript/language configuration if applicable
- Dependency management

### 3. Directory Structure

Establish a standard structure based on the technology:

```
# Example for Python project:
AI-Essential/
├── src/
│   └── ai_essential/
│       ├── __init__.py
│       └── core.py
├── tests/
│   └── test_core.py
├── pyproject.toml
├── README.md
└── LICENSE

# Example for Node.js/TypeScript project:
AI-Essential/
├── src/
│   └── index.ts
├── tests/
│   └── index.test.ts
├── package.json
├── tsconfig.json
├── README.md
└── LICENSE
```

### 4. Essential Configuration Files

Create these files early:
- `.gitignore` - Exclude build artifacts, dependencies, IDE files
- `.editorconfig` - Consistent coding style across editors
- Environment configuration (`.env.example` if needed)

## Coding Conventions

### General Guidelines

1. **Code Quality**
   - Write clean, readable, self-documenting code
   - Follow the language's official style guide
   - Keep functions small and focused
   - Use meaningful variable and function names

2. **Documentation**
   - Document public APIs and complex logic
   - Keep README.md updated with setup instructions
   - Include inline comments only when necessary

3. **Security**
   - Never commit secrets, API keys, or credentials
   - Validate all external inputs
   - Follow OWASP security guidelines

### Git Workflow

1. **Commits**
   - Write clear, descriptive commit messages
   - Use conventional commits format: `type(scope): description`
   - Types: `feat`, `fix`, `docs`, `style`, `refactor`, `test`, `chore`

2. **Branches**
   - `main` - Stable production code
   - `develop` - Integration branch for features
   - `feature/*` - New features
   - `fix/*` - Bug fixes
   - `claude/*` - AI assistant working branches

3. **Pull Requests**
   - Include clear description of changes
   - Reference related issues
   - Ensure all tests pass before merging

## License Compliance (GPL-3.0)

This project uses the GNU General Public License v3. When contributing:

1. **Source Code Availability**
   - All modifications must remain open source
   - Derived works must also be GPL-3.0 licensed

2. **Attribution**
   - Preserve copyright notices
   - Document significant changes

3. **Distribution**
   - Include license text with any distribution
   - Provide access to complete source code

## AI Assistant Instructions

### When Working on This Project

1. **Read First**
   - Always read existing code before making changes
   - Understand the project structure and conventions
   - Check for existing implementations before creating new ones

2. **Minimal Changes**
   - Make only the changes requested
   - Avoid over-engineering or unnecessary abstractions
   - Don't add features beyond what's asked

3. **Testing**
   - Write tests for new functionality when a test framework exists
   - Run existing tests before committing
   - Don't break existing functionality

4. **Documentation**
   - Update this CLAUDE.md when project structure changes significantly
   - Keep README.md current with setup instructions
   - Document breaking changes

### Commands Reference

Since no build system is configured yet, commands will be added as the project develops.

```bash
# Placeholder - update when technology stack is chosen

# Install dependencies
# [command here]

# Run tests
# [command here]

# Build project
# [command here]

# Run development server
# [command here]
```

## Project Roadmap

### Phase 1: Foundation (Current)
- [ ] Define project purpose and scope
- [ ] Choose technology stack
- [ ] Initialize build system
- [ ] Create basic directory structure
- [ ] Set up development environment

### Phase 2: Core Development
- [ ] Implement core functionality
- [ ] Add unit tests
- [ ] Set up CI/CD pipeline
- [ ] Create initial documentation

### Phase 3: Enhancement
- [ ] Add advanced features
- [ ] Performance optimization
- [ ] Comprehensive testing
- [ ] User documentation

## Contributing

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Write/update tests
5. Submit a pull request

## Contact

**Maintainer:** krushnadattaswain
**Repository:** AI-Essential

---

*Last updated: 2026-02-04*
*Document version: 1.0.0*
