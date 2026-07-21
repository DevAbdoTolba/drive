# Instructions for Codex

## Working style

- Read MVP1.md before changing anything.
- Begin by inspecting the repository and writing a short implementation plan.
- Make small, reviewable changes.
- Explain important architectural decisions.
- Never place real secrets in tracked files.
- Never execute destructive commands without explicit permission.
- Do not expand the scope beyond MVP1.

## Validation

After making changes:

1. Validate Docker Compose configuration.
2. Check configuration files for syntax errors.
3. Confirm all referenced environment variables exist in `.env.example`.
4. Review exposed ports and storage paths.
5. Update README.md with exact commands.

## Platform

The primary development machine runs Windows 11 using Docker Desktop and WSL2.
Commands and paths must work in that environment.