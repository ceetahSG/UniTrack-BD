# UniTrack BD Development Workflow

## Branch Strategy

main
- Production-ready code only

feature/frontend
- Frontend development

feature/backend-auth
- Authentication module

feature/backend-payment
- Wallet & payment module

feature/backend-api
- General backend APIs

feature/testing
- Testing related work

---

## Rules

1. Never push directly to main.
2. Create a feature branch before starting work.
3. Commit regularly.
4. Open a Pull Request before merging.
5. Update Trello when task status changes.
6. Inform the team if blocked for more than 24 hours.

---

## Commit Message Format

feat: add login screen

fix: resolve authentication bug

docs: update SRS

refactor: improve API structure

test: add login test cases