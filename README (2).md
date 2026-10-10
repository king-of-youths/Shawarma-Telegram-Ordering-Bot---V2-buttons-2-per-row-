# De BIG JOINT Shawarma — Telegram Ordering Bot (Version 1)

A Telegram bot that takes shawarma orders automatically. Customers browse the menu, place an order with tappable buttons (or by typing), and check their order status. The business owner manages everything from a Google Sheet.

**Stack:** n8n (backend) · Telegram Bot API (customer interface) · Google Sheets (database) · JavaScript in n8n Code nodes

**Version 1 is fully deterministic.** There is no AI agent. Menu, prices, totals, orders, customers and conversation state are all handled by normal n8n nodes and plain logic.

---

## What the bot does

| Customer picks | What happens |
|---|---|
| **View Menu & Prices** | Shows every item marked `Yes` in the Menu sheet, with current prices |
| **Place an Order** | Guided flow: product → quantity → extras → drinks → delivery/pickup → name → phone → address (delivery only) → summary → confirm |
| **Check My Order** | Customer enters an Order ID (e.g. `ORD-1001`) and sees items, total and status |
| **Delivery Information** | Shows delivery fees and time, then offers to start an order |
| **Speak to Someone** | Shows phone, WhatsApp and business hours |

Every question shows **tappable buttons** (two per row) and also accepts typed answers. Typing `cancel` at any step returns to the main menu. Typing `hi` or `/start` always restarts.

---

## Files in this project

| File | Purpose |
|---|---|
| `shawarma_bot_v2_buttons_2col.json` | The complete n8n workflow. Import this. |
| `Shawarma_Business_Bot_template.xlsx` | Google Sheet template with the 4 tabs, headers, example menu and dropdowns |
| `normalize_input_v2.js` | Code of the **Normalize Input** node (for patching an existing workflow) |
| `process_message_v2.js` | Code of the **Process Message** node (the bot's "brain") |
| `normalize_input_v2.json`, `process_message_v2.json` | The same two nodes as pasteable n8n JSON |

---

## How it works

```
Telegram Trigger
   ↓
Normalize Input        → turns a typed message OR a button tap into { chatId, userId, text }
   ↓                       (a parallel branch answers the button tap so Telegram stops the spinner)
Business Config        → business name, phone, hours, delivery fees, spreadsheet ID
   ↓
Read Session           → where is this customer in the conversation?
   ↓
Read Menu              → current products and prices
   ↓
Process Message        → state machine: decides the reply, the next state and the buttons
   ↓
Save Session
   ↓
Route Action ──┬── reply        → Keyboard Size → Send Reply (with 0–6 buttons)
               ├── lookup_order → Read Order By ID → Format Order Result → Send Order Status
               └── place_order  → Read Orders → Build Order Record → Append Order
                                  → Read Customer → Build Customer Record → Upsert Customer
                                  → Reset Session → Send Order Confirmation
```

Any Google Sheets or Code node that fails sends its red **error** output to **Send Error Message**, so the customer gets a friendly message instead of silence.

### Conversation states

`MAIN_MENU` · `WAITING_FOR_DELIVERY_CHOICE` · `WAITING_FOR_ORDER_ID` · `WAITING_FOR_PRODUCT` · `WAITING_FOR_QUANTITY` · `WAITING_FOR_EXTRA` · `WAITING_FOR_DRINK` · `WAITING_FOR_ORDER_TYPE` · `WAITING_FOR_NAME` · `WAITING_FOR_PHONE` · `WAITING_FOR_ADDRESS` · `WAITING_FOR_ORDER_CONFIRMATION`

The state and the draft order are stored per customer in the **Sessions** tab, so progress is not lost between messages. The session is reset only **after** an order has been saved.

### Buttons

Each button carries `STATE|value`. Tapping it is treated exactly like typing `value`. If a customer taps a button from an old message, the bot replies that it belongs to an earlier step and does nothing else.

| Buttons on screen | Layout |
|---|---|
| 1 | one full-width button |
| 2 | one line of 2 |
| 3 | 2, then 1 |
| 4 | 2, 2 |
| 5 | 2, 2, 1 |
| 6 | 2, 2, 2 |

---

## Google Sheet

Spreadsheet name: **Shawarma Business Bot**. Row 1 of every tab holds the headers, spelled exactly as below. Do not rename, reorder or add spaces to them.

### Menu (you edit this)

| Product ID | Product | Description | Category | Price | Available |
|---|---|---|---|---|---|
| SH001 | Chicken Shawarma | Chicken shawarma | Shawarma | 3000 | Yes |
| SH002 | Beef Shawarma | Beef shawarma | Shawarma | 3500 | Yes |
| SH003 | Chicken + Beef Shawarma | Mixed shawarma | Shawarma | 4000 | Yes |
| EX001 | Extra Chicken | Extra chicken | Extra | 1000 | Yes |
| EX002 | Extra Cheese | Extra cheese | Extra | 700 | Yes |
| DR001 | Soft Drink | Chilled drink | Drink | 700 | Yes |

- **Category** must be exactly `Shawarma`, `Extra` or `Drink`.
- **Available** must be `Yes` to be offered. Set `No` to hide an item.
- **Price** is a plain number (no ₦ sign, no comma).
- Change a price or availability in the sheet and the bot uses it on the next message. No workflow edit needed.

### Orders (the bot fills this)

`Order ID` · `Telegram User ID` · `Telegram Username` · `Customer Name` · `Phone` · `Items` · `Quantity` · `Extras` · `Drinks` · `Total` · `Order Type` · `Delivery Address` · `Payment Status` · `Order Status` · `Created At` · `Updated At`

New orders start as `Unpaid` / `Received`. Change **Order Status** by hand (dropdown in the template) and customers see it under *Check My Order*:
`Received` → `Preparing` → `Out for Delivery` / `Ready for Pickup` → `Completed` (or `Cancelled`)

### Customers (the bot fills this)

`Customer ID` · `Telegram User ID` · `Telegram Username` · `Name` · `Phone` · `Address` · `First Order Date` · `Last Order Date` · `Total Orders`

### Sessions (the bot fills this)

`Telegram User ID` · `State` · `Draft Order` · `Updated At`

Leave Orders, Customers and Sessions empty below the header. Do not type test rows in them.

### Column formats

Set these columns to **Plain text** so phone numbers keep their leading 0 and IDs are not reformatted: Telegram User ID, Phone, Order ID, Customer ID, State, Draft Order, and the date columns. Set Price, Total, Quantity and Total Orders to numbers. The template already does all of this.

---

## Setup

### 1. Create the Telegram bot
1. In Telegram, open **@BotFather** and send `/newbot`.
2. Choose a name and a username ending in `bot`.
3. Copy the **token** it gives you. Keep it private.

### 2. Create the Google Sheet
1. In Google Sheets choose **File → Import → Upload** and select `Shawarma_Business_Bot_template.xlsx`.
2. Choose **Create new spreadsheet** and name it `Shawarma Business Bot`.
3. Copy the spreadsheet ID from the URL: the long string between `/d/` and `/edit`.

### 3. Add credentials in n8n
- **Telegram API:** paste the BotFather token. Name it clearly (e.g. `big joint`).
- **Google Sheets OAuth2:** connect your Google account.

### 4. Import the workflow
1. Create a **new, empty workflow**.
2. Open the `…` menu → **Import from file** → choose `shawarma_bot_v2_buttons_2col.json`.
3. Open the 9 Telegram nodes and the Google Sheets nodes and confirm each shows the right credential.

> Import into a blank workflow. Pasting into a canvas that already has nodes with the same names makes n8n rename them (`Process Message1`) and breaks the expressions.

### 5. Edit Business Config
Open the **Business Config** node and set:

| Field | Meaning |
|---|---|
| `spreadsheetId` | The ID from step 2 |
| `businessName` | Shown in the welcome message |
| `phone`, `whatsapp` | Shown in *Speak to Someone* and in error messages |
| `businessHours` | Shown in *Speak to Someone* |
| `deliveryFees` | Shown in *Delivery Information* (one line per tier) |
| `deliveryTime` | Estimated delivery time |

### 6. Publish
Turn the workflow on (**Publish** / **Active**). Telegram sends messages to the live version only, so edits take effect after you save and publish again. Only **one** workflow at a time can be live for the same bot.

> Telegram needs a public HTTPS address for n8n. n8n Cloud works directly. A local install needs a tunnel (for example ngrok) or a domain.

### 7. Test
Send `hi` to your bot, then place a test order end to end. Check that a row appears in **Orders** and **Customers**, and that **Sessions** returns to `MAIN_MENU`.

---

## Business rules built into the workflow

- **Prices only come from the Menu sheet.** Totals are calculated in n8n and recalculated from the sheet when the customer confirms. A customer can never supply a price.
- **Unavailable or unknown items** are never offered, and an item that becomes unavailable mid-order cancels that order with a friendly message.
- **Order IDs** are sequential: the highest existing number plus one, starting at `ORD-1001`.
- **Customer ID** is `CUS-` plus the Telegram user ID. Repeat customers update their row and their order count.
- **Customers only see their own orders** in *Check My Order*.
- **Quantity** is 1–20. Each extra is added once per order. The drink question uses the first available `Drink` row in the Menu.
- **Delivery fees** are shown as information but are **not** added to the total.
- **Timestamps** use the Africa/Lagos time zone.

## Error handling

| Situation | Bot behaviour |
|---|---|
| Unrecognised text | "Sorry, I didn't understand that" plus the options again |
| Invalid quantity, phone, name, address | Asks again with a hint |
| Unknown Order ID (or someone else's) | "Sorry, I couldn't find that order" |
| Photo, sticker, voice note | Asks the customer to type |
| Stale button from an old message | Explains it is from an earlier step |
| Missing session | Treated as a new customer at the main menu |
| Google Sheets or Code failure | Friendly "something went wrong" message with the business phone |
| Order saving fails | Customer is told to retry; the session is **not** reset |

---

## Security

- Keep the Telegram token and Google login **only** in n8n **Credentials**. Never paste them into node fields, Code nodes, chat messages or screenshots.
- If a token is ever exposed, send `/revoke` to BotFather and update the n8n credential.
- Share the Google Sheet only with the people who need it. Orders and Customers contain names, phone numbers and addresses.
- Use a separate Google account or Sheet for demos so real customer data is never shown.

---

## Troubleshooting

| Symptom | Likely cause and fix |
|---|---|
| No reply at all | Workflow is not published, or two workflows are live for the same bot. Publish one and unpublish the other. |
| Reply comes from a different bot | A Telegram node is using another bot's credential. Select the correct one on every Telegram node. |
| "Our menu is not available right now" | **Read Menu** returns nothing. Set its Sheet to **Menu**, remove any filters, and check the headers and `Available = Yes`. |
| The main menu keeps repeating after every message | **Read Session** is reading the wrong tab (it must be **Sessions**), or **Send Reply** has fixed text instead of expressions. |
| "Referenced node doesn't exist" | A node was renamed on import (`Process Message1`). Re-import into a blank workflow. |
| Buttons do not appear | Telegram Trigger must listen for **Message** and **Callback Query**. Re-publish after changing it. |
| Phone numbers lose the leading 0 | Format the Phone column as **Plain text**. |
| New prices do not appear | Check the Price cell is a number and `Available` is `Yes`. |

---

## Known limits of Version 1

- Whole tabs are read on each message. This is fine for a small business and gets slower with thousands of rows.
- Two orders confirmed at the same second could receive the same Order ID. Unlikely at small volume.
- No online payment: **Payment Status** is edited by hand.
- Order status is updated manually in the sheet; customers are not notified automatically.
- One drink product, and extras are quantity 1.

## Ideas for later versions

Owner alert on every new order · automatic status notifications to the customer · payment gateway · delivery fee in the total · admin dashboard · AI for free-text ordering (Version 2/3).
