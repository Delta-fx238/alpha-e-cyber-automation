# Database Schema & Design

## Overview

The Alpha E-Cyber Services platform uses a relational database to manage clients, orders, payments, files, and workflow states. This document provides the complete database schema design.

## Entity Relationship Diagram (ERD)

```
┌─────────────┐         ┌──────────────┐
│   Users     │────────→│   Clients    │
│ (Admins)    │  1:N    │              │
└─────────────┘         └──────┬───────┘
                                │ 1:N
                                ▼
                         ┌──────────────┐
                         │   Addresses  │
                         │ (Shipping)   │
                         └──────────────┘

┌──────────────┐         ┌──────────────┐
│   Services   │────────→│   Orders     │
│              │  1:N    │              │
└──────────────┘         └──────┬───────┘
                                │ 1:1
                                ▼
                         ┌──────────────┐
                         │  Payments    │
                         │              │
                         └──────────────┘

┌──────────────┐         ┌──────────────┐
│   Orders     │────────→│    Files     │
│              │  1:N    │              │
└──────────────┘         └──────────────┘

┌──────────────┐         ┌──────────────────┐
│   Orders     │────────→│  OrderStateHist  │
│              │  1:N    │                  │
└──────────────┘         └──────────────────┘

┌──────────────┐         ┌──────────────┐
│   Orders     │────────→│Notifications │
│              │  1:N    │              │
└──────────────┘         └──────────────┘

┌──────────────┐
│   Settings   │
│ (Config)     │
└──────────────┘
```

---

## Detailed Schema

### 1. Users Table (System Administrators)

```sql
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email VARCHAR(255) UNIQUE NOT NULL,
  password_hash VARCHAR(255) NOT NULL,
  first_name VARCHAR(100) NOT NULL,
  last_name VARCHAR(100) NOT NULL,
  phone VARCHAR(20),
  role VARCHAR(50) NOT NULL,  -- 'admin', 'super_admin'
  is_active BOOLEAN DEFAULT true,
  last_login TIMESTAMP,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  
  INDEX (email),
  INDEX (role),
  INDEX (is_active)
);
```

**Purpose**: Store system administrators and staff who manage the platform

---

### 2. Clients Table

```sql
CREATE TABLE clients (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email VARCHAR(255) UNIQUE NOT NULL,
  phone VARCHAR(20) UNIQUE NOT NULL,
  first_name VARCHAR(100) NOT NULL,
  last_name VARCHAR(100) NOT NULL,
  
  -- Contact Information
  alternative_email VARCHAR(255),
  alternative_phone VARCHAR(20),
  
  -- Account Status
  status VARCHAR(20) DEFAULT 'active',  -- 'active', 'inactive', 'banned'
  account_type VARCHAR(20) DEFAULT 'standard',  -- 'standard', 'premium'
  
  -- Account Metadata
  total_orders INT DEFAULT 0,
  total_spent DECIMAL(10,2) DEFAULT 0.00,
  average_rating DECIMAL(3,2),
  notes TEXT,
  
  -- Timestamps
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  last_order_at TIMESTAMP,
  
  INDEX (email),
  INDEX (phone),
  INDEX (status),
  INDEX (created_at)
);
```

**Purpose**: Store client information and profiles

---

### 3. Addresses Table

```sql
CREATE TABLE addresses (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  client_id UUID NOT NULL,
  address_type VARCHAR(20) NOT NULL,  -- 'billing', 'delivery'
  
  -- Address Details
  street_address VARCHAR(255) NOT NULL,
  city VARCHAR(100) NOT NULL,
  region VARCHAR(100),
  postal_code VARCHAR(20),
  country VARCHAR(100) DEFAULT 'Kenya',
  
  is_primary BOOLEAN DEFAULT false,
  
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  
  FOREIGN KEY (client_id) REFERENCES clients(id) ON DELETE CASCADE,
  INDEX (client_id),
  INDEX (address_type)
);
```

**Purpose**: Store multiple addresses per client

---

### 4. Services Table

```sql
CREATE TABLE services (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(100) NOT NULL UNIQUE,
  description TEXT,
  category VARCHAR(50) NOT NULL,  -- 'documents', 'applications', 'online_forms', etc.
  
  -- Pricing
  base_price DECIMAL(10,2) NOT NULL,
  currency VARCHAR(3) DEFAULT 'KES',
  is_variable_price BOOLEAN DEFAULT false,
  min_price DECIMAL(10,2),
  max_price DECIMAL(10,2),
  
  -- Service Details
  estimated_duration_hours INT,
  icon_url VARCHAR(255),
  form_json TEXT,  -- JSON schema for intake form
  
  -- Status
  is_active BOOLEAN DEFAULT true,
  is_featured BOOLEAN DEFAULT false,
  display_order INT,
  
  -- Metadata
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  
  INDEX (category),
  INDEX (is_active),
  INDEX (display_order)
);
```

**Purpose**: Define available services and pricing

---

### 5. Orders Table (Core)

```sql
CREATE TABLE orders (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  order_number VARCHAR(50) UNIQUE NOT NULL,  -- e.g., ORD-2026-001
  client_id UUID NOT NULL,
  service_id UUID NOT NULL,
  
  -- Status & Workflow
  status VARCHAR(30) NOT NULL,  -- NEW_REQUEST, AWAITING_PAYMENT, PAID, IN_PROGRESS, DELIVERED, COMPLETED
  previous_status VARCHAR(30),
  status_changed_at TIMESTAMP,
  
  -- Order Details
  title VARCHAR(255),
  description TEXT,
  requirements TEXT,
  special_instructions TEXT,
  
  -- Pricing
  base_price DECIMAL(10,2) NOT NULL,
  discount_amount DECIMAL(10,2) DEFAULT 0.00,
  total_price DECIMAL(10,2) NOT NULL,
  currency VARCHAR(3) DEFAULT 'KES',
  
  -- Timing
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  submitted_at TIMESTAMP,
  payment_requested_at TIMESTAMP,
  paid_at TIMESTAMP,
  started_at TIMESTAMP,
  completed_at TIMESTAMP,
  estimated_completion TIMESTAMP,
  actual_completion TIMESTAMP,
  
  -- Admin Notes
  internal_notes TEXT,
  admin_assigned_to UUID,
  
  -- Flags
  is_urgent BOOLEAN DEFAULT false,
  needs_revision BOOLEAN DEFAULT false,
  on_hold_reason VARCHAR(255),
  
  FOREIGN KEY (client_id) REFERENCES clients(id) ON DELETE CASCADE,
  FOREIGN KEY (service_id) REFERENCES services(id),
  FOREIGN KEY (admin_assigned_to) REFERENCES users(id),
  
  INDEX (order_number),
  INDEX (client_id),
  INDEX (status),
  INDEX (created_at),
  INDEX (service_id),
  INDEX (admin_assigned_to)
);
```

**Purpose**: Track all service orders and their lifecycle

---

### 6. Payments Table

```sql
CREATE TABLE payments (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  order_id UUID NOT NULL UNIQUE,
  
  -- Payment Details
  amount DECIMAL(10,2) NOT NULL,
  currency VARCHAR(3) DEFAULT 'KES',
  payment_method VARCHAR(50) NOT NULL,  -- 'm-pesa', 'bank_transfer', 'cash', etc.
  
  -- M-Pesa Specific
  mpesa_transaction_id VARCHAR(100),
  mpesa_receipt_number VARCHAR(50),
  mpesa_phone_number VARCHAR(20),
  
  -- Payment Status
  status VARCHAR(30) NOT NULL,  -- 'pending', 'confirmed', 'failed', 'cancelled', 'refunded'
  
  -- Timestamps
  initiated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  confirmed_at TIMESTAMP,
  failed_at TIMESTAMP,
  refunded_at TIMESTAMP,
  
  -- Refund Details
  refund_amount DECIMAL(10,2),
  refund_reason VARCHAR(255),
  refund_transaction_id VARCHAR(100),
  
  -- Metadata
  notes TEXT,
  error_message TEXT,
  
  FOREIGN KEY (order_id) REFERENCES orders(id) ON DELETE CASCADE,
  
  INDEX (order_id),
  INDEX (status),
  INDEX (mpesa_phone_number),
  INDEX (initiated_at)
);
```

**Purpose**: Track all financial transactions

---

### 7. Files Table

```sql
CREATE TABLE files (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  order_id UUID NOT NULL,
  
  -- File Information
  original_filename VARCHAR(255) NOT NULL,
  stored_filename VARCHAR(255) NOT NULL,
  file_type VARCHAR(50),  -- 'pdf', 'docx', 'jpg', etc.
  file_size INT,  -- in bytes
  mime_type VARCHAR(100),
  
  -- File Purpose
  category VARCHAR(50) NOT NULL,  -- 'upload', 'deliverable', 'draft', 'reference'
  
  -- Cloud Storage
  cloud_provider VARCHAR(50) DEFAULT 'google_drive',
  cloud_file_id VARCHAR(255),
  cloud_folder_id VARCHAR(255),
  share_link VARCHAR(500),
  share_link_expires_at TIMESTAMP,
  
  -- Access Control
  is_public BOOLEAN DEFAULT false,
  is_encrypted BOOLEAN DEFAULT true,
  encryption_key_id UUID,
  
  -- Status
  upload_status VARCHAR(20) DEFAULT 'pending',  -- 'pending', 'completed', 'failed'
  virus_scan_status VARCHAR(20),  -- 'pending', 'clean', 'infected'
  
  -- Metadata
  description TEXT,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  deleted_at TIMESTAMP,  -- Soft delete
  
  FOREIGN KEY (order_id) REFERENCES orders(id) ON DELETE CASCADE,
  
  INDEX (order_id),
  INDEX (category),
  INDEX (created_at),
  INDEX (cloud_provider)
);
```

**Purpose**: Track all files associated with orders

---

### 8. Order State History Table

```sql
CREATE TABLE order_state_history (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  order_id UUID NOT NULL,
  
  -- State Transition
  previous_status VARCHAR(30),
  new_status VARCHAR(30) NOT NULL,
  
  -- Reason & Context
  reason VARCHAR(255),
  transition_reason_code VARCHAR(50),  -- 'payment_received', 'admin_action', 'auto_expire', etc.
  
  -- User Information
  changed_by UUID,  -- admin_id or NULL if system action
  changed_by_type VARCHAR(20),  -- 'admin', 'system', 'client'
  
  -- Metadata
  metadata JSON,  -- Additional context as needed
  notes TEXT,
  
  -- Timestamp
  changed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  
  FOREIGN KEY (order_id) REFERENCES orders(id) ON DELETE CASCADE,
  FOREIGN KEY (changed_by) REFERENCES users(id),
  
  INDEX (order_id),
  INDEX (changed_at),
  INDEX (new_status),
  INDEX (changed_by)
);
```

**Purpose**: Maintain audit trail of all order status changes

---

### 9. Notifications Table

```sql
CREATE TABLE notifications (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  order_id UUID,
  client_id UUID,
  
  -- Notification Details
  type VARCHAR(50) NOT NULL,  -- 'order_confirmation', 'payment_reminder', 'delivery', etc.
  title VARCHAR(255) NOT NULL,
  message TEXT NOT NULL,
  
  -- Channel
  channel VARCHAR(50) NOT NULL,  -- 'email', 'sms', 'in_app'
  recipient_address VARCHAR(255),  -- email or phone
  
  -- Status
  status VARCHAR(20) DEFAULT 'pending',  -- 'pending', 'sent', 'failed', 'read'
  
  -- Timestamps
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  sent_at TIMESTAMP,
  read_at TIMESTAMP,
  
  -- Retry Logic
  retry_count INT DEFAULT 0,
  max_retries INT DEFAULT 3,
  last_retry_at TIMESTAMP,
  
  -- Error Handling
  error_message TEXT,
  
  FOREIGN KEY (order_id) REFERENCES orders(id) ON DELETE SET NULL,
  FOREIGN KEY (client_id) REFERENCES clients(id) ON DELETE SET NULL,
  
  INDEX (order_id),
  INDEX (client_id),
  INDEX (status),
  INDEX (created_at),
  INDEX (channel)
);
```

**Purpose**: Track all notifications sent to clients

---

### 10. System Settings Table

```sql
CREATE TABLE system_settings (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  setting_key VARCHAR(100) UNIQUE NOT NULL,
  setting_value TEXT NOT NULL,
  setting_type VARCHAR(50),  -- 'string', 'integer', 'boolean', 'json'
  description TEXT,
  
  -- Control
  is_configurable BOOLEAN DEFAULT true,
  is_secret BOOLEAN DEFAULT false,
  
  -- Timestamps
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_by UUID,
  
  FOREIGN KEY (updated_by) REFERENCES users(id),
  
  INDEX (setting_key)
);
```

**Example Settings**:
- `payment_timeout_days: 7`
- `delivery_confirmation_timeout_days: 5`
- `auto_expiration_enabled: true`
- `max_file_upload_size_mb: 50`
- `email_notification_enabled: true`
- `m_pesa_consumer_key: ****`

**Purpose**: Store system configuration and feature flags

---

## Indexing Strategy

### Primary Indexes
```sql
CREATE INDEX idx_orders_client_status ON orders(client_id, status);
CREATE INDEX idx_orders_created_at ON orders(created_at DESC);
CREATE INDEX idx_payments_order_id ON payments(order_id);
CREATE INDEX idx_payments_status ON payments(status);
CREATE INDEX idx_files_order_id ON files(order_id);
CREATE INDEX idx_state_history_order_id ON order_state_history(order_id);
CREATE INDEX idx_notifications_order_id ON notifications(order_id);
```

---

## Data Integrity Constraints

### Unique Constraints
```sql
ALTER TABLE clients ADD CONSTRAINT uk_clients_email UNIQUE (email);
ALTER TABLE users ADD CONSTRAINT uk_users_email UNIQUE (email);
ALTER TABLE orders ADD CONSTRAINT uk_orders_order_number UNIQUE (order_number);
ALTER TABLE services ADD CONSTRAINT uk_services_name UNIQUE (name);
```

### Check Constraints
```sql
ALTER TABLE orders ADD CONSTRAINT ck_orders_total_price CHECK (total_price >= 0);
ALTER TABLE payments ADD CONSTRAINT ck_payments_amount CHECK (amount > 0);
ALTER TABLE services ADD CONSTRAINT ck_services_price CHECK (base_price > 0);
```

---

## Backup & Maintenance

### Backup Strategy
- Daily automated backups at 02:00 UTC
- Weekly backups retained for 4 weeks
- Point-in-time recovery enabled for 30 days

### Archive Strategy
- Orders completed > 90 days ago: Move to archive table
- Files deleted > 30 days ago: Permanent removal
- Notifications > 90 days old: Archive

---

This schema is designed to be:
✅ Scalable for growth
✅ Secure with proper constraints
✅ Maintainable with clear structure
✅ Efficient with strategic indexing
✅ Auditable with history tracking
