# Research Rank: AI-Powered Medical Research Evaluation Platform

**Research Rank** is an AI-powered web application that enables medical professionals and academic institutions to evaluate and rank medical students based on their research portfolios. The platform uses advanced scoring algorithms, OpenAI analysis, and impact factor databases to provide comprehensive research assessment and comparative ranking.

**Tech Stack:** React 18 · Vite · Firebase (Auth, Firestore, Storage) · OpenAI API · Chart.js · Plotly.js · jsPDF · XLSX

## Table of Contents

1. [Project Architecture](#project-architecture)
2. [Authentication and User Management](#authentication-and-user-management)
3. [Current Implementation Status](#current-implementation-status)
4. [Feature Breakdown: End-to-End Flows](#feature-breakdown-end-to-end-flows)
5. [OpenAI Integration](#openai-integration)
6. [Data Models](#data-models)
7. [Backend Complete, Frontend Not Yet Built](#backend-complete-frontend-not-yet-built)
8. [Known Gaps and Issues](#known-gaps-and-issues)

---

## Project Architecture

Research Rank is a **frontend-only React application** deployed with Vite, using Firebase as the backend-as-a-service (BaaS) for:

- User authentication and session management
- Data persistence (Firestore database)
- File storage (Cloud Storage)

The application is structured around the following routing layers:

- **Public route:** Landing page (/) - marketing and entry point
- **App entry:** /app - authentication gateway
- **Protected routes:** /app/dashboard and related flows - require Firebase authentication and user profile

### Routing Map

```
/                       → Landing (public)
/app                    → AppEntry (auth check + redirect)
/app/login              → Firebase email/password login
/app/signup             → Firebase account creation
/app/register           → Profile completion (first name, last name, specialty, institution)
/app/dashboard          → Protected dashboard with ResearchRatingComponent
/test                   → Development utility for impact factor lookup
```

---

## Authentication and User Management

### User Authentication Flow

1. User navigates to landing page (/)
2. If not authenticated, clicking "Get Started" redirects to /app/login or /app/signup
3. **Sign Up** (/app/signup):
   - User provides email and password (6+ chars)
   - Firebase creates auth account
   - Redirects to /app/register (profile completion page)
4. **Login** (/app/login):
   - User provides email and password
   - Firebase authenticates
   - System checks if user profile exists in Firestore
   - If profile exists: redirects to /app/dashboard
   - If profile missing: redirects to /app/register
5. **Register** (/app/register):
   - User completes profile: firstName, lastName, institution, specialty
   - Profile saved to Firestore `users` collection
   - Redirects to /app/dashboard

### User Data Model

**Firestore Collection: `users`**

```
Document ID: Auto-generated
Fields:
  - email: string (user's Firebase auth email)
  - firstName: string
  - lastName: string
  - institution: string (medical school or university)
  - specialty: string (e.g., "Radiology", "Neurology", "Surgery")
```

### Role-Based Access

The platform does not yet implement role-based access control (RBAC). All authenticated users with complete profiles have full access to the dashboard.

---

## Current Implementation Status

The application is in a **partial implementation state**:

### Implemented Features

1. **User Authentication** - Complete
   - Signup, login, profile creation
   - Protected routes
   - Session management via Firebase Auth

2. **Landing Page** - Complete
   - Marketing website with animations
   - Feature highlights
   - Workflow visualization
   - YouTube demo link
   - CTA buttons

3. **Dashboard Entry** - Partial
   - User profile display with welcome message
   - Logout functionality
   - ResearchRatingComponent (rating form only)

4. **Research Rating Form** - Partial
   - 7 weighted rating scales (0-10) for research evaluation criteria
   - 100+ characteristic traits selection (5-10 selection requirement)
   - Form submission and Firestore persistence
   - No next step after submission

### Not Yet Integrated to UI

The following features exist in code but are commented out and not accessible through the UI:

- Research product data entry (title, authors, publication type, journal, impact factor)
- Student information management (DOB, medical school, profile photo)
- Supporting document input (Statement of Purpose, Letters of Recommendation)
- Research scoring calculations and result display
- PDF report generation
- Excel bulk import/export
- Student image upload
- Scoring results and ranking visualization
- Comparative analysis and bell curve charts

---

## Feature Breakdown: End-to-End Flows

### 1. Characteristic Rating Setup (Currently Implemented)

**User Flow:**

1. User logs in and lands on /app/dashboard
2. Sees the "Research Ratings" page with:
   - 7 rating categories with descriptions (0-10 scale each):
     * Total Number of Research Products (📊)
     * Research Relates to Your Specialty (🎯)
     * Being First Author on Project (👤)
     * Peer-Reviewed Journal Articles (📝)
     * Abstract or Presentation Research (🎤)
     * Published Research Products (📚)
     * Impact Factor of Journals (⭐)
   - Scrollable list of 100+ professional characteristics
   - Select 5-10 characteristics that matter most
3. Enters rating values (0-10) for each criterion
4. Selects preferred characteristics
5. Clicks "Submit Ratings"

**Data Stored:**

Firestore `ratings` collection:

```
Document ID: user's email
Fields:
  - totalNumberOfResearchProducts: number (0-10)
  - researchRelatesToSpecialty: number (0-10)
  - firstAuthorOnProject: number (0-10)
  - peerReviewedJournalArticles: number (0-10)
  - abstractResearch: number (0-10)
  - publishedResearch: number (0-10)
  - impactFactorOfJournals: number (0-10)
  - selectedCheckboxes: string[] (array of characteristic names)
```

**After Submission:**

Currently the form closes. No redirect or next step is implemented.

### 2. Research Product Entry (Code Exists, Not in UI)

The ResearchProducts component (commented out) was designed to accept:

- Student Information: First name, last name, DOB, medical school, profile photo
- Research Products (multiple): Title, authors, research type, publication status, publication venue
- Supporting Documents: Statement of Purpose (SOP), Letters of Recommendation (LOR)
- Bulk Import: Excel file support

This data was intended to be saved to Firestore `researchProducts` collection.

### 3. AI-Powered Scoring (Code Exists, Not in UI)

The Result component (commented out) contains logic to:

- **Specialty Matching:** OpenAI Assistant evaluates research titles against user's specialty
- **First Author Count:** OpenAI Assistant counts how many research products list the student as first author
- **SOP/LOR Analysis:** OpenAI Assistant analyzes supporting documents against selected characteristics

Scoring formula (from code):

```
researchScore = 
  (count of research products) * rating[totalNumberOfResearchProducts] +
  (specialty-aligned count) * rating[researchRelatesToSpecialty] +
  (first author count) * rating[firstAuthorOnProject] +
  (peer-reviewed count) * rating[peerReviewedJournalArticles] +
  (abstract/presentation count) * rating[abstractResearch] +
  (published count) * rating[publishedResearch] +
  (impact factor sum) * rating[impactFactorOfJournals]
```

Additionally:
- **sopScore:** Percentage match of SOP against selected characteristics
- **lorScore:** Percentage match of each LOR against selected characteristics

### 4. Report Generation (Code Exists, Not in UI)

The `createPdf` utility is available but not called by any UI. It would generate professional PDF reports with:

- Student summary (name, medical school, DOB)
- Research score breakdown
- SOP score and LOR scores
- Charts: bell curve distribution, percentile ranking
- Comparative analysis

---

## OpenAI Integration

The application integrates OpenAI's Assistants API for content analysis. **All OpenAI API calls occur in the browser** (marked with `dangerouslyAllowBrowser: true` in calculationsHelper.js), exposing the API key in client-side code.

### OpenAI Setup

**Environment Variables Required:**

```
VITE_OPENAI_KEY              - OpenAI API key
VITE_ASST_ID_1               - Assistant ID for specialty matching
VITE_THREAD_ID_1             - Thread ID for specialty matching
VITE_ASST_ID_2               - Assistant ID for first author counting
VITE_THREAD_ID_2             - Thread ID for first author counting
VITE_ASST_ID_3               - Assistant ID for SOP/LOR analysis
VITE_THREAD_ID_3             - Thread ID for SOP/LOR analysis
```

### OpenAI Functions (in calculationsHelper.js)

1. **countSpecialty(specialty, products)**
   - Input: User specialty (string) and array of research titles
   - Output: Count of research products aligned to specialty
   - Uses: VITE_ASST_ID_1, VITE_THREAD_ID_1
   - Status: Code complete, not integrated to UI

2. **countFirstName(firstName, lastName, products)**
   - Input: Student name and array of author lists
   - Output: Count of first-author publications
   - Uses: VITE_ASST_ID_2, VITE_THREAD_ID_2
   - Status: Code complete, not integrated to UI

3. **sopInfo(characteristics, sopText)**
   - Input: Selected characteristics and statement of purpose text
   - Output: Percentage match and detailed analysis
   - Uses: VITE_ASST_ID_3, VITE_THREAD_ID_3
   - Status: Code complete, not integrated to UI

4. **lorInfo(characteristics, lorText)**
   - Input: Selected characteristics and letter of recommendation text
   - Output: Percentage match and detailed analysis
   - Uses: VITE_ASST_ID_3, VITE_THREAD_ID_3
   - Status: Code complete, not integrated to UI

### Impact Factor Database

The application includes a hardcoded database of 1000+ journals with 2022 Journal Impact Factor (JIF) values:

- Location: src/data/ImpactFactor.js
- Format: Array of objects: `{ "Journal name": string, "2022 JIF": number }`
- Function: searchImpact(journalName) looks up impact factor by journal name
- Status: Available for scoring but not yet exposed to research product entry

---

## Data Models

### Firebase Firestore Collections

#### 1. `users` Collection

Stores user profile information. Created during /app/register flow.

```
Document ID: Auto-generated
{
  email: string,              // User's Firebase auth email
  firstName: string,          // First name
  lastName: string,           // Last name
  institution: string,        // Medical school/University
  specialty: string           // Medical specialty (required for specialty matching)
}
```

#### 2. `ratings` Collection

Stores user's research evaluation weights and preferred characteristics. Created on first research rating form submission.

```
Document ID: User's email address
{
  totalNumberOfResearchProducts: number,        // 0-10
  researchRelatesToSpecialty: number,           // 0-10
  firstAuthorOnProject: number,                 // 0-10
  peerReviewedJournalArticles: number,          // 0-10
  abstractResearch: number,                     // 0-10
  publishedResearch: number,                    // 0-10
  impactFactorOfJournals: number,               // 0-10
  selectedCheckboxes: array<string>             // Array of characteristic names (5-10 items)
}
```

#### 3. `researchProducts` Collection (Code-only, Not in Current UI)

Designed to store research evaluation records (not actively populated).

```
Document ID: User's email address
{
  savedData: array<{
    fName: string,                    // Student first name
    lName: string,                    // Student last name
    dob: string,                      // Date of birth
    collegeName: string,              // Medical school
    studentImage: string,             // URL to profile image
    researchProducts: array<{
      title: string,
      authors: array<string>,         // Author names
      researchType: string,           // "Peer-reviewed publication", "Abstract", "Presentation"
      publicationStatus: string,      // "Published", "Accepted", "Submitted"
      publicationName: string         // Journal or conference name
    }>,
    sop: string,                      // Statement of Purpose text
    lors: array<string>,              // Letters of Recommendation
    researchScore: number,            // Calculated score
    sopScore: object,                 // SOP analysis result
    lorScore: object                  // LOR analysis result
  }>
}
```

---

## Backend Complete, Frontend Not Yet Built

The following backend logic and OpenAI integrations are implemented but have no corresponding UI screens:

### 1. Research Product Entry and Management

**What's Ready in Backend:**

- Data model designed and tested
- Add/edit/delete research products
- Support for multiple research types: Peer-reviewed publication, Abstract, Presentation
- Publication status tracking: Published, Accepted, Submitted
- Author list parsing from semicolon-separated format
- Form validation for all fields

**Suggested UI Location:** `/app/dashboard/research-products`

**UI Flow to Implement:**

1. **Research Products List Page** (/app/dashboard/research-products)
   - Show existing research products for current student (if editing)
   - List view with edit/delete buttons
   - "Add Research Product" button

2. **Add/Edit Research Product Form** (/app/dashboard/research-products/add or /edit/:id)
   - Form fields:
     * Title: Text input
     * Authors: Textarea (format: "First Last; First Last; ...")
     * Research Type: Dropdown (Peer-reviewed publication, Abstract, Presentation)
     * Publication Name: Text input (journal or conference name)
     * Publication Status: Dropdown if peer-reviewed (Published, Accepted, Submitted)
   - Success: Product added to Firestore, return to list
   - Error: Display error message

3. **Journal Impact Factor Lookup** (within product form)
   - Text input to search journal name
   - Auto-suggest from ImpactFactor database
   - Display JIF value when selected
   - Store with research product

### 2. Student Information Management

**What's Ready in Backend:**

- Data model for student details: DOB, medical school, profile photo
- Image upload utility (html2canvas + Firebase Storage)
- Form validation for all fields
- Bulk import from Excel files

**Suggested UI Location:** `/app/dashboard/student-profile`

**UI Flow to Implement:**

1. **Student Profile Page** (/app/dashboard/student-profile)
   - Current view: Evaluator's profile (already built in Register)
   - Suggested new section: Evaluate Student Profile
     * Student name: Text inputs (first, last)
     * Date of birth: Date picker
     * Medical school: Text input
     * Profile photo: File upload with preview
     * Photo upload status and image display

2. **Bulk Import from Excel** (/app/dashboard/import)
   - File upload input for .xlsx files
   - Expected columns: FirstName, LastName, DOB, MedicalSchool, [optional: Photo URLs]
   - Upload and parse logic
   - Show preview of parsed data with validation results
   - Import button to save to Firestore

### 3. Supporting Documents (SOP and LOR)

**What's Ready in Backend:**

- OpenAI Assistant integration for SOP analysis
- OpenAI Assistant integration for LOR analysis
- Percentage match calculation against selected characteristics
- Support for multiple LORs per student

**Suggested UI Location:** `/app/dashboard/student-documents`

**UI Flow to Implement:**

1. **Statement of Purpose (SOP) Section** (/app/dashboard/student-documents/sop)
   - Text area for entering SOP
   - Save button
   - After save: Show AI analysis result:
     * Percentage match against evaluator's selected characteristics
     * Key strengths identified
     * Areas of concern (if any)

2. **Letters of Recommendation (LOR) Section** (/app/dashboard/student-documents/lors)
   - Add multiple LOR text areas (1-3 typical)
   - Each LOR has:
     * Dropdown to select recommender role (Professor, Advisor, Research Mentor)
     * Large text area for LOR content
     * Save button
   - After save, each LOR shows:
     * Percentage match against selected characteristics
     * Confidence score
   - LOR list view with edit/delete per LOR

### 4. Scoring and Results Dashboard

**What's Ready in Backend:**

- Comprehensive scoring algorithm combining all metrics
- Bell curve distribution calculation
- Percentile ranking
- PDF report generation with charts
- Historical data tracking

**Suggested UI Location:** `/app/dashboard/results`

**UI Flow to Implement:**

1. **Evaluation Summary** (/app/dashboard/results/:studentId)
   - Student name, medical school, DOB
   - Score breakdown by category:
     * Total Research Products Count
     * Specialty-Aligned Count
     * First Author Count
     * Peer-Reviewed Count
     * Abstract/Presentation Count
     * Published Count
     * Impact Factor Total
   - Research Score: Large number/visual gauge
   - SOP Score: Percentage with bar chart
   - LOR Scores: Percentage for each LOR with bar chart

2. **Comparative Ranking** (/app/dashboard/results/:studentId/ranking)
   - Bell curve chart (Plotly.js ready, in code)
   - Show student's position in percentile
   - Show comparative scores against peer group
   - Downloadable ranking data

3. **Generate Report** (button on results page)
   - PDF download with all information
   - Includes all charts and breakdowns
   - Professional formatting

---

## Characteristics List

The application includes a curated list of 97 professional characteristics used for evaluator preference rating and AI-powered matching. Located in src/data/charateristicsList.js:

Sample characteristics include:
- Clinical: Compassionate, Empathetic, Patient, Good bedside manner, Strong clinical skills
- Leadership: Strong leadership skills, Mentor-oriented, Good teaching skills, Takes initiative
- Communication: Good communicator, Clear communicator, Good public speaking skills
- Resilience: Resilient, Emotionally resilient, Handles stress well, Persistent
- Ethics: Ethical, Respects patient autonomy, Good judgment, Accountable
- Teamwork: Team player, Collaborative, Values teamwork, Works well under supervision

Full list has 97 traits covering medical, interpersonal, technical, and professional competencies.

---

## Known Gaps and Issues

### Critical Issues

1. **OpenAI API Key Exposure**
   - File: src/utils/calculationsHelper.js line 9: `dangerouslyAllowBrowser: true`
   - Issue: API key is exposed in client-side code
   - Impact: Security risk, API key can be extracted from browser
   - Fix: Move OpenAI calls to a backend service (Node.js, Cloud Functions, etc.)

2. **Incomplete Feature Integration**
   - Heavy components (ResearchProducts, Result, UploadStudentImage) exist but are entirely commented out
   - ResearchRatingComponent form submission has no callback or next step
   - Dashboard shows only the rating form, no navigation to other features

3. **Browser Storage of Sensitive Data**
   - File: src/pages/login/login.jsx lines 72-73
   - Code: `localStorage.setItem('token', user.accessToken)` and `localStorage.setItem('user', JSON.stringify(user))`
   - Issue: Firebase accessToken stored in localStorage (already session-managed by Firebase)
   - Fix: Remove localStorage calls; rely on Firebase Auth session management

### Code Issues

1. **Undefined Variable in calculationsHelper.js**
   - Line 18: `fName,lName` statement without assignment in countFirstName1 function
   - Function countFirstName1 is never used; consider removing

2. **Broken Reference in calculationsHelper.js**
   - Line 86: References `VITE_THREAD_ID_2` but function is defined for VITE_THREAD_ID_1 context
   - Thread ID mismatch for different assistant purposes

3. **Test Component References Wrong Import**
   - File: src/pages/test.jsx line 2
   - Code: `import { impactData } from '../data/ImpactFactor'` 
   - Impact: Component assumes it's one level higher than it actually is in directory structure
   - Fix: Use correct relative path

### Missing Functionality

1. **No Error Handling for OpenAI Rate Limits**
   - OpenAI calls have no retry logic or rate limit handling
   - User receives no feedback if API quota exceeded

2. **No Loading State During AI Analysis**
   - When countSpecialty, countFirstName, sopInfo, lorInfo are called, user sees no progress
   - Can appear frozen for 10+ seconds per call

3. **No Fallback for Missing Environment Variables**
   - Application will crash at runtime if any VITE_* variable is undefined

4. **Protected Route Does Not Redirect Incomplete Profiles**
   - Protected.jsx checks authentication but does not enforce profile completion
   - Users can still access /app/dashboard without completing /app/register

5. **Result Component Has Commented Mock Data**
   - File: src/components/Result/Result.jsx line 4
   - Mock data available but no way to test results flow without uncommenting

### UI/UX Gaps

1. **No Student List Management**
   - Evaluators cannot view or manage multiple student evaluations
   - No dashboard for batching evaluations
   - No history of past evaluations

2. **No Multi-Student Comparison**
   - Scoring is designed for comparing students but UI does not implement comparison view

3. **No Undo/Save Draft**
   - Form submissions are immediate; no draft save option
   - User cannot modify ratings after initial submission without manual Firestore edit

4. **No Role Separation**
   - All users appear as evaluators; no student view or read-only access
   - No permission system for managing who can see which evaluations

---

## Development Notes

### Setup

```bash
npm install
npm run dev        # Start Vite dev server (localhost:5173)
npm run build      # Production build
npm run preview    # Preview production build
npm run lint       # ESLint check
```

### Environment Setup

Create .env file with all VITE_* variables listed above. OpenAI API key must be valid; Assistant IDs and Thread IDs must exist in the associated OpenAI account.

### Development Utilities

- `/test` route: Journal impact factor lookup (development only)
- `src/mocks/mockData.js`: Dummy result data for testing (not used in current UI)

### Key Dependencies

- **react-router-dom**: Client-side routing
- **firebase**: Auth, Firestore, Storage
- **openai**: OpenAI API client
- **chart.js & react-chartjs-2**: Interactive charts
- **plotly.js & react-plotly.js**: Statistical plotting
- **jspdf & html2canvas**: PDF generation
- **xlsx**: Excel import/export
- **uuid**: Unique IDs for research products
- **react-tooltip**: Hover tooltips

