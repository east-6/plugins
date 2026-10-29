# Whop Ads

Read this before creating or launching ads. Enums and field lists live in each
tool's own schema — do not invent them. When a field here differs from the schema,
the schema wins.

Whop Ads runs the ads on Whop's own ad accounts and bills spend to the business's
ads payment method (its Whop balance or a card). Drafts cost nothing and need no
setup. Launching spends money, so every launch, unpause, and budget change is a
two-step tool: show the preview with the budget and wait for the user's yes.

## Before launching

- **Payment method.** Launching needs an ads payment method on the account. It is
  set in the user's Whop dashboard (Ads settings, Billing); this plugin cannot set
  it. If a launch fails with `Connect an ads payment method before launching`, keep
  the campaign as a draft and tell the user to add one there.
- **Destination.** A website ad needs a `url`. The business's whop.com store page
  works as-is, and so does a site Whop hosts on a whop.site address, which already
  has the pixel. For any other destination, check it with `events_validate_pixel`
  and the `url` before launching. An external page needs the Whop pixel first; a 400 saying the pixel was
  not detected means send traffic to the store page or have the user install the
  pixel.

## Creatives

An ad needs at least one image or video file id (`file_…`) in `creatives`, or, if
the schema has it, an `existing_post_id` instead of `creatives`.

- **The user's own file:** `files_create` with the `filename` and
  `visibility: "public"`, then PUT the bytes to the returned `upload_url` with its
  `upload_headers` using your own tools, then poll `files_get` until
  `upload_status` is `ready`, and use its `id`. `files_complete` is only for a
  `multipart` upload. Uploading needs a workspace that can make HTTP requests, such
  as Codex. In a ChatGPT conversation without one, say the user's own file has to be
  uploaded from Codex. Mention AI images only if the user asks for one.
- **A post the business already published:** only if the `ads_create` schema has
  `existing_post_id`. Find the business's Facebook page or Instagram account with
  `external-accounts_list` (`platform`), list its posts with
  `external-accounts_posts`, and pass that post's id as `existing_post_id`, with
  `post_source` (`facebook` or `instagram`), instead of `creatives`.
- **An AI image or video:** `media_generate` with a prompt. Use `type: "image"`
  unless the user asks for a video, and for a video keep the default
  `duration_seconds` and `resolution` unless the user asks for more. It bills the
  Whop balance, and the exact amount (`amount_charged`) is known only after it runs:
  say so, and get a yes to the preview (it is two-step). Poll `media_get` until
  `status` is `ready`, then use `file.id`. A `failed` job refunds the charge. A 402
  means the balance is too low: tell the user to add funds in their Whop dashboard,
  and do not pass on the deposit link.

## Create, then launch

Build the campaign as a draft first, so nothing spends while the user reviews it.

1. `ad-campaigns_create` with `account_id`, `title`, and `objective`. It always
   creates a `draft`. If the schema asks for `platform`, use the one value it lists.
2. `ads_create` with an inline `ad_group` that points at the draft campaign. The ad
   group carries the targeting and, by default, the budget. On a draft campaign
   this launches nothing; the tool is still two-step, so show its preview and wait
   for a yes before the second call.

```json
{
  "title": "Launch ad",
  "primary_texts": [{ "text": "Join 10,000 runners chasing their next PR together." }],
  "headlines": [{ "text": "Find your stride" }],
  "call_to_action": "sign_up",
  "url": "https://whop.com/your-store",
  "creatives": [{ "id": "file_x" }],
  "ad_group": {
    "ad_campaign_id": "adcamp_x",
    "title": "US broad",
    "budget_amount": 25,
    "budget_type": "daily",
    "regions": { "include": { "countries": ["US"] } }
  }
}
```

3. **Preview, yes, then launch.** Call `ad-campaigns_update` with
   `status: "active"`; this first call is only a preview. Show it with the
   campaign, the daily or lifetime budget in USD, the targeting, and the
   destination URL. Wait for a clear yes, then call it again with the token.

- To add an ad to an existing ad group, pass `ad_group_id` instead of `ad_group`.
  Adding to a campaign that is not a draft launches the new ad at once.
- If the schema has `external_accounts`, the ad needs the business's Facebook page:
  find it with `external-accounts_list`. Creating a new page
  (`external-accounts_create`) makes a public page, so confirm first.
- Budget: one level owns it. `budget_amount` on the ad group (default), or on the
  campaign with `budget_optimization: "ad_campaign"`, never both. Budgets are USD.
- Omit `demographics` and `devices` for automatic optimization. `regions` takes
  `include` and `exclude` objects: countries as ISO 3166-1 codes in `countries`
  (`"US"`), US states as ISO 3166-2 codes in `regions` (`"US-CA"`).
- `creatives`: for a single-image or single-video ad, pass exactly one entry with no
  `format` (the base asset; one per language when the ad has `translations`), plus optional crops `square`, `vertical`, and
  `horizontal`, each at most once. Two or more entries with no `format` make a
  carousel of 2-10 cards; use that only when the user asks for a carousel, not to
  offer alternatives.

## Manage what exists

| Job | Tools |
| --- | --- |
| Read | `ad-campaigns_list`, `ad-campaigns_get`, `ad-groups_list`, `ad-groups_get`, `ads_list`, `ads_get` |
| Plan targeting | `ad-groups_targeting_options`, `ad-groups_estimate_reach` (nothing is saved) |
| Stop delivery | `ad-campaigns_pause`, `ad-groups_pause`, `ads_pause` |
| Resume delivery | `ad-campaigns_unpause`, `ad-groups_unpause`, `ads_unpause` — two-step, spends money |
| Change budget or targeting | `ad-campaigns_update`, `ad-groups_update` — two-step |
| Edit an ad | `ads_update` — confirm first when the ad is live |
| Copy | `ad-campaigns_duplicate` (copies become drafts); `ad-groups_duplicate`, `ads_duplicate` keep the source's status — two-step |
| Remove | `ad-campaigns_delete` (cascades to its ad groups and ads), `ad-groups_delete`, `ads_delete` — confirm first |

- `ads_update` with `creatives` replaces the whole set on the ad — keep exactly one
  entry with no `format` (one per language when the ad has `translations`) unless
  the user wants a carousel.
- `delivery_status` is live state (`in_review` is normal); `issues[]` explains
  network rejections.
- **Failed ads payment.** `delivery_status: "payment_failed"` blocks every campaign
  on the account. The user fixes the payment method in their Whop dashboard first.
  Then `accounts_retry_ads_payment` once for the account (two-step: it charges the
  ads payment method). It queues a retry and does not confirm payment; check the
  campaigns again afterwards. Do not retry per campaign.
