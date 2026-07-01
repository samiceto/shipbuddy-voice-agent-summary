# Channel-Specific Response Templates
## ShipBuddy Customer Success FTE

These templates show how the same resolved answer is adapted per channel.
Implemented in `src/agent/formatter.py` → `format_for_channel()`.

---

## The Same Answer, Three Ways

**Scenario:** Customer asks how to enable returns portal.
**Raw agent answer (pre-formatting):**
> To enable the Returns Portal, go to Settings → Returns Portal → Enable Portal and toggle it on. Under Label Funding, choose either Merchant Pays or Customer Pays. Your returns URL will be yourstore.shipbuddy.io/returns once enabled. If you're on Starter plan, note that the Returns Portal requires Growth or above.

---

### Email version

```
Rachel Kim —

To enable the Returns Portal, go to **Settings → Returns Portal → Enable Portal**
and toggle it on. Under **Label Funding**, choose either Merchant Pays or
Customer Pays. Your returns URL will be yourstore.shipbuddy.io/returns once enabled.

If you're on Starter plan, note that the Returns Portal requires Growth or above.

ShipBuddy Support
ShipBuddy Customer Success
support@shipbuddy.io | status.shipbuddy.io
```

**Rules applied:**
- Menu paths wrapped in `**bold**`
- Customer name prepended
- ShipBuddy sign-off appended
- Full detail kept

---

### WhatsApp version

```
To enable Returns Portal go to Settings, then Returns Portal, then Enable Portal.
Under Label Funding choose Merchant Pays or Customer Pays.
Your returns URL will be yourstore.shipbuddy.io/returns once enabled.
Note: Returns Portal requires Growth plan or above.
```

**Rules applied:**
- Bullet points removed (would render as literal `-` on mobile)
- Markdown bold stripped
- Blank lines collapsed
- Truncated to 5 sentences max
- No sign-off (WhatsApp is conversational)
- No customer name greeting

---

### Web form version

```
To enable the Returns Portal:

- Go to **Settings → Returns Portal → Enable Portal** and toggle it on
- Under **Label Funding**, choose either Merchant Pays or Customer Pays
- Your returns URL will be yourstore.shipbuddy.io/returns once enabled

Note: Returns Portal requires Growth plan or above.
```

**Rules applied:**
- Menu paths bold
- Bullet points kept (web renders markdown)
- No sign-off
- Medium detail

---

## Escalation Acknowledgment Templates

### Email (data loss, Pro plan)

```
Patrick Walsh —

Your orders from Tuesday April 29th have been flagged and our engineering team
is investigating now. A specialist will be in touch within 1 hour with an update.

ShipBuddy Support
ShipBuddy Customer Success
support@shipbuddy.io | status.shipbuddy.io
```

### WhatsApp (data loss, Pro plan)

```
Your missing orders from Tuesday have been flagged with our engineering team.
A specialist will reach out within 1 hour.
```

### Web form (billing dispute, Growth plan)

```
Your billing dispute has been passed to our billing team for review.
You'll hear back within 4 hours.
```

---

## Tone Examples by Sentiment

### Neutral / curious (how-to question)

```
The bulk label printing shortcut is Ctrl+P (Windows) or Cmd+P (Mac) from the
Orders page. Select the orders first, then use the shortcut — it batches all
selected labels into one PDF.
```
No opener. Direct answer first.

---

### Frustrated (Shopify sync still broken after force sync)

```
That's a known issue — Shopify webhooks during peak hours (9am–2pm EST) can
delay by 15–30 minutes even after a force sync. The inventory count will
correct itself, but I understand that's not helpful when you're seeing oversells.

In **Settings → Inventory → Shopify Sync**, set the sync interval to 10 minutes
as a temporary fix. Engineering is tracking this for a permanent fix.
```
One acknowledgment sentence, then straight to solution.

---

### Panicked (5 oversells today)

```
5 oversells means the sync gap is actively costing you — turn on the 10-minute
sync interval now: **Settings → Inventory → Shopify Sync → Interval → 10 min**.
This will stop new oversells while the webhook delay is active.
I'm also flagging this with the team given the volume today.
```
Immediate action first. No pleasantries.

---

### Angry (threatening to leave)

```
That's a fair reaction given what you've been dealing with. I'm passing this
to our customer success manager now — they'll reach out within 4 hours and
work through this with you directly.
```
Even tone. Not defensive. No solution attempt — route to human.

---

## Channel Rules Summary

| Rule | Email | WhatsApp | Web Form |
|---|---|---|---|
| Menu paths bolded | ✅ | ❌ (stripped) | ✅ |
| Bullet points | ✅ | ❌ (stripped) | ✅ |
| Customer name greeting | ✅ | ❌ | ❌ |
| Sign-off | ✅ | ❌ | ❌ |
| Max length | None | 5 sentences | None |
| Markdown bold | ✅ | ❌ | ✅ |
| Blank lines | ✅ | Collapsed | ✅ |
