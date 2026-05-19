# Order Workflow & State Management

## Overview

Orders in the Alpha E-Cyber Services platform progress through a defined lifecycle. This document details all possible states, transitions, and automation rules.

## Order State Diagram

```
                     ┌─────────────────┐
                     │  NEW_REQUEST    │
                     │  (Form Submitted)│
                     └────────┬────────┘
                              │
                              │ Auto-transition (10 mins)
                              ▼
                     ┌──────────────────────┐
                     │ AWAITING_PAYMENT     │
                     │ (Payment Pending)    │
                     └────────┬─────────────┘
                              │
                         ┌────┴─────────────────────┐
                         │                          │
                    (Payment OK)            (Payment Fails)
                         │                          │
                         ▼                          ▼
            ┌──────────────────┐      ┌─────────────────────┐
            │     PAID         │      │   ON_HOLD / FAILED  │
            │ (Work Begins)    │      │  (Retry or Cancel)  │
            └────────┬─────────┘      └─────────────────────┘
                     │
                     │ Admin action (mark started)
                     ▼
            ┌──────────────────┐
            │   IN_PROGRESS    │
            │ (Work in Progress)│
            └────────┬─────────┘
                     │
                     │ Admin action (mark delivered)
                     ▼
            ┌──────────────────┐
            │   DELIVERED      │
            │ (Files Ready)    │
            └────────┬─────────┘
                     │
                     │ Auto-transition (7 days) / Client action
                     ▼
            ┌──────────────────┐
            │   COMPLETED      │
            │ (Order Finished) │
            └──────────────────┘

Special States:
    ON_HOLD ──→ (Can return to PAID or CANCELLED)
    CANCELLED ──→ (Terminal state)
    EXPIRED ──→ (Auto from AWAITING_PAYMENT after 7 days)
```

---

## Order States

### 1. NEW_REQUEST
**Description**: Client has submitted a service request form but hasn't been asked for payment yet.

**Entry Conditions**:
- Client completes and submits service request form
- All required fields filled
- Files uploaded (if required)

**Activities**:
- System validates the request
- Generates order number (ORD-YYYY-NNN)
- Sends confirmation email to client
- Admin receives notification

**Exit Conditions**:
- Auto-transition to AWAITING_PAYMENT after 10 minutes
- Manual admin action (skip to AWAITING_PAYMENT immediately)

**Client Visibility**: ✅ Can see order status
**Client Actions**: View order, message admin

---

### 2. AWAITING_PAYMENT
**Description**: Order is waiting for payment confirmation from client.

**Entry Conditions**:
- NEW_REQUEST auto-transition OR admin triggers payment request
- Payment invoice generated
- Payment details sent to client (SMS/Email)

**Activities**:
- M-Pesa payment link/prompt displayed
- System polls M-Pesa for payment confirmation every 30 seconds
- Automated reminders sent:
  - At 3 hours: First reminder
  - At 6 hours: Second reminder (if unpaid)
- Payment timeout counter starts (7 days)

**Exit Conditions**:
- ✅ Payment received → **PAID**
- ❌ Payment fails/rejected → **ON_HOLD**
- ⏰ 7 days without payment → **EXPIRED**
- 🔧 Admin action: Cancel → **CANCELLED**

**Client Visibility**: ✅ Can see payment status
**Client Actions**: Initiate payment, contact admin

**Auto-Expiration Rules**:
- If AWAITING_PAYMENT for 7 days without payment
- Order marked EXPIRED
- Client notified
- Admin notified for follow-up

---

### 3. PAID
**Description**: Payment has been confirmed. Work can begin.

**Entry Conditions**:
- Payment successfully verified via M-Pesa webhook
- Payment confirmation recorded in database
- Order total matches payment amount

**Activities**:
- Send thank-you email to client
- Notify admin: "Payment received for [Order]"
- Admin can now mark order as IN_PROGRESS
- Work begins (admin assigned)

**Exit Conditions**:
- ✅ Admin marks as IN_PROGRESS
- 🔧 Admin action: Place ON_HOLD
- 🔧 Admin action: Cancel → **CANCELLED**

**Client Visibility**: ✅ Can see payment confirmed
**Client Actions**: Cannot make changes; wait for work to begin

**Auto-Transitions**:
- None - requires manual admin action

---

### 4. IN_PROGRESS
**Description**: Work is actively being done on the order.

**Entry Conditions**:
- Order is in PAID state
- Admin clicks "Start Work" / "Mark In Progress"
- Work assignment recorded

**Activities**:
- Admin can upload draft files
- Admin can send progress updates to client
- Client may request revisions
- Estimated completion timestamp visible

**Exit Conditions**:
- ✅ Work complete, ready for delivery → **DELIVERED**
- 🔧 Client requests revision → remains IN_PROGRESS
- 🔧 Admin requests more info → **ON_HOLD**
- 🔧 Admin cancels → **CANCELLED**

**Client Visibility**: ✅ Can see work in progress
**Client Actions**: Request updates, ask questions

---

### 5. DELIVERED
**Description**: Work is complete and files are ready for client pickup/download.

**Entry Conditions**:
- Work completely done
- Final files uploaded to cloud storage
- Shareable links generated
- Admin clicks "Mark Delivered"

**Activities**:
- Generate and send download links to client (Email + SMS)
- Access link expires after 30 days
- Client notified: "Your files are ready!"
- System starts 7-day auto-completion timer

**Exit Conditions**:
- ✅ Auto-transition to COMPLETED after 7 days
- 🔧 Client confirms received → **COMPLETED** immediately
- 🔧 Client requests revision → **IN_PROGRESS**
- 🔧 Admin action: Cancel → **CANCELLED**

**Client Visibility**: ✅ Can access download links
**Client Actions**: Download files, confirm receipt, request revision

**Auto-Transition Logic**:
```
If order.status = "DELIVERED" AND days_since_delivered >= 7:
  order.status = "COMPLETED"
  send_notification_to_admin("Order auto-completed")
  send_notification_to_client("Order completed - thank you!")
```

---

### 6. COMPLETED
**Description**: Order is fully complete. Client has received deliverables.

**Entry Conditions**:
- Either: 7 days passed in DELIVERED state
- Or: Client manually confirmed receipt
- Or: Admin manually marked complete

**Activities**:
- Order archived from active view
- Download links deactivated (if not already expired)
- Files moved to long-term storage
- Client feedback/rating request sent
- Admin notified: "Order completed successfully"

**Exit Conditions**:
- ❌ Terminal state - cannot revert

**Client Visibility**: ✅ Can see completed order in history
**Client Actions**: Leave review/rating, reorder similar service

---

### Special States

#### ON_HOLD
**Description**: Order paused temporarily, waiting for resolution.

**Reasons**:
- `payment_failed` - Payment declined or failed
- `more_info_needed` - Admin needs additional information
- `revision_requested` - Client requested revisions
- `admin_review` - Admin paused for manual review
- `custom_reason` - Admin-specified reason

**Entry Conditions**:
- From AWAITING_PAYMENT: Payment fails
- From PAID/IN_PROGRESS: Admin action
- From DELIVERED: Client requests revision

**Activities**:
- Work paused (if applicable)
- Clear notification sent to relevant party
- Reason recorded in database
- Follow-up reminder set for 3 days

**Exit Conditions**:
- ✅ Issue resolved → return to previous state or PAID
- ❌ Cancelled → **CANCELLED**
- ⏰ 14 days in ON_HOLD → auto-cancel warning

**Client Visibility**: ✅ Can see order on hold
**Client Actions**: Take corrective action (e.g., retry payment)

---

#### CANCELLED
**Description**: Order cancelled by client or admin. No further work.

**Reasons**:
- `client_requested` - Client cancelled
- `payment_failed_final` - Payment ultimately failed
- `admin_cancelled` - Admin cancelled
- `no_response` - Client didn't respond for 14 days
- `duplicate` - Duplicate order

**Entry Conditions**:
- Client action from any active state
- Admin action from any state
- Automatic after 14 days in ON_HOLD

**Activities**:
- Work immediately stops
- Notification sent to both parties
- Refund processed (if applicable):
  - Full refund if not started
  - Partial refund if started (per policy)
- Files archived (not deleted)

**Exit Conditions**:
- ❌ Terminal state - cannot be reversed

**Client Visibility**: ✅ Can see cancelled order
**Client Actions**: None - order is final

---

#### EXPIRED
**Description**: Order automatically expired due to inaction (no payment received in 7 days).

**Entry Conditions**:
- Order in AWAITING_PAYMENT for 7+ days
- Payment still not received

**Triggers**:
```
Daily Job (runs at 2 AM UTC):
  For each order in AWAITING_PAYMENT:
    If created_at + 7 days <= NOW():
      order.status = 'EXPIRED'
      send_notification_to_client("Your order has expired")
      send_notification_to_admin("Order expired - no payment")
```

**Activities**:
- Order removed from active queue
- Client notified: "Order expired due to lack of payment"
- Admin can manually extend deadline if needed
- Files held for 30 days then deleted

**Exit Conditions**:
- ❌ Terminal state unless admin manually reactivates

---

## State Transition Rules

### Valid Transitions

```
NEW_REQUEST         → AWAITING_PAYMENT (auto or manual)
                    → CANCELLED (admin)

AWAITING_PAYMENT    → PAID (payment confirmed)
                    → ON_HOLD (payment failed)
                    → CANCELLED (admin or auto after failed)
                    → EXPIRED (auto after 7 days)

PAID                → IN_PROGRESS (admin)
                    → ON_HOLD (admin)
                    → CANCELLED (admin)

IN_PROGRESS         → DELIVERED (admin)
                    → ON_HOLD (admin)
                    → CANCELLED (admin)

DELIVERED           → COMPLETED (auto after 7 days or manual)
                    → IN_PROGRESS (client revision request)
                    → CANCELLED (admin)

ON_HOLD             → PAID (resolved)
                    → IN_PROGRESS (resolved, if past PAID)
                    → CANCELLED (admin)
                    → EXPIRED (auto after 14 days)

COMPLETED           → ❌ No transitions (terminal)
CANCELLED           → ❌ No transitions (terminal)
EXPIRED             → ❌ No transitions (terminal)
```

### Invalid Transitions
```
❌ AWAITING_PAYMENT → IN_PROGRESS (must go through PAID)
❌ IN_PROGRESS → AWAITING_PAYMENT (cannot reverse)
❌ COMPLETED → any state (terminal)
❌ CANCELLED → any state (terminal)
❌ NEW_REQUEST → DELIVERED (must follow sequence)
```

---

## Workflow Automation Rules

### Auto-Transition Rules

```yaml
NEW_REQUEST → AWAITING_PAYMENT:
  Trigger: Order created OR admin manually initiates
  Delay: 10 minutes after creation
  Action: Send payment request to client
  
AWAITING_PAYMENT → EXPIRED:
  Trigger: Scheduled job runs daily at 2 AM UTC
  Condition: created_at + 7 days <= NOW()
  Action: Mark as EXPIRED, notify both parties

DELIVERED → COMPLETED:
  Trigger: Scheduled job runs daily at midnight
  Condition: delivered_at + 7 days <= NOW()
  Action: Mark complete, archive order

ON_HOLD → CANCELLED:
  Trigger: Scheduled job runs daily at 3 AM UTC
  Condition: on_hold_since + 14 days <= NOW()
  Action: Auto-cancel with reason "No response after 14 days"
```

### Notification Rules

```yaml
NEW_REQUEST:
  - Email to client: "Order received"
  - Notification to admin: "New order submitted"
  
AWAITING_PAYMENT:
  - SMS to client: Payment link + amount
  - Email to client: Detailed payment instructions
  - At 3 hours: First payment reminder
  - At 6 hours: Second payment reminder

PAID:
  - Email to client: "Payment confirmed, thank you!"
  - Notification to admin: "Payment received - [amount]"

IN_PROGRESS:
  - Notification to client: "Work has started"
  - Admin can send progress updates

DELIVERED:
  - Email to client: Download link + instructions
  - SMS to client: "Your files are ready!"
  - Notification to admin: "Order delivered"

COMPLETED:
  - Email to client: "Order completed, thank you for using our service!"
  - Request for review/rating
  - Notification to admin: "Order archived"

CANCELLED/EXPIRED/ON_HOLD:
  - Email to both parties explaining reason
  - Notification to admin for follow-up
```

---

## Client Visibility Matrix

| State | Can View | Can Update | Can Download | Can Message |
|-------|----------|-----------|--------------|-------------|
| NEW_REQUEST | ✅ | ❌ | ❌ | ✅ |
| AWAITING_PAYMENT | ✅ | ❌ | ❌ | ✅ |
| PAID | ✅ | ❌ | ❌ | ✅ |
| IN_PROGRESS | ✅ | ❌ | ✅ (drafts) | ✅ |
| DELIVERED | ✅ | ❌ | ✅ (final) | ✅ |
| COMPLETED | ✅ (history) | ❌ | ✅ (30 days) | ❌ |
| ON_HOLD | ✅ | ✅ (pay/info) | ❌ | ✅ |
| CANCELLED | ✅ (history) | ❌ | ❌ | ❌ |
| EXPIRED | ✅ | ✅ (reorder) | ❌ | ✅ |

---

## Order State Database Schema

```sql
CREATE TABLE orders (
  -- ... other fields ...
  status VARCHAR(30) NOT NULL DEFAULT 'NEW_REQUEST',
  previous_status VARCHAR(30),
  status_changed_at TIMESTAMP,
  
  -- Timestamps for each state
  created_at TIMESTAMP,
  submitted_at TIMESTAMP,
  payment_requested_at TIMESTAMP,
  paid_at TIMESTAMP,
  started_at TIMESTAMP,
  delivered_at TIMESTAMP,
  completed_at TIMESTAMP,
  expired_at TIMESTAMP,
  cancelled_at TIMESTAMP,
  
  -- State-specific fields
  on_hold_reason VARCHAR(255),
  on_hold_since TIMESTAMP,
  cancellation_reason VARCHAR(255),
  expiration_reason VARCHAR(255),
  
  -- Tracking
  last_reminder_at TIMESTAMP,
  auto_expire_warning_sent BOOLEAN DEFAULT false,
  
  INDEX (status),
  INDEX (created_at),
  INDEX (client_id, status)
);

CREATE TABLE order_state_history (
  id UUID PRIMARY KEY,
  order_id UUID NOT NULL,
  previous_status VARCHAR(30),
  new_status VARCHAR(30) NOT NULL,
  transition_reason_code VARCHAR(50),
  changed_by UUID,
  changed_by_type VARCHAR(20),  -- 'admin', 'system', 'client'
  notes TEXT,
  changed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  
  FOREIGN KEY (order_id) REFERENCES orders(id),
  INDEX (order_id),
  INDEX (changed_at)
);
```

---

## Handling Edge Cases

### What if payment comes in after EXPIRED?
- System should check for late payments
- If valid M-Pesa webhook received:
  - Reactivate order
  - Move to PAID
  - Send "Order reactivated" notification

### What if client requests revision in COMPLETED state?
- Cannot modify completed orders
- Create NEW order for revision service
- Link to original order in notes

### What if admin forgets to mark order complete?
- Automatic transition after 7 days in DELIVERED
- Prevents orders from being stuck
- Admin still gets notification

### What if payment is double-charged?
- Payment service should check for duplicates
- If duplicate detected:
  - Log in order notes
  - Trigger refund process
  - Notify both parties

---

## Admin Dashboard State Indicators

**Summary Dashboard**:
```
🟢 Active Orders: 5 (NEW_REQUEST: 2, PAID: 2, IN_PROGRESS: 1)
🟡 Awaiting Payment: 8
🔴 On Hold: 2
⏳ Ready for Delivery: 3
✅ Completed This Week: 12
```

**Quick Actions**:
- Filter by status
- Sort by created_at, deadline
- Bulk status updates
- Auto-reminder trigger
- Export to CSV

---

This workflow ensures:
✅ No orders are lost
✅ Clear client communication
✅ Automated processes reduce manual work
✅ Flexible for edge cases
✅ Audit trail maintained
✅ Scalable for growth
