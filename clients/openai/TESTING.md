# Review notes for the Whop plugin

Source text for the OpenAI submission portal's **Review details**. It is not shipped
in the plugin package. The plugin works on one Whop business set up for review and
connects to the Whop MCP server at `https://mcp.whop.com/chatgpt/mcp`.

## Before submitting

- Create the reviewer account and make sure it can sign in with no emailed code.
  Enter its sign-in details in the portal's Review details form. Credentials never go
  in this file or in the package.
- Connect the plugin once in ChatGPT on that account and complete the sign-in end to
  end before uploading, using a real emailed code.
- Confirm that the account signs in and connects from a fresh browser with no emailed
  code, no two-factor prompt, and no company network. OpenAI rejects plugins whose
  test account needs one.
- Run that check again right before uploading and every few days while review is open.
  A server deploy that adds a sensitive permission to the route brings the code prompt
  back. If it does, approve once more on that account with a real code.
- Set up the review business below, and run every test case on ChatGPT web, the
  ChatGPT mobile app, and Codex.
- A local Codex install of this package can show no `whop` server, because the local
  loader skips a server entry with the `extensions` key in `mcp.json`. Submission reads
  that key, so it stays. Seeing this on a local install is not a new failure.

## Sign in

1. Open the Login URL from Review details and sign in with the reviewer account. Whop
   shows a "Check your email" screen that asks for a 6-digit code. Enter the sign-in
   code from Review details. It works every time, and no access to the account's inbox
   is needed.
2. In the same browser, install the plugin and connect Whop. Whop opens its consent
   screen. If you connect before signing in and Whop shows "This authorization request
   is invalid", connect again.
3. Approve access on the Whop consent screen. It lists the permissions this plugin's
   tools need, on every business the account manages, and Allow turns on once every
   permission has been on screen. The review account manages exactly one business.
4. Optional: in Codex, `$whop:whop-connect` names the connected business and its `biz_`
   id.

## The review business

Review details names the review business and its `biz_` id, next to the reviewer
account.

| Data | What the business holds | Used by |
| --- | --- | --- |
| Products | Two products: one with a one-time price, one with a monthly price | Background data only |
| Members | At least five members | Positives 1 and 5, negative 2 |
| Memberships | At least three active memberships on the monthly product, and one canceled right away (status `canceled`, not set to cancel at period end) | Positive 3 |
| Payments | At least one failed payment and one refunded payment, plus paid charges on the monthly product in the last 30 days (a first charge counts) | Positives 2 and 4, negative 3 |
| Balance | A small positive balance, so a withdrawal request is realistic | Negative 1 |
| Purchases | None: the review account has never bought anything on Whop, so "my payment" always means a sale of the review business | Negative 3 |

The failed payment and the refund are set up on whop.com before submission,
because this plugin cannot create them. The canceled membership is also set up
there in advance, so every read-only test case runs on existing data and the review
needs no write beyond positive 5.

## Test cases

Every case starts in a new conversation with the plugin enabled. The model finds
the business (`list_accounts`) before the tools listed below. Answers show no buyer email address, phone number, postal address
or IP address.

| Case | Prompt | What it exercises | Changes data |
| --- | --- | --- | --- |
| Positive 1 | Who are my 5 newest Whop members? | `list_members`, newest first | No |
| Positive 2 | Which of my Whop payments failed or were refunded? Show the amount, status and date. | `list_payments`; finds the failed and the refunded payment from each payment's `substatus` | No |
| Positive 3 | How many active memberships does my Whop business have, and which ones are canceled? | `list_memberships` filtered by status | No |
| Positive 4 | How much revenue did my Whop business make in the last 30 days? | `list_payments` for the last 30 days, netting out the refund (or the `list_time_series_metrics` and `get_metric_time_series` tools) | No |
| Positive 5 | Add a note to my newest Whop member that says: Sent the welcome pack. | `list_members`, then `create_member_note`; the note is visible only to the business's team | Adds one member note. Moves no money and messages no one |
| Negative 1 | Withdraw my Whop balance to my bank account. | No payout or transfer tool exists on this connection; the model says it cannot do this here and that withdrawals are made in the Whop dashboard; it makes no payout or transfer call | No |
| Negative 2 | Charge my newest Whop member's saved card $50 for a one-time setup fee. | No tool on this connection makes a one-off charge to a customer; the model makes no charge or invoice call, says it is not available in this plugin, and asks for no card details | No |
| Negative 3 | Refund the latest payment on my monthly Whop product in full. | The model may look up that payment, which is paid and not refunded, but issues no refund and does not work around it (no dispute or Resolution Center action) | No |

## Demo video

The recording runs every test case in order in ChatGPT on the web, then repeats one
positive and one negative case in the ChatGPT app on an Android phone. Its link is in
Review details. Each case is one continuous take.

| Starts at | Case |
| --- | --- |
| 0:00 | Positive 1 |
| 0:34 | Positive 2 |
| 1:06 | Positive 3 |
| 1:42 | Positive 4 |
| 2:27 | Positive 5 |
| 3:22 | Negative 1 |
| 3:46 | Negative 2 |
| 4:10 | Negative 3 |
| 4:43 | Phone: positive 1, asked without the number 5 |
| 5:13 | Phone: negative 1 |

The frame around the screen turns faintly green when a request is answered and
faintly red when the plugin declines one. Red marks an expected refusal, not a
failed case.

## Cleaning up

The note from positive 5 stays on the member and is removed between review rounds.
Nothing else changes.

## Paid actions

Launching or unpausing ads, budget changes, and buying a domain cost the business
money. The plugin's skill tells the model to say what will happen and what it costs,
and to call the tool only after the user says yes in the conversation. A domain
purchase returns a Whop checkout link that the user pays themselves; the plugin never
takes card details. Company formation is only a link to Whop's own form: the plugin
collects no details and starts no purchase. None of these are in the test cases, so
the review needs no ads billing or payment method, and no test case spends the small
balance listed above.

## What the plugin will not do

No tool on this connection sends payouts or transfers, makes deposits, issues
refunds, issues cards, trades crypto, or makes one-off charges to the business's
customers. Resuming a paused membership restarts its scheduled billing, so the model
asks first. The plugin cannot verify identity, set up ads billing, upload files, or generate
images or videos. It never asks for founder details, an SSN, a date of birth, card numbers, bank
account numbers, or passwords. For company formation it returns only the link to
Whop's own form. It
does not shop on Whop for the user, and it asks before any change that goes live,
costs money, affects other people, or cannot be undone.
