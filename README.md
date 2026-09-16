# Mattermost User Management — cURL Guide

A copy-paste-ready reference for managing Mattermost users from the command line using the **REST API v4** — look users up, list them, and activate or deactivate accounts, with nothing but `curl` and a Personal Access Token.

> **Interactive guide:** https://ali-m07.github.io/mattermost/

---

## Contents

1. [Prerequisites](#1-prerequisites)
2. [Quick Start: Set Up Your Token](#2-quick-start-set-up-your-token)
3. [Finding Users](#3-finding-users)
4. [Reading a User Object](#4-reading-a-user-object)
5. [Activating a User](#5-activating-a-user)
6. [Deactivating a User](#6-deactivating-a-user)
7. [Verifying the Result](#7-verifying-the-result)
8. [Error Reference](#8-error-reference)
9. [Shell Quoting: Bash vs. Windows CMD](#9-shell-quoting-bash-vs-windows-cmd)
10. [Appendix: Python Equivalent](#10-appendix-python-equivalent)

---

## 1. Prerequisites

| Requirement | Details |
| --- | --- |
| Mattermost server | Any reachable instance, for example `https://mattermost.example.com` |
| Personal Access Token (PAT) | Owned by a **System Admin** or **User Manager** account |
| `curl` | Any recent version; Git Bash is recommended on Windows |

**Creating a token:** open Mattermost → *Account Settings → Security → Personal Access Tokens → Create New Token*. The server must allow personal access tokens (`ServiceSettings.EnableUserAccessTokens`).

**Permission notes**

- Normal users: an account with user-management rights can activate or deactivate them.
- Target is a **System Admin**: full `manage_system` permission is required; a limited token returns `403`.
- **LDAP/AD-managed users** (`auth_service: "ldap"`): their status is controlled by LDAP, not the API. Enable or disable them in LDAP itself.

---

## 2. Quick Start: Set Up Your Token

Set two variables once per terminal session. Avoid pasting the token into shared scripts or screenshots.

**Bash / Git Bash**

```bash
export MM_URL="https://mattermost.example.com"
export MM_TOKEN="<your-personal-access-token>"
```

**Windows CMD**

```cmd
set MM_URL=https://mattermost.example.com
set MM_TOKEN=<your-personal-access-token>
```

**Verify the token** — this must return the token owner's profile as JSON:

```bash
curl -s -H "Authorization: Bearer $MM_TOKEN" "$MM_URL/api/v4/users/me"
```

```cmd
curl -s -H "Authorization: Bearer %MM_TOKEN%" "%MM_URL%/api/v4/users/me"
```

If this returns `401 ... session_expired`, the header or token is wrong — see the [Error Reference](#8-error-reference).

---

## 3. Finding Users

The activation endpoint requires the **user ID**, so look the user up first.

### By username

```bash
curl -s -H "Authorization: Bearer $MM_TOKEN" \
  "$MM_URL/api/v4/users/username/<username>"
```

```cmd
curl -s -H "Authorization: Bearer %MM_TOKEN%" "%MM_URL%/api/v4/users/username/<username>"
```

> In many deployments, the Mattermost username is the **local part of the email address**: `ali@corp.com` → `ali`.

### By user ID

```bash
curl -s -H "Authorization: Bearer $MM_TOKEN" "$MM_URL/api/v4/users/<user_id>"
```

### Search active users

```bash
curl -s -X POST -H "Authorization: Bearer $MM_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"term": "ali"}' \
  "$MM_URL/api/v4/users/search"
```

### List all users, including deactivated accounts

```bash
curl -s -H "Authorization: Bearer $MM_TOKEN" \
  "$MM_URL/api/v4/users?page=0&per_page=200&include_deleted=true"
```

- `page` starts at `0`; increment it until fewer than `per_page` users are returned.
- `include_deleted=true` makes deactivated users visible.
- With `jq`, trim the output to the essentials:

```bash
curl -s -H "Authorization: Bearer $MM_TOKEN" \
  "$MM_URL/api/v4/users?page=0&per_page=200&include_deleted=true" \
  | jq -r '.[] | [.username, .id, .delete_at] | @tsv'
```

---

## 4. Reading a User Object

```json
{
  "id": "nkn7n3rc1if87bbdq9kf73553e",
  "username": "hardik",
  "email": "hardik@corp.com",
  "roles": "system_user",
  "auth_service": "",
  "delete_at": 0
}
```

| Field | Meaning |
| --- | --- |
| `id` | 26-character user ID required for status changes |
| `username` / `email` | Identification fields |
| `roles` | Contains `system_admin` when the user is a System Admin |
| `auth_service` | A non-empty value such as `ldap` means an external provider manages the account |
| `delete_at` | **`0` means active; a non-zero timestamp means deactivated** |

---

## 5. Activating a User

First, obtain the user ID. Bash can capture it directly with `jq`:

```bash
USER_ID=$(curl -s -H "Authorization: Bearer $MM_TOKEN" \
  "$MM_URL/api/v4/users/username/<username>" | jq -r .id)
```

Then activate the account:

```bash
curl -s -X PUT -H "Authorization: Bearer $MM_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"active": true}' \
  "$MM_URL/api/v4/users/$USER_ID/active"
```

```cmd
curl -s -X PUT -H "Authorization: Bearer %MM_TOKEN%" -H "Content-Type: application/json" -d "{\"active\": true}" "%MM_URL%/api/v4/users/<user_id>/active"
```

A successful response contains the updated user object with `"delete_at": 0`.

---

## 6. Deactivating a User

Use the same endpoint with the opposite value:

```bash
curl -s -X PUT -H "Authorization: Bearer $MM_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"active": false}' \
  "$MM_URL/api/v4/users/$USER_ID/active"
```

```cmd
curl -s -X PUT -H "Authorization: Bearer %MM_TOKEN%" -H "Content-Type: application/json" -d "{\"active\": false}" "%MM_URL%/api/v4/users/<user_id>/active"
```

> **There is no `/deactive` endpoint.** Both operations use `PUT /api/v4/users/{user_id}/active`; only the `active` value changes. Calling `/users/<id>/deactive` returns `404`.

Mattermost also supports a soft-delete with `DELETE /api/v4/users/{user_id}`, but it returns an empty body. The `PUT` endpoint is usually preferable because it returns the updated user object.

---

## 7. Verifying the Result

Fetch the user again and inspect `delete_at`:

```bash
curl -s -H "Authorization: Bearer $MM_TOKEN" \
  "$MM_URL/api/v4/users/username/<username>" | jq .delete_at
```

- `0` → active
- Any other value → deactivated

---

## 8. Error Reference

| HTTP | Typical body | Meaning | Fix |
| --- | --- | --- | --- |
| `401` | `api.context.session_expired.app_error` | Token missing, invalid, expired, or the `Authorization: Bearer` header is malformed | Recheck the header and regenerate the token if needed |
| `403` | Permission error | Token lacks rights; the target is a System Admin; or the user is LDAP-managed | Use a fully privileged token or manage LDAP users in LDAP |
| `404` | `api.context.404.app_error` | Wrong route, such as `/users/<id>/deactive` | Use `/users/<id>/active` |
| `404` | `app.user.get.not_found` | Unknown username or user ID | Check the username and ID |

---

## 9. Shell Quoting: Bash vs. Windows CMD

| | Bash / Git Bash | Windows CMD |
| --- | --- | --- |
| Variables | `$MM_TOKEN` | `%MM_TOKEN%` |
| Line breaks | Trailing `\` | Keep commands on one line |
| JSON body | `-d '{"active": true}'` | `-d "{\"active\": true}"` |
| Echo a variable | `echo $MM_TOKEN` | `echo %MM_TOKEN%` |

Sending `-d '{"active": true}'` from CMD is a common failure: CMD treats single quotes literally and sends an invalid JSON body. Escape double quotes as shown above.

---

## 10. Appendix: Python Equivalent

Install `requests` once:

```bash
python -m pip install requests
```

```python
import os
import requests

BASE = "https://mattermost.example.com/api/v4"
HEADERS = {"Authorization": f"Bearer {os.environ['MM_TOKEN']}"}


def get_user(username):
    response = requests.get(f"{BASE}/users/username/{username}", headers=HEADERS)
    response.raise_for_status()
    return response.json()


def list_users(page=0):
    response = requests.get(
        f"{BASE}/users",
        headers=HEADERS,
        params={"page": page, "per_page": 200, "include_deleted": "true"},
    )
    response.raise_for_status()
    return response.json()


def set_active(user_id, active):
    response = requests.put(
        f"{BASE}/users/{user_id}/active",
        headers=HEADERS,
        json={"active": active},
    )
    response.raise_for_status()
    return response.json()


user = get_user("hardik")
print(user["id"], user["delete_at"])
print(list_users())
set_active(user["id"], True)     # Activate
# set_active(user["id"], False)  # Deactivate
```

---

## Security

- Never commit a real Personal Access Token, include it in an issue, or show it in screenshots.
- Store it in an environment variable such as `MM_TOKEN` or a git-ignored configuration file.
- Rotate the token immediately if it may have been exposed.
