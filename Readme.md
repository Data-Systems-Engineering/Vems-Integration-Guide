# VEMS API Integration Guide
## Vendor Equipment Management System (VEMS) Integration Documentation

**Version:** 1.0  
**Date:** November 17, 2025  
**Audience:** Apeiro & HMIS Integration Teams

---

## Table of Contents
1. [Introduction](#introduction)
2. [Integration Overview](#integration-overview)
3. [Prerequisites](#prerequisites)
4. [Authentication](#authentication)
5. [API Endpoints](#api-endpoints)
6. [Request/Response Formats](#requestresponse-formats)
7. [Cost Distribution Logic](#cost-distribution-logic)
8. [Sample Integration Flow](#sample-integration-flow)
9. [Error Handling](#error-handling)

---

## 1. Introduction

The Vendor Equipment Management System (VEMS) is a specialized platform designed to manage vendor-supplied medical equipment and services across healthcare facilities in Kenya. This document provides comprehensive guidelines for integrating external systems (Apeiro, HMIS) with VEMS to enable seamless cost distribution calculations and claims processing.

### Purpose
This integration enables external systems to:
- Query service availability at VEMS-enabled facilities
- Submit patient service requests with facility and service details
- Receive real-time cost distribution breakdowns (Vendor Share, Facility Share, Service Fee)
- Track claims and bookings through their lifecycle

---

## 2. Integration Overview

### Integration Architecture

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

### Key Concepts

1. **VEMS-Enabled Facilities**: Healthcare facilities that have contracted with vendors to provide specialized equipment and services (e.g., CT Scans, MRI, Dialysis)

2. **Service Contracts**: Each facility has specific contracts with vendors for defined services

3. **Cost Distribution**: For each service, costs are split between:
   - **Vendor Share**: Payment to the equipment/service vendor
   - **Facility Share**: Payment to the healthcare facility
   - **Service Fee**: Additional administrative/processing fee
   - **VEMS Total**: Sum of all three components

4. **Claims Integration**: External systems submit claims that reference VEMS bookings for accurate cost tracking

---

## 3. Prerequisites

### Required Information

Before integrating with VEMS, you need:

1. **Facility Information**
   - Facility FR Code (Health Information Exchange Facility Registry Code)
   - Example: `FID-32-121583-3`

2. **Service Information**
   - Service Codes for VEMS-managed services
   - Example: `F8735` (CT Scan), `F8736` (MRI)

3. **Patient Information**
   - Patient CR Number (Client Registry Number from HIE)
   - Example: `CR1234567`

4. **Claim Information**
   - Unique Claim ID from your system
   - Example: `3663-gdhs-TAIFA-CARE`

### API Access Credentials

Contact VEMS technical team to obtain:
- Base URL: `https://api.vems.go.ke/api/v1`
- HMIS-specific endpoint: `https://api.vems.go.ke/hmis/v1`
- API Key/OAuth credentials (if required)

---

## 4. Authentication

### Authentication Method

Currently, VEMS API uses facility-based authentication where requests are validated based on the `facilityFrCode` provided.

**Future Enhancement**: OAuth 2.0 Bearer Token authentication will be implemented for enhanced security.

### Request Headers

```http
Content-Type: application/json
Accept: application/json
```

---

## 5. API Endpoints

### 5.1 Get Available Services at Facility

**Purpose**: Check which VEMS services are available at a specific facility

**Endpoint**: `GET /hmis/v1/services`

**Query Parameters**:
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `fr_code` | string | Yes | Facility FR Code |

**Request Example**:
```http
GET /hmis/v1/services?fr_code=FID-32-121583-3
```

**Response Example**:
```json
{
  "success": true,
  "data": [
    {
      "serviceCode": "F8735",
      "serviceName": "CT Scan - Contrast Enhanced",
      "vendorCode": "V364",
      "vendorName": "Medical Imaging Solutions Ltd",
      "shaRate": 15000.00,
      "vendorShare": 10000.00,
      "facilityShare": 4500.00,
      "serviceFee": 500.00,
      "isCapitated": false,
      "isActive": true
    }
  ]
}
```

---

### 5.2 Submit Cost Distribution Request

**Purpose**: Submit patient service details and receive cost distribution breakdown

**Endpoint**: `POST /hmis/v1/bookings/cost-distribution`

**Request Body**:

```json
{
  "patientCR": "CR1234567",
  "paymentMode": "sha",
  "facilityFrCode": "FID-32-121583-3",
  "claimID": "3663-gdhs-TAIFA-CARE",
  "services": [
    {
      "serviceCode": "F8735",
      "serviceDate": "2025-11-02T10:51:00.000Z"
    },
    {
      "serviceCode": "F8736",
      "serviceDate": "2025-11-03T14:30:00.000Z"
    }
  ]
}
```

**Request Field Descriptions**:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `patientCR` | string | Yes | Patient's Client Registry Number from HIE |
| `paymentMode` | string | Yes | Payment method: `sha`, `cash`, `other_insurances` |
| `facilityFrCode` | string | Yes | Healthcare facility's FR Code |
| `claimID` | string | Yes | Unique claim identifier from your system |
| `services` | array | Yes | List of services requested |
| `services[].serviceCode` | string | Yes | VEMS service code |
| `services[].serviceDate` | string (ISO 8601) | Yes | Date/time of service delivery |

**Response Example**:

```json
{
  "success": true,
  "claimID": "3663-gdhs-TAIFA-CARE",
  "bookingNo": "VB673D4F2A1B3E",
  "bookingStatus": "pending",
  "totalAmount": 31000.00,
  "services": [
    {
      "serviceCode": "F8735",
      "serviceName": "CT Scan - Contrast Enhanced",
      "serviceDate": "2025-11-02T10:51:00.000Z",
      "vendorCode": "V364",
      "vendorName": "Medical Imaging Solutions Ltd",
      "vendorShare": 10000.00,
      "facilityShare": 4500.00,
      "serviceFee": 500.00,
      "vemsTotal": 15000.00
    },
    {
      "serviceCode": "F8736",
      "serviceName": "MRI Scan - Full Body",
      "serviceDate": "2025-11-03T14:30:00.000Z",
      "vendorCode": "V364",
      "vendorName": "Medical Imaging Solutions Ltd",
      "vendorShare": 11000.00,
      "facilityShare": 4500.00,
      "serviceFee": 500.00,
      "vemsTotal": 16000.00
    }
  ]
}
```

**Response Field Descriptions**:

| Field | Type | Description |
|-------|------|-------------|
| `success` | boolean | Request success status |
| `claimID` | string | Your original claim ID (for reference tracking) |
| `bookingNo` | string | VEMS booking reference number |
| `bookingStatus` | string | Status: `pending`, `confirmed`, `completed` |
| `totalAmount` | decimal | Sum of all VEMS totals |
| `services[].vendorShare` | decimal | Amount payable to vendor |
| `services[].facilityShare` | decimal | Amount payable to facility |
| `services[].serviceFee` | decimal | Administrative/processing fee |
| `services[].vemsTotal` | decimal | Total cost for this service (vendor + facility + fee) |

---

### 5.3 Query Booking Status

**Purpose**: Check the status of a previously submitted booking

**Endpoint**: `GET /hmis/v1/bookings/{bookingNo}`

**Path Parameters**:
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `bookingNo` | string | Yes | VEMS booking reference number |

**Request Example**:
```http
GET /hmis/v1/bookings/VB673D4F2A1B3E
```

**Response Example**:
```json
{
  "bookingId": "019a7c5b-b296-712d-bb4a-64d9f666f669",
  "bookingNo": "VB673D4F2A1B3E",
  "bookingDate": "2025-11-02T10:51:00.000Z",
  "bookingStatus": "confirmed",
  "approvalStatus": "approved",
  "paymentMode": "sha",
  "override": false,
  "createdAt": "2025-11-02T08:30:00.000Z",
  "updatedAt": "2025-11-02T09:15:00.000Z",
  "patient": {
    "name": "John Kamau Mwangi",
    "phone": "+254712345678",
    "dateOfBirth": "1985-05-15",
    "gender": "Male",
    "identificationType": "National ID",
    "identificationNo": "12345678"
  },
  "bookedServices": [
    {
      "serviceBookingId": "019a7c5b-b296-712d-bb4a-64d9f666f670",
      "serviceDate": "2025-11-02T10:51:00.000Z",
      "serviceStatus": "completed",
      "vendorShare": 10000.00,
      "facilityShare": 4500.00,
      "service": {
        "name": "CT Scan - Contrast Enhanced",
        "code": "F8735",
        "shaRate": 15000.00,
        "isCapitated": false
      },
      "vendor": {
        "name": "Medical Imaging Solutions Ltd",
        "code": "V364"
      },
      "lot": {
        "number": "LOT-2024-001",
        "name": "Diagnostic Imaging Services"
      }
    }
  ]
}
```

---

## 6. Request/Response Formats

### 6.1 Date/Time Format

All date/time fields use **ISO 8601** format:
```
2025-11-02T10:51:00.000Z
```

**Timezone**: UTC (Coordinated Universal Time)

**Example Conversions**:
- EAT (East Africa Time) is UTC+3
- 10:51 EAT = 07:51 UTC
- Submit as: `2025-11-02T07:51:00.000Z`

### 6.2 Decimal Format

All monetary amounts use **decimal format** with 2 decimal places:
```
15000.00
```

### 6.3 Payment Modes

Accepted values:
- `sha` - Social Health Authority
- `cash` - Cash payment
- `mpesa` - M-Pesa mobile money
- `other_insurances` - Private insurance

---

## 7. Cost Distribution Logic

### 7.1 Cost Calculation Formula

For each service:

```
VEMS Total = Vendor Share + Facility Share + Service Fee
```

**Example**:
- Vendor Share: KES 10,000.00
- Facility Share: KES 4,500.00
- Service Fee: KES 500.00
- **VEMS Total**: KES 15,000.00

### 7.2 SHA Rate vs VEMS Total

- **SHA Rate**: The standard rate registered with Social Health Authority
- **VEMS Total**: The actual cost distributed among vendor, facility, and VEMS
- VEMS Total ≤ SHA Rate (VEMS optimizes cost distribution)

### 7.3 Cost Distribution by Stakeholder

| Stakeholder | Share | Description |
|-------------|-------|-------------|
| **Vendor** | ~65-70% | Equipment owner/service provider |
| **Facility** | ~28-32% | Healthcare facility hosting the service |
| **VEMS** | ~2-5% | Platform administration & management |

---

## 8. Sample Integration Flow

### Step-by-Step Integration Process

#### Step 1: Query Available Services

Before submitting a claim, check if the service is available at the facility:

```http
GET /hmis/v1/services?fr_code=FID-32-121583-3
```

#### Step 2: Submit Cost Distribution Request

Submit patient and service details:

```json
POST /hmis/v1/bookings/cost-distribution

{
  "patientCR": "CR1234567",
  "paymentMode": "sha",
  "facilityFrCode": "FID-32-121583-3",
  "claimID": "3663-gdhs-TAIFA-CARE",
  "services": [
    {
      "serviceCode": "F8735",
      "serviceDate": "2025-11-02T10:51:00.000Z"
    }
  ]
}
```

#### Step 3: Receive Cost Distribution

VEMS responds with detailed cost breakdown:

```json
{
  "success": true,
  "claimID": "3663-gdhs-TAIFA-CARE",
  "bookingNo": "VB673D4F2A1B3E",
  "totalAmount": 15000.00,
  "services": [
    {
      "serviceCode": "F8735",
      "vendorShare": 10000.00,
      "facilityShare": 4500.00,
      "serviceFee": 500.00,
      "vemsTotal": 15000.00
    }
  ]
}
```

#### Step 4: Store Booking Reference

Store the `bookingNo` in your system linked to your `claimID` for future reference and reconciliation.

#### Step 5: Query Status (Optional)

Check booking status at any time:

```http
GET /hmis/v1/bookings/VB673D4F2A1B3E
```

---

## 9. Error Handling

### 9.1 HTTP Status Codes

| Status Code | Meaning | Description |
|-------------|---------|-------------|
| `200` | OK | Request successful |
| `201` | Created | Resource created successfully |
| `400` | Bad Request | Invalid request format or parameters |
| `401` | Unauthorized | Authentication required |
| `404` | Not Found | Resource not found |
| `422` | Unprocessable Entity | Validation errors |
| `500` | Internal Server Error | Server-side error |

### 9.2 Error Response Format

```json
{
  "success": false,
  "message": "Service with code 'F9999' not found",
  "error": "Service not found in facility contracts"
}
```

### 9.3 Common Error Scenarios

#### Facility Not Found

```json
{
  "success": false,
  "message": "Facility not found",
  "error": "No facility found with FR code 'FID-INVALID-CODE'"
}
```

**Solution**: Verify the facility FR code is correct and facility is registered in VEMS

#### Service Not Available

```json
{
  "success": false,
  "message": "Service 'F8735' is not available in this facility's contracts",
  "error": "Service not contracted at this facility"
}
```

**Solution**: Query available services first using the services endpoint

#### Patient Not Found

```json
{
  "success": false,
  "message": "Patient not found in HIE",
  "error": "No patient record found with CR number 'CR9999999'"
}
```

**Solution**: Verify patient CR number is correct and patient is registered in HIE

#### Invalid Date Format

```json
{
  "success": false,
  "errors": {
    "services.0.serviceDate": [
      "The services.0.serviceDate must be a valid date."
    ]
  }
}
```

**Solution**: Use ISO 8601 format: `2025-11-02T10:51:00.000Z`

---

## Appendix A: Sample Integration Code

### PHP Example

```php
<?php

class VemsIntegration
{
    private $baseUrl = 'https://api.vems.go.ke/hmis/v1';
    
    public function getCostDistribution($claimId, $patientCR, $facilityFrCode, $services)
    {
        $payload = [
            'claimID' => $claimId,
            'patientCR' => $patientCR,
            'paymentMode' => 'sha',
            'facilityFrCode' => $facilityFrCode,
            'services' => $services
        ];
        
        $ch = curl_init($this->baseUrl . '/bookings/cost-distribution');
        curl_setopt($ch, CURLOPT_POST, true);
        curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($payload));
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
        curl_setopt($ch, CURLOPT_HTTPHEADER, [
            'Content-Type: application/json',
            'Accept: application/json'
        ]);
        
        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);
        
        if ($httpCode === 200) {
            return json_decode($response, true);
        }
        
        throw new Exception("VEMS API Error: " . $response);
    }
}

// Usage
$vems = new VemsIntegration();
$result = $vems->getCostDistribution(
    '3663-gdhs-TAIFA-CARE',
    'CR1234567',
    'FID-32-121583-3',
    [
        [
            'serviceCode' => 'F8735',
            'serviceDate' => '2025-11-02T10:51:00.000Z'
        ]
    ]
);

echo "Total Amount: KES " . $result['totalAmount'];
```

### Python Example

```python
import requests
import json
from datetime import datetime

class VemsIntegration:
    def __init__(self):
        self.base_url = 'https://api.vems.go.ke/hmis/v1'
    
    def get_cost_distribution(self, claim_id, patient_cr, facility_fr_code, services):
        payload = {
            'claimID': claim_id,
            'patientCR': patient_cr,
            'paymentMode': 'sha',
            'facilityFrCode': facility_fr_code,
            'services': services
        }
        
        headers = {
            'Content-Type': 'application/json',
            'Accept': 'application/json'
        }
        
        response = requests.post(
            f'{self.base_url}/bookings/cost-distribution',
            json=payload,
            headers=headers
        )
        
        if response.status_code == 200:
            return response.json()
        
        raise Exception(f"VEMS API Error: {response.text}")

# Usage
vems = VemsIntegration()
result = vems.get_cost_distribution(
    '3663-gdhs-TAIFA-CARE',
    'CR1234567',
    'FID-32-121583-3',
    [
        {
            'serviceCode': 'F8735',
            'serviceDate': datetime.utcnow().isoformat() + 'Z'
        }
    ]
)

print(f"Total Amount: KES {result['totalAmount']}")
```

---

## Appendix B: Glossary

| Term | Definition |
|------|------------|
| **VEMS** | Vendor Equipment Management System - Platform for managing vendor-supplied medical equipment |
| **HIE** | Health Information Exchange - National health data exchange platform in Kenya |
| **FR Code** | Facility Registry Code - Unique identifier for healthcare facilities in HIE |
| **CR Number** | Client Registry Number - Unique patient identifier in HIE |
| **SHA** | Social Health Authority - Kenya's national health insurance authority |
| **Vendor Share** | Portion of service cost paid to equipment/service vendor |
| **Facility Share** | Portion of service cost paid to healthcare facility |
| **Service Fee** | VEMS platform administration fee |
| **VEMS Total** | Total service cost (vendor + facility + service fee) |
| **Capitated** | Pre-paid services under insurance coverage |
| **Claim ID** | Unique identifier for insurance/payment claims |
| **Booking Number** | VEMS reference number for service bookings |

---

## Document Control

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | November 17, 2025 | VEMS Technical Team | Initial release |

---

**End of Document**

For questions or clarifications, please contact: emekubo@datasystems.co.ke | manzolo@datasystems.co.ke | chepkwony@datasystems.co.ke | cc: woki@datasystems.co.ke
