# Gmail backend

Gmail is read and written through an authenticated Gmail connector. Select one complete adapter and use it for the whole run:

- **Codex Gmail:** `gmail_get_profile`, `gmail_search_emails`, `gmail_read_email_thread`, `gmail_list_labels`, `gmail_create_label`, `gmail_apply_labels_to_emails`, `gmail_batch_modify_email`, and `gmail_create_draft`. `gmail_read_attachment` is optional.
- **Claude/Google Gmail:** `search_threads`, `get_thread`, `list_labels`, `create_label`, `label_message`, `unlabel_message`, and `create_draft` from the claude.ai connector or Google's Gmail MCP server (`gmailmcp.googleapis.com`).

The connector supplies authentication. If neither complete adapter is available, report that no compatible Gmail connector is attached. Use only the selected adapter: the allowed writes are label creation, labeling, removal of `INBOX`, and draft creation.

## Fetch

### Codex Gmail

1. Call `gmail_get_profile` and keep its `email` as the authenticated mailbox address.
2. Call `gmail_search_emails` with:
   - `query`: `in:inbox -category:promotions -category:social newer_than:1d`
   - `max_results`: `50`
3. Repeat with `next_page_token` until the response has no next token.
4. For each search result, call `gmail_read_email_thread` with its `id` as `message_id` and `max_messages: 20`. Deduplicate messages by ID across thread results.

For other windows, replace `newer_than:1d` with `newer_than:7d` or `after:YYYY/MM/DD`; add `is:unread` when requested. Keep only messages whose `label_ids` contain `INBOX` and whose `internal_date` is inside the requested window. Read sender, recipients, subject, and date from the payload headers; read the plain-text body from the MIME payload. Record each message's `id`, `thread_id`, read state (`UNREAD` in `label_ids`), and attachment metadata.

### Claude/Google Gmail

Call `search_threads` with:

- `query`: `in:inbox -category:promotions -category:social newer_than:1d`
  - Other windows: `newer_than:7d`, or `after:YYYY/MM/DD`.
  - Unread only: add `is:unread`.
- `pageSize`: 50

Repeat with each returned `nextPageToken` until none is returned. An empty `{}` means no matches.

For each thread, call `get_thread` with `messageFormat: PLAIN_TEXT`. Keep only messages that are in the inbox (`label_ids` contains `INBOX`) and inside the window — a thread can include older or already-filed messages. Record each message's `id`, sender, subject, date, read state (`UNREAD` in `label_ids`), plain-text body, and attachment names.

## Attachments

With Codex Gmail, call `gmail_read_attachment` only when the selected attachment says `read_attachment_supported: true`; pass the parent message ID and the exact attachment ID, or the exact filename when no complete attachment ID is available. Inspect the returned content or file before classification. With the Claude/Google adapter, or for an unsupported attachment, classify from the body and attachment names, mark the item unverified, and say the attachment could not be read.

## File

Label first, then remove `INBOX`, so a failure never leaves a message unfiled and absent from the inbox.

### Codex Gmail

1. Call `gmail_list_labels` once. For each missing group label, call `gmail_create_label` with its `name`. Record labels created during this run.
2. Group the fetched message IDs by destination label. For each group, call `gmail_apply_labels_to_emails` with `message_ids`, `add_label_names: [<group label name>]`, and `create_missing_labels: false`.
3. After that call succeeds, call `gmail_batch_modify_email` for the same message IDs with `remove_labels: ["INBOX"]`. This operates on individual messages; do not use the thread-wide archive tool.

### Claude/Google Gmail

1. Call `list_labels` once to map label names to IDs.
2. For each group label that doesn't exist yet, call `create_label` with `displayName` set to the label name. Nested names like `Triage/P1` work; parents are created automatically.
3. For each message, call `label_message` with `messageId` and `labelIds: [<group label id>]`, then `unlabel_message` with `messageId` and `labelIds: ["INBOX"]`.

If a message is no longer found, report its sender and subject, skip it, and keep filing the rest.

## Draft

Use the `[gmail]` section's `primary_smtp_address` when configured; otherwise use the authenticated address returned by Codex `gmail_get_profile` or the mailbox address returned by the Claude/Google connector.

With Codex Gmail, call `gmail_create_draft` with:

- `to`: the mailbox address above
- `subject`: `Triage Report: Gmail — <YYYY-MM-DD>`
- `payload`: `{ "mime_type": "text/plain", "charset": "UTF-8", "body": { "content": "<report>" } }`

With Claude/Google Gmail, call `create_draft` with:

- `to`: the mailbox address above
- `subject`: `Triage Report: Gmail — <YYYY-MM-DD>`
- `body`: the report as plain text (no Markdown)

This only saves a draft. Don't label or move it.

## Failures

- **Rate limit or quota error:** stop, and report the exact error. Don't retry in a tight loop.
- **Any other tool error:** report the exact error and which step it happened in.
