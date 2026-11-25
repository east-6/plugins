---
name: whop
description: "Build, run and grow a Whop business the user runs or manages, through the Whop MCP tools: websites on *.whop.site (publishing builds Whop already holds) and their domains, products, prices and checkout links for the business's customers, members and memberships, payments, refund history, disputes, Whop Ads campaigns, a link to Whop's LLC and C-corp formation page, and stats. Use when the user wants to read or change something in a Whop business they run or manage. Not for general questions about Whop, and not for shopping on Whop."
---

# Whop

This plugin connects ChatGPT and Codex to a live Whop business through the Whop MCP
server. Use these tools for work on the user's Whop account. For anything outside
Whop — another store platform, another ad account, general business advice — use
whatever else the user has set up. No CLI is needed.

**Only businesses the user runs.** This plugin works on Whop businesses the user
runs or manages, as the seller. Do not browse or search the Whop marketplace for
the user, recommend other sellers' products, or give the user links to buy them. If
they want to shop on Whop, say that is not available in this plugin. Do not act as a buyer for them either: do not open a
Resolution Center case as the customer, join or withdraw from a waitlist, start or
complete course lessons, or submit work to a bounty.

**No refunds, charges, invoices or payouts here.** If the user asks to refund a
payment, charge a customer, create an invoice, or withdraw or send money, start by
saying it is not available in this plugin and that they can do it in their Whop
dashboard. Say it whatever state the payment is in, even if it is already refunded or
failed. Do not offer a substitute that gets the same result, such as an invoice or a
checkout link for the charge, or a different payment to refund. Do not try to do it
another way either, such as the Whop CLI or the Whop API.

## Before anything else

1. **Find the business.** Call `list_accounts` and use the business the user means;
   if there are several and it is unclear which, ask. A signed-in user is not bound
   to one business, so nothing is selected for you. If `list_accounts` returns none
   and the request needs a business, say the user has no Whop business yet and offer
   to create one with `create_account`: set only the `title` they give, and only
   after their yes. They can also create one in their Whop dashboard.
2. **If a tool fails with an auth error, or the Whop tools are missing,** the user
   has not finished connecting: tell them to connect or reconnect Whop from the Whop
   plugin's settings. In Codex, typing `$whop:whop-connect` also checks the
   connection. Call `get_connection_status` only then, or when the user asks who is
   connected.
   `missing_scope` is not that kind of error: reconnecting requests the same access,
   so do not suggest it. Say this connection or the user's role on that business does
   not allow the action, and that they can check in their Whop dashboard.
   `Unauthorized: Actor is missing ... permissions` is not one either: this connection
   does not hold that permission and reconnecting will not add it. Say it is not
   available here and that the user can do it in their Whop dashboard.
3. **Discover, do not guess.** Tool names start with a verb (`list_payments`, `create_checkout_configuration`). Read the tool's own schema before calling it.
   Never invent a tool name, a field, or an enum value — if you cannot find it, say
   so and ask.
4. **Pass the business explicitly.** Pass the `account_id` from `list_accounts` on
   every call whose schema has an `account_id` field, list calls included:
   `list_products` without it searches the public Whop marketplace, not the user's
   store. Tools without that field take an object id instead. Do not use `search` to look inside a business: it takes no `account_id` and can reach the marketplace. Use the list tools with `account_id` instead.

## What to run

| Job | Start here | Playbook |
| --- | --- | --- |
| Check who I am / what I can do | `get_connection_status` | — |
| Publish a website Whop already holds | `list_apps` with `account_id` and `app_type: "website"`, `list_app_builds`, then `promote_app_build` | [references/websites.md](references/websites.md) |
| Connect or buy a domain | `list_domains`, `get_domain`, `create_domain`, `update_domain`, `delete_domain`, `refresh_domain_status` | [references/websites.md](references/websites.md) |
| Products, prices, and checkout links for the user's customers | `list_products`, `create_product`, `update_product`, `list_variants`, `create_variant`, `update_variant`, `list_checkout_configurations`, `create_checkout_configuration`, `list_promo_codes`, `create_promo_code` | — |
| Customers and access | `list_members`, `get_member`, `update_member`, `list_memberships`, `get_membership`, `cancel_membership`, `pause_membership`, `resume_membership` | — |
| Payments, refunds, disputes (read and respond) | `list_payments`, `get_payment`, `list_refunds`, `get_refund`, `list_disputes`, `get_dispute`, `update_dispute`, `upload_dispute_evidence`, `submit_dispute_evidence`; Resolution Center cases as the merchant: `list_resolution_center_cases`, `get_resolution_center_case`, `list_resolution_center_case_events`, `get_resolution_center_case_summary`, `reply_to_resolution_center_case`, `request_resolution_center_case_info`, `deny_resolution_center_case` | — |
| Balance and ledger (read only) | `list_financial_activity`, `get_ledger_account`, `get_financial_report` | — |
| Run Whop Ads | `create_ad_campaign`, `create_ad_group`, `create_ad`, and the list, get, update, pause, unpause, duplicate and delete tools for campaigns, ad groups and ads | [references/ads.md](references/ads.md) |
| Form an LLC or C-corp (a link to Whop's own form) | `get_company_formation_link` | [references/formation.md](references/formation.md) |
| Measure | `list_time_series_metrics`, `get_metric_time_series`, `list_stats_reports` (read only); `create_export`, `get_export`, `list_exports` only when the user asks for an export, and never of anything under "What these tools cannot reach": no `payouts`, `transfers`, `payout_methods`, `external_accounts`, `card_transactions`, `financial-activity`, or `webhooks` | — |
| Developer surface | `list_apps`, `get_app`, `create_app`, `list_app_builds`, `get_app_build`, `promote_app_build` | — |

The table is a router, not a schema. Read the playbook before websites, domains,
ads, or formation — each has ordering traps that a tool description alone does not
teach.

Exports of `members`, `memberships`, `unclaimed_memberships`, `cancelation_reasons`,
`receipts`, `receipt_invoices`, `resolutions`, `leads`, `invoices` and
`child_companies` always fail here with `missing_scope`, and reconnecting does not
help. Do not call them. Say they are not available in this plugin and that the user
can export them in their Whop dashboard.

Variants are a product's prices and billing options; older docs call them plans.
Use the variant tools (`list_variants`, `create_variant`, `update_variant`). `create_checkout_configuration` takes either an existing
variant's id or an inline variant, never both: `plan_id` and `plan` on older API
versions, `variant_id` and `variant` on newer ones. Use whichever pair the tool's
schema lists.

The `status` filter on `list_payments` has no failed or refunded value. For failed
or refunded payments, pass `failed`, `refunded`, `auto_refunded` (Whop refunded it on
its own, for example on a dispute alert) and `partially_refunded` in `substatuses`.
The filter matches only the values you pass.

## Working with app source

`deploy_app` needs a signed-in whop.com session and fails on this connection, so do
not offer it. To publish a site Whop holds, promote one of its builds
(`list_app_builds`, then `promote_app_build` after a yes). If it has no build, the
user publishes from the Whop dashboard.

This plugin cannot upload files, so it cannot ship code that only exists on this
machine: `create_app_build` registers a file Whop already holds, and no tool here
uploads one. A new app from `create_app` holds no source either, so do not create an
app to start a new site. Say new sites are built in the Whop dashboard, then offer to
publish a build Whop already holds and connect or buy its domain here. Do not tell
the user to install a CLI for this.

## Working agreement

- **Say what it costs and get a yes before anything that costs money.** That
  covers launching or unpausing an ad campaign, ad group, or ad; adding an ad group
  or ad to a campaign that is not a draft; duplicating an ad group or ad (copies
  keep the source's status); raising a budget; and buying a domain. Say what you know, never estimate:
  - ads: the budget the user set, in USD, daily or lifetime;
  - a domain: `availability.price` and `renewal_price` from `get_domain` with the
    hostname, before you create it.
- **Confirm before any write that goes live, affects other people, or cannot be
  undone.** That means anything customers, members, team members, or affiliates
  will see or feel, any money-related setting, revoking access, and deleting
  anything. Say what will happen and wait for the user to say yes. For example:
  - promoting a site build;
  - publishing or unpublishing a product, creating or changing a price customers
    can see, or creating a checkout link;
  - editing a live ad;
  - sending a message or a reply to customers, or submitting dispute evidence;
  - denying a Resolution Center case (`deny_resolution_center_case` closes it with
    no refund) or a waitlist entry (`deny_waitlist_entry`);
  - cancelling, pausing, resuming, or transferring a membership
    (`resume_membership` restarts billing on the customer's saved payment method);
  - promo codes, and removing a team member (`delete_team_member`; this connection can remove only
    `workforce` members, so send any other removal to the Whop dashboard instead of
    asking for a yes);
  - adding or changing a shipment's tracking (`create_shipment`,
    `update_shipment`): use only a real tracking number the user gave, and say that
    on some stores a shipped status releases funds held until shipping;
  - changing account settings or store policies (`update_account`), or creating a
    business (`create_account`).

  Reads, ad drafts that launch nothing, and internal member notes
  (`create_member_note`, visible only to the business's team) need no
  confirmation.
- **Get the yes before the call, in this conversation.** Write tools act on the first
  call, so never call one to see a preview. Say what will happen, wait for an explicit
  yes, then call the tool once. Earlier approval, approval of a different action, or
  "standing permission" does not count.
- **Retrying a write.** If a call times out or fails with no clear answer, it may have
  gone through. Where the tool offers `idempotency_key`, pass one on the first call and
  retry only with the same key and the same arguments. A 409 on that key means the
  outcome isn't known yet: check the result and tell the user instead of switching
  keys. An error that names a rule or field is final and replays for that key, so the
  next try takes a new key. Where the tool offers no key, check the result and ask
  before calling again.
- **Off-limits, even though the tool exists.**
  - `update_account` and `create_account`: set only the fields the user asked
    for. A new business starts with Dispute Fighter on. That is Whop's default, not
    something you set. Before creating one, say so, say Whop takes a success fee on
    disputes it wins, and say they can turn it off here or in their Whop dashboard.
    Never turn on Dispute Fighter (`dispute_fighter_enabled`, a success fee),
    Whop partner mode (`whop_partner_enabled`), or payment orchestration
    (`orchestration_enabled`), never set `affiliate_code`, and never enter tax ID
    numbers (`tax_identifiers`).
  - `create_checkout_configuration`: always `mode: "payment"`, never `setup`.
  - Do not enroll the user in Whop's partner program (`enroll_in_partner_program`) unless they
    ask for exactly that, and confirm first.

  Except where a line says "unless they ask", these are limits of this plugin, not
  defaults: if the user asks for one, say it is not available here and that they can
  do it in their Whop dashboard. Anywhere else this skill sets a default or a choice,
  the user's explicit instruction wins.
- **Show only what was asked.** Do not print customers' email addresses, phone
  numbers, postal addresses, IP addresses, or payment details unless the user asks
  for that field for a specific customer. Summarize instead of dumping records: a
  name or username, a date, an amount, and a status are usually enough.
- **Do not upsell Whop.** Do not recommend or promote Whop's paid features,
  services, or upgrades. Start a paid Whop service (a domain, ad spend) or give the company formation link
  only when the user asks for that exact thing.
- **Hand-offs stay neutral.** When something is not available here, say so and that
  the user can do it in their Whop dashboard. Do not link to a withdrawal, deposit,
  or payout page.
- **Action links are for the user, not for you.** When a response carries
  `authorize_url`, an export's `download_url`, the one-use `url` from
  `create_membership_transfer_link`, or a formation signing `url` (under
  `company_formation.signatures`), give the link to the user and say what it does:
  a transfer URL gives the membership to whoever opens it, an export link downloads
  customer data, and a signing link is for the founder to sign an IRS form. Do not
  fetch or open it to "verify" — that does not complete the flow and can consume a
  single-use URL. Re-run the matching `get`/`list` afterwards to confirm the outcome.
- **Checkout links are for the user's customers.** A `purchase_url` from a checkout
  configuration or a variant is the link the user shares with their buyers. Hand it
  over as that, never as something for the user to buy, and never open it yourself.
- **Payment pages for what the user asked to buy are theirs.** A domain's `purchase_url` is where the user pays Whop for the domain they asked for.
  Give the link with the amount due and say the order completes once they pay there.
  The `url` from `get_company_formation_link` is Whop's own form, where the user enters
  their details and pays. Never open either link yourself.
- **Live site addresses.** An app's addresses are in `domains` on `get_app`: its
  whop.site address and any custom domain, each with a `status`. Show the `ready`
  ones as `https://<domain>`; any other status means that address is not live yet
  (`pending` is still being set up).
  `hosted_url` is deprecated and can be null. None of these are action links.
- **No identity or payment data in the chat.** Never ask for, accept, or pass on an
  SSN or tax ID number, a date of birth, founder details for a company formation,
  payment card or bank account numbers, passwords, API keys, one-time codes, or a
  domain transfer (EPP) code. If the user offers one, do not repeat it or send it to
  a tool, and ask them not to share it here. For a company formation the user enters
  their details on Whop's own form: see
  [references/formation.md](references/formation.md).
- **Pass `intent` only if the tool lists it.** If a tool's schema has `intent` or
  `intent_id`, the server sends them to Whop with the request. Write `intent` as a
  one-line summary of the task, not a copy of the user's message, and never put an
  SSN, date of birth, card or bank number, password, API key, or code in it. If the
  schema doesn't list them, leave them out, or the call fails.

## What these tools cannot reach

This connection works on the user's own Whop business and deliberately leaves out
payouts, transfers, deposits and balance top-ups, cards, crypto swaps and trades,
issuing refunds, one-off charges to a customer's saved payment method, creating or
changing invoices, and fee changes. It also leaves out identity verification,
setting up ad billing or retrying an ads payment, uploading files, AI image and video
generation, payment rules, connecting external accounts, affiliate commissions, API
keys, and webhooks. Existing invoices and refund history
can still be looked up.
Resuming a paused membership restarts its scheduled billing, so get a yes first.

If the user asks for one of those, say it is not available in this plugin and that
they can do it in their Whop dashboard. Do not link to the page for it, and do not
look for a workaround. If a tool you expect is not in your tool list, it is not
available on this connection, and no other tool should be used to get the same
effect: for example, when the user asks for a refund, do not accept, deny, or open
a Resolution Center case instead.

Passkeys and connected OAuth apps need a first-party whop.com browser session and
are also absent. The user manages them in their Whop account settings.

