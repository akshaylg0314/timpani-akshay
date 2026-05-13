#<!--
#* SPDX-FileCopyrightText: Copyright 2026 LG Electronics Inc.
#* SPDX-License-Identifier: MIT
#-->
# GitHub Development Workflow Guidelines

**Date**: 2026-02-17
**Author**: basheer_LGSI

## Table of Contents
1. [Issue Registration Rules](#1-issue-registration-rules)
2. [Branch Creation Rules](#2-branch-creation-rules)
3. [Commit Rules](#3-commit-rules)
4. [Labeling Rules by Stage](#4-labeling-rules-by-stage)
5. [Step-by-Step Workflow Guide](#5-step-by-step-workflow-guide)
6. [Automation Setup Guide](#6-automation-setup-guide)
7. [Documentation Metadata Standards](#7-documentation-metadata-standards)

---

## 1. Issue Registration Rules

### Issue Type Classification
- **FEATURE**: Requirement Issue (Parent Issue)
- **TASK**: Development Task Issue (Child Issue)
- **BUG**: Bug Fix Issue


### Issue Title Format

[Type] Title

Example:
- `[FEATURE] User Authentication System Implementation`
- `[TASK] Login Page UI Development`
- `[BUG] Password Reset Email Sending Failure`

### Issue Body Template

#### Requirement (REQ) Issue Template
```markdown
---
name: Requirement
about: New feature requirement
title: '[FEATURE] '
labels: requirement, status:backlog
assignees: ''
---

## 📝 Requirement Description
<!-- Detailed description of the requirement -->

## 📋 Acceptance Criteria
- [ ] Criterion 1
- [ ] Criterion 2

## 📎 Related Documents/References
<!-- Links to related documents -->

## 📌 Subtasks
<!-- Automatically updated -->

## 🧪 Testing Plan
- [ ] Unit Test:
- [ ] Integration Test:
- [ ] Performance Test:

## 📊 Test Results
<!-- Automatically updated after issue closure -->
```
## Development Task (TASK) Issue Template
```markdown
---
name: Development Task
about: Development task to be implemented
title: '[TASK] '
labels: task, status:todo
assignees: ''
---

## 📝 Task Description
<!-- Description of the task to be performed -->

## 📋 Checklist
- [ ] Item 1
- [ ] Item 2

## 🔗 Related Requirement
<!-- Link to parent requirement in "Relates to #issue_number" format -->
Relates to #

## 📐 Implementation Guidelines
<!-- Reference material for implementation -->

## 🧪 Testing Method
<!-- Testing method after implementation -->
```

## Issue Relationship Setup

    Connect Requirement (REQ) and Development Task (TASK): Specify Relates to #requirement_number in the TASK issue description.
    Track subtasks in the requirement issue:

    ## 📌 Subtasks
    - [ ] #123 Login Page UI Development
    - [ ] #124 Backend Authentication API Implementation

## 2. Branch Creation Rules
### Branch Naming Convention
```
<type>/<issue_number>-<short-description>
```
### Branch Types

- **feat**: New feature development
- **fix**: Bug fix
- **refactor**: Code refactoring
- **docs**: Documentation work
- **test**: Test code work
- **chore**: Other maintenance work

### Examples

- `feat/123-user-authentication`
- `fix/145-password-reset-bug`
- `docs/167-api-documentation`

### Branch Creation Procedure

  1. Use "Development" > "Create a branch" on the issue page, or
  2. From the command line:
```bash
    git checkout -b feat/123-user-login main
```
## 3. Commit Rules

## Commit Message Format
```
<type>(<scope>): <description> [#issue-number]
```

## Commit Types

- **feat**: New feature
- **fix**: Bug fix
- **docs**: Documentation changes
- **style**: Code formatting, missing semicolons, etc.
- **refactor**: Code refactoring
- **test**: Test-related code
- **chore**: Build tasks, package manager configuration, etc.

## Examples

- `feat(auth): Implement social login [#123]`
- `fix(ui): Fix button overlap on mobile [#145]`
- `docs(api): Update API documentation [#167]`

## Detailed Commit Description (Optional)
```
<type>(<scope>): <description> [#issue-number]

<Detailed explanation>

<Caveats or Breaking Changes>

<Related Issues (Closes, Fixes, Resolves)>

<Related Issues (Closes, Fixes, Resolves)>
```

## PR Body Format

```markdown
## 📝 PR Description
<!-- Description of the changes -->

## 🔗 Related Issue
<!-- Link to the issue this PR resolves (Use Closes, Fixes, Resolves keywords) -->
Closes #

## 🧪 Test Method
<!-- Description of the test method -->

## 📸 Screenshots
<!-- Attach screenshots if there are UI changes -->

## ✅ Checklist
- [ ] Code conventions are followed
- [ ] Tests are added/modified
- [ ] Documentation is updated (if necessary)
```
---

## 4. Labeling Rules By Stage

### Label System

#### 1. Status Labels (status:*)
- `status:backlog` - Issue in the backlog
- `status:todo` - Issue in the to-do list
- `status:in-progress` - Issue in progress
- `status:review` -  Under review
- `status:blocked` - Blocked
- `status:done` - Completed

#### 2. Type Labels  (type:*)
- `type:requirement` - Requirement issue
- `type:task` - Development task issue
- `type:bug` - Bug issue
- `type:enhancement` - Feature enhancement
- `type:documentation` - Documentation task

#### 3. Priority Labels (priority:*)
- `priority:critical` - Highest priority
- `priority:high` - High priority
- `priority:medium` - Medium priority
- `priority:low` - Low priority

#### 4. Test Status Labels(test:*)
- `test:pending` - Test pending
- `test:running` - Test running
- `test:passed` - Test passed
- `test:failed` - Test failed

### Label Color Guide
```
Status labels: Blue shades
Type labels: Green shades
Priority labels: Red/Yellow shades
Complexity labels: Purple shades
Test status labels: Gray/Black shades
```
---

## 5. Step-by-Step Workflow  Guide

### 1. Create Requirement Issue
- Title: [REQ] Requirement Title
- Labels: type:requirement, status:backlog
- Write detailed description


### 2. Create Development Task Issue
- Title: [TASK] Task Title
- Labels: type:task, status:todo
- Link to parent issue: Relates to #requirement_number


### 3. Create Branch and Develop
- Branch name: feat/issue_number-task_name
- Change issue status: status:in-progress


### 4. Commit and Push
- Commit message: feat(scope): Implementation details [#issue_number]

### 5. Create Pull Request
- Title: [Issue Type] Issue Title (#issue_number)
- Include Closes #issue_number in the body
- Label: status:review


### 6. Code Review and Merge
- Assign reviewers
- Merge after approval
- Issue automatically closes


### 7.  Run Tests
- Trigger test execution
- Update labels based on test results: test:passed or test:failed
- Update the requirement issue with test results


---

## 6. Automation Setup Guide

### Branch Protection Rules
1. Repository > Settings > Branches > Branch protection rules
2. Configure protection rules for the main/master branch:
  - Require pull request reviews
  - Require status checks to pass
  - Require linear history


### Label Automation Workflow
Implement the following automation using GitHub Actions:
  - Set initial labels when creating issues/PRs
  - Update issue status when creating a branch
  - Run tests and update labels when merging a PR


---

## Workflow Diagram

```
Create Requirement Issue (adminstrator)
       ↓
  Create Sub-tasks (adminstrator)
       ↓
  Create Branch (adminstrator)
       ↓
    Development Work (developer)
       ↓
  Commit and Push (developer)
       ↓
    Create PR (developer)
       ↓
  Code Review (adminstrator)
       ↓
  Approve and Merge PR (adminstrator)
       ↓
  Run Automated Tests (adminstrator)
       ↓
  Close Issue and Update Results (adminstrator)
```

---

## 7. Documentation Metadata Standards

### Overview

All documentation files in the Eclipse timpani project must include standardized metadata headers to ensure traceability, version control, and proper attribution. This applies to all files in the `doc/` directory.

### Required Metadata Header Template

Every documentation file must start with the following structure (after the SPDX license header):

```markdown
<!--
* SPDX-FileCopyrightText: Copyright 2026 LG Electronics Inc.
* SPDX-License-Identifier: MIT
-->

# [Document Title]

**Document Information:**
- **Issuing Author:** [Author Name/Team]
- **Configuration ID:** [Configuration ID following naming convention]
- **Document Status:** [Draft | Review | Approved | Published]
- **Last Updated:** [YYYY-MM-DD]

---

## Revision History

| Version | Date | Comment | Author | Approver |
|---------|------|---------|--------|----------|
| 0.0a | YYYY-MM-DD | Initial document creation | [Author] | [Approver] |

---

[Rest of document content...]
```

### Configuration ID Naming Convention

#### LLD Documents
Format: `timpani-[component]-lld-[number]`

**Examples:**
- `timpani-o-lld-01` - timpani-o SchedInfo Service LLD
- `timpani-o-lld-02` - timpani-o Fault Service Client LLD
- `timpani-n-lld-01` - timpani-n Initialization & Main LLD
- `timpani-n-lld-02` - timpani-n Configuration Management LLD
- `timpani-o-lld-index` - timpani-o LLD README
- `timpani-n-lld-index` - timpani-n LLD README

#### Architecture Documents
Format: `timpani-arch-[type]`

**Examples:**
- `timpani-arch-system` - System Architecture
- `timpani-arch-grpc` - gRPC Integration Architecture

#### Other Documentation
Format: `timpani-[category]-[type]`

**Examples:**
- `timpani-api-reference` - API Documentation
- `timpani-doc-structure` - Project Structure Documentation
- `timpani-doc-index` - Main Documentation Index (README)

### Document Status Values

| Status | Description | When to Use |
|--------|-------------|-------------|
| `Draft` | Initial creation, work in progress | Document is being written |
| `Review` | Under review | Document is complete and awaiting review |
| `Approved` | Reviewed and approved | Document has been reviewed and approved |
| `Published` | Final, published version | Document is complete and publicly available |

### Revision History Guidelines

1. **Version Numbering:** Use semantic versioning with alpha designation for initial versions
   - Alpha version (0.0a): Initial document creation, pre-release
   - Major version (1.0 → 2.0): Significant restructuring or content changes
   - Minor version (1.0 → 1.1): Content updates, additions, corrections
   - Patch version (1.0.0 → 1.0.1): Typo fixes, formatting (optional third digit)

2. **Date Format:** Always use `YYYY-MM-DD` format (ISO 8601)

3. **Comment Field:** Brief description of changes made in this version

4. **Author Field:** Person or team who made the changes

5. **Approver Field:** Person who approved the changes (use `-` if not yet approved)

### Example Revision History

```markdown
## Revision History

| Version | Date | Comment | Author | Approver |
|---------|------|---------|--------|----------|
| 1.1 | 2026-05-20 | Added error handling section | Eclipse timpani Team | John Doe |
| 0.0a | 2026-05-13 | Initial LLD document creation | Eclipse timpani Team | - |
```

### Files Requiring Metadata

The following types of files must include metadata headers:

1. **LLD Documents** (`doc/architecture/LLD/`)
   - All component LLD files (timpani-o/*.md, timpani-n/*.md)
   - README files in each directory

2. **Architecture Documents** (`doc/architecture/`)
   - System architecture documents
   - Integration architecture documents

3. **API Documentation** (`doc/docs/api.md`)

4. **Project Documentation** (`doc/docs/`)
   - Structure documentation
   - Development guides
   - Release documentation

5. **Main Documentation Index** (`doc/README.md`)

### Metadata Maintenance

1. **Update "Last Updated" date** whenever document content changes
2. **Add revision history entry** for significant changes
3. **Update document status** as document progresses through lifecycle
4. **Keep Configuration ID unchanged** after initial creation
5. **Preserve SPDX headers** - never remove or modify license information

### Verification Checklist

Before committing documentation changes, verify:

- [ ] SPDX license header is present and correct
- [ ] Document Information section is complete
- [ ] Configuration ID follows naming convention
- [ ] Document Status is accurate
- [ ] Last Updated date is current (YYYY-MM-DD format)
- [ ] Revision History table is present
- [ ] Revision History has at least one entry (version 1.0)
- [ ] All dates use YYYY-MM-DD format
- [ ] Revision comments are meaningful and concise

---
