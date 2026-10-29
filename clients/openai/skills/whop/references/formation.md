# Company formation

The tool is `accounts_form_company`. State enums and founder field lists live in its
schema — do not invent them.

It creates a **hosted checkout**. It does not charge and it does not file. The
response to the confirmed call carries `checkout_url` and the `total` (USD cents).
Give the user both, and say nothing is charged until they pay there and filing
starts after payment. Track progress on the account's `company_formation` field via
`accounts_get`.

It is a two-step tool. The first call is a preview, and it echoes every argument
back, identity numbers included. Show the user only the company name, state, entity
type, and founders: names and ownership for an LLC; names and roles, plus the share
count and par value, for a C-corp. Wait for their yes before the second call.

## Checkout

Required: `id` (the `biz_` business id from `accounts_list`; this tool takes `id`,
not `account_id`), `business_name`, `formation_state`, and `founders`. Each founder
needs `first_name`, `last_name`, `email`, `phone` (E.164), and `address`, and exactly
one founder is `is_primary`. Whop recommends `WY`, where formations complete the
same day.

```json
{
  "id": "biz_xxx",
  "business_name": "Acme Holdings",
  "formation_state": "WY",
  "entity_type": "llc",
  "business_address": {
    "line1": "1 Market St", "city": "San Francisco",
    "state": "CA", "postal_code": "94105", "country": "US"
  },
  "business_phone": "+12125550100",
  "founders": [{
    "is_primary": true, "first_name": "Jane", "last_name": "Doe",
    "email": "jane@example.com", "phone": "+12125550100",
    "address": {
      "line1": "1 Market St", "city": "San Francisco",
      "state": "CA", "postal_code": "94105", "country": "US"
    },
    "ownership_percentage": 100
  }]
}
```

Always pass a unique idempotency key. If a result is uncertain, retry with the
**same** key — a new key would create a second checkout.

## Founder identity

Each founder's home `address` is required. `ssn` and `date_of_birth` are optional in
the API and go into the filing; `ssn` is for US residents, so leave it empty for a
founder who is not one. Ask for them only when the formation needs them. When the
user gives them, pass them to the tool as given. Do not infer a missing value,
repeat them back, show them in a preview or summary, write them to a file, or put
them in `intent`. If the user would rather not share them here, do not ask again:
they can start the formation in their Whop dashboard instead.

## LLC vs C-corp vs registered agent

| Path | Extra rules |
| --- | --- |
| LLC (default `entity_type: "llc"`) | Each founder needs `ownership_percentage`; they must total 100. |
| C-corp (`entity_type: "c_corp"`) | Requires `share_structure`: `number_of_shares`, and `value`, the par value per share in USD (for example `0.01`, not the total capital). Every founder needs at least one entry in `roles`. Across the founders, every role must be covered: `president`, `secretary`, and `treasurer` each by exactly one founder, and `director` by at least one (it can repeat). A solo founder holds all four. Ask before executing. |
| Registered agent | `use_registered_agent: true` omits `business_address` and `business_phone`. Otherwise both are required. |

`expedite_ein` (an extra fee) is only valid when no founder supplies an SSN.

The preview does not check these rules, so get them right before showing it: a
mistake fails only on the confirmed call, after the user has approved. A request that
breaks these rules fails and creates no checkout. Re-read the tool's
schema and fix the field the error names.

## After payment

The IRS forms that still need a founder's signature appear under
`company_formation.signatures` on `accounts_get`: `ss4` (the EIN application) and
`form8821` (tax information authorization), each with a signing `url`. Give the
founder that link and say what it is for. Do not open it yourself.
