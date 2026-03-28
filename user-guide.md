# LionCRM — User Guide

**Version:** 1.0 | **Audience:** Sales reps, managers, and admins

This guide walks you through every major workflow in LionCRM. If you can't find what you need, contact your admin.

---

## Table of Contents

1. [Getting Started](#getting-started)
2. [Roles and Permissions](#roles-and-permissions)
3. [Navigation](#navigation)
4. [Dashboard](#dashboard)
5. [Contacts](#contacts)
6. [Deal Pipeline](#deal-pipeline)
7. [Activities](#activities)
8. [Settings](#settings)

---

## Getting Started

### Logging In

1. Open LionCRM in your browser.
2. Enter your email address and password.
3. Click **Sign in**.

You are redirected to the Dashboard after a successful login.

> **Demo mode:** If a demo accounts panel appears at the bottom of the login form, click any listed email to pre-fill the form, then click Sign in.

### Logging Out

Click **Sign out** at the bottom of the left sidebar. This clears your session immediately.

### Forgotten Password

Contact your admin to reset your password.

---

## Roles and Permissions

Your role controls what you can see and do. If a button doesn't appear or an action is blocked, your role doesn't permit it.

| Action | Viewer | Sales Rep | Manager | Admin |
|--------|--------|-----------|---------|-------|
| View contacts, deals, activities | ✓ | ✓ | ✓ | ✓ |
| Create and edit contacts | — | ✓ | ✓ | ✓ |
| Create and edit deals | — | ✓ | ✓ | ✓ |
| Delete contacts and deals | — | — | ✓ | ✓ |
| Manage pipeline stages | — | — | — | ✓ |
| Manage team members (Settings) | — | — | ✓ | ✓ |
| Create and deactivate users | — | — | — | ✓ |
| View reports and dashboard | ✓ | ✓ | ✓ | ✓ |

**Viewer** accounts see a yellow "You have view-only access" banner on pages where editing is available to other roles.

---

## Navigation

The left sidebar is your main navigation. It contains:

| Item | What it does |
|------|-------------|
| **LionCRM** (logo) | Always visible at the top |
| **Dashboard** | KPI summary, charts, recent activity |
| **Contacts** | All people in your CRM |
| **Pipeline** | Kanban deal board |
| **Activities** | Log of calls, emails, meetings, notes, and tasks |
| **Settings** | Your profile and team management |
| **Sign out** | End your session |

Your name and role badge are shown at the very bottom of the sidebar.

---

## Dashboard

The Dashboard gives you a real-time snapshot of your pipeline.

### KPI Cards

| Card | What it shows |
|------|--------------|
| **Total Contacts** | Count of all contacts in the CRM |
| **Pipeline Value** | Sum of all open deal values |
| **Closed This Month** | Revenue closed in the current calendar month |
| **Win Rate** | Percentage of deals won vs. total closed |

Each card shows a trend comparison against the previous period.

### Charts

- **Pipeline by Stage** — bar chart showing deal value distributed across stages
- **Revenue by Month** — monthly revenue trend line

### Recent Activity

A feed of the 5 most recent activities logged across the team.

---

## Contacts

Contacts are the people you track in the CRM — leads, prospects, customers, or churned accounts.

### Viewing Contacts

The Contacts page shows all contacts in a card grid. Each card displays the contact's name, title, company, tags, status badge, email, and phone.

**Search and filter:**

- **Search bar** — type to filter by name, email, company, or tag in real time
- **Segment dropdown** — filter to contacts in a specific segment
- Result count updates as you type

### Adding a Contact

Requires **Sales Rep** role or above.

1. Click **New contact** (top-right of Contacts page).
2. A form modal appears. Fill in:
   - **First name** and **Last name** (required)
   - **Email** (optional)
   - **Phone** (optional)
   - **Job title** (optional)
   - **Company** (optional — links to a company record)
   - **Tags** (optional — comma-separated labels, e.g. `vip`, `enterprise`)
   - **Segments** (optional — audience segments for filtering)
   - **Source** (optional — e.g. `website`, `referral`, `cold_outreach`)
   - **Status** — `Lead` (default), `Prospect`, `Customer`, or `Churned`
   - **Notes** (optional)
3. Click **Save**.

### Editing a Contact

Click a contact card to open the Contact Detail page, then click **Edit**. Or click the edit icon directly on the card. All fields are editable.

### Contact Detail Page

Click any contact card to open its detail view. The detail page shows:

- **Profile header** — full name, title and company, tags, segment badge
- **Contact info** — email (clickable mailto link), phone, company, date added
- **Deals** — a list of all deals associated with this contact, with stage and value
- **Activity History** — all activities (calls, emails, meetings, notes, tasks) logged against this contact

### Deleting a Contact

Requires **Manager** role or above. On a contact card or detail page, click the delete icon and confirm.

---

## Deal Pipeline

The Pipeline page shows your deals as a Kanban board organized by stage.

### Reading the Board

- Each **column** is a pipeline stage (Prospecting → Qualification → Proposal → Negotiation → Closed Won → Closed Lost)
- Each **card** shows the deal title, value, expected close date, and probability
- The header shows total open deals and total pipeline value

### Moving a Deal Between Stages

Requires **Sales Rep** role or above. Drag a deal card from one column and drop it into another. The deal's stage updates immediately.

> **Viewer accounts** see a yellow banner: "You have view-only access. Contact an admin to edit deals." Dragging is disabled.

### Adding a Deal

Requires **Sales Rep** role or above.

1. Click **New deal** (top-right of Pipeline page).
2. Fill in:
   - **Title** (required)
   - **Stage** (required — choose which pipeline stage to place the deal in)
   - **Contact** (optional — link to an existing contact)
   - **Company** (optional)
   - **Assignee** (optional — defaults to you)
   - **Value** (optional — deal size in currency)
   - **Currency** (optional — default USD)
   - **Expected close date** (optional)
   - **Description** (optional)
   - **Tags** (optional)
3. Click **Save**.

### Closing a Deal

Move the deal card to **Closed Won** or **Closed Lost** using drag-and-drop. The deal's `closedAt` timestamp is set automatically.

---

## Activities

Activities are the logged touchpoints with contacts and companies — calls you made, emails you sent, meetings you held, notes you wrote, and tasks you set.

### Viewing Activities

The Activities page shows all activities in a chronological feed (newest first).

**Filter by type** using the pill buttons at the top:
- **All** — everything
- **Calls** — logged phone calls
- **Emails** — sent/received emails
- **Meetings** — meetings and demos
- **Notes** — free-text notes
- **Tasks** — action items and follow-ups

Each activity item shows: type icon, subject, linked contact/deal, date and duration, and any logged outcome.

### Logging an Activity

Activities can be logged from multiple places:

- From the **Activities page**: click **New activity**
- From a **Contact detail page**: use the Activity History section

Fill in:
- **Type** (required) — call, email, meeting, note, or task
- **Subject** (required) — brief description
- **Linked to** (at least one required) — contact, deal, or company
- **Body** (optional) — full notes or email body
- **Outcome** (optional) — result of the interaction
- **Assigned to** (optional) — defaults to you
- **Scheduled at / Completed at** (optional) — for scheduled or past activities
- **Duration** (optional) — in minutes (useful for calls and meetings)

---

## Settings

Click **Settings** in the sidebar to access your account and workspace settings.

### Your Profile

Displays your name, email address, and role badge. Use this to verify you're logged in as the correct user.

### Team Members

Visible to **Managers and Admins**. Shows all active team members with their name, email, and role. Admins can manage roles and create/deactivate users from the backend admin interface.

### API Integration

If you're building integrations, the CRM exposes a full REST API at `/api/v1`. See the [API Documentation](/LIOAA/issues/LIOAA-19#document-api-docs) for the complete reference.

---

## Quick Reference

### Contact Status Values

| Status | Meaning |
|--------|---------|
| Lead | Initial contact, not yet qualified |
| Prospect | Qualified interest, actively engaged |
| Customer | Active or past customer |
| Churned | Was a customer, no longer active |

### Activity Types

| Type | Use when… |
|------|----------|
| Call | You made or received a phone call |
| Email | You sent or received an email |
| Meeting | You met in person or via video |
| Note | You want to record information about a contact or deal |
| Task | You want to track a follow-up action item |

### Keyboard Shortcuts

| Shortcut | Action |
|----------|--------|
| `?` | Open help |
| `Esc` | Close modal |

