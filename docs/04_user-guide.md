# User Guide

**Language**: English | [简体中文](./04_user-guide_CN.md) | [繁體中文](./04_user-guide_TW.md) | [Русский](./04_user-guide_RU.md) | [فارسی](./04_user-guide_FA.md)

This document is intended for regular EzPanel users and covers how to register, log in, obtain subscriptions, purchase plans, and more.

---

## Registration and Login

### Creating an Account

1. Open the panel URL and click "Register"
2. Enter your email address and password
3. If the site requires email verification, check your inbox and click the verification link
4. If the site requires an invite code, enter it before submitting

### Logging In

Enter the email address and password you registered with. Check "Remember Me" to extend the session duration.

### Forgot Password

1. Click "Forgot Password" on the login page
2. Enter your registered email address; the system will send a reset email
3. Click the link in the email to set a new password

---

## Two-Factor Authentication (2FA)

Enabling two-factor authentication effectively protects your account from unauthorized access.

### Enabling 2FA

1. Go to "Profile Settings" → "Security Settings"
2. Click "Enable Two-Factor Authentication"
3. Scan the QR code with an app such as Google Authenticator or Aegis
4. Enter the 6-digit verification code shown in the app to confirm

### Using 2FA at Login

After entering your email and password, you will be prompted to enter a 6-digit dynamic verification code.

> **Note**: Be sure to save your backup recovery codes. They cannot be retrieved if lost.

---

## Subscriptions

Subscription links are used to import node information into clients such as Clash, Sing-box, and others.

### Getting Your Subscription Link

1. Log in and go to "My Subscription" or the home page
2. Copy the subscription link
3. Paste it into your client's "Add Subscription" field and update

### Subscription Formats

EzPanel supports multiple subscription formats. Common clients and their recommended formats:

| Client | Recommended Format |
|--------|-------------------|
| Clash / Mihomo | Clash format |
| Sing-box | Sing-box format |
| v2rayN / Nekoray | Universal Base64 |
| Quantumult X | Universal format |

> Specific format parameters may vary depending on site configuration. Please check with the site administrator.

### Refreshing Subscriptions

Subscription content updates as nodes change. It is recommended to enable automatic periodic updates in your client (e.g., hourly or daily).

---

## Purchasing Plans (Pro Edition Feature)

> The following features require the site to have a Pro edition license installed.

### Buying a Plan

1. Go to the "Plans" page and browse available plans
2. Select a plan and billing cycle (monthly / quarterly / annual, etc.)
3. Confirm the order and choose a payment method
4. The plan takes effect immediately after payment is complete

### Renewal

Before a plan expires, you can go to "My Subscription" to renew early. The renewal extends from the current expiration date.

### What to Do When Traffic Runs Out

When traffic is exhausted, the subscription will be suspended. You can:
- Purchase a traffic top-up package (if available)
- Wait for the next billing cycle to reset automatically
- Manually renew to trigger a traffic reset

---

## Redeem Card

Some sites support redeeming plans or account balance with a card code.

1. Go to the "Redeem" or "Card Redeem" page
2. Enter the card code
3. Click "Redeem"; balance or plan will be credited immediately upon success

---

## Invite Friends

1. Go to the "Invite" or "Referral" page
2. Copy your unique invite link or invite code
3. Share it with friends; you will earn a commission when friends register and make purchases

### Viewing Commissions

On the "Referral / Invite" page you can see:
- Number of users invited
- Total commission earned
- Withdrawable balance

### Requesting a Withdrawal (Pro Edition)

Once the minimum withdrawal amount is met, submit a request on the "Withdrawal" page. The administrator will process and transfer the funds after review.

---

## Support Tickets (Pro Edition Feature)

If you encounter an issue and need to contact site support:

1. Go to "Tickets" → "New Ticket"
2. Select the issue type, fill in the title and a detailed description
3. Submit and wait for a support reply
4. Track progress in the ticket list

---

## Notifications

EzPanel sends notifications through the following channels:

- **In-app messages**: View by clicking the notification bell icon after logging in
- **Email**: Sent to your registered email address (requires the administrator to configure SMTP)
- **Telegram**: Received after binding Telegram (requires the administrator to configure a Bot)

### Binding Telegram

1. Find the site's Bot on Telegram
2. Send `/bindaccount`
3. Follow the prompts to obtain a verification code from the panel
4. Send `/bindcode <verification_code>` to complete the binding

---

## Profile Settings

Click the avatar in the top-right corner → "Profile Settings":

| Setting | Description |
|---------|-------------|
| Email | Change login email (verification required) |
| Password | Change login password |
| Two-Factor Authentication | Enable / Disable 2FA |
| Device Management | View and log out other active device sessions |
| Telegram Binding | Bind / Unbind Telegram account |

---

## Keyboard Shortcuts

| Shortcut | Function |
|----------|----------|
| `Ctrl + K` / `Cmd + K` | Global search |

---

## Language Switching

Switch the interface language from the top-right corner:

- 简体中文
- English
- 繁體中文
- Русский
- فارسی
