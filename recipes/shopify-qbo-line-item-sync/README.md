[![Maintained by Avalux](https://img.shields.io/badge/Maintained%20by-Avalux.io-3b82f6?style=flat-square)](https://avalux.io)

# Recipe: Idempotent Shopify → QuickBooks Online Line-Item Sync

> The classic problem with one-click Shopify-to-QBO integrations is that they aggregate. Your bookkeeper opens QuickBooks and sees one mystery line per day labeled "Shopify daily summary $4,182.17." When a customer disputes a charge, you have nothing. The fix is to sync each order as a proper line-item transaction with idempotency tokens so you can replay safely.

## The problem

You're running Shopify, doing $500k-$30M/year in revenue. Your bookkeeper uses QuickBooks Online. You connected one of the popular bridges (A2X, Bookkeep, Synder) and it works — except:

1. Every day in QBO is one summary line. No customer-level data, no SKU-level data, no refund detail.
2. When the bridge breaks (it does), there's no easy way to backfill — you either get duplicates or missing days.
3. Multi-currency stores get the conversion wrong on roughly 1 in 10 orders.
4. Subscription revenue (Recharge, Bold, Skio) is decoupled from order creation and bridges handle it inconsistently.

You're paying $200-500/month for the bridge and still spending hours each month reconciling.

## Who has this problem

- Shopify merchants doing $500k-$30M/year (under $200k it's cheaper to just use A2X)
- E-commerce bookkeepers and fractional CFOs
- Multi-store operators (parallel Shopify storefronts rolling up to one QBO file)
- Subscription-based DTC brands

## When this fix makes sense

- You want line-item accuracy in QBO
- You're tired of vendor lock-in / per-month bridge fees
- You have someone who can deploy a small Node service (or want us to do it)
- You care enough about the accounting to invest a few days getting it right

## When it doesn't make sense

- Volume under 50 orders/day — A2X at $50-100/month is genuinely cheaper than self-hosting
- You have no internal tech and no budget — start with A2X and migrate later

## The free DIY path

Three moving parts:

1. **Shopify webhook subscriber** — listens for `orders/paid`, `orders/fulfilled`, `refunds/create`
2. **Transformer** — maps Shopify line items to QBO accounts per your chart of accounts
3. **QBO poster** — posts each transaction to QBO with an idempotency token tied to the Shopify order ID

Below is the core transformer + poster in Node. Full middleware (with retries, multi-currency, multi-store) lives in our [shopify-quickbooks-sync](https://github.com/elikem2021/shopify-quickbooks-sync) repo.

```javascript
// shopify_to_qbo.js — minimum viable handler
import express from "express";
import crypto from "crypto";
import OAuthClient from "intuit-oauth";

const app = express();
app.use(express.json({ verify: (req, res, buf) => { req.rawBody = buf; } }));

const SHOPIFY_SECRET = process.env.SHOPIFY_WEBHOOK_SECRET;
const QBO_TOKEN = process.env.QBO_ACCESS_TOKEN;
const QBO_REALM = process.env.QBO_REALM_ID;
const QBO_BASE = "https://quickbooks.api.intuit.com/v3/company";

const accountMap = {
  revenue_default: "Sales of Product Income",
  revenue_by_product_type: {
    subscription: "Recurring Revenue",
    digital_download: "Digital Product Income",
  },
  shipping_income: "Shipping Income",
  sales_tax_by_jurisdiction: {
    ON_HST: "GST/HST Payable",
    BC_PST: "PST Payable",
  },
  discount_expense: "Promotional Discounts",
  refunds: "Sales Refunds Contra",
};

function verifyShopifyHmac(req) {
  const hmac = req.headers["x-shopify-hmac-sha256"];
  const expected = crypto
    .createHmac("sha256", SHOPIFY_SECRET)
    .update(req.rawBody)
    .digest("base64");
  return hmac === expected;
}

async function postToQBO(transaction, idempotencyKey) {
  // QBO has a SalesReceipt entity for paid orders. We attach the Shopify order ID
  // in CustomField so the next poll/replay is safely idempotent.
  const body = {
    ...transaction,
    CustomField: [{ DefinitionId: "1", Name: "ShopifyOrderID", Type: "StringType", StringValue: idempotencyKey }],
  };
  const res = await fetch(`${QBO_BASE}/${QBO_REALM}/salesreceipt`, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${QBO_TOKEN}`,
      Accept: "application/json",
      "Content-Type": "application/json",
    },
    body: JSON.stringify(body),
  });
  return res.json();
}

function shopifyOrderToQBOSalesReceipt(order) {
  const lines = order.line_items.map((li) => ({
    DetailType: "SalesItemLineDetail",
    Amount: parseFloat(li.price) * li.quantity,
    SalesItemLineDetail: {
      ItemRef: { name: li.sku || li.title },
      Qty: li.quantity,
      UnitPrice: parseFloat(li.price),
      AccountRef: { name: accountMap.revenue_by_product_type[li.product_type] || accountMap.revenue_default },
    },
    Description: li.title,
  }));

  if (parseFloat(order.shipping_lines?.[0]?.price || 0) > 0) {
    lines.push({
      DetailType: "SalesItemLineDetail",
      Amount: parseFloat(order.shipping_lines[0].price),
      SalesItemLineDetail: {
        ItemRef: { name: "Shipping" },
        AccountRef: { name: accountMap.shipping_income },
      },
    });
  }

  return {
    Line: lines,
    CustomerRef: { name: order.customer?.email || "Shopify Guest" },
    TxnDate: order.processed_at.split("T")[0],
    DocNumber: `SHO-${order.order_number}`,
    PrivateNote: `Shopify order ${order.id} | ${order.financial_status}`,
  };
}

app.post("/webhook/shopify/orders-paid", async (req, res) => {
  if (!verifyShopifyHmac(req)) return res.sendStatus(401);
  const order = req.body;
  const idempotencyKey = `shopify_order:${order.id}`;
  const txn = shopifyOrderToQBOSalesReceipt(order);
  const result = await postToQBO(txn, idempotencyKey);
  console.log(`Synced order ${order.id} → QBO`, result);
  res.sendStatus(200);
});

app.listen(3000, () => console.log("Shopify→QBO sync listening on :3000"));
```

Configure Shopify webhooks to POST to `https://your-domain/webhook/shopify/orders-paid` with the secret. Configure QBO OAuth with refresh-token rotation (Intuit's `intuit-oauth` library handles this).

## Common gotchas

1. **Idempotency.** Always tag each QBO transaction with the Shopify order ID in a custom field. Before creating a new transaction, query QBO for any existing record with that ID. Replays then become safe.
2. **QBO token rotation.** QBO access tokens expire every 60 minutes. Refresh tokens expire every 100 days of inactivity. Build in automatic rotation early or you'll wake up to a broken sync.
3. **Multi-currency.** Shopify reports order amounts in both the presentment currency (what the buyer paid) and shop currency. Pick one and stick with it — typically shop currency, with the conversion as a memo.
4. **Sales tax.** Shopify tax lines map to QBO Tax accounts, not to revenue. Get the mapping table right or your bookkeeper will hate you.
5. **Refunds.** Shopify fires `refunds/create` separately. Don't try to deduce refunds from `orders/cancelled` — they're different events.
6. **Multi-store.** If you run multiple Shopify stores into one QBO file, prefix the `DocNumber` with store name (`STORE_A-SHO-1234`) so you can audit.

## Variations

- **Daily summary mode.** If line-item-per-order is too much volume for your QBO file (5000+ orders/day), aggregate into daily summaries with per-channel line items. Keep order-level detail in a separate Postgres for audit.
- **Multi-currency with FX memo.** Store the original currency + amount + rate in a custom field, post the converted amount in shop currency.
- **NetSuite / Sage / Xero.** Same pattern, different API. The transformer logic ports cleanly.

## Production version

[Avalux](https://avalux.io) builds the full production middleware on top of this pattern — multi-store, multi-currency, idempotent replay endpoint for backfills, OAuth rotation, alerting when sync fails, NetSuite/Sage variants. Pricing starts at $5,000. See [avalux.io/contact.html](https://avalux.io/contact.html) or email [eli@avalux.io](mailto:eli@avalux.io).

## License

MIT — part of the [solved-by-avalux](https://github.com/elikem2021/solved-by-avalux) cookbook.
