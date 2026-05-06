# MatchMaker Platform Technical Overview

This platform is an umbrella of three distinct products sharing a single backend, authentication system, and billing infrastructure:

| Service | Type | Description |
|---|---|---|
| **MatchMaker** | Web App | AI-powered residency application builder |
| **MedWrite Academy** | Web App | Medical manuscript writing course (7-module) |
| **ScrubHub** | Mobile App | Medical exam prep app (separate codebase, billing integrated here) |

**Stack:** Node.js/Express backend · React + Vite frontend · MongoDB · AWS S3 · OpenAI · Stripe · RevenueCat · Twilio · Nodemailer

---

## Table of Contents

1. [MatchMaker](#1-matchmaker)
2. [MedWrite Academy](#2-medwrite-academy)
3. [ScrubHub](#3-scrubhub)
4. [Payments & Subscriptions](#4-payments--subscriptions)

---

## 1. MatchMaker

MatchMaker is a web application that guides medical graduates through building their residency application. It collects their background, runs AI-assisted personal statement generation, and helps them discover and match with residency programs.

### 1.1 What It Covers

- User registration and account management
- Personal statement generation (AI-assisted, multi-step)
- Experience management (with CV parsing)
- Research publications management (with CV parsing)
- Miscellaneous application questions (education, honors, professionalism)
- Program preferences and matching
- Application progress dashboard with PDF export

### 1.2 Authentication & Account

**Registration** is a multi-step form (5 steps):

1. Personal info: first name, last name, gender, email, phone, password
2. Address: country, state, city, zip, street
3. Medical info: nationality, work authorization, specialty, preferred locations
4. File uploads: CV (PDF/DOC), profile image
5. Service selection: which products the user wants access to

On submission, the backend:
- Validates inputs
- Hashes the password (bcrypt)
- Uploads files to AWS S3 (profile-images/ and resumes/ folders)
- Creates the User document in MongoDB
- Returns a JWT token

The frontend stores the token in localStorage and the AuthContext manages the global user state for the session.

**Login** is email + password. The backend validates credentials and returns a fresh JWT.

**Email OTP Verification:**

After login, users are prompted to verify their email. The frontend calls `POST /api/auth/send-otp`, which generates a 6-digit OTP, stores its hashed value on the User document with an expiry, and emails it via Nodemailer. The user enters the code and `POST /api/auth/verify-otp` validates it and sets `isEmailVerified: true`.

**Profile Management:**

Users can update name, bio, location, specialty, social links, and upload a new profile image or CV at any time via `PUT /api/auth/profile`.

---

### 1.3 Dashboard

The dashboard is the main hub of the MatchMaker application. It shows the user's profile and tracks completion across five application sections.

**Sections tracked:**
1. Personal Statement
2. Research Products
3. Experiences
4. Miscellaneous Questions
5. Program Preferences

Each section shows a status (Not Started / In Progress / Completed) with navigation buttons to jump directly to that section. An overall progress percentage is computed from how many sections are complete.

**PDF Export:** A "Download PDF" button calls `GET /api/dashboard/download-pdf`, which generates a formatted PDF of all the user's application data (profile, experiences, research, statement, etc.) using pdfkit and returns it as a file download.

**Readiness Check:** `GET /api/dashboard/check-readiness` evaluates whether all required sections are sufficiently filled in before the user proceeds to program matching.

---

### 1.4 Personal Statement Builder

The personal statement builder is a guided, multi-step workflow that produces a 700-800 word personal statement using GPT-4.

**Steps:**

1. **Specialty Selection:** User picks their target residency specialty/specialties from a curated list
2. **Motivation:** User explains in their own words why they chose this specialty
3. **Characteristics:** User selects exactly 3 personal characteristics (e.g., Compassion, Leadership, Resilience)
4. **Experiences per Characteristic:** For each of the 3 characteristics, the user describes a specific experience that demonstrates it
5. **Thesis Generation:** Frontend calls `POST /api/openai/thesis-statements`; GPT-4 returns 5 distinct thesis statement options based on the user's inputs
6. **Thesis Selection:** User picks one (or edits) and saves via `PUT /api/personal-statement/select-thesis`
7. **Final Statement Generation:** Frontend calls `POST /api/openai/personal-statement`; GPT-4 generates the full 700-800 word statement using the selected thesis and all prior inputs
8. **Edit:** User reviews the draft and can edit it inline
9. **Preview & Save:** User previews the final result and saves via `PUT /api/personal-statement/save-final`
10. **Completion:** `POST /api/personal-statement/complete` marks the section complete on the dashboard

**PDF Download:** At any point after the statement is generated, `GET /api/personal-statement/download-pdf` returns a formatted PDF.

**State persistence:** All intermediate state (specialties, motivation, characteristics, selected thesis, final statement) is saved to the `PersonalStatement` MongoDB document, so users can leave and return without losing progress.

---

### 1.5 Experience Manager

The Experience Manager lets users build their experience list either manually or by parsing their CV.

**CV Parsing Flow:**

1. User uploads their CV (PDF, DOC, DOCX, TXT, RTF; max 10MB)
2. Frontend calls `POST /api/experiences/parse-cv` with the file
3. Backend uses GPT-3.5-turbo to extract structured experience records from the CV text
4. The parsed experiences are returned as a JSON array
5. Frontend displays them in a paginated, editable review screen

**Manual Entry:** Users can also add experiences one at a time via `POST /api/experiences`.

**Experience fields:** Organization, position title, experience type, start/end date, current status, country, state, participation frequency, setting (Hospital, Clinic, Lab, etc.), focus area, description (max 750 chars).

**Bulk Save:** After reviewing parsed experiences, `POST /api/experiences/bulk` saves all confirmed entries at once.

**Most Meaningful Experiences:**

Users can mark up to 3 experiences as "most meaningful." Each selected experience gets an additional expanded description field (max 300 chars). These expanded descriptions feed into the personal statement AI generation. Saved via `PUT /api/experiences/:id/most-meaningful`.

---

### 1.6 Research Publications Manager

The Research Publications Manager handles the user's academic output.

**CV Parsing Flow:**

1. User uploads their CV
2. Frontend calls `POST /api/research/parse-cv`
3. Backend sends CV text to GPT-3.5-turbo, which extracts publication records (title, type, status, authors, journal, volume, issue, pages, PMID, date)
4. For any record with a PubMed ID (PMID), the backend calls the PubMed API to enrich the citation with verified data
5. Parsed records are returned for user review

**Manual Entry:** `POST /api/research/product` for individual entries.

**Research product types:** Peer-reviewed publication, non-peer-reviewed publication, poster presentation, oral presentation.

**Statuses:** Published, submitted, accepted.

After review, `POST /api/research/save-products` saves all products. `POST /api/research/complete-section` marks the research section as complete on the dashboard.

---

### 1.7 Miscellaneous Application Questions

This section collects supplementary application data across four areas:

- **Professionalism:** Has the user had any professionalism issues? (yes/no with explanation)
- **Education:** Undergraduate and graduate institutions with start/end dates and field of study
- **Honors & Awards:** Title, date, description (full CRUD: add, edit, delete)
- **Impactful Experience & Hobbies:** Free-text fields

Data is saved to the `MiscellaneousQuestion` document in MongoDB and reflected on the dashboard as a completed section.

---

### 1.8 Program Preferences

**Frontend (implemented):**

Users fill out a preferences form at `/programs`:

- Whether they are only searching in their chosen specialty (yes/no)
- Other specialties they are applying to (free text)
- Preferred US states (multi-select dropdown, all 50 states)
- Hospital type preference (Academic Centers vs. Community-Level Hospitals)
- Resident count preference (Many vs. Fewer)
- Three valued program characteristics (checkboxes, max 3 from a curated list)

The form loads any previously saved preferences on mount, validates required fields, and on submit calls `POST /api/programs/preferences`. The user is redirected to the dashboard where the section is marked complete.

**Backend (implemented, not yet connected to a frontend screen):**

The backend has a full program matching and discovery layer that is not yet surfaced in the frontend:

- `GET /api/programs/preferences/recommendations` runs `programRecommendationService`, which takes the saved preferences and scores each program in the database against them, returning a ranked list of matches.
- `GET /api/programs/search` supports full-text search across programs with filters.
- `POST /api/programs/save/:id` and `GET /api/programs/preferences/saved` allow users to bookmark programs and retrieve their saved list.
- Individual program records include compensation ranges, application deadlines, location, custom questions, and company info.

**Frontend screens to build (backend APIs already exist):**

**Screen 1: Recommended Programs**

After saving preferences, instead of redirecting to the dashboard, redirect the user to a new `/programs/results` screen. This screen calls `GET /api/programs/preferences/recommendations` on load and renders the returned programs as cards. Each card should display:
- Program name and specialty
- Location (city, state)
- Hospital type (academic / community)
- Application deadline

Each card has a "Save Program" button that calls `POST /api/programs/save/:id`. On success, the button changes to "Saved" (call `DELETE /api/programs/save/:id` to undo).

**Screen 2: Program Search**

A search bar on the same results screen calls `GET /api/programs/search?q=<query>` as the user types (debounced). The same program cards are rendered for search results. This allows users to find programs beyond their recommendations.

**Screen 3: Program Detail**

Clicking a program card navigates to `/programs/:id` and calls `GET /api/programs/:id`. Display all available fields:
- Full description
- Requirements and qualifications
- Compensation range (min/max salary, benefits)
- Application deadline and start date
- Contact email and phone
- Company name, logo, and website
- A "Save Program" / "Saved" toggle button (same save/unsave endpoints as above)

**Screen 4: Saved Programs**

A "Saved Programs" tab or button on the results screen calls `GET /api/programs/preferences/saved` and lists only the programs the user has bookmarked. Same card layout with an "Unsave" button calling `DELETE /api/programs/save/:id`.

---

## 2. MedWrite Academy

MedWrite Academy is an online course platform that teaches medical professionals how to write and publish research manuscripts. It is a separate service under the same umbrella. Users access it at `/medwrite/app` after subscribing.

### 2.1 What It Covers

- 7-module structured curriculum
- AI-assisted content generation (research questions, body sections)
- Mentor review workflow (modules 2 and 5)
- Final draft submission (module 7)
- Per-user progress tracking

### 2.2 Landing Page & Waitlist

Before subscribing, users can visit the MedWrite landing page (`/medwrite`) to learn about the product. A waitlist modal collects their email via `POST /api/medwrite/waitlist`.

---

### 2.3 Course Structure & Module Flow

The academy is organized into 7 modules. Each module has paginated lesson content served from the database (`MedWriteLesson` model). Progress is tracked per user in the `MedWriteProgress` document.

**Dashboard:**
`GET /api/medwrite/dashboard` bootstraps the full UI. It returns modules, lessons, and the user's current progress in one payload. The frontend renders a module card grid showing which modules are completed (green badge) and which is active.

**Navigation:**
Users move through modules page by page. `PUT /api/medwrite/progress` saves the current page and module state with a 1.2-second debounce to avoid excessive API calls. `POST /api/medwrite/modules/:moduleId/complete` marks a module as completed.

---

### 2.4 Module-by-Module Detail

#### Module 1: Course Introduction
Orientation content. No AI features, no mentor submission. User reads through lesson pages and completes the module.

#### Module 2: Topic & Research Question

This is the first interactive module with AI and mentor involvement.

**Flow:**
1. User enters two research topic areas and subtopics for each
2. Saved via `PUT /api/medwrite/module-2`
3. User requests AI assistance via `POST /api/medwrite/ai/research-question`; GPT-4o takes the topics and subtopics and generates focused research questions
4. User reviews the AI-generated questions and optionally writes their own
5. User submits to mentor via `POST /api/medwrite/module-2/submit-to-mentor`; backend sends an email (Nodemailer) to the assigned mentor with the user's questions and sets `sentToMentor` to true on the progress document

#### Module 3: Introduction Writing

**Flow:**
1. User fills in the three parts of a manuscript introduction: problem statement, status quo, and unmet need
2. User optionally requests AI-generated body section content via `POST /api/medwrite/ai/body-sections`; GPT-4o generates draft body sections based on the research question from Module 2
3. Saved via `PUT /api/medwrite/module-3`

#### Module 4: Sources & Discussion

**Flow:**
1. User provides intro sources and discussion sources for their manuscript
2. User can request AI assistance for body sections (same AI endpoint as Module 3)
3. Saved via `PUT /api/medwrite/module-4`

#### Module 5: Outline

**Flow:**
1. User writes section notes and assembles their manuscript outline
2. Saved via `PUT /api/medwrite/module-5`
3. User submits outline to mentor via `POST /api/medwrite/module-5/submit-outline`; backend sends the outline via email to the mentor and sets `outlineSent` on the progress document

#### Module 6: Writing the Draft
Guided writing module with content-based lessons. No AI endpoints or mentor submission.

#### Module 7: Final Draft Submission

**Flow:**
1. User uploads their completed manuscript draft (PDF, DOC, etc.)
2. Frontend calls `POST /api/medwrite/module-7/draft` with the file
3. Backend uploads the file to AWS S3 (research-drafts/ folder) and creates a `MedWriteDraftSubmission` record
4. Status begins as `submitted`; mentor can update to `under-review` or `reviewed` with notes

---

---

## 3. ScrubHub

ScrubHub is a **mobile application** (iOS and Android) for medical exam preparation. Its primary codebase lives in a separate repository. This platform integrates with ScrubHub only at two points:

**Marketing:** A landing page at `/scrubhub` describes the product (quiz banks, daily challenges, leaderboards, performance analytics) and links to the App Store and Google Play.

**Billing:** The `/scrubhub/billing` page handles ScrubHub subscription payments through Stripe using the same billing infrastructure as the other services (covered in Section 4).

The cross-service link is maintained via the `SubService` model, which stores the user's Firebase UID (`externalId`) alongside `serviceName: "scrubhub"`. This allows ScrubHub's separate backend to verify that a web-registered user has an active subscription.

There are no ScrubHub-specific API routes or business logic endpoints in this codebase.

### AI Tutor

The backend exposes an AI Tutor endpoint at `POST /api/ai/ask`, powered by GPT-4o. This is the backend integration point for ScrubHub's in-app AI Tutor feature, consumed by the mobile app.

- **Limit:** 100 queries per user per month (tracked in `User.aiUsage.monthlyCount`, reset monthly via `lastResetDate`)
- **Per-session limit:** 5 follow-up questions per MCQ interaction

---

## 4. Payments & Subscriptions

All three services are monetized through a shared billing system. Web subscriptions are handled by Stripe; mobile in-app purchases (ScrubHub on iOS/Android) are handled by RevenueCat.

### 4.1 Service Pricing (Stripe)

| Service | Price | Stripe Price ID |
|---|---|---|
| MatchMaker | $49/month | (configured in env) |
| ScrubHub | $7.99/month | (configured in env) |
| MedWrite Academy | (configured in env) | (configured in env) |

### 4.2 Stripe Subscription Flow (Web)

This flow applies to MatchMaker, MedWrite, and ScrubHub when subscribing through the web:

1. **Phone Verification (Pre-Checkout)**
   - Before accessing the billing page, the user verifies their phone number
   - `POST /api/subservice/initiate` generates a 6-digit OTP and sends it via Twilio SMS
   - `POST /api/subservice/verify` validates the code and stores a `PhoneVerification` record

2. **Checkout Session Creation**
   - User clicks Subscribe on the billing page
   - Frontend calls `POST /api/subservice` with the service name and price ID
   - Backend calls the Stripe API to create a Checkout Session
   - Returns `sessionId` and `checkoutUrl` to the frontend
   - Frontend redirects the user to Stripe's hosted checkout page

3. **Payment on Stripe**
   - User enters card details on Stripe's hosted page
   - Stripe processes the payment

4. **Webhook Processing**
   - Stripe sends events to `POST /api/stripe/webhook` (signature verified using `STRIPE_WEBHOOK_KEY`)
   - Backend handles the following events:

   | Event | Action |
   |---|---|
   | `checkout.session.completed` | Creates `Subscription` record, activates `SubService` entry |
   | `customer.subscription.created` | Sets trial and billing start/end dates |
   | `customer.subscription.updated` | Syncs status changes (e.g., trial to active) |
   | `customer.subscription.deleted` | Marks subscription as canceled or expired |
   | `invoice.payment_succeeded` | Extends the billing period end date |

5. **Subscription Record**
   - A `Subscription` document is created in MongoDB
   - Stores Stripe customer ID, subscription ID, plan, status, trial/billing dates
   - Status lifecycle: `trialing` -> `active` -> `canceled` / `past_due` / `expired`

6. **SubService Activation**
   - A `SubService` document is created or updated linking the user to the service name
   - `hasActivatedTrial` is tracked per service

### 4.3 Subscription Management

Users can manage their subscriptions from the billing pages:

| Action | Endpoint | Effect |
|---|---|---|
| Cancel | `POST /api/subservice/cancel/:subscriptionId` | Sets `cancelAtPeriodEnd: true`; access continues until billing period ends |
| Reactivate | `POST /api/subservice/reactivate/:subscriptionId` | Removes scheduled cancellation |
| Retry payment | `POST /api/subservice/retry-payment/:subscriptionId` | Retries a `past_due` payment |

The **Unified Dashboard** (`/subscription`) shows all three services side by side with their status and links to each billing page.

### 4.4 RevenueCat (Mobile - ScrubHub)

For users who subscribe to ScrubHub through the iOS App Store or Google Play:

1. RevenueCat handles the in-app purchase flow natively within the ScrubHub mobile app
2. RevenueCat sends events to `POST /api/subservice/revenuecathook`
3. Backend creates or updates an `InAppSubscription` document with: store (PLAY_STORE / APP_STORE), product ID, transaction ID, period type (TRIAL / NORMAL), expiration date
4. This record is used to verify active mobile subscriptions server-side

### 4.5 Data Models

| Model | Purpose |
|---|---|
| `Subscription` | Stripe web subscription record (all services) |
| `OneTimePayment` | One-time Stripe charges (non-recurring) |
| `InAppSubscription` | RevenueCat mobile subscription records |
| `SubService` | Activation record linking a user to a service name |
| `PhoneVerification` | Temporary OTP record for phone verification at billing |

### 4.6 Email Utilities at Billing

Nodemailer handles two billing-related emails:
- `POST /api/subservice/check-email`: checks if the user has an account and sends an account creation link if they do not
- `POST /api/subservice/account-delete`: sends an account deletion confirmation email

---

## 5. Backend Complete, Frontend Not Yet Built

This section covers every feature where the backend API is fully implemented and ready to use, but no frontend screen exists yet. Items are organized by service and include the exact API calls, suggested route, and how a user would experience the feature.

---

### 5.1 MatchMaker: User-Facing Features

---

#### User Application Stats

**Backend:** `GET /api/users/stats` returns the logged-in user's application breakdown by status (draft, submitted, interview, accepted, rejected) and their 5 most recent applications.

**Where it lives on FE:** A stats widget or card on the MatchMaker dashboard (`/dashboard`).

**User flow:**
1. Dashboard calls `GET /api/users/stats` on load alongside the existing `GET /api/dashboard` call.
2. A small stats row is displayed below the progress tracker showing counts: Submitted, In Interview, Accepted, Rejected.
3. A "Recent Applications" list shows the 5 most recent with their status badge and program name.

---

#### Application Tracking (Full System)

This is the most complete backend feature with no frontend at all. The backend supports the full lifecycle of a residency application: creating, submitting, interview scheduling, and final decision. Roles are enforced throughout.

**Applicant-side screens:**

**Screen: My Applications** (`/applications`)

On load calls `GET /api/applications`. The backend filters by role automatically so applicants only see their own. Each application card shows:
- Program name and specialty
- Current status badge (Draft / Submitted / Interview / Accepted / Rejected)
- Submission date
- A "View Details" button

**Screen: Application Detail** (`/applications/:id`)

Calls `GET /api/applications/:id`. Shows:
- Program name, deadline, required documents
- Status timeline (Draft → Submitted → Interview → Decision)
- Uploaded documents (resume, cover letter, portfolio link)
- Answers to any custom program questions
- Interview details if scheduled (date, round number, interviewers list)
- A "Submit Application" button (visible only when status is `draft`) that calls `PUT /api/applications/:id/submit`

**Screen: Create Application** (triggered from a Program detail page)

A "Apply Now" button on a program card calls `POST /api/applications` with `{ program: programId }`. This creates the application in `draft` status. The user is then redirected to `/applications/:id` to fill in required documents and answers before submitting.

---

#### Update Password (from Profile)

**Backend:** `PUT /api/users/updatepassword` accepts `{ currentPassword, newPassword }`, verifies the current password, and updates it.

**Where it lives on FE:** A "Change Password" section on the user profile/settings page.

**User flow:**
1. User navigates to their profile settings.
2. A "Change Password" section has three fields: Current Password, New Password, Confirm New Password.
3. On submit, frontend calls `PUT /api/users/updatepassword` with `{ currentPassword, newPassword }`.
4. On success, show "Password updated successfully."
5. On failure (wrong current password), show the error returned by the API.

---

### 5.2 MatchMaker: Program Manager Role

Program managers are users assigned the `program_manager` role. They manage their own programs and review applications submitted to those programs. None of this has a frontend yet.

---

#### Program Manager Dashboard

**Where it lives on FE:** A protected route `/manager` accessible only to users with `role === program_manager` or `role === admin`.

**Screen: My Programs** (`/manager/programs`)

Calls `GET /api/programs` — the backend already filters by the logged-in user's created programs for program managers. Each program card shows:
- Program title, status (draft / published / closed), application count
- "Edit" button → opens the program edit form
- "View Applications" button → navigates to `/manager/programs/:id/applications`
- "Create New Program" button at the top → opens the program creation form

**Screen: Create / Edit Program** (`/manager/programs/new` or `/manager/programs/:id/edit`)

A form that calls `POST /api/programs` (create) or `PUT /api/programs/:id` (edit). Fields to render:

| Field | Input Type |
|---|---|
| Title | Text |
| Description | Textarea |
| Department / Specialty | Dropdown |
| Type (Internship, Fellowship, etc.) | Dropdown |
| Location (city, state, country) | Text fields |
| Application Deadline | Date picker |
| Start Date / End Date | Date pickers |
| Compensation (min salary, max salary, currency) | Number inputs |
| Benefits | Text |
| Required Documents (resume, cover letter, etc.) | Checkboxes |
| Custom Questions | Add/remove question builder (text, multiple choice, yes/no) |
| Visibility (public / private / unlisted) | Dropdown |
| Status (draft / published / closed) | Dropdown |
| Featured | Toggle |

On submit calls `POST /api/programs` or `PUT /api/programs/:id`. On delete calls `DELETE /api/programs/:id`.

---

#### Application Review (Program Manager)

**Screen: Applications for My Program** (`/manager/programs/:id/applications`)

Calls `GET /api/applications` — backend returns only applications for the manager's programs. A list/table view showing:
- Applicant name and email
- Status badge
- Submission date
- Application rating (1-5 stars if rated)
- "View" button

Filter controls: by status (All / Submitted / Interview / Accepted / Rejected), sort by date or rating.

**Screen: Application Detail (Review View)** (`/manager/applications/:id`)

Calls `GET /api/applications/:id`. Shows all applicant data. Manager-specific actions:

| Action | Button Label | API Call |
|---|---|---|
| Update status | Status dropdown | `PUT /api/applications/:id` with `{ status }` |
| Add internal note | "Add Note" button + textarea | `POST /api/applications/:id/notes` |
| Schedule interview | "Schedule Interview" button | `POST /api/applications/:id/interviews` |
| Update interview | Edit interview | `PUT /api/applications/:id/interviews/:interviewId` |
| Rate application | Star rating (1-5) | `PUT /api/applications/:id` with `{ rating }` |

**Schedule Interview modal** fields: Round number, Scheduled date/time, Interviewers (text or user search), Notes.

**Screen: Application Stats** (`/manager/stats`)

Calls `GET /api/applications/stats`. Displays a summary card:
- Total applications across all managed programs
- Breakdown by status: Draft, Submitted, Interview, Accepted, Rejected
- Can be shown as a bar chart or number cards.

**Batch Actions:**

A checkbox on each application row in the list enables bulk selection. A "Batch Update" button appears when items are selected, calling `PUT /api/applications/batch` with `{ ids: [...], status: 'rejected' }` (or any status). Useful for bulk rejecting or shortlisting.

---

### 5.3 MatchMaker: Admin Role

Admins have full access to everything program managers can do, plus user management.

---

#### Admin User Management

**Screen: All Users** (`/admin/users`)

Calls `GET /api/users` (admin only). Note: the `advancedResults` pagination middleware is not currently applied to this route, so the endpoint returns `undefined` data as-is. A one-line backend fix is needed first — adding `advancedResults(User)` as middleware on the `GET /` route in `backend/src/routes/users.js`, the same way it is applied in `programRoutes.js`. Once fixed, the endpoint returns paginated results with filtering and sorting support.

A paginated table of all registered users showing:
- Name, email, role, registration date, last login
- "Edit" button to change role or update user details → calls `PUT /api/users/:id`
- "Delete" button → calls `DELETE /api/users/:id` with a confirmation dialog

**Screen: Create User** (`/admin/users/new`)

A form calling `POST /api/users` with fields: firstName, lastName, email, password, role (dropdown: applicant / program_manager / admin).

**Role assignment:** An "Edit Role" dropdown on the user row or detail page lets admins change a user's role between `applicant`, `program_manager`, and `admin`. This is the mechanism by which someone becomes a program manager.

---

#### Admin Program Management

Admins see all programs (not just their own) via `GET /api/programs`. The create/edit/delete flow is the same as the program manager's but with no ownership restriction.

- `DELETE /api/programs/:id` is admin-only (program managers can only delete their own programs).

---

### 5.4 Cross-Service: Register Service (External App Link)

**Backend:** `POST /api/subservice/registerService` accepts `{ serviceName, externalId }` and creates or updates a `SubService` record. The `externalId` is the user's Firebase UID from the ScrubHub mobile app. This is how the mobile app links a web account to a mobile account.

**Where it lives on FE:** This is called by the ScrubHub mobile app, not the web frontend. However, on the web Unified Dashboard (`/subscription`), a "Link Mobile Account" section could display the user's `externalId` (or show a QR code / deep link) so the mobile app can associate the accounts. No additional backend work is needed.