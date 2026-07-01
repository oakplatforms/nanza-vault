# PR Template

This document outlines the standard format for creating Pull Request descriptions in the Nanza Mobile app.

## Overview

The PR template follows a consistent structure to ensure all pull requests contain the necessary information for reviewers and maintainers.

## Structure

### Jira Ticket
NANZA-{TICKET_NUMBER} - https://oakplatforms.atlassian.net/browse/NANZA-{TICKET_NUMBER}

### What did you implement?
- Bullet point 1
- Bullet point 2
- Bullet point 3
- Bullet point 4
- Bullet point 5

### Additional resources

## Instructions for Assistant

1. **Detect branch name** - Take the last part of the branch name (e.g., "NANZA-71" from "feature/NANZA-71")
2. **Use markdown format** - Use `###` for headings as shown above
3. **Structure the implementation list** - Use concise bullet points with `-` for each change
4. **Keep it focused** - Include only the main features/components added or modified
5. **Follow the exact format** - Use the three headings: "Jira Ticket", "What did you implement?", "Additional resources"
6. **Output format** - ALWAYS output TWO separate markdown code blocks:
   - First block: PR description with "markdown" language specification
   - Second block: Git commit command with "bash" language specification

```markdown
### Jira Ticket
NANZA-{TICKET_NUMBER} - https://oakplatforms.atlassian.net/browse/NANZA-{TICKET_NUMBER}

### What did you implement?
- Bullet point 1
- Bullet point 2
- Bullet point 3

### Additional resources
```

```bash
git commit -m "Insert commit message here"
```

## Examples

### Form Components Enhancement

```markdown
### Jira Ticket
NANZA-157 - https://oakplatforms.atlassian.net/browse/NANZA-157

### What did you implement?
- Added EditListing component for updating existing product listings
- Added CreateListing component for creating new product listings
- Created reusable TextArea component with character validation, real-time counter display, and maxLength enforcement
- Added new form components (NumericField, DropdownField, ToggleField, ImageField) with help text support and style variants
- Added comprehensive unit test coverage for EditListing component with 20+ test cases covering form interactions, validation, and API calls
- Added comprehensive unit test coverage for CreateListing component with 15+ test cases covering form inputs, validation, and submission

### Additional resources
```

```bash
git commit -m "Added EditListing and CreateListing components, moved form functionality to global Form component, and added unit test coverage for all new components and form variants"
```

### Icon Component Enhancement

```markdown
### Jira Ticket
NANZA-{TICKET_NUMBER} - https://oakplatforms.atlassian.net/browse/NANZA-{TICKET_NUMBER}

### What did you implement?
- Enhanced Icon component with width/height props for independent dimension control
- Added strokeWidth prop for SVG stroke thickness control
- Created dedicated icons section in theme structure
- Added comprehensive unit test suite with 30+ test cases covering all prop combinations
- Maintained backward compatibility with existing size prop usage

### Additional resources
```

```bash
git commit -m "Enhanced Icon component with width/height props, added strokeWidth support, and created comprehensive unit tests"
```

## Best Practices

1. **Be concise** - Use clear, focused bullet points that describe the main changes
2. **Focus on features** - Emphasize what was added or built, not just technical details
3. **Include testing** - Mention unit test coverage when applicable
4. **Group related changes** - Keep related functionality in the same bullet point
5. **Use active voice** - Start bullet points with action verbs like "Added", "Created", "Enhanced"

## Troubleshooting

### Common Issues

- **Missing Jira ticket** - Always include the Nanza ticket number in the format shown above
- **Too verbose** - Keep bullet points concise and focused on main features
- **Missing test coverage** - Document any testing performed to validate changes
- **Technical jargon** - Use clear language that reviewers can understand

### Verification

Before submitting a PR, verify that your description includes:
- [ ] Jira ticket number with proper link
- [ ] Concise list of implemented features
- [ ] Main components or functionality added
- [ ] Testing coverage information when applicable
- [ ] Clear, focused bullet points 