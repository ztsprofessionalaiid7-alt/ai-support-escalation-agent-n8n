# AI Customer Support Agent with Escalation (n8n): n8n Workflow Code

Importable n8n workflow (n8n Cloud). Copy the JSON below into a file named `support-agent-aiagent-cloud.json`, then in n8n use **Workflows → Import from File** (or paste it onto the canvas with Ctrl+V).

After import, connect your own credentials (no secrets are stored in this file) and follow the setup steps in the README.

````json
{
  "name": "AI customer support agent (AI Agent, n8n Cloud)",
  "nodes": [
    {
      "parameters": {
        "content": "## Setup (n8n Cloud)\n1. Credentials (free tiers): Google Gemini, Slack, Postgres (Supabase)\n2. Create tables kb_articles (with seed rows) and support_tickets in Supabase SQL Editor\n3. Slack channel: support-escalations (change in the Slack node)\n4. Click Execute workflow: 4 mock tickets (tracking, angry refund, password, cancel)\nReal client use: replace Mock Tickets with a Webhook or helpdesk trigger and the knowledge base with a vector store\nAI Agent node (Gemini) with a tool: order_lookup",
        "height": 260,
        "width": 560
      },
      "id": "6ba400d5-d116-427c-9775-b9401185e900",
      "name": "Setup notes",
      "type": "n8n-nodes-base.stickyNote",
      "typeVersion": 1,
      "position": [
        -60,
        -220
      ]
    },
    {
      "parameters": {},
      "id": "aed95e1b-5fe9-4d53-9427-8c75604145bf",
      "name": "When clicking Execute workflow",
      "type": "n8n-nodes-base.manualTrigger",
      "typeVersion": 1,
      "position": [
        0,
        200
      ]
    },
    {
      "parameters": {
        "jsCode": "return [\n  { json: { ticket_id: 'T-1001', customer: 'Lina', order_id: '5521', message: 'Where is my order #5521? It was supposed to arrive three days ago.' } },\n  { json: { ticket_id: 'T-1002', customer: 'Omar', order_id: '5530', message: 'I want a refund for order #5530. The product arrived damaged and I am very angry.' } },\n  { json: { ticket_id: 'T-1003', customer: 'Sara', order_id: '', message: 'How do I reset my password?' } },\n  { json: { ticket_id: 'T-1004', customer: 'Ken', order_id: '5544', message: 'Please cancel my order #5544, I ordered by mistake.' } }\n];"
      },
      "id": "dfa51cd2-2aaf-4a3b-8164-5179cc22cc1a",
      "name": "Mock Tickets",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        220,
        200
      ]
    },
    {
      "parameters": {
        "operation": "select",
        "schema": {
          "__rl": true,
          "value": "public",
          "mode": "list",
          "cachedResultName": "public"
        },
        "table": {
          "__rl": true,
          "value": "kb_articles",
          "mode": "name"
        },
        "options": {},
        "returnAll": true
      },
      "id": "8eb36cce-42b3-421d-92a3-e5371161c655",
      "name": "Get KB Articles",
      "type": "n8n-nodes-base.postgres",
      "typeVersion": 2.5,
      "position": [
        440,
        200
      ],
      "credentials": {
        "postgres": {
          "id": "REPLACE_PG",
          "name": "Postgres account"
        }
      },
      "executeOnce": true
    },
    {
      "parameters": {
        "jsCode": "const articles = $input.all().map((a) => a.json);\nreturn $('Mock Tickets').all().map((t) => {\n  const x = t.json;\n  const text = String(x.message).toLowerCase();\n  const hits = articles.filter((a) => String(a.keywords).split(',').some((k) => text.includes(k.trim().toLowerCase())));\n  const kb = hits.map((a) => a.topic + ': ' + a.answer).join('\\n') || 'No matching article.';\n  return { json: Object.assign({}, x, { kb }) };\n});"
      },
      "id": "5f89f40d-a087-4232-9ef7-34b184afac0c",
      "name": "Build Context",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        660,
        200
      ]
    },
    {
      "parameters": {
        "promptType": "define",
        "text": "=You are a polite customer support agent. Use ONLY the knowledge base and order info below. If you cannot fully resolve the issue (refund requests, angry customers, anything unclear), set escalate to true.\n\nTicket: {{ $json.ticket_id }} from {{ $json.customer }}\nMessage: {{ $json.message }}\n\nKnowledge base:\n{{ $json.kb }}\n\nOrder number (use the order_lookup tool if there is one): {{ $json.order_id || 'none' }}\n\nReturn category (shipping, refund, account, order_change or other), sentiment (positive, neutral or negative), escalate (true or false) and a reply to the customer (2-3 sentences).",
        "hasOutputParser": false,
        "options": {
          "systemMessage": "You are a polite customer support agent. Use ONLY the knowledge base text you are given and the order_lookup tool. When the customer mentions an order number, call order_lookup with that number. If you cannot fully resolve the issue (refund requests, angry customers, anything unclear), set escalate to true. Respond with ONLY a JSON object, no markdown, no extra text: {\"category\":\"shipping|refund|account|order_change|other\",\"sentiment\":\"positive|neutral|negative\",\"escalate\":true,\"reply\":\"2-3 sentence reply to the customer\"}"
        }
      },
      "id": "ea6068d0-eb75-4e5a-8a6e-e49295f8c791",
      "name": "AI Agent",
      "type": "@n8n/n8n-nodes-langchain.agent",
      "typeVersion": 1.7,
      "position": [
        880,
        200
      ],
      "retryOnFail": true,
      "maxTries": 3,
      "waitBetweenTries": 5000
    },
    {
      "parameters": {
        "modelName": "models/gemini-2.5-flash",
        "options": {}
      },
      "id": "791eb2f4-5cec-4e0c-b196-c04e36807cab",
      "name": "Google Gemini Chat Model",
      "type": "@n8n/n8n-nodes-langchain.lmChatGoogleGemini",
      "typeVersion": 1,
      "position": [
        860,
        420
      ],
      "credentials": {
        "googlePalmApi": {
          "id": "REPLACE_GEMINI",
          "name": "Google Gemini(PaLM) Api account"
        }
      }
    },
    {
      "parameters": {
        "mode": "runOnceForEachItem",
        "jsCode": "const ctx = $('Build Context').item.json;\nlet r = $json.output;\nif (typeof r === 'string') {\n  try { const m = r.match(/\\{[\\s\\S]*\\}/); r = m ? JSON.parse(m[0]) : {}; } catch (e) { r = {}; }\n}\nr = r || {};\nconst esc = r.escalate === true || String(r.escalate).toLowerCase() === 'true';\nreturn { json: {\n  ticket_id: ctx.ticket_id,\n  customer: ctx.customer,\n  message: ctx.message,\n  category: r.category || 'other',\n  sentiment: r.sentiment || 'neutral',\n  status: esc ? 'escalated' : 'resolved',\n  reply: String(r.reply || '').slice(0, 800)\n}};"
      },
      "id": "b399013e-6549-4e66-bdd0-1a06da666d0f",
      "name": "Parse Reply",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        1240,
        200
      ]
    },
    {
      "parameters": {
        "conditions": {
          "options": {
            "caseSensitive": true,
            "leftValue": "",
            "typeValidation": "strict",
            "version": 2
          },
          "conditions": [
            {
              "id": "6cdf4fbb-e1a5-4d92-a66f-1b99e7c58f16",
              "leftValue": "={{ $json.status }}",
              "rightValue": "escalated",
              "operator": {
                "type": "string",
                "operation": "equals"
              }
            }
          ],
          "combinator": "and"
        },
        "options": {}
      },
      "id": "d0c1670d-2c77-4c86-ab85-2f9b338a0c23",
      "name": "Escalate?",
      "type": "n8n-nodes-base.if",
      "typeVersion": 2.2,
      "position": [
        1460,
        200
      ]
    },
    {
      "parameters": {
        "select": "channel",
        "channelId": {
          "__rl": true,
          "value": "support-escalations",
          "mode": "name"
        },
        "text": "=:sos: *Ticket escalated: {{ $json.ticket_id }}*\nCustomer: {{ $json.customer }} ({{ $json.sentiment }})\nCategory: {{ $json.category }}\nMessage: {{ $json.message }}\nDraft reply: {{ $json.reply }}",
        "otherOptions": {}
      },
      "id": "9736f0e9-d4ff-4970-bc37-613a32675c2b",
      "name": "Slack Escalation",
      "type": "n8n-nodes-base.slack",
      "typeVersion": 2.3,
      "position": [
        1720,
        120
      ],
      "credentials": {
        "slackApi": {
          "id": "REPLACE_SLACK",
          "name": "Slack account"
        }
      },
      "onError": "continueRegularOutput"
    },
    {
      "parameters": {
        "operation": "insert",
        "schema": {
          "__rl": true,
          "value": "public",
          "mode": "list",
          "cachedResultName": "public"
        },
        "table": {
          "__rl": true,
          "value": "support_tickets",
          "mode": "name"
        },
        "options": {},
        "columns": {
          "mappingMode": "defineBelow",
          "value": {
            "ticket_id": "={{ $json.ticket_id }}",
            "customer": "={{ $json.customer }}",
            "category": "={{ $json.category }}",
            "sentiment": "={{ $json.sentiment }}",
            "status": "={{ $json.status }}",
            "reply": "={{ $json.reply }}"
          },
          "matchingColumns": [],
          "schema": [
            {
              "id": "ticket_id",
              "displayName": "ticket_id",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "string",
              "canBeUsedToMatch": true
            },
            {
              "id": "customer",
              "displayName": "customer",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "string",
              "canBeUsedToMatch": true
            },
            {
              "id": "category",
              "displayName": "category",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "string",
              "canBeUsedToMatch": true
            },
            {
              "id": "sentiment",
              "displayName": "sentiment",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "string",
              "canBeUsedToMatch": true
            },
            {
              "id": "status",
              "displayName": "status",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "string",
              "canBeUsedToMatch": true
            },
            {
              "id": "reply",
              "displayName": "reply",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "string",
              "canBeUsedToMatch": true
            }
          ]
        }
      },
      "id": "1f4800aa-2fbf-4548-b5b5-0280e9cb4bdf",
      "name": "Save Ticket",
      "type": "n8n-nodes-base.postgres",
      "typeVersion": 2.5,
      "position": [
        1720,
        320
      ],
      "credentials": {
        "postgres": {
          "id": "REPLACE_PG",
          "name": "Postgres account"
        }
      }
    },
    {
      "parameters": {
        "name": "order_lookup",
        "description": "Looks up an order by order number. Input: the order number only, for example '5521'. Returns the order status, tracking number and ETA as JSON.",
        "jsCode": "const orders = {\n  '5521': { status: 'in transit', tracking: 'TRK-88231', eta: '2026-11-12' },\n  '5530': { status: 'delivered', tracking: 'TRK-77410', eta: '' },\n  '5544': { status: 'processing', tracking: '', eta: '' }\n};\nconst key = String(query).replace(/[^0-9]/g, '');\nreturn JSON.stringify(orders[key] || { error: 'order not found' });"
      },
      "id": "93f13ac8-d78c-4ee6-be66-7c2a7695f25c",
      "name": "Order Lookup Tool",
      "type": "@n8n/n8n-nodes-langchain.toolCode",
      "typeVersion": 1.3,
      "position": [
        1080,
        420
      ]
    }
  ],
  "connections": {
    "When clicking Execute workflow": {
      "main": [
        [
          {
            "node": "Mock Tickets",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Mock Tickets": {
      "main": [
        [
          {
            "node": "Get KB Articles",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Get KB Articles": {
      "main": [
        [
          {
            "node": "Build Context",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Build Context": {
      "main": [
        [
          {
            "node": "AI Agent",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Parse Reply": {
      "main": [
        [
          {
            "node": "Escalate?",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Escalate?": {
      "main": [
        [
          {
            "node": "Slack Escalation",
            "type": "main",
            "index": 0
          },
          {
            "node": "Save Ticket",
            "type": "main",
            "index": 0
          }
        ],
        [
          {
            "node": "Save Ticket",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Google Gemini Chat Model": {
      "ai_languageModel": [
        [
          {
            "node": "AI Agent",
            "type": "ai_languageModel",
            "index": 0
          }
        ]
      ]
    },
    "AI Agent": {
      "main": [
        [
          {
            "node": "Parse Reply",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Order Lookup Tool": {
      "ai_tool": [
        [
          {
            "node": "AI Agent",
            "type": "ai_tool",
            "index": 0
          }
        ]
      ]
    }
  },
  "pinData": {},
  "active": false,
  "settings": {
    "executionOrder": "v1"
  },
  "tags": []
}
````
