# VEMS Integration Workflow
## Detailed Integration Process and System Communication

**Version:** 1.0  
**Date:** November 17, 2025  
**Audience:** Technical Integration Teams

---

## Table of Contents
1. [Integration Architecture](#integration-architecture)
2. [Detailed Workflow Steps](#detailed-workflow-steps)
3. [Key Integration Benefits](#key-integration-benefits)

---

## Integration Architecture

### System Communication Flow

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                     Health Information Exchange (HIE)                         │
│                   • Facility Registry (FR Codes)                             │
│                   • Client Registry (CR Numbers)                             │
│                   • Master Facility & Service Data                           │
└────────────────────────────────────┬─────────────────────────────────────────┘
                                     │
                    ┌────────────────┼────────────────┐
                    │                │                │
                    ▼                ▼                ▼
         ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
         │      HMIS       │  │Provider Portal  │  │  Payer System   │
         │   (Hospital)    │  │  (Facility)     │  │     (SHA)       │
         └────────┬────────┘  └────────┬────────┘  └────────┬────────┘
                  │                    │                    │
                  └────────────────────┼────────────────────┘
                                       │
                                       ▼
                            ┌─────────────────────┐
                            │     VEMS API        │
                            │  • Cost Calculator  │
                            │  • Booking Engine   │
                            │  • Status Polling   │
                            └──────────┬──────────┘
                                       │
                    ┌──────────────────┼──────────────────┐
                    ▼                  ▼                  ▼
         ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
         │     Vendor      │  │    Facility     │  │   VEMS Admin    │
         │   Settlement    │  │   Settlement    │  │    Dashboard    │
         └─────────────────┘  └─────────────────┘  └─────────────────┘
```

### Integration Workflow Overview

The VEMS integration follows a bidirectional communication pattern that enables real-time cost distribution and claim status synchronization across healthcare systems:

#### **Phase 1: Discovery & Service Query**
External systems (HMIS, Provider Portals, Payer Systems) query the Health Information Exchange (HIE) to discover VEMS-enabled facilities and their contracted services. This ensures that only eligible facilities and services are presented to users during the booking process.

#### **Phase 2: Booking & Cost Distribution Request**
When a patient requires a VEMS-managed service, the external system submits a booking request to VEMS with patient, facility, and service details. VEMS immediately calculates and returns the cost distribution breakdown (Vendor Share, Facility Share, Service Fee), enabling transparent cost visibility before claim submission.

#### **Phase 3: Claim Status Polling & Real-Time Updates**
After the payer processes the claim, VEMS actively polls the payer system to retrieve claim status updates (approved, rejected, partially paid). This bidirectional synchronization ensures VEMS maintains accurate, real-time booking and payment status, enabling automated vendor and facility settlements.

This architecture eliminates manual reconciliation, reduces payment delays, and provides end-to-end visibility across all stakeholders in the healthcare payment ecosystem.

---

## Detailed Workflow Steps

### Step-by-Step Integration Process

#### Workflow Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ Step 1: Facility & Service Discovery (via HIE)                              │
└─────────────────────────────────────────────────────────────────────────────┘

    HMIS/Provider/Payer                HIE                        VEMS
           │                            │                          │
           │──── Query Facilities ─────>│                          │
           │     (VEMS-enabled)         │                          │
           │                            │                          │
           │<─── Facility List ─────────│                          │
           │     (FR Codes)             │                          │
           │                            │                          │
           │──── Query Services ────────┼────────────────────────> │
           │     (fr_code=FID-XX)       │                          │
           │                            │                          │
           │<─── Available Services ────┼───────────────────────── │
           │     (Codes, Rates, Shares) │                          │
           │                            │                          │

┌─────────────────────────────────────────────────────────────────────────────┐
│ Step 2: Patient Service Request & Cost Distribution                         │
└─────────────────────────────────────────────────────────────────────────────┘

    HMIS/Provider/Payer                HIE                        VEMS
           │                            │                          │
           │──── Validate Patient CR ───>│                          │
           │                            │                          │
           │<─── Patient Details ───────│                          │
           │                            │                          │
           │──── POST Booking Request ──┼────────────────────────> │
           │     {patientCR,            │                          │
           │      facilityFrCode,       │                          │
           │      claimID,              │                          │
           │      services[]}           │                          │
           │                            │                          │
           │                            │      [Cost Calculation]  │
           │                            │      • Vendor Share      │
           │                            │      • Facility Share    │
           │                            │      • Service Fee       │
           │                            │                          │
           │<─── Cost Distribution ─────┼───────────────────────── │
           │     {bookingNo,            │                          │
           │      vendorShare: 10000,   │                          │
           │      facilityShare: 4500,  │                          │
           │      serviceFee: 500,      │                          │
           │      vemsTotal: 15000}     │                          │
           │                            │                          │
           │──── Store bookingNo ───────│                          │
           │     Link to claimID        │                          │
           │                            │                          │

┌─────────────────────────────────────────────────────────────────────────────┐
│ Step 3: Claim Submission & Processing                                       │
└─────────────────────────────────────────────────────────────────────────────┘

    HMIS/Provider                  Payer System                   VEMS
           │                            │                          │
           │──── Submit Claim ─────────>│                          │
           │     (with bookingNo)       │                          │
           │                            │                          │
           │                            │──── Claim Processing ────│
           │                            │     (Adjudication)       │
           │                            │                          │
           │<─── Claim Acknowledgment ──│                          │
           │     (Claim Reference)      │                          │
           │                            │                          │

┌─────────────────────────────────────────────────────────────────────────────┐
│ Step 4: Real-Time Status Polling & Updates (Bidirectional Sync)            │
└─────────────────────────────────────────────────────────────────────────────┘

    HMIS/Provider                  Payer System                   VEMS
           │                            │                          │
           │                            │<──── Poll Claim Status ──│
           │                            │      (Every X minutes)   │
           │                            │                          │
           │                            │──── Claim Status ───────>│
           │                            │     {status: approved,   │
           │                            │      amountApproved,     │
           │                            │      paymentDate}        │
           │                            │                          │
           │                            │                      [Update Booking]
           │                            │                      • Status: Approved
           │                            │                      • Trigger Settlements
           │                            │                          │
           │<──── Booking Update ───────┼───────────────────────── │
           │     (Optional webhook)     │                          │
           │                            │                          │

┌─────────────────────────────────────────────────────────────────────────────┐
│ Step 5: Automated Settlement & Reconciliation                               │
└─────────────────────────────────────────────────────────────────────────────┘

         Vendor                    VEMS System                  Facility
           │                            │                          │
           │<──── Vendor Payment ───────│                          │
           │      KES 10,000           │──── Facility Payment ───>│
           │      (Direct Transfer)     │      KES 4,500           │
           │                            │      (Direct Transfer)   │
           │                            │                          │
           │<──── Payment Notification ─│──── Payment Notification>│
           │      (SMS/Email)           │      (SMS/Email)         │
           │                            │                          │
```

---

## Workflow Step Descriptions

### Step 1: Facility & Service Discovery

**Objective**: Identify VEMS-enabled facilities and available services

**Process**:
1. External systems (HMIS, Provider Portal, Payer) query HIE to identify healthcare facilities that have VEMS contracts
2. Systems retrieve facility FR codes from HIE's Facility Registry
3. Using the FR codes, systems query VEMS directly via `GET /hmis/v1/services?fr_code=FID-XX` to retrieve:
   - Available service codes
   - Service names and descriptions
   - SHA rates
   - Cost distribution details (vendor share, facility share, service fee)
4. This ensures only VEMS-eligible services are displayed to healthcare providers during patient registration

**Key Data Elements**:
- Facility FR Code (e.g., `FID-32-121583-3`)
- Service Codes (e.g., `F8735`, `F8736`)
- Vendor information
- Cost breakdowns

**Benefits**:
- Prevents submission of ineligible claims
- Provides upfront cost transparency
- Reduces claim rejections

---

### Step 2: Patient Service Request & Cost Distribution

**Objective**: Submit patient service request and receive real-time cost breakdown

**Process**:
1. When a patient needs a VEMS service, the external system validates patient identity through HIE using CR Number
2. HIE returns patient demographic and insurance eligibility details
3. The system submits a booking request to VEMS via `POST /hmis/v1/bookings/cost-distribution` with:
   - Patient CR Number
   - Facility FR Code
   - Claim ID from originating system
   - Service codes and dates
4. VEMS calculates real-time cost distribution based on:
   - Active vendor-facility contracts
   - Current service rates
   - Payment mode (SHA, cash, insurance)
5. VEMS returns detailed breakdown showing:
   - Vendor share amount
   - Facility share amount
   - Service fee amount
   - Total VEMS amount
   - Unique booking number
6. External system stores the VEMS `bookingNo` linked to their `claimID` for future reconciliation

**Key Data Elements**:
- Patient CR Number (e.g., `CR1234567`)
- Facility FR Code (e.g., `FID-32-121583-3`)
- Claim ID (e.g., `3663-gdhs-TAIFA-CARE`)
- Booking Number (e.g., `VB673D4F2A1B3E`)
- Cost distribution components

**Benefits**:
- Real-time cost transparency
- Accurate billing before claim submission
- Automated cost calculation
- Traceability via booking number

---

### Step 3: Claim Submission & Processing

**Objective**: Submit claim to payer with VEMS booking reference

**Process**:
1. HMIS or Provider Portal submits the claim to the Payer (SHA or Insurance) through standard claim submission channels
2. Claim includes:
   - Patient information
   - Service codes
   - VEMS booking number (for reference)
   - Total amount claimed
3. Payer processes the claim through their adjudication workflow:
   - Verify patient eligibility
   - Validate service coverage
   - Check VEMS booking reference
   - Approve, reject, or request additional information
4. Claim acknowledgment is returned to the submitting system with claim reference number

**Key Data Elements**:
- Claim ID
- VEMS Booking Number
- Service codes and amounts
- Adjudication status

**Benefits**:
- Standardized claim submission
- VEMS booking provides audit trail
- Facilitates automated reconciliation

---

### Step 4: Real-Time Status Polling & Updates

**Objective**: Maintain synchronized claim and booking status across all systems

**Process**:
1. VEMS automatically polls the Payer system at regular intervals (configurable, e.g., every 15-30 minutes)
2. Polling queries claim status using the claim ID associated with the booking
3. When claim status updates are detected (approved, rejected, partially paid, etc.), VEMS:
   - Updates internal booking status
   - Records approval amounts
   - Logs payment dates
   - Triggers settlement workflows (if approved)
4. This bidirectional sync ensures VEMS maintains accurate real-time data without manual intervention
5. Optional webhooks can notify external systems of booking status changes for their own record updates

**Key Data Elements**:
- Claim status (pending, approved, rejected, partially paid)
- Approval amount
- Payment date
- Rejection reason (if applicable)

**Benefits**:
- Eliminates manual status updates
- Real-time visibility for all stakeholders
- Automated settlement triggers
- Complete audit trail
- Reduced reconciliation effort

**Technical Implementation**:
- Scheduled polling jobs (cron/queue-based)
- Exponential backoff for failed polls
- Webhook notifications for status changes
- Event-driven settlement initiation

---

### Step 5: Automated Settlement & Reconciliation

**Objective**: Automatically distribute payments to vendors and facilities upon claim approval

**Process**:
1. Upon claim approval (detected via polling), VEMS automatically initiates settlement processes
2. VEMS triggers payment transactions based on pre-calculated cost distribution:
   - Vendor receives their contracted share (e.g., KES 10,000) via direct bank transfer
   - Facility receives their contracted share (e.g., KES 4,500) via direct bank transfer
   - VEMS retains the service fee (e.g., KES 500) for platform operations
3. Payment instructions are sent to integrated payment gateway (e.g., M-Pesa, bank transfers)
4. All stakeholders receive automated payment notifications via:
   - SMS
   - Email
   - In-app notifications
5. Complete transaction records are maintained for financial reporting and auditing

**Key Data Elements**:
- Settlement amounts per stakeholder
- Payment transaction IDs
- Payment dates
- Bank account/M-Pesa numbers
- Transaction status

**Benefits**:
- Eliminates payment delays
- Reduces manual reconciliation
- Transparent payment distribution
- Automated financial reporting
- Complete audit trail

**Payment Methods Supported**:
- Direct bank transfers (EFT)
- M-Pesa business payments
- Integrated payment gateways

---

## Key Integration Benefits

### For Healthcare Facilities (HMIS/Provider Portals)

✅ **Real-time cost transparency**: Know exact vendor and facility shares before claim submission

✅ **Reduced administrative burden**: No manual cost calculation or vendor coordination

✅ **Faster settlements**: Automated payment distribution upon claim approval

✅ **Accurate billing**: VEMS-validated service codes and rates prevent claim rejections

✅ **Audit compliance**: Complete digital trail from booking to settlement

---

### For Payers (SHA/Insurance Companies)

✅ **Standardized cost structure**: Consistent pricing across VEMS-contracted facilities

✅ **Reduced fraud risk**: Verified vendor-facility relationships and service delivery

✅ **Simplified reconciliation**: Clear cost distribution for audit trails

✅ **Real-time booking visibility**: Link claims to verified VEMS bookings

✅ **Improved claims accuracy**: Pre-validated service codes and rates

---

### For VEMS Platform

✅ **Bidirectional synchronization**: Automatic status updates from payer systems

✅ **Complete audit trail**: Track bookings from request to settlement

✅ **Automated settlements**: Trigger vendor and facility payments based on claim approvals

✅ **Performance analytics**: Monitor service delivery, approval rates, and payment cycles

✅ **Scalable architecture**: Handle multiple external system integrations simultaneously

---

### For Vendors & Facilities

✅ **Guaranteed payments**: Automated settlement upon claim approval

✅ **Payment transparency**: Clear visibility into share calculations

✅ **Reduced payment delays**: Eliminate manual reconciliation bottlenecks

✅ **Digital payment records**: Complete transaction history for financial reporting

✅ **Predictable cash flow**: Faster payment cycles improve financial planning

---

## Integration Patterns Summary

| Pattern | Description | Benefit |
|---------|-------------|---------|
| **Discovery via HIE** | Query facility and service availability through Health Information Exchange | Ensures data consistency and prevents invalid submissions |
| **Real-time Cost Calculation** | Immediate cost distribution upon booking request | Eliminates manual calculations and pricing errors |
| **Bidirectional Sync** | VEMS polls payer systems for claim status updates | Maintains real-time accuracy across all systems |
| **Automated Settlement** | Trigger payments based on claim approval events | Reduces payment delays and manual reconciliation |
| **Booking Reference Linking** | Link VEMS bookingNo to external claimID | Enables end-to-end traceability and audit compliance |

---

**For technical implementation details, API specifications, and code samples, refer to the main [Integration Guide](Readme.md).**

---

**End of Workflow Document**
