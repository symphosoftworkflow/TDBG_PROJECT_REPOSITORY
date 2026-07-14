# เอกสาร การออกแบบระบบ (SYSTEM SOFTWARE DESIGN)

**ชื่อระบบงาน [TH]** : ระบบยกระดับประสิทธิภาพกระบวนการบริหารจัดการข้อมูลภาษีและต้นทุนการนำเข้าด้วยเทคโนโลยีดิจิทัล  
**ชื่อระบบงาน [EN]** : TradeBridge Digital — Import Tax & Cost Intelligence Platform

**ชื่อย่อโครงการ** : TBDG  
**เวอร์ชัน** : 1.0  
**จัดทำโดย** : นายปริญญา พงษ์ดนตรี (PM)  
**วันที่อนุมัติเอกสาร** : [____ เดือน พ.ศ. ____]

---

## ประวัติการจัดทำเอกสาร

| ลำดับ | เวอร์ชัน | รายละเอียดการดำเนินการ | ผู้ดำเนินการ | วันที่ดำเนินการ |
| ----- | -------- | ---------------------- | ------------ | --------------- |
| 1 | 0.1 | จัดทำเอกสารการออกแบบระบบ | นายปริญญา พงษ์ดนตรี (PM) | [____] |
| 2 | 1.0 | อนุมัติเอกสารการออกแบบระบบ | ________________________ (TL) | [____] |

---

## สารบัญ

1. [ภาพรวมการออกแบบระบบ](#1-ภาพรวมการออกแบบระบบ-system-and-software-design-overview)
2. [การออกแบบระบบ (System Design)](#2-การออกแบบระบบ-system-design)
3. [การออกแบบซอฟต์แวร์ (Software Design)](#3-การออกแบบซอฟต์แวร์-software-design)
4. [แผนภาพประกอบ (Diagrams)](#4-แผนภาพประกอบ-diagrams)
5. [API Payload Specification](#5-api-payload-specification)
6. [User Interface Design](#6-user-interface-design)
7. [ภาคผนวก](#7-ภาคผนวก)

---

## 1. ภาพรวมการออกแบบระบบ (System and Software Design Overview)

TradeBridge Digital ออกแบบเป็นเว็บแอปพลิเคชันที่ให้ผู้นำเข้า (บุคคลธรรมดา บริษัท หรือ broker) สมัครใช้งาน เลือกแพ็กเกจ ชำระเงินผ่าน Payment Gateway แล้วเข้าใช้โมดูลธุรกิจตามสิทธิ์ ได้แก่ การวิเคราะห์ต้นทุนนำเข้า การแนะนำพิกัดศุลกากร การเตรียมใบขน และการตรวจสอบย้อนหลัง

เอกสารนี้แปลงความต้องการใน TBDG_SRS / TBDG_SOW เป็นสถาปัตยกรรม ข้อมูล API และหน้าจอ (DES) สำหรับการพัฒนา โดยใช้แนวทาง **modular monolith** ฝั่ง backend เพื่อให้ส่งมอบได้ภายในกรอบระยะเวลาและงบประมาณของโครงการ

---

## 2. การออกแบบระบบ (System Design)

### 2.1 แนวคิดการออกแบบสถาปัตยกรรมและมาตรฐาน (Architecture Concept Design and Standard)

ระบบประกอบด้วยชั้นหลัก 3 ชั้น:

1. **Presentation** — Svelte 5 SPA + shadcn-svelte ทำงานบนเบราว์เซอร์
2. **Application API** — Go HTTP API (JSON REST) ยืนยันตัวตนด้วย JWT และบังคับสิทธิ์แพ็กเกจผ่าน entitlement middleware
3. **Data & External** — PostgreSQL เป็นระบบจัดเก็บหลัก; Payment Gateway สำหรับ checkout และ webhook

![Architecture Overview](https://symphosoftworkflow.github.io/TDBG_PROJECT_REPOSITORY/BASELINE/diagrams/01_architecture_overview.svg)

**มาตรฐานการสื่อสาร**

* HTTP/HTTPS + JSON
* Authorization: `Bearer <JWT>`
* สิทธิ์แพ็กเกจตรวจที่ middleware ก่อนเข้า endpoint ของโมดูลธุรกิจ
* Timezone จัดเก็บเป็น UTC; แสดงผลตามเขตเวลาท้องถิ่นที่ client

### 2.2 รายละเอียดการออกแบบระบบ (System Detail Design)

Backend แบ่งเป็น package ตามขอบเขตธุรกิจ (bounded contexts) ใน process เดียว:

| Go Package | หน้าที่ | REQ ที่เกี่ยวข้อง |
| ---------- | ------- | ----------------- |
| `auth` | สมัคร เข้าสู่ระบบ โปรไฟล์ | REQ0.1, REQ0.2, REQ6.1 |
| `billing` | แพ็กเกจ checkout webhook subscription ประวัติชำระเงิน | REQ0.3–REQ0.9 |
| `entitlement` | ตรวจสิทธิ์ตาม Package → Feature Matrix + Trial quota | REQ0.5, REQ0.6, REQ0.7 |
| `tariff` | HS DB ค้นหา แนะนำพิกัด (รวมรับรูปภาพ) | REQ2.* |
| `pretrade` | Cost Estimation / FTA / Green-Red | REQ1.* |
| `entryprep` | อัปโหลดเอกสาร สกัดข้อมูล preflight NSP payload | REQ3.* |
| `postaudit` | ตรวจสอบย้อนหลัง findings คำแนะนำ | REQ4.* |
| `history` | Dashboard / ประวัติการใช้งาน | REQ6.3 |
| `admin` | รายการผู้ใช้ สถานะข้อมูลอ้างอิง | REQ6.1, REQ6.2 |

Frontend จัดโครงสร้างตาม route ที่สอดคล้อง DES ในข้อ 6 โดยเรียก API ผ่าน typed client และจัดการ state แพ็กเกจ/โควต้าหลัง login

---

## 3. การออกแบบซอฟต์แวร์ (Software Design)

### 3.1 แนวคิดการออกแบบสถาปัตยกรรมและมาตรฐาน (Architecture Concept Design and Standard)

| ชั้น | เทคโนโลยี | หมายเหตุ |
| ---- | --------- | -------- |
| Frontend | Svelte 5 + shadcn-svelte | SPA, responsive web |
| Backend | Go | Modular monolith, REST JSON |
| Database | PostgreSQL | แหล่งข้อมูลหลัก |
| Auth | JWT (access) + password hash (bcrypt/argon2) | HTTPS บน production |
| Payment | Payment Gateway (vendor-agnostic) | Checkout session + webhook |
| Files | Object storage หรือ filesystem ที่กำหนดใน ops | เอกสารนำเข้า / รูปสินค้า |

**หลักการออกแบบ**

* Rule-based business logic (ไม่ใช้ ML ใน MVP)
* Idempotent webhook handling สำหรับการชำระเงิน
* Soft-delete / audit fields สำหรับเอกสารและการชำระเงินที่สำคัญ
* Tariff / FTA rates ระบุ `effective_date` และ `updated_at` ให้ API ส่งกลับไปแสดงที่ UI

### 3.2 รายละเอียดการออกแบบซอฟต์แวร์ (Software Detail Design)

สรุปการออกแบบตามกลุ่มผู้ใช้และโมดูล (รายละเอียดหน้าจอเต็มในข้อ 6):

#### 3.2.1 Access Layer (ทุกประเภทบัญชี)

| Design ID | การออกแบบ | REQ |
| --------- | --------- | --- |
| DES0.1 | หน้าสมัครบัญชี เลือกประเภท บุคคลธรรมดา / บริษัท / broker | REQ0.1 |
| DES0.2 | หน้าเข้าสู่ระบบ / ออกจากระบบ | REQ0.2 |
| DES0.3 | หน้า Package Offer (Trial / Importer / Broker) | REQ0.3 |
| DES0.4 | Checkout + แสดงสถานะชำระเงิน | REQ0.4, REQ0.9 |
| DES0.5 | Account / Subscription status + ประวัติชำระเงิน | REQ0.5–REQ0.8 |

#### 3.2.2 โมดูลธุรกิจ (หลังผ่าน entitlement)

| Design ID | การออกแบบ | Package | REQ |
| --------- | --------- | ------- | --- |
| DES1.1 | Dashboard Overview | Trial+ | REQ6.3 |
| DES1.2 | Pre-trade Cost Estimation | Trial+ | REQ1.* |
| DES2.1 | Tariff Intelligence (ข้อความ + รูปภาพ) | Trial+ | REQ2.* |
| DES3.1 | Document Prep อัปโหลดเอกสาร | Broker | REQ3.1, REQ3.5 |
| DES3.2 | Pre-flight + NSP JSON preview | Broker | REQ3.2–REQ3.4 |
| DES4.1 | Post-Audit verification | Broker | REQ4.* |
| DES5.1 | Trade / Analysis History | Trial+ (จำกัดบน Trial) | REQ6.3 |
| DES6.1 | Admin: users + data freshness | Admin | REQ6.1, REQ6.2 |

### 3.3 รายละเอียดส่วนติดต่อ (Interface Detail Design)

#### 3.3.1 Frontend routes (สรุป)

| Route | หน้าจอ | Auth |
| ----- | ------ | ---- |
| `/register` | DES0.1 | Public |
| `/login` | DES0.2 | Public |
| `/pricing` | DES0.3 | Public / Logged-in |
| `/checkout` | DES0.4 | Logged-in |
| `/account` | DES0.5 | Logged-in |
| `/app` | DES1.1 Dashboard | Entitled |
| `/app/pretrade` | DES1.2 | Trial+ |
| `/app/tariff` | DES2.1 | Trial+ |
| `/app/entry` | DES3.1–3.2 | Broker |
| `/app/postaudit` | DES4.1 | Broker |
| `/app/history` | DES5.1 | Trial+ |
| `/admin` | DES6.1 | Admin |

เมื่อ entitlement ไม่พอ: UI แสดง upgrade banner และ API คืน `403` พร้อม `code: ENTITLEMENT_REQUIRED`

#### 3.3.2 API base

* Base path: `/api/v1`
* Content-Type: `application/json` (ยกเว้น upload เป็น `multipart/form-data`)
* รายละเอียด payload ในข้อ 5

### 3.4 การออกแบบข้อมูล (Data Element Design)

#### 3.4.1 ตารางหลัก Access / Billing

| Table | Column | Type (แนวทาง) | หมายเหตุ |
| ----- | ------ | ------------- | -------- |
| `users` | `id` | ULID/UUID PK | |
| | `email` | varchar unique | login |
| | `password_hash` | varchar | bcrypt/argon2 |
| | `account_type` | enum | individual / company / broker |
| | `display_name` | varchar | |
| | `company_name` | varchar nullable | จำเป็นเมื่อ company/broker |
| | `tax_id` | varchar nullable | เลขผู้เสียภาษี |
| | `role` | enum | user / admin |
| | `created_at` / `updated_at` | timestamptz | |
| `packages` | `id` | PK | |
| | `code` | varchar unique | trial / importer / broker |
| | `name` | varchar | |
| | `price_thb` | numeric | 0 สำหรับ trial |
| | `duration_days` | int | 30 หรือ 365 |
| | `features_json` | jsonb | feature flags |
| | `is_active` | boolean | |
| `subscriptions` | `id` | PK | |
| | `user_id` / `package_id` | FK | |
| | `status` | enum | trial / active / expired / cancelled |
| | `starts_at` / `ends_at` | timestamptz | |
| `payments` | `id` | PK | |
| | `user_id` / `subscription_id` | FK | subscription อาจว่างตอน pending |
| | `provider_ref` | varchar | transaction ของ gateway |
| | `amount_thb` | numeric | |
| | `status` | enum | pending / success / failed / cancelled |
| | `raw_payload` | jsonb | สำหรับ audit (REQ0.9) |
| | `paid_at` | timestamptz nullable | |
| `usage_quotas` | `user_id` | FK | หนึ่งแถวต่อรอบ trial |
| | `period_start` / `period_end` | timestamptz | |
| | `pretrade_used` / `tariff_used` | int | |
| | `pretrade_limit` / `tariff_limit` | int | ค่าเริ่มต้น 10 / 10 |

#### 3.4.2 ตารางหลักธุรกิจ

| Table | Column | Type (แนวทาง) | หมายเหตุ |
| ----- | ------ | ------------- | -------- |
| `tariff_hs_codes` | `hs_code` | varchar PK/UK | 6–8 หลัก |
| | `stat_code` | varchar | 3 หลัก |
| | `description_th` / `description_en` | text | |
| | `updated_at` | timestamptz | แสดงบน UI |
| `fta_rates` | `hs_code` | FK/varchar | |
| | `scheme` | enum | MFN / ACFTA / AFTA / RCEP |
| | `origin_country` | varchar | ISO country |
| | `rate_pct` | numeric | |
| | `effective_date` | date | |
| | `updated_at` | timestamptz | |
| `analyses` | `id` | PK | Pre-trade run |
| | `user_id` | FK | |
| | `input_json` / `result_json` | jsonb | input ตาม REQ1.1; result ตาม REQ1.2–1.9 |
| | `hs_code` / `origin` / `cif_thb` | denorm | สำหรับ list/filter |
| | `created_at` | timestamptz | |
| `documents` | `id` | PK | |
| | `user_id` | FK | |
| | `entry_draft_id` / `audit_case_id` | FK nullable | ผูก workflow |
| | `doc_type` | varchar | invoice / packing_list / bl / coo / insurance / entry / other |
| | `storage_key` / `checksum` / `version` | | version รองรับ re-upload (REQ3.5) |
| | `created_at` | timestamptz | |
| `entry_drafts` | `id` | PK | |
| | `user_id` | FK | Broker only |
| | `fields_json` | jsonb | ชุดฟิลด์ภาคผนวก ก ของ SRS |
| | `status` | enum | draft / processed / exported |
| | `updated_at` | timestamptz | |
| `preflight_results` | `id` | PK | |
| | `entry_draft_id` | FK | |
| | `checklist_json` | jsonb | ความครบ/สอดคล้อง |
| | `duty_preview_json` | jsonb | foreshadow ภาษี |
| | `nsp_payload_json` | jsonb | passthrough |
| | `created_at` | timestamptz | |
| `audit_cases` | `id` | PK | |
| | `user_id` | FK | |
| | `input_meta_json` | jsonb | |
| | `status` | enum | open / completed |
| | `created_at` | timestamptz | |
| `audit_findings` | `id` | PK | |
| | `audit_case_id` | FK | |
| | `severity_code` | varchar | hs / value / fta / document |
| | `severity` / `recommendation` | text | |
| | `severity` | enum | info / warn / high |

#### 3.4.3 ชุดฟิลด์ `entry_drafts.fields_json` (สรุปจาก SRS ภาคผนวก ก)

อย่างน้อยต้องรองรับคีย์: `bl_no`, `vessel_name`, `invoice_no`, `invoice_date`, `marks_numbers`, `package_count`, `origin_country`, `export_country`, `import_port`, `hs_code`, `stat_code`, `preference_code`, `goods_value_fx`, `goods_value_thb`, `net_weight`, `exchange_rate`, `duty_customs`, `vat`, `excise`, `interior_tax`, `goods_description`, `brand`, `quantity`, `permit_nos`, `coo_form_nos`

#### 3.4.4 การออกแบบที่ตอบ Non-functional (SRS §4)

| NFR (SRS) | การออกแบบใน SDD |
| --------- | ---------------- |
| UI ครอบคลุม Access + โมดูล 1–4 + dashboard | DES0.*–DES6.1 |
| Usability | คำอธิบายผลลัพธ์สั้น ๆ บนหน้า Pre-trade / Pre-flight / Post-Audit |
| ≥ 100 concurrent sessions | Go API แบบ stateless JWT; connection pool Postgres |
| Downtime ≤ 0.5%/เดือน | Deployment health check + DB backup ตามแผน ops |
| TLS / เข้ารหัสข้อมูลสำคัญ / RBAC | HTTPS; hash รหัสผ่าน; JWT + role + entitlement middleware |

---

## 4. แผนภาพประกอบ (Diagrams)

> **Note (maintainers):** source Mermaid files live in `BASELINE/diagrams/*.mmd` — ไฟล์ต้นฉบับ Mermaid อยู่ที่โฟลเดอร์นี้สำหรับแก้ไดอะแกรม

### 4.1 แผนภาพ Component (Component Diagram)

![Component Diagram](https://symphosoftworkflow.github.io/TDBG_PROJECT_REPOSITORY/BASELINE/diagrams/02_component.svg)

### 4.2 แผนภาพโดเมนหลัก (Domain / Class Overview)

![Domain Class Diagram](https://symphosoftworkflow.github.io/TDBG_PROJECT_REPOSITORY/BASELINE/diagrams/03_domain_class.svg)

### 4.3 แผนภาพ Sequence — Payment → Entitlement

![Payment Sequence](https://symphosoftworkflow.github.io/TDBG_PROJECT_REPOSITORY/BASELINE/diagrams/04_seq_payment.svg)

### 4.4 แผนภาพ Sequence — Pre-trade Analysis

![Pre-Trade Sequence](https://symphosoftworkflow.github.io/TDBG_PROJECT_REPOSITORY/BASELINE/diagrams/05_seq_pretrade.svg)

### 4.5 แผนภาพ Sequence — Entry Preflight

![Entry Sequence](https://symphosoftworkflow.github.io/TDBG_PROJECT_REPOSITORY/BASELINE/diagrams/06_seq_entry.svg)

### 4.6 แผนภาพ Deployment (Deployment Diagram)

![Deployment Diagram](https://symphosoftworkflow.github.io/TDBG_PROJECT_REPOSITORY/BASELINE/diagrams/07_deployment.svg)

### 4.7 แผนภาพ Use Case (Use Case Diagram)

![Use Case Diagram](https://symphosoftworkflow.github.io/TDBG_PROJECT_REPOSITORY/BASELINE/diagrams/08_usecase.svg)

### 4.8 แผนภาพการไหลของผู้ใช้ (User Flow Diagram)

![User Flow](https://symphosoftworkflow.github.io/TDBG_PROJECT_REPOSITORY/BASELINE/diagrams/09_userflow.svg)

### 4.9 Enhanced Entity-Relationship (ER Model)

![ER Diagram](https://symphosoftworkflow.github.io/TDBG_PROJECT_REPOSITORY/BASELINE/diagrams/10_er.svg)

### 4.10 Data Dictionary (สรุป)

ดูรายละเอียดฟิลด์ในข้อ 3.4 — ความสัมพันธ์หลักคือ `users` เป็นศูนย์กลางของ `subscriptions`, `payments`, `analyses`, `entry_drafts`, `audit_cases` และ `usage_quotas`

---

## 5. API Payload Specification

Base URL: `/api/v1`  
Auth header: `Authorization: Bearer <token>` (ยกเว้น register/login และ webhook ที่ใช้ลายเซ็นของผู้ให้บริการชำระเงิน)

### 5.1 Auth

**POST `/auth/register`**

```json
{
  "email": "user@example.com",
  "password": "********",
  "account_type": "company",
  "display_name": "สมชาย ใจดี",
  "company_name": "บริษัท ตัวอย่าง จำกัด",
  "tax_id": "0105559000000"
}
```

**Response `201`**

```json
{
  "user": {
    "id": "ulid_user",
    "email": "user@example.com",
    "account_type": "company",
    "role": "user"
  },
  "access_token": "<jwt>"
}
```

**POST `/auth/login`**

```json
{ "email": "user@example.com", "password": "********" }
```

**GET `/auth/me`** — คืนโปรไฟล์ + สิทธิ์แพ็กเกจปัจจุบัน (ถ้ามี)

```json
{
  "user": {
    "id": "ulid_user",
    "email": "user@example.com",
    "account_type": "company",
    "display_name": "สมชาย ใจดี",
    "role": "user"
  },
  "subscription": {
    "package_code": "trial",
    "status": "trial",
    "starts_at": "2026-07-01T00:00:00Z",
    "ends_at": "2026-07-31T00:00:00Z"
  },
  "quota": {
    "pretrade_used": 2,
    "pretrade_limit": 10,
    "tariff_used": 1,
    "tariff_limit": 10
  }
}
```

**POST `/auth/logout`** — invalidate client token (stateless JWT: client ทิ้ง token; optional denylist ตามนโยบาย ops)

### 5.2 Billing

**GET `/billing/packages`**

```json
{
  "items": [
    {
      "code": "trial",
      "name": "Trial 30 วัน",
      "price_thb": 0,
      "duration_days": 30,
      "features": ["module1", "module2", "dashboard_limited"]
    },
    {
      "code": "importer",
      "name": "Importer",
      "price_thb": 9600,
      "duration_days": 365,
      "features": ["module1", "module2", "dashboard"]
    },
    {
      "code": "broker",
      "name": "Broker",
      "price_thb": 20000,
      "duration_days": 365,
      "features": ["module1", "module2", "module3", "module4", "dashboard"]
    }
  ]
}
```

> ราคาแพ็กเกจในตัวอย่างอ้างอิงแนวเสนอผลิตภัณฑ์เดิม สามารถปรับค่าจริงในการตั้งค่า `packages` ได้โดยไม่ต้องเปลี่ยนโค้ดหน้าจอ

**POST `/billing/trial/start`** — เริ่ม Trial (ถ้ายังไม่เคยใช้ตามนโยบาย)

**POST `/billing/checkout`**

```json
{ "package_code": "importer" }
```

**Response**

```json
{
  "payment_id": "ulid_pay",
  "checkout_url": "https://payment-gateway.example/checkout/sess_xxx",
  "status": "pending"
}
```

**POST `/billing/webhook`** — ผู้ให้บริการเรียกกลับเมื่อสถานะเปลี่ยน (ต้อง verify signature; idempotent ตาม `provider_ref`)

แนวทาง payload ภายในหลัง normalize:

```json
{
  "provider_ref": "sess_xxx",
  "payment_id": "ulid_pay",
  "status": "success",
  "amount_thb": 9600,
  "paid_at": "2026-07-14T04:00:00Z"
}
```

**GET `/billing/subscription`**

```json
{
  "package_code": "importer",
  "status": "active",
  "starts_at": "2026-07-14T04:00:00Z",
  "ends_at": "2027-07-14T04:00:00Z",
  "features": ["module1", "module2", "dashboard"]
}
```

**GET `/billing/payments`** — ประวัติชำระเงินของบัญชี (`provider_ref`, amount, status, paid_at)

### 5.3 Entitlement errors

```json
{
  "error": {
    "code": "ENTITLEMENT_REQUIRED",
    "message": "Package Broker is required for this feature",
    "required_package": "broker"
  }
}
```

```json
{
  "error": {
    "code": "QUOTA_EXCEEDED",
    "message": "Trial pretrade quota exceeded",
    "limit": 10,
    "used": 10
  }
}
```

### 5.4 Tariff

**GET `/tariff/hs?query=เสื้อ`**

```json
{
  "items": [
    {
      "hs_code": "6109.10",
      "stat_code": "000",
      "description_th": "เสื้อยืด ...",
      "updated_at": "2026-07-01T00:00:00Z"
    }
  ]
}
```

**POST `/tariff/suggest`** — `multipart/form-data`

| Part | Required | หมายเหตุ |
| ---- | -------- | -------- |
| `image` | No | รูปสินค้า (REQ2.1) |
| `technical_text` | Yes* | *อย่างน้อยหนึ่งใน image หรือ technical_text |
| `origin_hs` | No | พิกัดประเทศต้นทาง |
| `expected_hs` | No | พิกัดที่ผู้ใช้คาดหวัง |

**Response**

```json
{
  "suggestions": [
    { "hs_code": "6109.10", "stat_code": "000", "confidence": 0.86, "rationale": "cotton knitted T-shirt" }
  ],
  "rates": [
    { "scheme": "MFN", "rate_pct": 10 },
    { "scheme": "ACFTA", "rate_pct": 0 }
  ],
  "rates_updated_at": "2026-07-01T00:00:00Z"
}
```

**GET `/tariff/rates?hs_code=6109.10&origin=CN`**

### 5.5 Pre-trade

**POST `/pretrade/analyses`**

```json
{
  "product_name": "เสื้อยืด cotton",
  "hs_code": "6109.10",
  "stat_code": "000",
  "origin_country": "CN",
  "origin_port": "CNSHA",
  "dest_country": "TH",
  "dest_port": "THLCH",
  "incoterm": "FOB",
  "transport_mode": "sea",
  "invoice_thb": 650000,
  "freight_thb": 25000,
  "insurance_thb": 5000
}
```

**Response `201`**

```json
{
  "id": "ulid_analysis",
  "cif_thb": 680000,
  "duties": { "customs": 0, "vat": 47600, "excise": 0, "interior": 0 },
  "logistics_estimate": {
    "doc_fee_thb": 1500,
    "terminal_fee_thb": 8000
  },
  "fta_options": [
    { "scheme": "ACFTA", "rate_pct": 0, "saving_thb": 68000 },
    { "scheme": "RCEP", "rate_pct": 5, "saving_thb": 34000 }
  ],
  "oga_notes": [{ "agency": "example", "permit_hint": "ตรวจเงื่อนไขนำเข้าเพิ่มเติมถ้าเข้าข่าย" }],
  "carrier_suggestions": [{ "name": "Carrier A", "mode": "sea" }],
  "broker_suggestions": [{ "name": "Licensed Broker B" }],
  "transit_days_est": 12,
  "green_red_indicator": "green",
  "rates_updated_at": "2026-07-01T00:00:00Z"
}
```

**GET `/pretrade/analyses/{id}`** — ดึงผลเดิมสำหรับหน้า History / Dashboard

### 5.6 Entry preparation (Broker)

**POST `/entry/documents`** — `multipart/form-data`

| Part | หมายเหตุ |
| ---- | -------- |
| `files[]` | หลายไฟล์ |
| `doc_types[]` | ขนานกับไฟล์: invoice / packing_list / bl / customs_value / coo / insurance / other |
| `entry_draft_id` | optional — สร้างใหม่ถ้าไม่ส่ง |

**POST `/entry/drafts/process`**

```json
{ "entry_draft_id": "ulid_draft", "voyage": { "vessel_name": "...", "bl_no": "..." } }
```

**Response**

```json
{
  "entry_draft_id": "ulid_draft",
  "fields": {
    "bl_no": "...",
    "hs_code": "6109.10",
    "stat_code": "000",
    "goods_value_thb": 680000
  },
  "preflight": {
    "checklist": [
      { "item": "invoice", "status": "ok" },
      { "item": "coo", "status": "missing" }
    ],
    "duty_preview": { "customs": 0, "vat": 47600 }
  },
  "nsp_payload": { "schema_version": "1.0", "declaration": {} }
}
```

**GET `/entry/drafts/{id}/nsp-payload`** — คืน JSON สำหรับ copy/export ไป NSP

### 5.7 Post-audit (Broker)

**POST `/postaudit/cases`** — multipart: ใบขน + เอกสารประกอบ + `meta` JSON  
**GET `/postaudit/cases/{id}`**

```json
{
  "id": "ulid_case",
  "status": "completed",
  "findings": [
    {
      "severity_code": "hs",
      "severity": "high",
      "severity": "พิกัดที่สำแดงไม่สอดคล้องกับคำอธิบายสินค้า",
      "recommendation": "ทบทวน HS กับเอกสารเทคนิคและ Invoice"
    }
  ]
}
```

### 5.8 History / Admin

**GET `/history/items?q=&origin=&type=&status=&limit=`**

```json
{
  "items": [
    {
      "type": "analysis",
      "id": "ulid_analysis",
      "title": "เสื้อยืด cotton",
      "origin": "CN",
      "created_at": "2026-07-10T10:00:00Z"
    }
  ],
  "quota_note": "trial_limited"
}
```

**GET `/admin/users`** — รายการผู้ใช้ + role + package สรุป  
**PATCH `/admin/users/{id}`** — ปรับ `role` หรือสถานะบัญชีเบื้องต้น  
**GET `/admin/reference-freshness`**

```json
{
  "tariff_hs_codes_updated_at": "2026-07-01T00:00:00Z",
  "fta_rates_updated_at": "2026-07-01T00:00:00Z",
  "hs_count": 200
}
```

### 5.9 API endpoint catalogue (สรุป)

| Method | Path | Auth | Entitlement |
| ------ | ---- | ---- | ----------- |
| POST | `/auth/register` | Public | — |
| POST | `/auth/login` | Public | — |
| POST | `/auth/logout` | JWT | — |
| GET | `/auth/me` | JWT | — |
| GET | `/billing/packages` | Public/JWT | — |
| POST | `/billing/trial/start` | JWT | — |
| POST | `/billing/checkout` | JWT | — |
| POST | `/billing/webhook` | Signature | — |
| GET | `/billing/subscription` | JWT | — |
| GET | `/billing/payments` | JWT | — |
| GET | `/tariff/hs` | JWT | Trial+ |
| POST | `/tariff/suggest` | JWT | Trial+ (+quota) |
| GET | `/tariff/rates` | JWT | Trial+ |
| POST | `/pretrade/analyses` | JWT | Trial+ (+quota) |
| GET | `/pretrade/analyses/{id}` | JWT | Trial+ |
| POST | `/entry/documents` | JWT | Broker |
| POST | `/entry/drafts/process` | JWT | Broker |
| GET | `/entry/drafts/{id}/nsp-payload` | JWT | Broker |
| POST | `/postaudit/cases` | JWT | Broker |
| GET | `/postaudit/cases/{id}` | JWT | Broker |
| GET | `/history/items` | JWT | Trial+ |
| GET | `/admin/users` | JWT | Admin |
| PATCH | `/admin/users/{id}` | JWT | Admin |
| GET | `/admin/reference-freshness` | JWT | Admin |

---

## 6. User Interface Design

แนว UI อิง workflow และองค์ประกอบจาก mockup เดิม (Overview / Pre-trade / Document Prep / History / Pricing) แต่ปรับเป็นหน้าจอผลิตภัณฑ์จริงบน Svelte 5 + shadcn-svelte และเพิ่ม Auth/Account ที่ mockup ยังไม่มี

**ไม่อยู่ใน UI ของ MVP:** หน้าเปรียบเทียบ TB vs TCS, CBAM tracker, ROI calculator แบบการตลาด (คงเฉพาะตารางเปรียบเทียบฟีเจอร์ใน Package Offer)

### 6.1 Access Layer

#### DES0.1 Register — `/register`

* **Purpose:** สมัครบัญชีผู้นำเข้า
* **Layout (wireframe):**

```
[ Brand TradeBridge Digital ]
  Account type: ( ) บุคคลธรรมดา  ( ) บริษัท  ( ) Broker
  Email / Password / Display name
  [Company name] [Tax ID]   ← แสดงเมื่อเลือกบริษัท/broker
  [ สมัครใช้งาน ]
  มีบัญชีแล้ว? เข้าสู่ระบบ
```

* **Primary components:** Card, Input, RadioGroup/Select, Button, Alert
* **Actions:** Submit → auto-login → `/pricing`
* **States:** validation error, email already exists
* **REQ:** REQ0.1

#### DES0.2 Login — `/login`

* **Purpose:** เข้าสู่ระบบ / ออกจากระบบทำที่เมนูบัญชี
* **Fields:** email, password
* **Actions:** Login → `/app` ถ้ามี entitlement ใช้งานได้, ไม่เช่นนั้น `/pricing`
* **REQ:** REQ0.2

#### DES0.3 Package Offer — `/pricing`

* **Purpose:** แสดง Trial / Importer / Broker + feature matrix (แนว Pricing mockup)
* **Layout:**

```
[ Trial 30d ] [ Importer ] [ Broker ]
   ฟีเจอร์     ฟีเจอร์       ฟีเจอร์
   [เริ่มทดลอง] [ชำระเงิน]  [ชำระเงิน]
--------------- feature comparison table ---------------
```

* **Actions:** เริ่ม Trial / เลือกแผนแล้วไป Checkout
* **Empty/Error:** โหลดแพ็กเกจไม่สำเร็จ → retry
* **REQ:** REQ0.3

#### DES0.4 Checkout — `/checkout`

* **Purpose:** สร้างรายการชำระและพาไป Payment Gateway
* **UI:** สรุปแพ็กเกจ ราคา ปุ่มชำระเงิน; กลับจาก gateway แสดง pending/success/failed/cancelled
* **REQ:** REQ0.4, REQ0.9

#### DES0.5 Account / Subscription — `/account`

* **Purpose:** แพ็กเกจปัจจุบัน วันหมดอายุ โควต้า Trial ประวัติชำระเงิน
* **Layout:** สถานะแพ็กเกจด้านบน + ตาราง payments + ปุ่มอัปเกรด/ต่ออายุ
* **REQ:** REQ0.5–REQ0.8

### 6.2 Business screens

#### DES1.1 Dashboard Overview — `/app`

* **Purpose:** KPI สรุป + รายการ analysis/shipment ล่าสุด (แนว Overview mockup)
* **Layout:**

```
Sidebar: Overview | Pre-trade | Tariff | Entry* | Post-Audit* | History | Account
* Entry / Post-Audit แสดงเฉพาะ Broker (หรือ disabled + upgrade)
Main: [Package chip] [Quota Trial]
      [KPI: analyses] [KPI: tariff] [KPI: open entries]
      Recent list ...
```

* **Omit from MVP UI:** CBAM card / ROI widgets
* **REQ:** REQ6.3

#### DES1.2 Pre-trade Cost Estimation — `/app/pretrade`

* **Purpose:** วิเคราะห์ต้นทุนนำเข้า (แนว Pre-trade mockup)
* **Input fields (REQ1.1):** product_name, origin/dest country+port, incoterm, origin, transport_mode, hs_code+stat_code, invoice/freight/insurance
* **Result panes (REQ1.2–1.9):** CIF, duties breakdown, logistics estimate, FTA comparison, OGA notes, transit days, carrier/broker suggestions, Green/Red, `rates_updated_at`
* **Upgrade/Quota:** Trial เต็ม → Alert + CTA Pricing
* **REQ:** REQ1.1–REQ1.9

#### DES2.1 Tariff Intelligence — `/app/tariff`

* **Purpose:** ค้นหา/แนะนำพิกัด รวมอัปโหลดรูปภาพ (ขยายจาก autocomplete ใน mockup)
* **Layout:** ซ้าย — dropzone รูป + technical text + origin/expected HS; ขวา — suggestions + confidence + rates
* **REQ:** REQ2.1–REQ2.4

#### DES3.1 Document Prep — `/app/entry`

* **Purpose:** อัปโหลดเอกสาร + เมตาเที่ยวเรือ (แนว Document Prep mockup)
* **Uploads:** Invoice, Packing List, B/L, customs value docs, importer/tax id docs, COO/benefit docs
* **Gate:** Broker; อื่น ๆ เห็น upgrade wall
* **REQ:** REQ3.1, REQ3.5

#### DES3.2 Pre-flight & NSP Payload — `/app/entry` (ขั้นตอนถัดจาก DES3.1)

* **Purpose:** checklist ความครบ, duty preview, JSON preview + Copy
* **Actions:** Process → แสดงผล; Export / Copy NSP JSON
* **REQ:** REQ3.2–REQ3.4

#### DES4.1 Post-Audit — `/app/postaudit`

* **Purpose:** ตรวจสอบย้อนหลัง (ออกแบบใหม่จากแท็บ Compliance เดิมที่ไม่ใช่แค่ feedback form)
* **Layout:** อัปโหลดใบขน+เอกสาร → รายการ findings (severity) + recommendation ต่อรายการ → บันทึกเคส
* **Gate:** Broker
* **REQ:** REQ4.1–REQ4.4

#### DES5.1 Trade / Analysis History — `/app/history`

* **Purpose:** ค้นหา/กรองประวัติ (แนว History mockup)
* **Filters:** q, origin, type, status
* **Trial:** จำกัด 10 รายการล่าสุด + ข้อความอัปเกรด
* **REQ:** REQ6.3

#### DES6.1 Admin Console — `/admin`

* **Purpose:** รายการผู้ใช้ (role) + วันที่อัปเดต tariff/FTA + จำนวน HS ในฐาน
* **REQ:** REQ6.1, REQ6.2

### 6.3 shadcn-svelte component map

| ความต้องการ UI | Component แนะนำ |
| -------------- | ---------------- |
| ฟอร์ม/อินพุต | Input, Select, Textarea, Checkbox, Label, RadioGroup |
| การ์ดแพ็กเกจ/เมตริก | Card, Badge, Separator |
| ตารางประวัติ | Table, Pagination |
| อัปโหลด | custom dropzone บน Button/Card |
| สถานะ/แจ้งเตือน | Alert, Toast, Dialog |
| นำทางแอป | Sidebar / Tabs / Breadcrumb |

### 6.4 แผนภาพทางจอหลัก (UI navigation)

![UI Navigation](https://symphosoftworkflow.github.io/TDBG_PROJECT_REPOSITORY/BASELINE/diagrams/11_ui_nav.svg)

---

## 7. ภาคผนวก

### 7.1 Trial quota (ค่าเริ่มต้น MVP)

| รายการ | โควต้าต่อรอบ Trial 30 วัน |
| ------ | ------------------------: |
| Pre-trade analyses (`POST /pretrade/analyses`) | 10 |
| Tariff suggestions (`POST /tariff/suggest`) | 10 |
| History visibility | 10 รายการล่าสุด |

Importer / Broker: ไม่จำกัดโควต้ารายการแบบ Trial โดยยังอยู่ภายใต้ NFR 100 concurrent sessions

### 7.2 Package → Feature → API entitlement matrix

| Feature | Trial | Importer | Broker | API / UI |
| ------- | :---: | :------: | :----: | -------- |
| Cost Estimation | Y (quota) | Y | Y | DES1.2 / `/pretrade/*` |
| Tariff Intelligence | Y (quota) | Y | Y | DES2.1 / `/tariff/*` |
| Customs Entry Prep | N | N | Y | DES3.* / `/entry/*` |
| Post-Audit | N | N | Y | DES4.1 / `/postaudit/*` |
| Dashboard | Y limited | Y | Y | DES1.1 |
| History | Y limited | Y | Y | DES5.1 / `/history/*` |

### 7.3 REQ → Design / API Traceability (Must)

| REQ | Design / API |
| --- | ------------ |
| REQ0.1 | DES0.1 / `POST /auth/register` |
| REQ0.2 | DES0.2 / `POST /auth/login`, `GET /auth/me`, logout |
| REQ0.3 | DES0.3 / `GET /billing/packages` |
| REQ0.4 | DES0.4 / `POST /billing/checkout` |
| REQ0.5 | webhook → `subscriptions` / `GET /billing/subscription` |
| REQ0.6 | entitlement middleware + DES upgrade states / `403 ENTITLEMENT_REQUIRED` |
| REQ0.7 | `POST /billing/trial/start` + `usage_quotas` + expire policy |
| REQ0.8 | DES0.5 / subscription + `GET /billing/payments` |
| REQ0.9 | `payments.raw_payload` / provider_ref |
| REQ1.1–1.9 | DES1.2 / `POST /pretrade/analyses` (fields + result panes) |
| REQ2.1–2.4 | DES2.1 / `/tariff/hs`, `/tariff/suggest`, `/tariff/rates` |
| REQ3.1 | DES3.1 / `POST /entry/documents` |
| REQ3.2 | DES3.2 / `fields_json` ตามภาคผนวก ก |
| REQ3.3 | DES3.2 / preflight checklist + duty_preview |
| REQ3.4 | DES3.2 / `nsp_payload` + export endpoint |
| REQ3.5 | `documents.version` / storage_key |
| REQ4.1–4.4 | DES4.1 / `/postaudit/cases` + `audit_findings` |
| REQ6.1 | DES6.1 / `GET|PATCH /admin/users` |
| REQ6.2 | DES6.1 / `GET /admin/reference-freshness` |
| REQ6.3 | DES1.1, DES5.1 / `/history/items` |

### 7.4 NFR (SRS §4) → Design mapping

| SRS NFR topic | Design reference |
| ------------- | ---------------- |
| UI coverage | §6 DES catalog |
| Usability | result explanations on DES1.2 / DES3.2 / DES4.1 |
| Efficiency / 100 sessions | §2–3 Go stateless + DB pool |
| Reliability ≤0.5% downtime | §4.6 deployment + backup ops note |
| Security TLS / RBAC / encryption at rest for secrets | §3.1 JWT/TLS + roles + hashed passwords |

### 7.5 สิ่งที่จงใจไม่รวมใน SDD / Out-of-Scope DES

| รายการ | สถานะใน MVP product UI |
| ------ | ---------------------- |
| หน้า TB vs TCS comparison | ไม่รวม (marketing-only) |
| CBAM tracker | ไม่รวม |
| ROI calculator | ไม่รวม (เก็บเฉพาะ feature matrix ใน DES0.3) |
| Add-on คิดเงินรายใบขน | ไม่รวม |
| Mobile Native | ไม่รวม |
| ML / predictive / CBAM engine | ไม่รวม |
| Multi-seat enterprise billing | ไม่รวม |

---

## ส่วนอนุมัติเอกสาร

* [ ] อนุมัติ
* [ ] ไม่อนุมัติ

__________________________  
[Signature]

**อนุมัติโดย**: ________________________  
**ตำแหน่ง**: ________________________  
**วันที่อนุมัติ**: [____ เดือน พ.ศ. ____]

---

*ชื่อย่อเอกสาร: TBDG_SDD_v1_0*
