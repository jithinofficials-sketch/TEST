# Understanding Webhooks in Bucks Currency Converter

## What Are Webhooks?

Think of webhooks as **automatic notifications** that Shopify sends to the Bucks app whenever something important happens in your store. 

**Simple Example:**
- When you uninstall the Bucks app, Shopify automatically notifies our system
- Our app receives this notification and cleans up your data properly
- You don't need to do anything - it happens automatically in the background!

---

## How Webhooks Work in Bucks

### The Simple Flow:

1. **Something happens in your Shopify store** (e.g., you update store settings)
2. **Shopify sends a notification** to the Bucks app
3. **Bucks receives and processes** the notification
4. **Your data stays up-to-date** automatically

```
Your Store → Shopify Detects Change → Sends Notification → Bucks Updates Data
```

---

## Main Webhooks Used in Bucks

### 1. **APP_UNINSTALLED** ✅ (Active)

**What it does:**
- Triggers when you uninstall the Bucks app from your store

**What happens automatically:**
- Your active sessions are cleared
- Your account is marked as inactive
- Your subscription is downgraded to Free plan
- Trial days are preserved for future reinstallation
- Onboarding progress is partially saved (money format & theme extension settings)
- Email notifications are sent
- Analytics events are tracked

**Why it's important:**
- Ensures clean data management
- Protects your trial period if you reinstall later
- Keeps your account secure

---

### 2. **SHOP_UPDATE** ⚠️ (Configured but Currently Blocked)

**What it does:**
- Triggers when you update your shop settings (store name, domain, currency, etc.)

**What happens automatically:**
- Bucks updates your store information in our database
- Ensures currency conversion works with latest settings

**Current Status:**
- The webhook is configured but currently blocked from processing
- Your store data may need manual refresh in some cases

---

### 3. **APP_SUBSCRIPTIONS_UPDATE** 🚧 (In Development)

**What it does:**
- Triggers when your Bucks subscription plan changes

**What happens automatically:**
- Updates your plan status in real-time
- Tracks payment success/failure

**Current Status:**
- Handler is set up but logic is still being implemented

---

### 4. **GDPR Compliance Webhooks** ✅ (Active)

These webhooks help Bucks comply with privacy regulations:

#### **CUSTOMER_DATA_REQUEST**
- Triggers when a customer requests their data
- Allows you to fulfill GDPR data requests

#### **CUSTOMER_REDACT**
- Triggers when customer data needs to be deleted
- Ensures compliance with data deletion requests

#### **SHOP_REDACT**
- Triggers when your entire shop data needs to be removed
- Handles complete data cleanup after 48 hours of uninstallation

---

## Benefits of Webhooks for You

✅ **Automatic Updates** - No manual work needed
✅ **Real-time Sync** - Your data is always current
✅ **Better Performance** - App responds faster to changes
✅ **Data Security** - Proper cleanup when you uninstall
✅ **GDPR Compliance** - Automatic handling of privacy requests

---

## Common Questions

### Q: Do I need to do anything to enable webhooks?
**A:** No! Webhooks work automatically in the background. You don't need to configure anything.

### Q: Can I see when webhooks are triggered?
**A:** Webhooks work behind the scenes. If you need to verify something is working, contact our support team.

### Q: What if a webhook fails?
**A:** Shopify automatically retries failed webhooks multiple times. If there's a persistent issue, our team is notified automatically.

### Q: Are webhooks secure?
**A:** Yes! Shopify uses encrypted HTTPS connections and verifies each webhook with a secret signature to ensure security.

---

## Technical Details (For Developers)

### Webhook Endpoints in Bucks:

| Webhook Topic | Endpoint | Status |
|--------------|----------|--------|
| APP_UNINSTALLED | `/api/webhooks/app_uninstalled` | ✅ Active |
| SHOP_UPDATE | `/api/webhooks/shop_update` | ⚠️ Blocked |
| APP_SUBSCRIPTIONS_UPDATE | `/api/webhooks/app_subscriptions/update` | 🚧 In Dev |
| GDPR Webhooks | `/api/webhooks/[topic]` | ✅ Active |

### Configuration Location:
- Main webhook registration: `utils/shopify.js`
- Individual handlers: `utils/webhooks/` directory
- Webhook routing: `pages/api/webhooks/` directory

---

## Need Help?

If you have questions about how webhooks work in Bucks or notice any issues:

📧 **Contact Support:** [Your support email]
📚 **Documentation:** [Your help center link]
💬 **Live Chat:** Available in the Bucks app dashboard

---

*Last Updated: February 2026*
*Version: 1.0*
