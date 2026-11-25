# Company formation

The tool is `get_company_formation_link`. It takes the business id as `account_id`
(the `biz_` id from `list_accounts`) and returns `url`, the link to Whop's company
formation page for that business.

The user forms the company themselves on that page. This plugin collects no founder
details, files nothing, and charges nothing.

- Give the user the link and say what it is for: on Whop's page they choose an LLC or
  a C-corp, enter the founders' details, and pay.
- Do not ask for founder details here: no names, home addresses, dates of birth,
  SSNs, ownership splits or roles. If the user offers any, say they enter them on
  Whop's page, and do not repeat them or send them to a tool.
- Do not quote a price. Whop's page shows the current price before the user pays.
- Do not open the link yourself.
- Give the link only when the user asks to form a company. Do not suggest a formation.

## Before you give the link

Read `business_type` and `industry_type` on `get_account`. Whop sets both, and
formation needs both. No tool field sets them, so do not look for one. If either is
null, say Whop has not classified the business yet, so its form may not let them
start. A new business has neither. Whop classifies from what the business sells, so
offer to add the real product they plan to sell (`create_product` with a real title
and description, after their yes), never a placeholder. Give the link either way.

## After the user starts a formation

No tool here starts, changes or cancels a formation. Progress shows on the account's
`company_formation` field via `get_account`. The IRS forms that still need a
founder's signature appear under `company_formation.signatures`: a `pending` form
has a signing `url`. Give the founder that link and say what it is for. Do not open
it yourself. If the field is missing, say the user can follow the formation in their
Whop dashboard.
