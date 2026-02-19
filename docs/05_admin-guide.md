# Administrator Guide

**Language**: English | [简体中文](./05_admin-guide_CN.md) | [繁體中文](./05_admin-guide_TW.md) | [Русский](./05_admin-guide_RU.md) | [فارسی](./05_admin-guide_FA.md)

This document is intended for EzPanel administrators and covers the operation of each backend module.

![Admin Dashboard](../images/dashboard.png)

---

## First Login

Default administrator credentials:

- **Email**: `admin@opine.work`
- **Password**: `admin123`

> **Security Notice**: After logging in, immediately change the default password and email in "Profile Settings" and enable two-factor authentication.

---

## User Management

Go to **User Management → User List**:

### Viewing Users

| Action | Description |
|--------|-------------|
| Search | Search by email or user ID |
| Filter | Filter by status (active / banned) or plan |
| Export | Export user data in CSV / Excel format |

### User Actions

- **View Details**: View a user's subscriptions, orders, traffic, balance, and more
- **Edit**: Modify email, password, balance, traffic quota, expiration date
- **Ban / Unban**: Prohibit / allow the user to log in and use the subscription
- **Top Up Balance**: Directly add balance to the user's account

### Manually Resetting a Password

You can set a new password directly from the user details page without verifying the old password.

---

## Plan Management

Go to **Plan Management**:

### Creating a Plan

| Field | Description |
|-------|-------------|
| Plan Name | Name displayed to users |
| Traffic Quota | Available traffic per billing cycle (GB) |
| Billing Cycles | Monthly / Quarterly / Semi-annual / Annual; each cycle has its own price |
| Stock | -1 means unlimited; a positive integer sets a stock limit |
| Device Limit | Maximum number of simultaneous online devices; 0 means no limit |
| Speed Limit | Speed cap (Mbps); 0 means no limit |
| Node Group | The node group accessible with this plan |
| Sort Order | Smaller numbers appear first |
| Status | Listed / Unlisted |

### Plan Visibility

Unlisted plans are not shown to users, but users who have already purchased them are not affected.

---

## Node Management

Go to **System Management → Node Management**:

### Creating a Node

**Basic Configuration**:

| Field | Description |
|-------|-------------|
| Node Name | Displayed to users; supports Emoji, e.g. "🇭🇰 Hong Kong 01" |
| Node Address | Node server IP or domain name |
| Node Type | XrayM (recommended) / XrayR |
| Node Group | Assigned group; affects plan access permissions |
| Speed Limit | Global speed cap (Mbps); 0 means no limit |
| Device Limit | Maximum simultaneous online devices; 0 means no limit |
| Sort Order | Smaller numbers appear first |

**Protocol Configuration (XrayM)**:

Click "Quick Add Protocol" to select a preset combination, or add protocols manually:

- **Transport layer**: TCP / WebSocket / gRPC / HTTPUpgrade
- **Security layer**: TLS / REALITY / None
- **Port**: Each protocol has its own port

After configuration, enter the Node ID and API key into the XrayM config file on the server. See [Node Setup](./03_node-setup.md).

### Node Monitoring

Go to **System Management → Health Check** to view the real-time status of all nodes:

- Node online / offline status
- CPU, memory, disk usage
- Real-time network speed
- Current number of online users

Click a node to view detailed historical charts. Data refreshes automatically every 10 seconds.

> Node monitoring is only supported by XrayM. XrayR only reports traffic data.

---

## Order Management (Pro Edition)

Go to **Order Management**:

| Action | Description |
|--------|-------------|
| View Orders | Filter by user, status, or date |
| Order Details | View payment records and plan information |
| Refund | Return the order amount to the user's balance |
| Cancel | Cancel an unpaid order |
| Mark as Complete | Manually mark an order as complete for offline payments |

---

## Coupon Management (Pro Edition)

Go to **Marketing → Coupons**:

| Field | Description |
|-------|-------------|
| Discount Type | Fixed amount / Percentage discount |
| Usage Limit | Total uses / Uses per person |
| Applicable Plans | Leave blank to apply to all plans |
| Validity | Start / end date |
| Bulk Generation | Generate multiple coupons with different codes at once |

---

## Ticket Management (Pro Edition)

Go to **Ticket Management**:

- View all tickets submitted by users
- Filter by status (Pending / In Progress / Closed)
- Reply to tickets with rich text support
- Close or reopen tickets

---

## Knowledge Base (Pro Edition)

Go to **Knowledge Base**:

- Create FAQ articles with Markdown / rich text support
- Category management
- Set article visibility (all users / paid users only)
- Users can view articles on the "Help" page

---

## Email System

### Configuring SMTP

Go to **System Settings → Email Settings**:

| Option | Description |
|--------|-------------|
| SMTP Server | e.g. `smtp.gmail.com` |
| Port | Typically 465 (SSL) or 587 (STARTTLS) |
| Username | Sender email account |
| Password | Email password or app-specific password |
| Sender Name | The sender name displayed to recipients |

After filling in the details, click "Send Test Email" to verify the configuration.

### Email Templates

Go to **System Management → Email Templates** to customize the following emails:

- Registration verification email
- Password reset email
- Order notifications
- Traffic usage alerts
- Plan expiration reminders

**Templates must be reset after switching languages**: After switching to the target language in the interface, click "Reset to Default" to update email templates to the corresponding language version.

> Resetting will overwrite custom content. Please back up your templates beforehand.

---

## System Settings

Go to **System Management → System Settings**:

| Category | Options |
|----------|---------|
| Basic Settings | Site name, Logo, site URL |
| Registration Settings | Open registration, email verification, invite code requirement |
| Subscription Settings | Subscription URL, subscription format configuration |
| Node Settings | Node API key |
| Security Settings | Login failure lockout, IP blocklist |

### Changing the Node API Key

After changing the key, the `ApiKey` in the XrayM/XrayR config files on all nodes must be updated accordingly, otherwise nodes will be unable to connect.

---

## Data Export

The following pages support data export:

- User List → Export CSV
- Order List → Export CSV / Excel
- Financial Reports → Export

---

## Audit Logs

Go to **System Management → Audit Logs** to view:

- User login records (IP, timestamp)
- Administrator operation records
- Key operations (password changes, balance adjustments, etc.)

---

## Language Switching and Template Reset

1. Click the language switcher in the top-right corner to select the target language
2. Go to **System Management → Email Templates** and click "Reset to Default"
3. Go to **System Management → Subscription Templates** and click "Reset to Default"

This will update all template content to the selected language version.
