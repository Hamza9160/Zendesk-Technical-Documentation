# Concierge Feature - Technical Documentation

> **Version:** 1.0  
> **Last Updated:** March 2026  
> **Platform:** Android (Kotlin + Jetpack Compose)

---

## Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Screen Flow Diagram](#screen-flow-diagram)
4. [Submit Button - Complete API Flow](#submit-button---complete-api-flow)
5. [Screens & Components](#screens--components)
6. [API Endpoints](#api-endpoints)
7. [Error Handling](#error-handling)

---

## Overview

The **Concierge** feature enables prescribers to submit requests directly from the mobile app. It integrates with **Zendesk Sunshine Conversations** for ticket creation and real-time messaging.

### Key Capabilities

| Feature | Description |
|---------|-------------|
| Request Free Samples | Request pharmaceutical samples |
| Schedule Rep Visit | Request a visit from a pharmaceutical representative |
| Custom Request | Submit any custom request (clinical trials, medication savings, etc.) |
| Request Tracking | View active and archived requests |
| Real-time Messaging | Continue conversations via Zendesk messaging |
| Contact Preferences | Set preferred contact hours |

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              MVVM ARCHITECTURE                               │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│      VIEW       │     │   VIEWMODEL     │     │     MODEL       │
│   (Composable)  │────▶│  (StateFlow)    │────▶│   (Repository)  │
└─────────────────┘     └─────────────────┘     └─────────────────┘
        │                       │                       │
        │   observes state      │   manages state       │   API calls
        │◀──────────────────────│◀──────────────────────│
        │                       │                       │
┌───────┴───────┐       ┌───────┴───────┐       ┌───────┴───────┐
│ Concierge     │       │ Concierge     │       │  HTTP Client  │
│ Screens       │       │ ViewModels    │       │  (Cronet)     │
└───────────────┘       └───────────────┘       └───────────────┘
```

### Component Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           CONCIERGE MODULE                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                            UI LAYER                                   │  │
│  │                                                                       │  │
│  │   ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐     │  │
│  │   │ Concierge       │  │ Request List    │  │ Archived        │     │  │
│  │   │ Screen          │  │ Screen          │  │ Screen          │     │  │
│  │   │                 │  │                 │  │                 │     │  │
│  │   │ • Request Types │  │ • Active List   │  │ • Solved List   │     │  │
│  │   │ • Text Input    │  │ • Tap to open   │  │ • Closed List   │     │  │
│  │   │ • Submit Button │  │ • Refresh       │  │ • View History  │     │  │
│  │   └────────┬────────┘  └────────┬────────┘  └────────┬────────┘     │  │
│  │            │                    │                    │               │  │
│  │   ┌────────┴────────┐  ┌────────┴────────────────────┴────────┐     │  │
│  │   │ Settings Screen │  │         Shared UI Components          │     │  │
│  │   │ • Preferences   │  │  • RequestCard • RadioButton • Dialogs│     │  │
│  │   └─────────────────┘  └──────────────────────────────────────┘     │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                      │                                      │
│                                      ▼                                      │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                         VIEWMODEL LAYER                               │  │
│  │                                                                       │  │
│  │   ┌─────────────────────────┐   ┌─────────────────────────┐         │  │
│  │   │   ConciergeViewModel    │   │  RequestsListViewModel  │         │  │
│  │   │                         │   │                         │         │  │
│  │   │ • Form validation       │   │ • Fetch active requests │         │  │
│  │   │ • API state machine     │   │ • Fetch archived reqs   │         │  │
│  │   │ • Submission flow       │   │ • Search by user email  │         │  │
│  │   └─────────────────────────┘   └─────────────────────────┘         │  │
│  │                                                                       │  │
│  │   ┌─────────────────────────────────────────────────────────────┐   │  │
│  │   │                     SharedViewModel                          │   │  │
│  │   │  • User Profile • JWT Token • Conversation ID • Test NPI    │   │  │
│  │   └─────────────────────────────────────────────────────────────┘   │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                      │                                      │
│                                      ▼                                      │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                          DATA LAYER                                   │  │
│  │                                                                       │  │
│  │   ┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐   │  │
│  │   │   HTTP Client   │   │  Zendesk SDK    │   │ Local Storage   │   │  │
│  │   │   (Cronet)      │   │  (Messaging)    │   │ (SharedPrefs)   │   │  │
│  │   └─────────────────┘   └─────────────────┘   └─────────────────┘   │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Screen Flow Diagram

```
                                    ┌─────────────────┐
                                    │   Home Page     │
                                    │   (Concierge    │
                                    │    Tab)         │
                                    └────────┬────────┘
                                             │
                                             ▼
                              ┌──────────────────────────┐
                              │    Concierge Screen      │
                              │                          │
                              │  ┌────────────────────┐  │
                              │  │ Request Samples ○  │  │
                              │  │ Schedule Rep    ○  │  │
                              │  │ Custom Request  ○  │  │
                              │  └────────────────────┘  │
                              │                          │
                              │  [Message Icon] [Submit] │
                              └──────────┬───────────────┘
                                         │
                    ┌────────────────────┼────────────────────┐
                    │                    │                    │
                    ▼                    ▼                    ▼
         ┌──────────────────┐  ┌─────────────────┐  ┌─────────────────┐
         │  Zendesk         │  │ Requests List   │  │ Settings Menu   │
         │  Messaging       │  │ Screen          │  │                 │
         │  (after submit)  │  │                 │  │ • Preferences   │
         └──────────────────┘  └────────┬────────┘  │ • Archived      │
                                        │           └────────┬────────┘
                          ┌─────────────┴─────────────┐      │
                          │                           │      │
                          ▼                           ▼      ▼
               ┌─────────────────┐           ┌─────────────────┐
               │ Tap Request     │           │ Contact         │
               │ Card            │           │ Preferences     │
               │                 │           │                 │
               │ Opens Zendesk   │           │ • Hours to Call │
               │ Messaging       │           │ • Do Not Contact│
               └─────────────────┘           └─────────────────┘
                                                     │
                                                     ▼
                                            ┌─────────────────┐
                                            │ Archived        │
                                            │ Requests        │
                                            │                 │
                                            │ View completed  │
                                            │ conversations   │
                                            └─────────────────┘
```

---

## Submit Button - Complete API Flow

When user clicks the **Submit** button, the following sequence executes:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    SUBMIT BUTTON - COMPLETE API FLOW                         │
└─────────────────────────────────────────────────────────────────────────────┘


    USER CLICKS SUBMIT
           │
           ▼
    ┌──────────────┐
    │  VALIDATION  │
    │              │
    │ 1. Request   │
    │    type      │
    │    selected? │
    │              │
    │ 2. Message   │
    │    not empty?│
    │              │
    │ 3. NPI       │
    │    present?  │
    │              │
    │ 4. Phone     │
    │    present?  │
    └──────┬───────┘
           │
           │ All Valid
           ▼
    ┌──────────────┐
    │  SHOW LOADER │
    │  (Immediate  │
    │   feedback)  │
    └──────┬───────┘
           │
           ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                        API 1: ZENDESK SDK LOGIN                              │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Purpose: Authenticate user with Zendesk SDK using JWT                       │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  Zendesk.instance.loginUser(jwt = jwtToken)                            │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
│  On Success → Proceed to API 2                                               │
│  On Failure → Show "Couldn't login" error dialog                            │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
           │
           │ Success
           ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                     API 2: CREATE CONVERSATION                               │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Endpoint: POST {BASE_URL}/sc/v2/apps/{appId}/conversations                  │
│  Auth: Basic Auth (appKey:appSecret)                                         │
│                                                                              │
│  Purpose: Create a new Sunshine Conversation with metadata                   │
│                                                                              │
│  Request includes:                                                           │
│  • type: "personal"                                                          │
│  • displayName: subject (e.g., "Request Free Samples")                       │
│  • metadata: tags, NPI, email, name, source                                  │
│  • participants: user's external ID (NPI)                                    │
│                                                                              │
│  Response returns: conversation.id                                           │
│                                                                              │
│  On Success → Store conversationId, proceed to API 3                         │
│  On Failure → Show error dialog                                              │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
           │
           │ Success (got conversationId)
           ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                       API 3: SEND MESSAGE                                    │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Endpoint: POST {BASE_URL}/sc/v2/apps/{appId}/conversations/{convId}/messages│
│  Auth: Basic Auth (appKey:appSecret)                                         │
│                                                                              │
│  Purpose: Post the user's request message to the conversation                │
│                                                                              │
│  Request includes:                                                           │
│  • author: { type: "user", userExternalId: NPI }                             │
│  • content: { type: "text", text: user's message }                           │
│                                                                              │
│  On Success → Proceed to API 4                                               │
│  On Failure → Show "Message sending failed" error dialog                     │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
           │
           │ Success (message sent)
           ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                      API 4: FETCH TICKET (POLLING)                           │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Endpoint: GET {BASE_URL}/api/v2/search.json?query=type:ticket tags:{tag}    │
│  Auth: Basic Auth (agentEmail/token:apiToken)                                │
│                                                                              │
│  Purpose: Find the ticket that Zendesk auto-created from our conversation    │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │                         POLLING LOGIC                                  │ │
│  │                                                                        │ │
│  │   ┌─────────┐     No ticket      ┌─────────┐                          │ │
│  │   │  POLL   │────────found──────▶│  WAIT   │                          │ │
│  │   │   API   │                    │  500ms  │                          │ │
│  │   └────┬────┘                    └────┬────┘                          │ │
│  │        │                              │                                │ │
│  │        │◀─────────────────────────────┘                                │ │
│  │        │                                                               │ │
│  │        │ Ticket found                                                  │ │
│  │        ▼                                                               │ │
│  │   ┌─────────┐                                                         │ │
│  │   │ SUCCESS │                                                         │ │
│  │   └─────────┘                                                         │ │
│  │                                                                        │ │
│  │   Timeout: 30 seconds (if ticket not found → show error)              │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
│  Response returns: results[0].id (ticketId)                                  │
│                                                                              │
│  On Success → Store ticketId, proceed to API 5                               │
│  On Timeout → Show "Ticket creation timed out" error dialog                  │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
           │
           │ Success (got ticketId)
           ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                      API 5: SET REQUESTER                                    │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Endpoint: PUT {BASE_URL}/api/v2/tickets/{ticketId}.json                     │
│  Auth: Basic Auth (agentEmail/token:apiToken)                                │
│                                                                              │
│  Purpose: Update ticket with prescriber's contact information                │
│                                                                              │
│  Request includes:                                                           │
│  • ticket.subject: request type                                              │
│  • ticket.requester: { name, external_id (NPI), phone, email }               │
│  • ticket.custom_fields: subject, NPI, conversationId                        │
│                                                                              │
│  On Success → Hide loader, open Zendesk Messaging UI                         │
│  On Failure → Show "Failed to set requester" error dialog                    │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
           │
           │ Success
           ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                     OPEN ZENDESK MESSAGING                                   │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Action: Open Zendesk Messaging SDK with the conversationId                  │
│                                                                              │
│  User can now:                                                               │
│  • See their submitted request                                               │
│  • Continue the conversation                                                 │
│  • Receive replies from support team                                         │
│  • Get push notifications for new messages                                   │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

### API Flow Summary Table

| Step | API | Method | Purpose | Depends On |
|------|-----|--------|---------|------------|
| 1 | Zendesk SDK Login | SDK Call | Authenticate user | Valid JWT |
| 2 | Create Conversation | POST | Create new conversation | Step 1 success |
| 3 | Send Message | POST | Post user's request | Step 2 success (conversationId) |
| 4 | Fetch Ticket | GET (polling) | Find auto-created ticket | Step 3 success |
| 5 | Set Requester | PUT | Add prescriber info to ticket | Step 4 success (ticketId) |

### State Machine Transitions

```
   ┌──────────────────────────────────────────────────────────────┐
   │                                                              │
   │    None ──▶ CreateConversation ──▶ SendMessage ──▶          │
   │                                                              │
   │         ──▶ FetchTicket ──▶ SetRequester ──▶ None           │
   │                                                              │
   │    (Each step sets the trigger for the next step)           │
   │                                                              │
   └──────────────────────────────────────────────────────────────┘
```

---

## Screens & Components

### 1. Concierge Screen

**Purpose:** Main request submission interface

**Features:**
- Three mutually exclusive request types (radio-button behavior)
- Character-limited text input (200 chars max)
- Validation before submission

### 2. Request List Screen

**Purpose:** Display active (open) requests

**Features:**
- Auto-refresh on screen resume
- Tap card to open Zendesk messaging
- Dropdown menu for settings access
- Loading state management

### 3. Archived Request Screen

**Purpose:** Display completed/closed requests

**Features:**
- Shows solved and closed tickets
- View-only conversation history
- Loading and error states

### 4. Settings Screen

**Purpose:** Navigation hub for settings

**Features:**
- Link to Contact Preferences
- Link to Archived Requests

### 5. Contact Preferences Screen

**Purpose:** Set contact hour preferences

**Features:**
- Day-by-day time range selection
- "Do Not Contact" toggle per day
- Save to Zendesk user profile

---

## API Endpoints

### Base URL

```
{BASE_URL} = Your Zendesk instance URL (e.g., https://yourcompany.zendesk.com)
```

### 1. Create Conversation (Zendesk Sunshine)
```
POST {BASE_URL}/sc/v2/apps/{appId}/conversations
Authorization: Basic {appKey}:{appSecret}
Content-Type: application/json

Request Body:
{
  "type": "personal",
  "displayName": "{subject}",
  "metadata": {
    "zen:ticket:tags": "mobile,android,concierge,prescriber,from_mobile,{uniqueTag}",
    "provider_npi": "{npi}",
    "provider_email": "{email}",
    "provider_name": "{name}",
    "source": "android_app",
    "client_conversation_tag": "{uniqueTag}"
  },
  "participants": [
    { "userExternalId": "{npi}" }
  ]
}

Response:
{
  "conversation": {
    "id": "conversation_id_here"
  }
}
```

### 2. Send Message (Zendesk Sunshine)
```
POST {BASE_URL}/sc/v2/apps/{appId}/conversations/{conversationId}/messages
Authorization: Basic {appKey}:{appSecret}
Content-Type: application/json

Request Body:
{
  "author": {
    "type": "user",
    "userExternalId": "{npi}"
  },
  "content": {
    "type": "text",
    "text": "{message}"
  }
}

Response:
{
  "message": {
    "id": "message_id_here"
  }
}
```

### 3. Search Ticket by Tag
```
GET {BASE_URL}/api/v2/search.json?query=type:ticket tags:{uniqueTag}
Authorization: Basic {agentEmail}/token:{apiToken}

Response:
{
  "results": [
    { 
      "id": 12345, 
      "subject": "Request Free Samples",
      "status": "new"
    }
  ]
}
```

### 4. Update Ticket (Set Requester)
```
PUT {BASE_URL}/api/v2/tickets/{ticketId}.json
Authorization: Basic {agentEmail}/token:{apiToken}
Content-Type: application/json

Request Body:
{
  "ticket": {
    "subject": "{subject}",
    "requester": {
      "name": "{name}",
      "external_id": "{npi}",
      "phone": "{phone}",
      "email": "{email}"
    },
    "custom_fields": [
      { "id": {customFieldId1}, "value": "{subject}" },
      { "id": {customFieldId2}, "value": "{npi}" },
      { "id": {customFieldId3}, "value": "{conversationId}" }
    ]
  }
}
```

### 5. Search User by Email (for Requests List)
```
GET {BASE_URL}/api/v2/users/search.json?query=email:{userEmail}
Authorization: Basic {agentEmail}/token:{apiToken}

Response:
{
  "users": [
    { "id": 12345, "email": "user@example.com" }
  ]
}
```

### 6. Get User's Tickets
```
GET {BASE_URL}/api/v2/users/{userId}/tickets/requested.json
Authorization: Basic {agentEmail}/token:{apiToken}

Response:
{
  "tickets": [
    { "id": 123, "subject": "...", "status": "open" },
    { "id": 124, "subject": "...", "status": "solved" }
  ]
}
```

### 7. Update User Contact Preferences
```
PUT {BASE_URL}/api/v2/users/{userId}.json
Authorization: Basic {agentEmail}/token:{apiToken}
Content-Type: application/json

Request Body:
{
  "user": {
    "user_fields": {
      "monday_contact_hours": "9:00 AM - 5:00 PM",
      "tuesday_contact_hours": "No",
      "wednesday_contact_hours": "10:00 AM - 6:00 PM"
    }
  }
}
```

---

## Error Handling

### Validation Errors

| Condition | Error Message | UI Action |
|-----------|---------------|-----------|
| No request type selected | "Please enter a request." | Show error dialog |
| Empty message content | "Please enter a request." | Show error dialog, focus input |
| Missing NPI | "Your NPI number is missing..." | Show hyperlink dialog (edit profile) |
| Missing phone | "Your phone number is missing..." | Show hyperlink dialog |
| Missing both | "Your phone number and NPI..." | Show hyperlink dialog |

### API Errors

| HTTP Code | Meaning | Handling |
|-----------|---------|----------|
| -1 | Network failure | "Network error" dialog |
| 11 | Connectivity issue | "Internet connectivity or server issue" |
| 200-299 | Success | Proceed to next step |
| 400-499 | Client error | "Something went wrong" |
| 500+ | Server error | "Something went wrong" |
| Timeout | Ticket not found in 30s | "Ticket creation timed out" |

### Error Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                      ERROR HANDLING FLOW                         │
└─────────────────────────────────────────────────────────────────┘

   Any API Call
        │
        ├──── Success ────▶ Continue to next step
        │
        └──── Failure ────▶ ┌─────────────────┐
                           │  Hide Loader     │
                           │  Show Error      │
                           │  Dialog          │
                           │                  │
                           │  User taps OK    │
                           │       │          │
                           │       ▼          │
                           │  Return to form  │
                           │  (data preserved)│
                           └─────────────────┘
```

---

## Security Considerations

1. **JWT Authentication:** Zendesk SDK login uses JWT tokens with 6-hour expiry
2. **Basic Auth:** Sunshine Conversations API uses app key/secret (Base64 encoded)
3. **Token Storage:** Access tokens should be stored securely (encrypted storage recommended)
4. **NPI as External ID:** User's NPI is used as the external identifier in Zendesk

---

*Document generated for Prescriber Android App - Concierge Module*
