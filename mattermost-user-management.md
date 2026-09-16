# Mattermost User Management — cURL Guide

A copy-paste-ready reference for managing Mattermost users from the command line using the **REST API v4** — look users up, list them, and activate or deactivate accounts, with nothing but `curl` and a Personal Access Token.

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
| Mattermost server | any reachable instance, e.g. `https://mattermost.example.com` |
| Personal Access Token (PAT) | owned by a **System Admin** or **User Manager** account |
| `curl` | any recent version. On Windows, Git Bash is recommended |

**Creating a token:** open Mattermost → *Account Settings → Security → Personal Access Tokens → Create New Token*. The server must allow personal access tokens (`ServiceSettings.EnableUserAccessTokens`).

**Permission notes**

- Normal users: any account with user-management rights can activate/deactivate them.
- Target is a **System Admin**: requires full `manage_system` (a limited token gets `403`).
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

**Verify the token** — this must return the token owner's own profile as JSON:

```bash
curl -s -H "Authorization: Bearer $MM_TOKEN" "$MM_URL/api/v4/users/me"
```

```cmd
curl -s -H "Authorization: Bearer %MM_TOKEN%" "%MM_URL%/api/v4/users/me"
```

If you get `401 ... session_expired` here, the header or the token is wrong — see the [Error Reference](#8-error-reference).

---

## 3. Finding Users

The activate/deactivate endpoint needs the **user id**, so you almost always look the user up first.

### By username

```bash
curl -s -H "Authorization: Bearer $MM_TOKEN" \
  "$MM_URL/api/v4/users/username/<username>"
```

```cmd
curl -s -H "Authorization: Bearer %MM_TOKEN%" "%MM_URL%/api/v4/users/username/<username>"
```

> In many deployments the Mattermost username is the **local part of the e-mail** (`ali@corp.com` → `ali`).

### By user id

```bash
curl -s -H "Authorization: Bearer $MM_TOKEN" "$MM_URL/api/v4/users/<user_id>"
```

### Search (active users only)

```bash
curl -s -X POST -H "Authorization: Bearer $MM_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"term": "ali"}' \
  "$MM_URL/api/v4/users/search"
```

### List all users (including deactivated ones)

```bash
curl -s -H "Authorization: Bearer $MM_TOKEN" \
  "$MM_URL/api/v4/users?page=0&per_page=200&include_deleted=true"
```

- `page` starts at `0`; keep incrementing until fewer than `per_page` users come back.
- `include_deleted=true` is what makes deactivated users visible.
- With `jq` you can trim the output to the essentials:

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
| `id` | 26-character user id — required for status changes |
| `username` / `email` | identification |
| `roles` | contains `system_admin` → the user is a System Admin |
| `auth_service` | non-empty (e.g. `ldap`) → managed by an external auth provider |
| `delete_at` | **`0` → active. Non-zero (a timestamp) → deactivated** |

---

## 5. Activating a User

Two steps: find the id, then flip the flag.

**Step 1 — get the id** (see [Finding Users](#3-finding-users)) and copy the `"id"` value from the JSON. With `jq` it can be captured directly:

```bash
USER_ID=$(curl -s -H "Authorization: Bearer $MM_TOKEN" \
  "$MM_URL/api/v4/users/username/<username>" | jq -r .id)
```

**Step 2 — activate:**

```bash
curl -s -X PUT -H "Authorization: Bearer $MM_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"active": true}' \
  "$MM_URL/api/v4/users/$USER_ID/active"
```

```cmd
curl -s -X PUT -H "Authorization: Bearer %MM_TOKEN%" -H "Content-Type: application/json" -d "{\"active\": true}" "%MM_URL%/api/v4/users/<user_id>/active"
```

**Success** returns the full user object with `"delete_at": 0`.

---

## 6. Deactivating a User

Same endpoint, opposite flag:

```bash
curl -s -X PUT -H "Authorization: Bearer $MM_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"active": false}' \
  "$MM_URL/api/v4/users/$USER_ID/active"
```

```cmd
curl -s -X PUT -H "Authorization: Bearer %MM_TOKEN%" -H "Content-Type: application/json" -d "{\"active\": false}" "%MM_URL%/api/v4/users/<user_id>/active"
```

> **There is no `/deactive` endpoint.** Both operations go through `PUT /users/{user_id}/active` — only the `active` value changes. A request to `/users/<id>/deactive` returns `404`.

An equivalent soft-delete also exists — `DELETE /api/v4/users/{user_id}` — but it returns an empty response body. The `PUT` form above is preferable because it returns the updated user object.

---

## 7. Verifying the Result

Fetch the user again and check `delete_at`:

```bash
curl -s -H "Authorization: Bearer $MM_TOKEN" \
  "$MM_URL/api/v4/users/username/<username>" | jq .delete_at
```

- `0` → active
- anything else → deactivated

---

## 8. Error Reference

| HTTP | Typical body | Meaning | Fix |
| --- | --- | --- | --- |
| `401` | `api.context.session_expired.app_error` — *Invalid or expired session* | token missing, wrong, or the `Authorization: Bearer` header is malformed | re-check the header; regenerate the token |
| `403` | permission error | token lacks the rights: the target is a System Admin, or the user is LDAP-managed | use a full System Admin token; manage LDAP users in LDAP |
| `404` | `api.context.404.app_error` — *no api call for the url* | wrong path (e.g. `/users/<id>/deactive`) | use `/users/<id>/active` |
| `404` | `app.user.get.not_found` | unknown username / id | check the spelling; the username is often the e-mail's local part |

---

## 9. Shell Quoting: Bash vs. Windows CMD

| | Bash / Git Bash | Windows CMD |
| --- | --- | --- |
| Variables | `$MM_TOKEN` | `%MM_TOKEN%` |
| Line breaks | trailing `\` | none — keep the command on one line |
| JSON body | `-d '{"active": true}'` | `-d "{\"active\": true}"` |
| Echo a variable | `echo $MM_TOKEN` | `echo %MM_TOKEN%` |

Sending `-d '{"active": true}'` from CMD is a classic silent failure: CMD treats `'` literally, so the server receives an invalid body. In CMD, escape the double quotes as shown above.

---

## 10. Appendix: Python Equivalent

The same operations with `requests`:

```python
import os
import requests

BASE = "https://mattermost.example.com/api/v4"
HEADERS = {"Authorization": "Bearer " + os.environ["MM_TOKEN"]}

def get_user(username):
    r = requests.get(f"{BASE}/users/username/{username}", headers=HEADERS)
    r.raise_for_status()
    return r.json()

def set_active(user_id, active):
    r = requests.put(f"{BASE}/users/{user_id}/active",
                     headers=HEADERS, json={"active": active})
    r.raise_for_status()

user = get_user("hardik")
set_active(user["id"], True)      # activate
# set_active(user["id"], False)   # deactivate
```

---

*Keep tokens out of repositories and shell history. This project keeps its token in `settings.json` (git-ignored) or injects it via the `MM_TOKEN` environment variable in OpenShift.*
