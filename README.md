# Mattermost User Management

A practical guide for looking up Mattermost users, listing accounts, and activating or deactivating them through the **REST API v4**.

Use a Personal Access Token from an account with the required user-management permissions. Do not commit tokens to this repository.

> **Interactive version:** [Open the GitHub Pages guide](https://ali-m07.github.io/mattermost/)

## What you can do

- Look up a user by username or user ID
- List users, including deactivated accounts
- Activate an account
- Deactivate an account
- Verify the resulting status
- Diagnose common `401`, `403`, and `404` errors

## Prerequisites

| Requirement | Details |
| --- | --- |
| Mattermost server | A reachable instance, for example `https://mattermost.example.com` |
| Personal Access Token | Created by an account with user-management rights |
| Client | `curl` for shell commands, or Python 3 with `requests` |

Create a token in Mattermost under **Account Settings → Security → Personal Access Tokens**. Your server must allow personal access tokens (`ServiceSettings.EnableUserAccessTokens`).

> **Permissions:** Managing a System Admin normally requires full `manage_system` permission. LDAP/AD-managed accounts (`auth_service: "ldap"`) are controlled by LDAP, not the Mattermost API.

## Choose your platform

<details>
<summary><strong>Windows CMD</strong></summary>

### 1. Configure and verify your token

```cmd
set MM_URL=https://mattermost.example.com
set MM_TOKEN=<your-personal-access-token>
curl -s -H "Authorization: Bearer %MM_TOKEN%" "%MM_URL%/api/v4/users/me"
```

The last command returns the token owner's profile when the token is valid.

### 2. Find or list users

Look up a username and copy the returned `id` value:

```cmd
curl -s -H "Authorization: Bearer %MM_TOKEN%" "%MM_URL%/api/v4/users/username/<username>"
set USER_ID=<user-id>
```

List active and deactivated users:

```cmd
curl -s -H "Authorization: Bearer %MM_TOKEN%" "%MM_URL%/api/v4/users?page=0&per_page=200&include_deleted=true"
```

### 3. Activate or deactivate

Activate a user:

```cmd
curl -s -X PUT -H "Authorization: Bearer %MM_TOKEN%" -H "Content-Type: application/json" -d "{\"active\": true}" "%MM_URL%/api/v4/users/%USER_ID%/active"
```

Deactivate a user by changing `true` to `false`:

```cmd
curl -s -X PUT -H "Authorization: Bearer %MM_TOKEN%" -H "Content-Type: application/json" -d "{\"active\": false}" "%MM_URL%/api/v4/users/%USER_ID%/active"
```

### 4. Verify the status

```cmd
curl -s -H "Authorization: Bearer %MM_TOKEN%" "%MM_URL%/api/v4/users/%USER_ID%"
```

In the response, `"delete_at": 0` means active. A non-zero timestamp means deactivated.

</details>

<details>
<summary><strong>macOS / Linux / Git Bash</strong></summary>

### 1. Configure and verify your token

```bash
export MM_URL="https://mattermost.example.com"
export MM_TOKEN="<your-personal-access-token>"
curl -s -H "Authorization: Bearer $MM_TOKEN" "$MM_URL/api/v4/users/me"
```

### 2. Find or list users

Look up a username and save its user ID:

```bash
USER_ID=$(curl -s -H "Authorization: Bearer $MM_TOKEN" \
  "$MM_URL/api/v4/users/username/<username>" | jq -r .id)
```

List active and deactivated users:

```bash
curl -s -H "Authorization: Bearer $MM_TOKEN" \
  "$MM_URL/api/v4/users?page=0&per_page=200&include_deleted=true"
```

### 3. Activate or deactivate

Activate a user:

```bash
curl -s -X PUT -H "Authorization: Bearer $MM_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"active": true}' \
  "$MM_URL/api/v4/users/$USER_ID/active"
```

Deactivate a user by changing `true` to `false`:

```bash
curl -s -X PUT -H "Authorization: Bearer $MM_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"active": false}' \
  "$MM_URL/api/v4/users/$USER_ID/active"
```

### 4. Verify the status

```bash
curl -s -H "Authorization: Bearer $MM_TOKEN" \
  "$MM_URL/api/v4/users/$USER_ID" | jq .delete_at
```

`0` means active. Any non-zero timestamp means deactivated.

</details>

<details>
<summary><strong>Python</strong></summary>

### 1. Install and configure

```bash
python -m pip install requests
```

Set your token as an environment variable, then use it in Python:

```python
import os
import requests

BASE = "https://mattermost.example.com/api/v4"
HEADERS = {"Authorization": f"Bearer {os.environ['MM_TOKEN']}"}

response = requests.get(f"{BASE}/users/me", headers=HEADERS)
response.raise_for_status()
```

### 2. Find or list users

```python
user = requests.get(f"{BASE}/users/username/<username>", headers=HEADERS)
user.raise_for_status()
user_id = user.json()["id"]

users = requests.get(
    f"{BASE}/users",
    headers=HEADERS,
    params={"page": 0, "per_page": 200, "include_deleted": "true"},
)
users.raise_for_status()
print(users.json())
```

### 3. Activate or deactivate

```python
response = requests.put(
    f"{BASE}/users/{user_id}/active",
    headers=HEADERS,
    json={"active": True},
)
response.raise_for_status()
```

Change `True` to `False` to deactivate the account.

### 4. Verify the status

```python
user = requests.get(f"{BASE}/users/{user_id}", headers=HEADERS)
user.raise_for_status()
print(user.json()["delete_at"])
```

</details>

## Important endpoint

There is **no** `/deactive` endpoint.

Both actions use the same endpoint and only the JSON value changes:

```text
PUT /api/v4/users/{user_id}/active
```

```json
{"active": true}
```

Activates the user. Replace `true` with `false` to deactivate them.

## Common errors

| HTTP | Meaning | What to do |
| --- | --- | --- |
| `401` | Token is missing, invalid, expired, or the `Authorization: Bearer` header is malformed | Recreate or recheck the token and header |
| `403` | Token lacks permission, the target is a System Admin, or the user is LDAP-managed | Use a properly privileged token; manage LDAP users in LDAP |
| `404` | Incorrect URL, unknown user ID, or use of `/deactive` | Use `/users/{user_id}/active` and verify the user ID |

## Security

- Never paste a real token into a repository, issue, or screenshot.
- Prefer environment variables such as `MM_TOKEN` over hard-coded credentials.
- Rotate the token immediately if it has been exposed.

## Detailed reference

The original long-form reference is retained in [mattermost-user-management.md](mattermost-user-management.md).
