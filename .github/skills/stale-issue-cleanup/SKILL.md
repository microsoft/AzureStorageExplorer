---
name: stale-issue-cleanup
description: "Find and close stale bug issues in microsoft/AzureStorageExplorer. Use this when asked to clean up stale issues, close inactive bugs, or perform issue hygiene."
---

## Definition of Stale

An issue is stale when ALL of the following are true:

1. The issue is open.
2. The most recent comment was posted by a member of the
   `microsoft/azure-storage-explorer` GitHub team.
3. That comment was posted **14 or more days ago**.
4. No one outside the team has commented since — including the original
   author and any other community members.

## Important Design Decisions

- **Prefer GraphQL over REST.** Use `gh api graphql` for all GitHub queries.
  GraphQL lets us fetch issue type, last comment, and author in a single
  request instead of N+1 REST calls.
- **Only target Bug-type issues.** Feature requests and tasks should be
  left alone, even if they are old. Filter by `issueType.name == "Bug"`.
- **Never close accessibility issues.** Any issue with a label starting
  with `:accessibility:` must be skipped, regardless of staleness.

## Step-by-Step Procedure

### Step 1: Get team members

Run this command to get the list of Storage Explorer team member logins:

```
gh api --paginate /orgs/microsoft/teams/azure-storage-explorer/members --jq '.[].login'
```

Store these logins for comparison throughout the process.

### Step 2: Find candidate bug issues

Use a single GraphQL query to get open Bug issues with their last comment:

```
gh api graphql -f query='
{
  repository(owner: "microsoft", name: "AzureStorageExplorer") {
    issues(first: 50, states: OPEN, orderBy: {field: UPDATED_AT, direction: ASC}) {
      nodes {
        number
        title
        updatedAt
        issueType { name }
        labels(first: 50) {
          nodes { name }
        }
        comments(last: 1) {
          nodes {
            author { login }
            createdAt
            body
          }
        }
      }
      pageInfo { hasNextPage endCursor }
    }
  }
}'
```

Use pagination (pass `after: "{endCursor}"`) to fetch additional pages if needed.

Filter results to issues where:

1. `issueType.name` is "Bug"
2. No label starts with `:accessibility:`
3. `updatedAt` is ≥14 days ago
4. The last comment exists and its author is a team member (from Step 1)
5. The last comment's `createdAt` is ≥14 days ago

### Step 3: Verify each candidate

For each candidate, read the last comment's body. Only mark as stale if the
team member was asking the reporter for something (e.g., repro steps, logs,
version info, confirmation, follow-up). If the team member's comment was
just an acknowledgment or internal note, **skip** the issue.

### Step 4: Close stale issues

For each confirmed stale issue, do the following **in order**:

#### 4a. Post a contextual closing comment

Read the team member's last comment to understand what they asked for.
Then post a comment using `gh issue comment`.

The comment should follow this template:

> Closing due to inactivity. If you're still experiencing this issue, [ACTION].

Where [ACTION] is tuned to what the team member asked for. Examples:

- "...please reply with the requested logs and we can revisit."
- "...please reply with the repro steps and we'll take another look."
- "...please reply to confirm whether the issue persists on the latest version."
- "...please reply and let us know if the suggested workaround resolved it."

Keep it to ONE sentence. Do not restate what was asked — just tell them what
to provide if they want to re-engage.

Use:

```
gh issue comment {number} --repo microsoft/AzureStorageExplorer --body "{comment}"
```

#### 4b. Close the issue as not planned

```
gh issue close {number} --repo microsoft/AzureStorageExplorer --reason "not planned"
```

#### 4c. Set the Resolution project field to "Stale"

Use GraphQL to update the project field:

1. Find the issue's project item ID on the Storage Explorer project.
2. Find the field ID for "Resolution" and the option ID for "Stale".
3. Update the field value.

Combine lookups into a single query where possible:

```
gh api graphql -f query='
  query {
    repository(owner: "microsoft", name: "AzureStorageExplorer") {
      issue(number: {NUMBER}) {
        projectItems(first: 10) {
          nodes {
            id
            project { title id }
          }
        }
      }
    }
  }
'
```

Then update:

```
gh api graphql -f query='
  mutation {
    updateProjectV2ItemFieldValue(input: {
      projectId: "{PROJECT_ID}"
      itemId: "{ITEM_ID}"
      fieldId: "{FIELD_ID}"
      value: { singleSelectOptionId: "{STALE_OPTION_ID}" }
    }) {
      projectV2Item { id }
    }
  }
'
```

If the token lacks `read:project` scope, note this in the summary and skip
this step.

### Step 5: Report

After processing all issues, print a summary:

- Total open issues scanned
- Number of stale issues found and closed
- List each closed issue: number, title, and what was requested

If no stale issues were found, say so.

## Important Rules

- **DRY RUN**: If the user says "dry run", list the stale issues and what you
  would do, but do NOT post comments, close issues, or update projects.
- **Never close issues where the last comment is from a non-team member.**
  The reporter may have replied — always verify.
- **Be conservative.** If you're unsure whether an issue is stale, skip it.
- **Rate limiting.** Pause briefly between API calls if processing many issues.
