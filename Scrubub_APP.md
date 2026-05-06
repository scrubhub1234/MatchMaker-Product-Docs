
# ScrubHub

A mobile medical education app for practicing clinical questions, challenging friends, and tracking progress across organ systems.

**Stack:** Expo 54 / React Native 0.81 · Expo Router 6 · NativeWind 4 / Tailwind CSS 3 · Firebase Auth + Firestore · Zustand · RevenueCat · React Native Reanimated · Lottie

---

## Table of Contents

1. [Project Type](#1-project-type)
2. [Authentication Flow](#2-authentication-flow)
3. [Home Screen and Navigation Structure](#3-home-screen-and-navigation-structure)
4. [Study Mode](#4-study-mode)
5. [Review Mode](#5-review-mode)
6. [Daily Challenge](#6-daily-challenge)
7. [Friend Challenges](#7-friend-challenges)
8. [Question Types](#8-question-types)
9. [Social Features](#9-social-features)
10. [Leaderboards](#10-leaderboards)
11. [Notifications](#11-notifications)
12. [User Profile and Account Management](#12-user-profile-and-account-management)
13. [Subscriptions and Payments](#13-subscriptions-and-payments)
14. [AI Tutor Integration](#14-ai-tutor-integration)
15. [Firebase Integration](#15-firebase-integration)
16. [State Management](#16-state-management)
17. [External Backend](#17-external-backend)
18. [Known Gaps and Issues](#18-known-gaps-and-issues)

---

## 1. Project Type

ScrubHub is a React Native mobile app (iOS and Android). There is no custom backend server in this repository. All data persistence is handled by Firebase (Auth, Firestore). Subscription management uses RevenueCat. An external REST API at `matchmaker-backend-iota.vercel.app` handles email validation, account deletion emails, subscription service registration, and AI tutor requests. That external backend is a separate repository not included here.

---

## 2. Authentication Flow

Authentication uses Firebase Phone Auth (SMS OTP). There is no email/password login.

**Screens involved:** `onboarding.jsx` → `emailScreen.jsx` → `register.jsx` → `otpScreen.jsx` → `userInfoScreen.jsx`

**Step-by-step flow:**

1. **Onboarding** (`app/onboarding.jsx`): Landing screen with a logo and "Get Started" button. No Firebase calls. Navigates to `emailScreen`.

2. **Email Entry** (`app/emailScreen.jsx`): User enters their email address. The app calls `POST /api/subservice/check-email` on the external backend to validate the email. If the email is new, the user cannot proceed (new registrations are gated). If the email is recognized, the app navigates to `register` with `{email}` as a route param.

3. **Phone Number Entry** (`app/register.jsx`): User enters their phone number with a country code picker. The app queries the `Users` Firestore collection (`where("email", "==", email)`) to check if the user already exists. If the user exists, it validates that the phone number matches the stored number. Then it calls `initiatePhoneVerification(email, phoneNumber)` which triggers `Firebase Auth signInWithPhoneNumber`. Navigates to `otpScreen` with `{phoneNumber, email}` params.

4. **OTP Verification** (`app/otpScreen.jsx`): User enters the 6-digit SMS code. Calls `verifyPhoneCode(email, phoneNumber, code)` which calls `confirmation.confirm(code)` from Firebase Auth. On success, reads the `Users` Firestore doc by UID. If the doc exists, calls `setUser()` in Zustand and navigates to `/` (home). If the doc does not exist (new user), navigates to `userInfoScreen` with `{phoneNumber, uid, email}` params.

5. **User Info Setup** (`app/userInfoScreen.jsx`): New users choose one of 8 avatars and enter a username. On confirm, writes a new doc to `Users/{uid}` with fields: `phoneNumber, avatarId, username, uid, totalScore: null, totalSolved: null, lastDailyChallengeID: null, solvedTopics: [], createdAt: serverTimestamp(), hasCompletedOnboarding: false, email`. Also calls `POST /api/subservice/registerService` to register the user with the external subscription service. Navigates to `/` (home).

6. **Auth redirect** (`app/firebaseauth/link.jsx`): Deep link handler that auto-redirects to `otpScreen`. No logic beyond redirect.

---

## 3. Home Screen and Navigation Structure

**File:** `app/(tabs)/index.jsx`

The home screen is the main entry point after login. It shows four large action buttons:

- **Study by System** (red): navigates to `details.jsx` in study mode
- **Review** (yellow): navigates to `details.jsx` in review mode
- **Daily Challenge** (purple): navigates to the daily challenge question flow
- **Play with Friends** (blue): navigates to `friends.jsx`

On first launch, an onboarding modal is shown. The app reads the `hasCompletedOnboarding` flag from the `Users` Firestore doc and writes `hasCompletedOnboarding: true` when the modal is dismissed.

Features are gated by subscription. If the user has no active subscription and no active trial, all four buttons show a lock icon and display a tooltip when tapped.

The app uses `Purchases.logIn(user.uid)` from RevenueCat on this screen to associate the device with the user account.

**Navigation structure overview:**

```
app/
  (tabs)/
    index.jsx           Home
    Notifications.jsx   Notifications tab
    profile.jsx         Profile tab
  onboarding.jsx
  emailScreen.jsx
  register.jsx
  otpScreen.jsx
  userInfoScreen.jsx
  details.jsx           System picker (study + review entry)
  topics.jsx            Topic picker within a system
  fourOptQues.jsx       4-option MCQ question screen
  multipleOptSelect.jsx Multi-select question screen
  matching.jsx          Drag-and-drop matching screen
  wordscrambled.jsx     Word scramble screen
  incompleteProcess.jsx Fill-in-the-blank flowchart screen
  casemystery.jsx       Case mystery menu (nav only)
  review.jsx            Review menu (nav only)
  scoreScreen.jsx       Score display after quiz
  reviewScoreScreen.jsx Score display after review quiz
  questionmark.jsx      Fallback/placeholder screen
  PaymentScreen.jsx     Subscription purchase screen
  friends.jsx           Friends list and invite
  UserContacts.jsx      Device contacts list
  ChallengeFriend.jsx   Friend picker for challenges
  DisplayChallenges.jsx Incoming and past challenges list
  challengeLeaderboard.jsx  Daily challenge leaderboard
  FriendsLeaderboard.jsx    Friends total score leaderboard
  firebaseauth/link.jsx     Deep link redirect
```

---

## 4. Study Mode

Study mode lets the user practice questions filtered by organ system and topic.

**Screens:** `details.jsx` → `topics.jsx` → question screen → `scoreScreen.jsx`

**Step-by-step flow:**

1. **System Picker** (`app/details.jsx`): Displays 9 organ system buttons (Cardiovascular, Hematology, GI, Musculoskeletal, Neurology, Pulmonary, Psychiatry, Renal, Reproductive). Locked with a padlock icon if no active subscription or trial. On tap, reads the `Topics` Firestore collection: `doc(db, "Topics", systemNameLowercase)` to retrieve the topics array for that system.

2. **Topic Picker** (`app/topics.jsx`): Displays the topics for the selected system as a scrollable list. A "Random" button is also available. On topic selection, calls `quesStore.fetchQuestions(system, topic)` which queries the `Questions` Firestore collection. On "Random", calls `fetchQuestions(system, allTopics)`. After questions are loaded, routes to the appropriate question screen based on `getQuestionType(question.questionStyle)`.

3. **Question screens** (see [Section 8](#8-question-types)): Users answer 10 questions. After the last question, the app navigates to `scoreScreen`.

4. **Score Screen** (`app/scoreScreen.jsx`): Displays the score out of 10. Score shown in green if 8 or more, red otherwise. A "Challenge a friend" button is available in study mode. Tap it to navigate to `ChallengeFriend.jsx`.

**Firestore reads:**
- `Topics/{system}` (topics array)
- `Questions/{system}/{topic}/{questionId}` (question documents)

**Firestore writes:**
- `Users/{uid}/solved/{system}/{topic}/solvedIds` (batch appended after submission)
- `Users/{uid}.totalScore` (incremented)
- `Users/{uid}.totalSolved` (incremented)
- `Users/{uid}.solvedTopics` (array union)

---

## 5. Review Mode

Review mode lets users re-practice questions they have already answered correctly.

**Screens:** `review.jsx` → `details.jsx` → `topics.jsx` → question screen → `reviewScoreScreen.jsx`

**Step-by-step flow:**

1. **Review Menu** (`app/review.jsx`): Navigation-only screen with two buttons: "Review All" and "Review By System". "Review All" fetches questions using `getReviewAllQuestions()` which samples from all of the user's `solvedTopics`. "Review By System" goes to `details.jsx` in review mode.

2. **System and Topic Picker** (same `details.jsx` and `topics.jsx` as study mode): In review mode, calls `quesStore.fetchReviewQuestions(system, topic)` instead of `fetchQuestions`.

3. **Question screens**: Same question screens as study mode, but driven by `reviewQuestions` from the store instead of `questions`.

4. **Review Score Screen** (`app/reviewScoreScreen.jsx`): Simpler score display showing the review score. Same coloring as study mode. No "Challenge a friend" button.

**Firestore reads:**
- `Users/{uid}/solved/{system}/{topic}/solvedIds` (to know which questions were previously solved)
- `Questions/{questionId}` (fetched by ID, up to 10 per session)

**Firestore writes:** None (review mode does not update solved counts).

---

## 6. Daily Challenge

A time-limited daily quiz available to all users. A single challenge document is active per day and is tracked so users cannot attempt it more than once.

**Screen flow:** Home "Daily Challenge" button → question screens → `scoreScreen.jsx` → `challengeLeaderboard.jsx`

**Step-by-step flow:**

1. Tapping "Daily Challenge" on the home screen calls `quesStore.fetchChallengeQuestions()`. This reads the `dailyChallenge` Firestore collection to find the active challenge document and loads its questions.

2. The app routes to the appropriate question screen. The store tracks answers under `challengeQuestions` and uses `currentChallengeIndex`.

3. After the last question, calls `quesStore.submitChallengeQuestions()` which writes the score to `Users/{uid}.lastChallengeScore` and `Users/{uid}.lastDailyChallengeID`.

4. `scoreScreen.jsx` displays the challenge score and offers a "Challenge a friend" button (sends that score as a challenge baseline).

5. Navigating to `challengeLeaderboard.jsx` shows all users who completed today's daily challenge, sorted by `lastChallengeScore`.

**Subscription gate:** Active subscription or trial required to access.

---

## 7. Friend Challenges

Users can challenge individual friends to a 10-question quiz. The challenger plays first, then the opponent attempts the same questions. Scores are compared when both sides have played.

**Screens involved:** `ChallengeFriend.jsx` → question screens → `scoreScreen.jsx` (challenger side); `DisplayChallenges.jsx` → question screens → `scoreScreen.jsx` (opponent side)

**Challenger flow:**

1. After a study or daily challenge session, the user taps "Challenge a Friend" on `scoreScreen.jsx`.

2. `ChallengeFriend.jsx` displays the user's friend list (loaded via `useGetFriends()`). The user taps a friend.

3. `quesStore.fetchChallengingFriendsQuestions(challenge, friendId)` loads 10 questions.

4. The user answers the questions. On submit, `quesStore.submitChallengingFriend()` writes a new `Challenges` document with fields: `challengerId, opponentId, challengerScore, opponentScore: null, status: "pending", questions[], timestamp`. Calls `useChallengeFriend.challengeFriend(friendId, myScore, questions)`.

5. A notification is sent to the opponent via `useAddNotification.addChallengeNotification(...)` which writes to the `Notifications/{opponentId}` Firestore document.

6. `scoreScreen.jsx` shows "Challenge sent to your friend!"

**Opponent flow:**

1. Opponent sees a notification badge on the Notifications tab. Tapping the challenge notification navigates to `DisplayChallenges.jsx`.

2. `DisplayChallenges.jsx` lists all challenges fetched via `useGetChallenges()`. Pending challenges (where the current user is the opponent) show an "Attempt" button.

3. Tapping "Attempt" calls `quesStore.fetchChallengeFriendQuestions(challenge)` which loads the same questions from the challenge document.

4. After answering, `quesStore.submitFriendChallenge()` updates the `Challenges` document: sets `opponentScore` and `status: "completed"`. Calls `useChallengeFriend.challengeCompleted()` which sends notifications to both the challenger and mutual friends via `addChallengeCompletedNotification()`.

5. `scoreScreen.jsx` compares both scores and shows win, loss, or draw.

**Firestore reads:**
- `Users/{uid}` (friend list, usernames, avatarIds)
- `Challenges` (where challengerId or opponentId equals uid)

**Firestore writes:**
- `Challenges/{challengeId}` (new doc on challenge creation, updated on completion)
- `Notifications/{uid}` (notificationsArray updated via batch)

---

## 8. Question Types

All question types are driven by `useQuesStore`. The store fields used depend on the current mode (study, review, daily challenge, friend challenge, challenging friends). All question screens use the `MCQFooter` component to handle submit, skip, and next navigation. An AI hint button opens `AiTutorChat` for hints.

### 4-Option MCQ (`app/fourOptQues.jsx`)

Handles question styles: `quickDiagnosis`, `shortFacts`, `firstLineTreatment`, `lab`, `testToOrder`.

The screen displays a question text and 4 colored option buttons. The user taps one to select it. On submit, the selected text is compared against the correct answer text. Correct and incorrect answers are highlighted. An explanation is shown in a slide-up modal. The screen supports all 5 quiz modes.

### Multiple Select MCQ (`app/multipleOptSelect.jsx`)

Handles question styles: `medicationUse`, `differentialDiagnosis`.

Multiple options can be selected simultaneously. The question includes a "Select X option(s)" hint. On submit, the selected index array is compared against the `isCorrect` flags on each option object. All selected options must match all correct options exactly.

### Matching (`app/matching.jsx`)

Handles question styles: `matchTheMicrobe`, `match`.

Displays a list of microbe names on the left and empty drop boxes on the right. Treatment buttons sit at the bottom. The user taps a treatment to select it, then taps a drop box to place it. All boxes must be filled before submitting. On submit, each placed treatment is checked against `question.correctMatches`.

### Word Scramble (`app/wordscrambled.jsx`)

Handles question style: `scrabble`.

Displays blank boxes equal to the length of the answer (from route param `answerLength`). Below are letter buttons consisting of the shuffled answer letters plus 4 random decoy letters. The user taps letters to fill blanks in sequence. On submit, each position is compared against the correct letter (case-insensitive).

### Incomplete Process / Flowchart (`app/incompleteProcess.jsx`)

Handles question style: `flowChart`.

Displays a process diagram where some steps contain `{{}}` markers for blanks. The user first taps a blank box to select it, then taps one of the answer choice buttons at the bottom to fill it. The diagram is parsed by splitting on `→` and identifying `{{}}` markers as blanks. On submit, the `answers` array is compared against `question.correctAnswers`.

### Fallback (`app/questionmark.jsx`)

Displayed when a question style maps to no recognized screen. Shows three large question marks. Navigation back only.

---

## 9. Social Features

### Friends

**Screens:** `friends.jsx`, `UserContacts.jsx`

The Friends screen has two tabs: "My Friends" and "Invite Friend".

**My Friends tab:** Loads the user's `friendList` array from `Users/{uid}`, then fetches each friend's doc from the `Users` collection in parallel. Displays friend cards with avatar, username. A long-press or swipe reveals a remove button that batch-updates `friendList` on both sides.

**Invite Friend tab:** User enters a phone number. The app queries `Users` where `phoneNumber == fullPhoneNumber`. Several states are handled:
- User not found: alert shown
- Already friends: alert shown
- Request already sent: alert shown
- Request received from that user: auto-accepts (batch updates `friendList`, removes from `friendRequestsReceived` and `friendRequestsSent` on both sides, sends an acceptance notification)
- New request: batch adds to `friendRequestsSent` on current user and `friendRequestsReceived` on target user, sends a friend request notification

**Add Friend button (bell icon):** Opens a slide-up showing pending incoming friend requests fetched via `useGetInvitations()`. Each request can be accepted (via `useAcceptRequest`) or declined (via `useCancelRequest`).

**UserContacts screen** (`app/UserContacts.jsx`): Requests device contacts permission via `expo-contacts`. Loads all device contacts, queries the `Users` collection to match by phone number, and annotates each contact with a relationship status: `invite`, `received`, `sent`, or `add`. Contacts already in the friend list are filtered out.

**Firestore reads:**
- `Users/{uid}` (friendList, friendRequestsReceived, friendRequestsSent)
- `Users/{friendId}` (for each friend and invitation)
- `Users` where `phoneNumber == X`

**Firestore writes (batch operations):**
- `Users/{uid}.friendList` (arrayUnion / arrayRemove)
- `Users/{uid}.friendRequestsSent` (arrayUnion / arrayRemove)
- `Users/{targetId}.friendRequestsReceived` (arrayUnion / arrayRemove)
- `Notifications/{uid}.notificationsArray` (arrayUnion)

---

## 10. Leaderboards

### Daily Challenge Leaderboard (`app/challengeLeaderboard.jsx`)

Shows all users who completed the current daily challenge, sorted by `lastChallengeScore` descending. Top 3 are displayed in a podium layout with overlapping circles. Positions 4 and below appear in a scrollable list with pagination (15 per page).

**Firestore reads:**
- `dailyChallenge` (to find the current challenge ID)
- `Users` where `lastDailyChallengeID == currentChallengeId`

### Friends Leaderboard (`app/FriendsLeaderboard.jsx`)

Shows the current user and all friends ranked by `totalScore` descending. The current user's entry is marked with "(You)". View-only, no pagination.

**Firestore reads:**
- `Users/{uid}` (to get friendList)
- `Users/{friendId}` (for each friend's totalScore)

---

## 11. Notifications

**Screen:** `app/(tabs)/Notifications.jsx`

Notifications are stored as an array within a single Firestore document per user at `Notifications/{uid}.notificationsArray`. Each notification object contains: `type`, `text`, `timestamp`, `avatars[]`, `read`, `documentId`.

The screen loads notifications via `useGetNotifications()` which reads this document on mount. Notifications are displayed with avatar icons, text, and a formatted timestamp. Pagination shows 15 per page.

Tapping a notification navigates based on type:
- `friendRequestRecieved` (note: misspelled in code): navigates to `friends.jsx` with `openFriendRequests: true`
- `challenge`: navigates to `DisplayChallenges.jsx`

The screen checks for an active subscription or trial before rendering. Non-subscribers see the `NotSubscribed` upsell component.

**Notification types created throughout the app:**
- Friend request sent (via `addFriendRequestNotification`)
- Friend request accepted (via `acceptFriendRequestNotification`)
- Challenge sent (via `addChallengeNotification`)
- Challenge completed (via `addChallengeCompletedNotification`, sent to challenger and mutual friends)

---

## 12. User Profile and Account Management

**Screen:** `app/(tabs)/profile.jsx`

Displays: user avatar (from `avatarId`), username, email, and two stat cards showing total questions attempted (`totalSolved`) and total score (`totalScore`), both read from `Users/{uid}`.

**Membership section:** Shows a different card based on subscription status:
- Active RevenueCat subscription: `InAppCard` component
- Stripe subscription (web): `MembershipCard` component
- Trial period: `TrialMembershipCard` component
- No subscription: upsell prompt linking to `PaymentScreen`

**Account settings:**
- "Contact Support": opens `https://www.matchmakergme.com/contactus` in browser
- "App Version": displays the current app version from `expo-constants`
- "Logout": calls `clearUser()` and `resetSubscription()` in Zustand, calls `Purchases.logOut()` from RevenueCat, navigates to `/onboarding`
- "Delete Account": shows a confirmation alert, then calls `useDeleteUser.handleDeleteUser()`

**Account deletion flow (`useDeleteUser`):**
1. Deletes `Users/{uid}/solved` subcollection (recursive batch delete)
2. Deletes all `Challenges` where `challengerId == uid` or `opponentId == uid`
3. Deletes `Notifications/{uid}` document
4. Deletes `Users/{uid}` document
5. Signs out of Firebase Auth
6. Clears Zustand user state
7. Sends account deletion confirmation email via `POST /api/subservice/account-delete`

The deletion does not cancel an active subscription automatically. The user is directed to the platform's subscription management screen via `openSubscriptionSettings()` (iOS: `App-Prefs:root=STORE`, Android: Play Store).

---

## 13. Subscriptions and Payments

**Screen:** `app/PaymentScreen.jsx`

**Payment provider:** RevenueCat (in-app purchases, iOS and Android).

**RevenueCat configuration:**
- iOS key: `appl_abcd`
- Android key: `goog_abcd`
- RevenueCat is initialized in `app/_layout.jsx` on app startup

**Subscription entitlement:** `ScrubHub Premium`

**Package identifiers offered:**
- `$rc_monthly` (standard monthly)
- `monthly_discount` (discounted monthly)
- `monthly_without_trial` (no trial variant)

**Purchase flow:**

1. User taps "Subscribe Now" on `PaymentScreen.jsx`
2. App calls `Purchases.getOfferings()` to fetch available packages
3. App calls `Purchases.purchasePackage(pkg)` to initiate the purchase
4. On success, RevenueCat entitlements are checked for `ScrubHub Premium`
5. Subscription info is written to `subscription_logs` Firestore collection for diagnostics
6. `paymentStore.updateSubscription()` is called to update global state

**Subscription state (paymentStore):**
- `hasSubscribed`: RevenueCat active entitlement
- `trialPeriod`: trial is active
- `stripeSub`: web-based Stripe subscription (fetched from external backend via `fetchSub(email)`)
- `hasAvailedTrial`: trial was previously used
- `servicesActive`: additional services active

**Subscription status is fetched on app load** via `paymentStore.fetchSubscription()` which calls both `Purchases.getCustomerInfo()` from RevenueCat and `fetchSub(email)` from the external backend.

**Subscription gates throughout the app:**
- Home screen action buttons (study, review, challenge, friends)
- Notifications tab
- Details screen (system buttons)
- Topics screen

**On logout/deletion:** `resetSubscription()` calls `Purchases.logOut()`.

---

## 14. AI Tutor Integration

An AI tutor is available during any question as a hint system.

**Component:** `components/AiTutorChat.jsx`  
**Trigger:** `components/AiHintButton.jsx` on all question screens (via `MCQFooter`)

The chat interface accepts user messages. Each message is sent via `chatBotAPI.askAiTutor(email, userMessage, history, questionContext)`.

**API call:** `POST https://matchmaker-backend-iota.vercel.app/api/ai/ask`

Payload includes:
- `email`: current user's email
- `userMessage`: the typed message
- `history`: previous messages in the conversation
- `questionContext`: the current question object (provides context to the AI)

The backend handles rate limiting; a 429 response is handled and shown to the user.

Each question also has optional fields `question.aiHint` and `question.memoryHook` that are passed as context to the AI tutor footer. These are stored in Firestore question documents at authoring time.

---

## 15. Firebase Integration

**Firebase project:** `scrubhub-001`

**Services used:**
- Firebase Auth (phone OTP sign-in)
- Firestore (all application data)
- Firebase App Check (security attestation, initialized via `util/initializeAppCheck.js`)

**No Firebase Storage or Cloud Functions are used.**

### Firestore Collections

**Users**
- Document ID: user UID
- Key fields: `uid`, `email`, `phoneNumber`, `username`, `avatarId`, `totalScore`, `totalSolved`, `friendList[]`, `friendRequestsSent[]`, `friendRequestsReceived[]`, `lastDailyChallengeID`, `lastChallengeScore`, `solvedTopics[]`, `createdAt`, `hasCompletedOnboarding`
- Subcollection: `solved/{system}/{topic}/solvedIds` (document with `batches[]` array containing `solvedIds[]` arrays)

**Topics**
- Document ID: system name (lowercase, e.g. `cardiovascular`)
- Key fields: `topics[]` array of topic objects with `name` field

**Questions**
- Hierarchical path: `Questions/{system}/{topic}/{questionId}`
- Key fields: `questionStyle`, `questionId`, `timestamp`, question text, answer fields, `aiHint`, `memoryHook`
- Question styles: `quickDiagnosis`, `shortFacts`, `firstLineTreatment`, `lab`, `testToOrder`, `matchTheMicrobe`, `match`, `medicationUse`, `differentialDiagnosis`, `scrabble`, `flowChart`

**dailyChallenge**
- Contains daily challenge documents
- Referenced by ID stored in `Users/{uid}.lastDailyChallengeID`

**Challenges**
- Document ID: auto-generated
- Key fields: `challengerId`, `opponentId`, `challengerScore`, `opponentScore`, `status` (`pending` or `completed`), `questions[]`, `timestamp`

**Notifications**
- Document ID: user UID
- Key fields: `notificationsArray[]` (each item: `type`, `text`, `timestamp`, `avatars[]`, `read`, `documentId`)

**Reports**
- Key fields: `questionId`, `reason`, `userId`, `timestamp`, `resolved: false`
- Written by `util/reportQuestion.js` (called from question screens via `ReportQuestion` component)

**subscription_logs**
- Diagnostic logging for RevenueCat events
- Key fields: `platform`, `context`, `uid`, `email`, `appUserId`, `entitlements`, `isSubscribed`, `error`, `timestamp`

**appcheck_logs**
- Firebase App Check initialization results
- Key fields: `timestamp`, `platform`, `status`, `hasToken`, `error`

### Firebase App Check

`util/initializeAppCheck.js` initializes App Check on startup. Provider selection:
- Android production: `PlayIntegrityProvider`
- Android development: `DebugAppCheckProvider`
- iOS: `AppAttestProvider` with `DeviceCheckProvider` fallback

Result is logged to `appcheck_logs` Firestore collection.

### Firebase Security Rules

No `firestore.rules` file is present in this repository. Security rules must be managed directly in the Firebase console or in a separate configuration repository.

---

## 16. State Management

All global state uses Zustand stores located in `store/`.

**currentUserStore.js** (persisted to AsyncStorage)
- Holds the authenticated user object and UID
- Holds `userNotifications` and `userChallenges` arrays (set by hooks)
- Actions: `setUser`, `updateUser`, `updateUserEmail`, `getUser`, `clearUser`, `setUserNotifications`, `setUserChallenges`

**quesStore.js** (in-memory only)
- Manages all question session state across 5 modes: study, review, daily challenge, friend challenge (opponent side), and challenging friends (challenger side)
- Each mode has its own questions array, current index, and score fields
- Key async actions: `fetchQuestions`, `fetchReviewQuestions`, `fetchChallengeQuestions`, `fetchChallengeFriendQuestions`, `fetchChallengingFriendsQuestions`
- Key submit actions: `submitQuestions`, `submitReviews`, `submitChallengeQuestions`, `submitFriendChallenge`, `submitChallengingFriend`
- Also holds `fetchedQuestionTopic`, `fetchedQuestionSystem`, and their review equivalents for display in question screens

**paymentStore.js** (in-memory only)
- Holds subscription state: `subscription`, `hasSubscribed`, `trialPeriod`, `stripeSub`, `stripeTrialInfo`, `hasAvailedTrial`, `servicesActive`
- Actions: `fetchSubscription`, `updateSubscription`, `resetSubscription`

**quesData.js**
- Exported constant `q` containing 90 hardcoded medical questions about atrial fibrillation
- This data is not fetched from Firestore; it appears to be legacy or seed data used in testing

**PreviousQuesStore.js**
- Appears to be a legacy or alternate store for previous questions; the active version is `quesStore.js`

---

## 17. External Backend

The app calls an external REST API at `https://matchmaker-backend-iota.vercel.app`. This is a separate repository not included here. The following endpoints are used:

| Endpoint | Method | Purpose |
|---|---|---|
| `/api/ai/ask` | POST | AI tutor chat with question context |
| `/api/subservice/check-email` | POST | Validate email before registration |
| `/api/subservice/account-delete` | POST | Send account deletion confirmation email |
| `/api/subservice/registerService` | POST | Register user with subscription service on signup |
| `/api/subservice/fetch-sub` | POST | Fetch Stripe subscription status by email |

The base URL is read from `EXPO_PUBLIC_API_BASE_URL` in the `.env` file.

---

## 18. Known Gaps and Issues

**Firestore security rules absent from repository:** No `firestore.rules` file exists. Rules are managed outside this repo, which makes it impossible to audit access control from the codebase alone.

**`auth-handler.jsx` is a 5-second delay with no logic:** The screen simply waits 5 seconds and redirects to `otpScreen`. There is no actual auth state checking. If this screen is shown during a real auth callback (e.g. a deep link), the fixed delay is fragile.

**Notification type is misspelled in the codebase:** The type `friendRequestRecieved` (misspelled "recieved" instead of "received") is used both when writing notifications (`useAddNotification.jsx`) and when reading them in `Notifications.jsx`. This is consistent and therefore not broken, but any future code checking this type must use the misspelled form.

**`quesData.js` contains 90 hardcoded questions:** These questions are about atrial fibrillation and are not fetched from Firestore. It is unclear whether these are used in production or are leftover seed data. No screen in `app/` imports this file directly; it is imported only by the store. If `quesStore.js` uses it as a fallback, offline behavior may return stale content.

**`PreviousQuesStore.js` appears unused:** The active store for question state is `quesStore.js`. `PreviousQuesStore.js` exists but its usage is not confirmed by any screen or hook import.

**`casemystery.jsx` is a nav-only screen registered in `_layout.jsx`:** It navigates to `fourOptQues` or `details` but no home screen button or tab links to it. The "case mystery" mode is not accessible from the main navigation.

**`review.jsx` references a `reviewall` route:** The "Review All" button navigates to a route named `reviewall`, but no file `app/reviewall.jsx` exists in the repository. Tapping "Review All" will crash with a navigation error.

**`questionmark.jsx` is a dead-end fallback:** When `getQuestionType(questionStyle)` returns an unrecognized type, the app routes to `questionmark.jsx` which only shows three question marks and a back button. No error reporting occurs and no user-facing explanation is shown.

**Account deletion does not cancel subscription:** `useDeleteUser.handleDeleteUser()` deletes all user data from Firestore and Firebase Auth but does not cancel any active RevenueCat or Stripe subscription. The user is shown a prompt to manually go to platform subscription settings, but the subscription itself remains active and billing continues.

**`firebaseauth/link.jsx` does not use Firebase Auth state:** The deep link handler simply redirects to `otpScreen` unconditionally. If a user is already authenticated when this link is opened, they will be sent back through the OTP flow unnecessarily.

**No Firestore pagination in `useGetFriends` or `useGetChallenges`:** Both hooks fetch all documents in a single call. For users with large friend lists or many challenge records, this will become slow and expensive.



