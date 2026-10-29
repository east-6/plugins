{{#if openai}}
# Websites and apps (`*.whop.site`)
{{else}}
# Websites and apps (`*.whop.app`)
{{/if}}

Read this before creating or deploying an app. Field lists and enums live in each
tool's own schema — do not invent them.

## Create the app

{{#if openai}}
A new app from `apps_create` holds no source, so `apps_deploy` on it has nothing to
publish, and its preview does not warn you. In a ChatGPT conversation without a
workspace, do not call `apps_create` to start a new site. Say the site has to be
built in Codex or in the Whop dashboard, then offer to deploy it and connect or buy
its domain here. In Codex, create the app and ship the build as described under
"Shipping code that only exists locally".

{{/if}}
`apps_create` takes only `name` as required. The fields that matter for a hosted
site:

| Field | Notes |
| --- | --- |
| `name` | Required. Shown to users on the app store. |
{{#if openai}}
| `app_type` | `website` (visitors browse `<route>.whop.site`), `b2c_app` (creators install it), plus `b2b_app`, `company_app`, `component`. Ask; do not guess. |
{{else}}
| `app_type` | `website` (visitors browse `<route>.whop.app`), `b2c_app` (creators install it), plus `b2b_app`, `company_app`, `component`. Ask; do not guess. |
{{/if}}
{{#if openai}}
| `domain_id` / `route` | Where the site is served. `domain_id` is a domain already on the business (`dom_` tag, from `domains_list`). `route` names a free `<route>.whop.site` address; the schema marks it deprecated in favor of `domain_id`, so use it only when the business has no domain for the site. |
| `account_id` | `biz_` tag. Optional in the API, but always pass the business id from `accounts_list`: a signed-in user is not bound to one business. |
{{else}}
| `route` | The subdomain the hosted build is served from. A hosted app with no route cannot go live. |
| `account_id` | `biz_` tag. Defaults to the account behind the credential. |
{{/if}}
| `base_url` | Only for an app you host yourself elsewhere. |

Ask which `app_type` before creating. The choice shapes how the app is surfaced, and
changing a website back to an app is not a supported move.

## Deploy

```
apps_deploy(id: "app_xxx", draft: false)
```

`apps_deploy` **builds the app's current source on Whop and ships it.** Nothing local
is required to deploy source Whop already holds.

- Only one deployment runs per app at a time. Calling it again while one is in flight
  reports that run rather than starting a second.
- Calling it with nothing to publish says so rather than starting a run.
{{#if claude}}
- `draft: true` uploads the build without making it live. The default deploys and
  promotes in one step, so ask the user before a non-draft deploy.
{{#elif openai}}
- `draft: true` uploads the build without making it live. The default deploys and
  promotes in one step. `apps_deploy` is two-step either way: show the preview, say
  whether it goes live, and wait for a yes.
{{else}}
- `draft: true` uploads the build without making it live. The default deploys and
  promotes in one step.
{{/if}}
- It returns the run it started. Follow progress on the app's `deployment` field via
  `apps_get`.

## Shipping code that only exists locally

`apps_deploy` builds what Whop already has. To get new local code up, do the local
half with your own tools and the rest over MCP:

1. Write and build the files yourself.
{{#if openai}}
2. Upload the archive: `files_create` (with `filename`), PUT the bytes to the
   returned `upload_url` with your own tools, then `files_get` until
   `upload_status` is `ready`. `files_complete` is only for a `multipart: true`
   upload. Web takes a JavaScript file or a `.zip` of the hosted site.
{{else}}
2. `files_create`, then `files_complete` for the archive. Web takes a JavaScript file
   or a `.zip` of the hosted site.
{{/if}}
3. `app-builds_create` referencing the uploaded file — `attachment` takes `{ id }`
   for an existing file or `{ direct_upload_id }` for a completed direct upload.
{{#if claude}}
4. Ask the user, then `app-builds_promote` to make that build live.
{{#elif openai}}
4. Ask the user, then `app-builds_promote` it (two-step). A new build is a draft,
   and promoting a draft sends it to review first; it goes live once approved.
   Check `app-builds_get` for its status.
{{else}}
4. `app-builds_promote` to make that build live.
{{/if}}

Promoting an older build rolls back to it.

Do not tell the user to install a CLI for any of this.
{{#if openai}}

Steps 1 and 2 need a workspace where you can write, build, and upload files, such as
Codex. In a ChatGPT conversation without one, deploy only what Whop already holds.

## Domains

`domains_*` puts a custom domain on the business. Pass `account_id` on
`domains_create` and `domains_list`. `domains_get`, `domains_update`, and
`domains_delete` take the domain's `id` (`dom_…` or the hostname); `domains_check`
needs the `dom_` id, so look it up with `domains_get` first if you only have the
hostname. `domains_create`, `domains_update`, and `domains_delete` are two-step:
show the preview and wait for a yes.

- **Connect a domain the user already owns:** `domains_create` with `domain` and
  `verification: {}` (add `website` with the app's `app_id` if it should serve a
  site, so every record comes back at once). Its `issues` list the TXT and routing
  records the user publishes at their registrar. Tell them to publish them within
  48 hours (`verification.verification_expires_at`), or the claim is dropped. Then
  `domains_check` and `domains_get` to see the result.
- **Buy a domain:** first look up the price. `domains_get` with the hostname
  returns `availability` for a domain nobody has: `price` is the first year and
  `renewal_price` each year after. `domains_list` with `search` (and `tlds`)
  suggests names. Then call `domains_create` with the root `domain` and
  `registration: {}`; this first call is only a free preview. Show that preview in
  the same message as both prices and the note that it renews yearly while
  `auto_renew` is on. The user's yes to that preview is the one approval; then make
  the second call with the token. Passing no capability also means `registration`,
  so always name the capability you want. Never pass
  `registration.payment_method_id`: the user pays at the returned
  `registration.purchase_url`. Give them that link with `registration.amount_due`,
  say Whop registers the domain once they pay, and do not open the link.
- **Serve a website on it:** `website` with the app's `app_id`, on create or with
  `domains_update`.
- **Transfer from another registrar:** that needs the registrar's transfer (EPP)
  code, which this plugin does not take. The user starts transfers in their Whop
  dashboard.
- **Remove:** `domains_delete` removes a connected domain or cancels an unpaid
  purchase. A bought domain cannot be deleted: turn off `registration.auto_renew`
  with `domains_update` and Whop releases it when it expires.
{{/if}}

## Reading state

- `apps_get` — live state, including `deployment`.
{{#if openai}}
- `apps_list` with `account_id` lists the account's apps but leaves out websites.
  Pass `app_type: "website"` to list its websites; to answer "what do I have",
  call it both ways.
- `app-builds_list` / `app-builds_get` — build history.
- Runtime logs are not available on this connection; the user reads them in their
  Whop dashboard.

An app's addresses are in `domains` on `apps_get`: its whop.site address and any
custom domain, each with a `status`. Show the `ready` ones as `https://<domain>`;
any other status means that address is not live yet.
`hosted_url` is deprecated and can be null. None of these are action links.
{{else}}
- `apps_list` — every app on the account.
- `app-builds_list` / `app-builds_get` — build history.
- `apps_logs` — runtime logs.

`hosted_url` on an app is the live URL. Display it; it is not a "finish this step"
action link.
{{/if}}
