# Whop Ads

Read this before creating or launching ads. Enums and field lists live in each
tool's own schema — do not invent them. When a field here differs from the schema,
the schema wins.

Whop Ads runs the ads on Whop's own ad accounts and bills spend to the business's
ads payment method (its Whop balance or a card). Drafts cost nothing and need no
billing setup. Launching spends money, and these tools act on the first call. Before any
launch, unpause, or budget change, say what will happen, with the budget, and wait
for the user's yes. Then call the tool once.

## Before launching

- **Payment method.** Launching needs an ads payment method on the account. It is
  set in the user's Whop dashboard (Ads settings, Billing); this plugin cannot set
  it. If a launch fails with an error about an ads payment method, such as
  `Account must have an ads payment method configured`, keep the campaign as a draft
  and tell the user to add or update it there.
- **Destination.** A website ad needs a `url`. The business's whop.com store page
  works as-is, and so does a site Whop hosts on a whop.site address, which already
  has the pixel. Any other destination needs the Whop pixel on the page, and no tool here checks
  for it. `create_ad` and `update_ad` refuse a `url` with no pixel, even on a draft:
  a 400 saying the pixel was not detected means send traffic to the store page or have
  the user install the pixel. A draft can leave out `url` until then.
- **Identity and agreement.** A launch also needs an approved identity verification,
  done by an owner of the business, and a signed Whop Ads agreement. Saving ads billing
  opens the agreement. Both are done in the user's Whop dashboard and this plugin
  cannot do them, so say so in words with no link. Read `capabilities.run_ads` from
  `get_account` before asking for the launch yes. Inactive means the launch will be
  refused, so say that and keep the draft. It can be null, and it reads active before
  billing is set up, so if a launch fails asking for either one, keep the draft and
  send the user to the dashboard.

## Creatives

An ad needs at least one image or video file id (`file_…`) in `creatives`.

This plugin cannot upload a file or generate an image or video, so `create_ad` works
only with a file id Whop already holds. If the user has none, say that a new ad with
new creative is made in their Whop dashboard. To reuse what is already there,
`duplicate_ad` copies an existing ad with its creative (the copy keeps the source's
status, see "Manage what exists"), and `update_ad` changes its text, link and call
to action.

## Create, then launch

Build the campaign as a draft first, so nothing spends while the user reviews it.
Pass an `idempotency_key` on `create_ad` and `create_ad_group`, and reuse it if a
result is unclear. With no key, check `list_ads` and `list_ad_groups` before calling
again: a second call can add a second ad group with its own budget.

1. `create_ad_campaign` with `account_id`, `title`, and `objective`. It always
   creates a `draft`.
2. `create_ad` with an inline `ad_group` that points at the draft campaign. The ad
   group carries the targeting and, by default, the budget. On a draft campaign
   this launches nothing.

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

3. **Say it, get a yes, then launch.** Show the campaign, the daily or lifetime
   budget in USD, the targeting, and the destination URL. For a new ads payer, also say
   that a balance or platform credits must cover the larger of $10 and one day's
   budget, or a card gets a temporary authorization of about $10 for each campaign
   launched, released after about a day. It is not a charge. Wait for a clear yes.
   Then call `update_ad_campaign` with `status: "active"`. That call launches the
   ads, so never make it to see a preview.

- To add an ad to an existing ad group, pass `ad_group_id` instead of `ad_group`.
  Adding to a campaign that is not a draft launches the new ad at once, so get a
  yes first.
- Budget: one level owns it. `budget_amount` on the ad group (default), or on the
  campaign with `budget_optimization: "ad_campaign"`, never both. Budgets are USD.
- Omit `demographics` and `devices` for automatic optimization. `regions` takes
  `include` and `exclude` objects: countries as ISO 3166-1 codes in `countries`
  (`"US"`), US states as ISO 3166-2 codes in `regions` (`"US-CA"`).
- `creatives`: for a single-image or single-video ad, pass exactly one entry with no
  `format` (the base asset), plus optional crops `square`, `vertical`, and
  `horizontal`, each at most once. Two or more entries with no `format` make a
  carousel of 2-10 cards; use that only when the user asks for a carousel, not to
  offer alternatives.

## Manage what exists

| Job | Tools |
| --- | --- |
| Read | `list_ad_campaigns`, `get_ad_campaign`, `list_ad_groups`, `get_ad_group`, `list_ads`, `get_ad` |
| Stop delivery | `pause_ad_campaign`, `pause_ad_group`, `pause_ad` |
| Resume delivery | `unpause_ad_campaign`, `unpause_ad_group`, `unpause_ad`: spends money, so get a yes first |
| Change budget or targeting | `update_ad_campaign`, `update_ad_group`: say what changes and get a yes first |
| Edit an ad | `update_ad` — confirm first when the ad is live |
| Copy | `duplicate_ad_campaign` (copies become drafts); `duplicate_ad_group`, `duplicate_ad` keep the source's status, so a copy of something live spends money: get a yes first |
| Remove | `delete_ad_campaign` (cascades to its ad groups and ads), `delete_ad_group`, `delete_ad` — confirm first |

- `update_ad` with `creatives` replaces the whole set on the ad — keep exactly one
  entry with no `format` unless the user wants a carousel.
- `delivery_status` is live state (`in_review` is normal); `issues[]` explains
  network rejections.
- **Failed ads payment.** `delivery_status: "payment_failed"` blocks every campaign
  on the account. The user fixes the payment method and retries the payment in their Whop dashboard.
  No tool here retries it. Check the campaigns again afterwards.
