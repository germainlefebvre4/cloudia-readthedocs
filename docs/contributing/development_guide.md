# Development Guidelines

## Commit convention

Every commit message should follow the [Conventional Commits](https://www.conventionalcommits.org/) specification.

## Branch strategy

Branch strategy is based on the simplest Git workflow:

* `main` branch is the main branch of the project
* `feature` branches are used to develop new features
* `fix` branches are used to fix bugs
* `chore` branches are used to perform maintenance tasks

Git branch strategy graph:

```mermaid
---
title: Example Git diagram
---
gitGraph
   commit
   commit tag: "v1.0.0"
   commit
   branch feat/my-feature
   checkout feat/my-feature
   commit
   commit
   checkout main
   merge feat/my-feature
   commit tag: "v1.1.0"
   commit
   branch fix/my-bugfix
   checkout fix/my-bugfix
   commit
   commit
   checkout main
   merge fix/my-bugfix
   commit tag: "v1.1.1"
   commit
   branch chore/no-code-change
   checkout chore/no-code-change
   commit
   commit
   checkout main
   merge chore/no-code-change
   commit
```

## Pull Requests

### Minimalist template

```markdown
## Describe your changes

## Related issue

## Checklist before requesting a review

- [ ] I have performed a self-review of my code
- [ ] If it is a core feature, I have added thorough tests.
- [ ] Do we need to implement analytics?
- [ ] Will this be part of a product update? If yes, please write one phrase about this update.
```

### Extended template

```markdown
## What type of PR is this?

## Description of the changes

## Related Tickets & Documents

## Tests

## Documentation

## Post-deployment tasks

## Checklist before requesting a review

- [ ] I have performed a self-review of my code
- [ ] If it is a core feature, I have added thorough tests.
- [ ] Do we need to implement analytics?
- [ ] Will this be part of a product update? If yes, please write one phrase about this update.
```
