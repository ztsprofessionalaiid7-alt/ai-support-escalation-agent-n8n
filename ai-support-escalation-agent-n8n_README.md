# AI Customer Support Agent with Escalation (n8n)

An n8n workflow where an AI Agent (Gemini) answers support tickets using a knowledge base and an order-lookup tool, and escalates refunds, angry customers and anything unclear to a human on Slack.

▶ **Demo video:** https://drive.google.com/file/d/1gDnS861VNb-DdNTJESyfLTqZIVbiqH5t/view

## How it works

1. Mock tickets arrive (tracking question, angry refund, password reset, order cancellation)
2. Knowledge base articles are read from Postgres and matched by keyword
3. The AI Agent answers using ONLY the matched articles and the `order_lookup` tool (status, tracking, ETA), and sets category, sentiment and an escalate flag
4. A Code node validates the reply. Escalated tickets go to Slack
5. Every ticket is saved to Postgres with category, sentiment, status and reply

```
Manual Trigger -> Mock Tickets -> Get KB Articles (Postgres) -> Build Context
-> AI Agent (Gemini + order_lookup tool) -> Parse Reply -> Escalate?
   yes -> Slack Escalation -> Save Ticket
   no  -> Save Ticket
```

## AI Agent

AI Agent node with one tool (`order_lookup`). The prompt restricts answers to the knowledge base, and refunds or angry customers always escalate.

## Tech stack

n8n, AI Agent, Gemini, Customer Support, Slack, Postgres

## Setup

1. Copy the JSON from `workflow.md` into a file named `support-agent-aiagent-cloud.json` and import it into n8n
2. Create credentials (all have free tiers):
   - Google Gemini (API key)
   - Slack (Bot token and signing secret)
   - Postgres (Supabase Session pooler, SSL: Require)
3. In the Slack node set channel `support-escalations`, then run `/invite @your-bot` in that channel
4. Create the database tables (below)
5. In the Postgres save node remove the `id` and `created_at` fields, the database fills them
6. Click **Execute workflow**: 4 mock inputs run through all branches

## Database

```sql
create table if not exists public.kb_articles (
  id bigint generated always as identity primary key,
  topic text,
  keywords text,
  answer text
);

create table if not exists public.support_tickets (
  id bigint generated always as identity primary key,
  ticket_id text,
  customer text,
  category text,
  sentiment text,
  status text,
  reply text,
  created_at timestamptz default now()
);

insert into public.kb_articles (topic, keywords, answer) values
  ('Password reset', 'password,reset,login,forgot', 'Go to Settings > Security > Reset password. A reset link is emailed within 2 minutes.'),
  ('Order tracking', 'where is my order,tracking,delivery,shipped', 'Use the tracking number from your order email. Delivery usually takes 3 to 5 business days.'),
  ('Refunds', 'refund,damaged,money back', 'Refunds are available within 30 days. Damaged items are replaced or refunded after review.'),
  ('Order cancellation', 'cancel,ordered by mistake', 'Orders can be cancelled while the status is processing. Shipped orders must be returned.');
```

## Demo data

| Mock input | Expected result |
|---|---|
| T-1001 Lina: where is order #5521? | Resolved with tracking from order_lookup |
| T-1002 Omar: refund, damaged, very angry | Escalated to Slack |
| T-1003 Sara: how to reset password? | Resolved from the knowledge base |
| T-1004 Ken: cancel order #5544 | Resolved, order is still processing |

## Notes

- Replace Mock Tickets with a Webhook or helpdesk trigger, and the keyword match with a vector store for real use
- The sample `kb_articles` rows are examples, replace them with your real help articles
- Slack message text must be in **Expression** mode so `{{ }}` values are filled in
- Never commit API keys. Use n8n credentials only

## Author

Zarin Tasnim Shaily, AI Automation Engineer
[LinkedIn](https://linkedin.com/in/5ha1l72) | [GitHub](https://github.com/ztsprofessionalaiid7-alt)
