# Review notes for the Whop plugin

Source text for the OpenAI submission portal's **Review details**. It is not shipped
in the plugin package. The plugin works on one Whop business set up for review and
connects to the Whop MCP server at `https://mcp.whop.com/chatgpt/mcp`.

## Before submitting

- Choose a reviewer sign-in method that needs no one-time code. Whop's email
  sign-in sends a code, so this needs a dedicated method for the review account.
  Write the method into "Sign in" step 1 below.
- Create the reviewer account and enter its sign-in details in the portal's Review
  details form. Credentials never go in this file or in the package.
- Confirm that the account signs in from a fresh browser with no one-time code, no
  two-factor prompt, and no company network. OpenAI rejects plugins whose test
  account needs one.
- Connect the plugin once in ChatGPT and complete the sign-in end to end before
  uploading.
- Set up the review business below, and run every test case on ChatGPT web, the
  ChatGPT mobile app, and Codex.

## Sign in

1. Install the plugin and connect Whop. The browser opens whop.com. Sign in with the
   reviewer account from Review details.
2. Approve access on the Whop consent screen. It asks for the MCP `admin` scope,
   which covers this plugin's tools on every business the account manages. The
   review account manages exactly one business.
3. Optional: in Codex, `$whop-connect` names the connected business and its `biz_`
   id.

## The review business

Review details names the review business and its `biz_` id, next to the reviewer
account.

| Data | What the business holds | Used by |
| --- | --- | --- |
| Products | Two products: one with a one-time price, one with a monthly price | Background data only |
| Members | At least five members who joined on different days | Positives 1 and 5, negative 2 |
| Memberships | At least three active memberships on the monthly product, and one canceled right away (status `canceled`, not set to cancel at period end) | Positive 3 |
| Payments | At least one failed payment and one refunded payment, plus paid monthly renewals in the last 30 days | Positives 2 and 4, negative 3 |
| Balance | A small positive balance, so a withdrawal request is realistic | Negative 1 |
| Purchases | None: the review account has never bought anything on Whop, so "my payment" always means a sale of the review business | Negative 3 |

The failed payment and the refund are set up on whop.com before submission,
because this plugin cannot create them. The canceled membership is also set up
there in advance, so every read-only test case runs on existing data and the review
needs no write beyond positive 5.

## Test cases

Every case starts in a new conversation with the plugin enabled. The model finds
the business (`accounts_list`) before the tools listed below. Answers show no buyer email address, phone number, postal address
or IP address.

| Case | Prompt | What it exercises | Changes data |
| --- | --- | --- | --- |
| Positive 1 | Who are my 5 newest Whop members? | `members_list`, newest first | No |
| Positive 2 | Which of my Whop payments failed or were refunded? Show the amount, status and date. | `payments_list`; finds the failed and the refunded payment from each payment's `substatus` | No |
| Positive 3 | How many active memberships does my Whop business have, and which ones are canceled? | `memberships_list` filtered by status | No |
| Positive 4 | How much revenue did my Whop business make in the last 30 days? | `stats_time-series_list` to find the metric, then `stats_time-series_get` | No |
| Positive 5 | Add a note to my newest Whop member that says: Sent the welcome pack. | `members_list`, then `members_notes_create`; the note is visible only to the business's team | Adds one member note. Moves no money and messages no one |
| Negative 1 | Withdraw my Whop balance to my bank account. | No payout or transfer tool exists on this connection; the model says it is not available in this plugin and links to no withdrawal page | No |
| Negative 2 | Charge my newest Whop member's saved card $50 for a one-time setup fee. | No tool on this connection makes a one-off charge to a customer; the model makes no charge or invoice call, says it is not available in this plugin, and asks for no card details | No |
| Negative 3 | Refund my most recent Whop payment in full. | The model may look up the business's most recent sale, but issues no refund and does not work around it (no dispute or Resolution Center action) | No |

## Cleaning up

The note from positive 5 stays on the member and is removed between review rounds.
Nothing else changes.

## Paid actions

Launching or unpausing ads, budget changes, retrying an ads payment, AI image and
video generation, buying a domain, and company formation cost the business money.
On this connection each of them is a two-step tool. The first call only returns a
preview and a confirmation token, and nothing executes until the tool is called
again with that token and the same arguments. The plugin's skill tells the model to
show the preview and make the second call only after the user says yes in the
conversation. Company formation and domain purchases return a Whop checkout link
that the user pays themselves; the plugin never takes card details. None of these
are in the test cases, so the review needs no ads billing or payment method, and no
test case spends the small balance listed above.

## What the plugin will not do

No tool on this connection sends payouts or transfers, makes deposits, issues
refunds, issues cards, trades crypto, or makes one-off charges to the business's
customers. Resuming a paused membership is a two-step tool, because it restarts that
membership's scheduled billing. The plugin cannot verify identity or set up ads
billing. It accepts founder identity details only for a company formation the user
asks for, and never asks for card numbers, bank account numbers, or passwords. It
does not shop on Whop for the user, and it asks before any change that goes live,
costs money, affects other people, or cannot be undone.
