# W.TECH Config System - Site Developer Spec

## Overview

The backend already contains the new config system used by the loader and Lua script.

This system supports:

- user-owned configs
- imported configs
- share links
- market публикации
- ratings
- comments
- imported config updates

Important:  
**The website must not expose loader API keys or loader auth tokens in client-side JavaScript.**  
For website integration, use **server-side PHP** and call the internal service layer directly.

---

## Backend location

All backend files are inside:

`/backend/`

Main internal config service:

`/backend/configs/service.php`

Main database class:

`/backend/database.php`

---

## Important architecture rules

### 1. Config library is unified
All user configs are stored in one table:

`config_library`

This includes:
- own configs
- imported configs from share links
- imported configs from market

### 2. Config names are unique per user
A user cannot have 2 configs with the same name.

This applies to:
- own configs
- imported share configs
- imported market configs

If name already exists, import/save must fail.

### 3. Share links are snapshots
When a user shares a config, the backend stores a snapshot of config data at that moment.

If the author later updates the original config:
- old share link still gives the old snapshot
- new share action creates a new share token

### 4. Imported configs are copies
When a user imports a config:
- a copy is added to their library
- it is not a live reference

If the author later updates the original:
- the recipient may update manually from the website
- loader/Lua does not handle update UI

### 5. Only own configs can be shared or published
Imported configs cannot:
- be shared as normal config links
- be published to market

If imported from market, website may offer:
- "copy market link"
but not "share config"

---

## Main tables

---

## 1) config_library

Stores all configs belonging to a user.

### Key columns
- `id`
- `user_id`
- `name`
- `data`
- `item_type`
- `version`
- `source_config_id`
- `source_user_id`
- `source_market_item_id`
- `source_share_link_id`
- `source_version_applied`
- `created_at`
- `updated_at`

### item_type values
- `own`
- `import_share`
- `import_market`

### Meaning
#### own
User's own config.
Can:
- save/update
- share
- publish to market

#### import_share
Imported from share link.
Cannot:
- share
- publish

#### import_market
Imported from market.
Cannot:
- share
- publish

If related market item still exists and is active, website may allow:
- copy market page link

---

## 2) config_share_links

Stores immutable share snapshots.

### Key columns
- `id`
- `owner_user_id`
- `owner_config_id`
- `token`
- `snapshot_name`
- `snapshot_data`
- `snapshot_version`
- `is_active`
- `uses_count`
- `created_at`
- `last_used_at`

### Notes
- token is 64 hex chars
- links do not expire
- links have no max-use limit
- if original config is deleted, links should be considered inactive

---

## 3) config_market_items

Stores published market configs.

### Key columns
- `id`
- `owner_user_id`
- `owner_config_id`
- `title`
- `description`
- `is_active`
- `rating_avg`
- `rating_count`
- `comment_count`
- `import_count`
- `created_at`
- `updated_at`

### Notes
- one own config -> one market item
- market item points to original own config
- if original config is deleted, market item should be hidden/inactive

---

## 4) config_market_ratings

Stores one rating per user per market item.

### Key columns
- `id`
- `market_item_id`
- `user_id`
- `rating`
- `created_at`

### Rules
- one user can rate one market item only once
- rating cannot be edited later in current version

---

## 5) config_market_comments

Stores one comment per user per market item.

### Key columns
- `id`
- `market_item_id`
- `user_id`
- `comment`
- `created_at`

### Rules
- one user -> one comment per market item
- comment cannot be edited in current version

---

## Internal PHP service

Main class:

`ConfigService`

Location:

`/backend/configs/service.php`

This class contains the real business logic.

### Main methods already implemented
- `save_own_config(int $user_id, string $name, string $data)`
- `get_user_library(int $user_id)`
- `load_config(int $user_id, int $config_id)`
- `delete_config(int $user_id, int $config_id)`
- `create_share_link(int $user_id, int $config_id)`
- `get_share_link_info(string $token)`
- `accept_share_link(int $user_id, string $token)`
- `update_imported_config(int $user_id, int $config_id)`
- `publish_to_market(int $user_id, int $config_id, string $title, string $description)`
- `unpublish_from_market(int $user_id, int $market_item_id)`
- `get_market_list(int $limit = 50, int $offset = 0)`
- `get_market_item_details(int $market_item_id)`
- `add_market_config_to_library(int $user_id, int $market_item_id)`
- `rate_market_item(int $user_id, int $market_item_id, int $rating)`
- `add_market_comment(int $user_id, int $market_item_id, string $comment)`
- `get_market_comments(int $market_item_id, int $limit = 50, int $offset = 0)`

---

## Important website integration rule

### Do NOT use loader endpoints directly from browser JS
Many backend endpoints require:
- API key
- loader token
- hwid

Those are for loader/Lua, not public website JS.

### Recommended approach
Website should use:
- server-side PHP session auth
- direct `require_once` of backend files
- direct call to `ConfigService`

Example pattern:

```php
define('WTECH_BACKEND', true);

require_once '/path/to/backend/configs/service.php';
require_once '/path/to/backend/database.php';
require_once '/path/to/backend/config.php';
```

Then use logged-in website user ID:
```php
$result = ConfigService::get_user_library($user_id);
```

This is the recommended secure integration.

---

## Website pages / features to implement

---

## 1) My Configs page

Should show user's entire library:
- own configs
- imported configs

Use:
`ConfigService::get_user_library($user_id)`

### For each config show:
- name
- type
- author
- created_at / updated_at
- share button only if own
- delete button
- update button only if imported and update available
- publish to market only if own
- if imported from market and source market item still active: copy market link

### Rules
#### own config
Allowed:
- share
- publish to market
- save/update
- delete

#### imported config
Allowed:
- load
- delete
- update from original if source still exists and newer

Not allowed:
- share
- publish to market

---

## 2) Share route

Current backend returns share links in format based on:

`SHARE_BASE_URL`

Example:
`http://domain/share/<token>`

### Required website route
Implement route:
`/share/{token}`

### Route logic
1. If user is not logged in:
   - store token in session
   - redirect to login page

2. After login:
   - use stored token
   - call `ConfigService::accept_share_link($user_id, $token)`

3. On success:
   - redirect to My Configs page
   - show success message

4. On failure:
   - show readable error

### Common error reasons
- `link_not_found`
- `link_inactive`
- `cannot_import_own_config`
- `name_taken`

---

## 3) Share preview page logic

Before accept, site may preview link using:

`ConfigService::get_share_link_info($token)`

Show:
- config name
- author
- version
- created_at
- uses_count

---

## 4) Imported config update logic

For imported configs:
- compare `source_version_applied`
- against original `config_library.version`

Use:
`ConfigService::update_imported_config($user_id, $config_id)`

### Website should show update button only when:
- config is imported
- source_config_id exists
- source config still exists
- source version > source_version_applied

### If source is deleted
- keep imported copy
- disable update button

---

## 5) Market list page

Use:
`ConfigService::get_market_list($limit, $offset)`

Show:
- title
- author
- rating
- rating_count
- comment_count
- import_count
- created_at

Only show active items.

---

## 6) Market details page

Use:
`ConfigService::get_market_item_details($market_item_id)`

Show:
- title
- description
- author
- config version
- rating stats
- comments
- import button
- rate form
- comment form

Comments list:
`ConfigService::get_market_comments($market_item_id, $limit, $offset)`

---

## 7) Publish to market

Only own configs can be published.

Use:
`ConfigService::publish_to_market($user_id, $config_id, $title, $description)`

### Publish rules
- config must exist
- config must belong to user
- item_type must be `own`
- imported configs must fail

Possible errors:
- `not_found`
- `cannot_publish_imported`
- `invalid_title`
- `description_too_long`
- `already_published`

---

## 8) Unpublish from market

Use:
`ConfigService::unpublish_from_market($user_id, $market_item_id)`

Rules:
- only owner can unpublish
- hidden item remains in DB, only `is_active = 0`

---

## 9) Import from market

Use:
`ConfigService::add_market_config_to_library($user_id, $market_item_id)`

Rules:
- active market item only
- cannot import own config
- cannot import if user already has config with same name

Errors:
- `not_found`
- `item_inactive`
- `cannot_import_own_config`
- `name_taken`

---

## 10) Rate market item

Use:
`ConfigService::rate_market_item($user_id, $market_item_id, $rating)`

Rules:
- rating must be 1..5
- one rating only
- cannot rate own item

Errors:
- `invalid_rating`
- `not_found`
- `item_inactive`
- `cannot_rate_own`
- `already_rated`

---

## 11) Comment on market item

Use:
`ConfigService::add_market_comment($user_id, $market_item_id, $comment)`

Rules:
- one comment per user per item
- no edit in current version
- cannot be empty
- max 500 chars

Errors:
- `invalid_comment`
- `not_found`
- `item_inactive`
- `already_commented`

---

## Recommended website wrappers

The website developer should create site-level wrappers/controllers using website session auth.

### Suggested wrappers
- `my_configs.php`
- `share_accept.php`
- `market_list.php`
- `market_view.php`
- `market_publish.php`
- `market_unpublish.php`
- `market_add.php`
- `market_rate.php`
- `market_comment.php`
- `config_update.php`

These wrappers should:
- verify website login
- map website user to backend `users.id`
- call `ConfigService`
- return HTML or JSON for the website UI

---

## Existing loader/Lua endpoints

These already exist and are used by Lua:

### Config library
- `/backend/configs/save.php`
- `/backend/configs/list.php`
- `/backend/configs/load.php`
- `/backend/configs/delete.php`

### Share
- `/backend/configs/share/create.php`
- `/backend/configs/share/info.php`
- `/backend/configs/share/accept.php`

### Imports
- `/backend/configs/imports/update.php`

### Market
- `/backend/configs/market/publish.php`
- `/backend/configs/market/unpublish.php`
- `/backend/configs/market/list.php`
- `/backend/configs/market/view.php`
- `/backend/configs/market/add.php`
- `/backend/configs/market/rate.php`
- `/backend/configs/market/comment_add.php`
- `/backend/configs/market/comments.php`

### Important
These endpoints are mostly designed for loader/Lua token flow, not for public browser use.

Preferred website integration:
- direct PHP service call
- or server-side private wrappers

---

## Admin panel

Admin panel has already been adapted to market moderation.

Current admin config tab works with:
- `config_market_items`
- comments moderation

Admin can:
- list market items
- hide/show item
- delete market item
- view comments
- delete comments

Admin does not manage:
- personal library configs
- imported user configs

---

## Security notes

1. Do not expose loader API key in frontend JavaScript.
2. Do not expose loader token/hwid in website JS.
3. Website should use server-side session auth.
4. Use direct `ConfigService` calls from PHP.
5. Keep `WTECH_BACKEND` define before including backend internal files.
6. Share links are public by token, so token secrecy matters.
7. Imported configs must never be allowed to publish/share as own.
8. Name uniqueness must be respected everywhere.

---

## Current limitations / future extensions

These may be added later:
- editable comments
- editable ratings
- multiple comments per user
- share link revoke UI
- market sorting/filtering
- screenshots/previews for market configs
- categories/tags

Current version keeps logic intentionally simple.

---

## Summary

### Done in backend
- unified config library
- share snapshots
- import update flow
- market items
- ratings
- comments
- admin market moderation
- Lua config integration

### Required from website developer
- implement `/share/{token}` route
- build My Configs page
- build Market pages
- use server-side PHP integration with `ConfigService`
- do not expose loader secrets in browser