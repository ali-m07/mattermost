# Mattermost User Management

A copy-paste-ready cURL cookbook for managing Mattermost users through the **REST API v4** — look users up, list them, activate and deactivate accounts, and troubleshoot errors.

## Contents

- **[Mattermost User Management — cURL Guide](mattermost-user-management.md)** — the full guide (Markdown)
- **[mattermost-user-management.html](mattermost-user-management.html)** — the same guide as a standalone, styled HTML page

## Highlights

- Find users by username, id, or search term
- List all users — including deactivated ones — with pagination
- Activate / deactivate via `PUT /users/{user_id}/active`
- Error reference (`401` / `403` / `404`) and Bash vs. Windows CMD quoting notes

No dependencies — just `curl` and a Personal Access Token.

> Keep tokens out of repositories and shell history. Store them in environment variables or git-ignored config files.
