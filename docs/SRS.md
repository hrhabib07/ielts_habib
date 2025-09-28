# 📄 Software Requirement Specification (SRS)

**Project Title:** IELTS HABIB – Learn Better, Score Higher  
**Type:** Web Application (Online IELTS Preparation Platform)  
**Prepared For:** Bangladeshi Students preparing for IELTS

---

## 1. Introduction

### 1.1 Purpose

The purpose of this system is to provide a complete IELTS preparation platform for Bangladeshi students. It will include IELTS modules (Reading, Writing, Listening, Speaking), vocabulary, mock tests, leaderboards, and progress tracking in one place.

### 1.2 Scope

**Roles:** Students, Teachers, Admins.

**MVP Strategy:**

- Use free/low-cost third-party services (Gmail SMTP, Google Drive/YouTube for videos, manual bKash/Bank payments).
- Keep only Chapter 1 free; the rest requires paid enrollment.
- Admins verify payments manually within 72 hours.

**Future upgrades:**

- Paid email APIs
- Secured video CDN
- Online payment gateway (SSLCommerz/Stripe)
- Advanced proctoring

---

## 2. Functional Requirements

### 2.1 Authentication & Authorization

- **FR-1:** Users (Student, Teacher, Admin) can log in/out securely.
- **FR-2:** All users can update passwords.
- **FR-3:** Forgot Password via free email service (SMTP); future → paid email API.
- **FR-4:** Role-Based Access Control (RBAC).
- **FR-5:** Passwords hashed securely.
- **FR-6:** MFA (future, not MVP).

### 2.2 Profile Management

- **FR-7:** Students manage profile (name, contact, picture).
- **FR-8:** Teachers manage profile (bio, qualifications, contact).
- **FR-9:** Admins manage all profiles and activate/deactivate users.

### 2.3 Academic Process

#### Students

- **FR-13:** Course enrollment workflow:
  - Free access: First chapter (with quiz & exam) is free for all students.
  - Paid access: For remaining chapters, students must pay manually (bKash/Bank) and submit transaction ID + payment details via the site.
  - Approval: Admin verifies payment and approves/rejects within 72 hours.
  - Future: Automatic approval after online gateway integration (SSLCommerz/Stripe).
- **FR-14:** Students view quiz/exam marks.
- **FR-15:** Students access the noticeboard.
- **FR-16:** Students view leaderboards.
- **FR-17:** Students track learning streaks.
- **FR-18:** Students access notes inside the platform only (no download in MVP).
- **FR-19:** Students review past quizzes/exams.

#### Teachers

- **FR-20:** Create/manage quizzes & exams.
- **FR-21:** Add chapters.
- **FR-22 (MVP):** Upload video lectures via Google Drive/YouTube links. (Future: Move to secured CDN/video hosting).
- **FR-23:** Grade Writing/Speaking manually.
- **FR-24:** View student profiles & performance.

#### Admins

- **FR-25:** Create/manage teacher accounts.
- **FR-26:** Approve/reject student enrollment requests (including payment verification).
- **FR-27:** Create courses.
- **FR-28:** Assign teachers to courses.
- **FR-29:** Generate reports (student performance, teacher activity, course usage).

### 2.4 User Management

- **FR-30:** Admins block/unblock users.
- **FR-31:** Admins reset passwords.
- **FR-32:** Admins soft-delete or permanently delete accounts.
- **FR-33:** System logs all activity (logins, quizzes, submissions).

### 2.5 Communication & Collaboration

- **FR-34:** Noticeboard for announcements.
- **FR-35 (MVP):** Q&A forum (comments + upvotes/downvotes).
- **FR-36:** Teachers send announcements/messages to enrolled students.
- **FR-37:** Notifications (MVP in-app; future push/email).

### 2.6 Assessments & Progress Tracking

#### Creation & Delivery

- **FR-38:** Support quizzes & exams.
- **FR-39:** Randomize questions/options.
- **FR-40:** Time limits + auto-submit.
- **FR-41:** Attempt limits + retake rules.
- **FR-42:** IELTS skill sections (R/L/W/S).
- **FR-42a:** Enforce access control → free users limited to Chapter 1 quiz/exam.

#### Grading & Feedback

- **FR-43:** Auto-grade objective items.
- **FR-44:** Manual grading with rubrics (Writing/Speaking).
- **FR-45:** IELTS band per skill + overall.
- **FR-46:** Review mode with feedback & timing.

#### User Progress

- **FR-47:** Store all attempts (id, date, device, score, bands).
- **FR-48:** Dashboard (charts, streaks, badges).
- **FR-49:** Analytics (skill/topic/question type).
- **FR-50:** Export results (CSV/PDF).
- **FR-51:** Teachers view student attempt history (their courses).
- **FR-52:** Admins generate cohort reports.

#### Chapter Deadlines & Readmission (New Rules)

- **FR-58 (Per-Chapter Deadline):** Each chapter shall have a teacher-defined deadline (time allowed to complete the chapter). The deadline is assigned when the student unlocks/starts the chapter.
- **FR-59 (Freeze on Missed Deadline):** If the student misses the chapter deadline, that chapter becomes frozen. The student can access previously completed chapters and already unlocked content in the frozen chapter (read-only), but cannot progress further.
- **FR-60 (Readmission):** To continue after a freeze, the student must pay 2% of the course enrollment fee via the manual payment process. After admin approval, the student’s chapter access is reactivated.
- **FR-61 (Deadline Reset on Readmission):** When a student is readmitted, the chapter’s deadline shall be reset from zero, starting fresh from the readmission date.
- **FR-62 (Final Exam Pass Requirement):** To complete a chapter, the student must score ≥ 60% on its final exam. If the score is below 60%, the chapter’s progress is reset and the student must restart the chapter from its first content.

#### Proctoring (future only)

- **FR-53:** Basic proctoring (fullscreen lock, copy-paste block).
- **FR-54:** Log suspicious events.

#### Notifications

- **FR-55:** Notify students after grading.
- **FR-56:** Update streaks + leaderboards after grading.

#### Localization

- **FR-57:** Support English + Bangla UI.

---

## 3. Non-Functional Requirements (NFRs)

- **NFR-1 Performance:** ≥500 concurrent users.
- **NFR-2 Security:** Data encrypted at rest + transit.
- **NFR-3 Scalability:** Scale to 10,000+ users.
- **NFR-4 Usability:** Responsive UI (desktop/mobile).
- **NFR-5 Availability:** ≥99% uptime annually.
- **NFR-6 Maintainability:** Modular updates, no downtime.
- **NFR-7 Reliability:** RTO = 5 min, RPO = 1 min, MTTR ≤5 min (server crash, DB outage).
- **NFR-8 Compliance:** GDPR-like privacy policy.
- **NFR-9 Data Retention:** Store attempts ≥24 months, deletable on request.
- **NFR-10 Privacy:** Attempts visible only to student, assigned teachers, admins.
- **NFR-11 Auditability:** Log lifecycle events (create/start/submit/grade).
- **NFR-12 Reporting Performance:** 50k attempts export ≤10s.
- **NFR-13 Portability:** Export in CSV + PDF.

---

## 4. Role Permissions (Summary Matrix with Free/Paid Access + Deadlines)

| Feature             | Free Student   | Paid Student                                        | Teacher       | Admin            |
| ------------------- | -------------- | --------------------------------------------------- | ------------- | ---------------- |
| Login/Logout        | ✅             | ✅                                                  | ✅            | ✅               |
| Profile Update      | ✅             | ✅                                                  | ✅            | ✅               |
| Course Enrollment   | ✅ (request)   | ✅ (request)                                        | ❌            | ✅ (approve)     |
| Course Access       | ✅ (only Ch.1) | ✅ (all after payment)                              | ❌            | ✅               |
| Attempt Quiz/Exam   | ✅ (Ch.1 only) | ✅ (all, with deadlines)                            | ❌            | ❌               |
| Chapter Deadlines   | ⏳ enforced    | ⏳ enforced                                         | Set deadlines | Monitor/override |
| Readmission         | ❌             | ✅ (manual payment, admin approval, deadline reset) | ❌            | ✅               |
| Add Lecture/Chapter | ❌             | ❌                                                  | ✅            | ✅               |
| Create Quiz/Exam    | ❌             | ❌                                                  | ✅            | ✅               |
| Manage Users        | ❌             | ❌                                                  | ❌            | ✅               |
| Noticeboard Access  | ✅             | ✅                                                  | ✅            | ✅               |
| Leaderboard/Streaks | ✅ (Ch.1 only) | ✅ (Chapter + Course)                               | View only     | ✅               |
| Generate Reports    | ❌             | ❌                                                  | ❌            | ✅               |

---

## 5. MVP vs Future (Cost Strategy)

- **Email:** MVP → Gmail SMTP; Future → SendGrid/SES.
- **Videos:** MVP → Google Drive/YouTube; Future → Secured CDN.
- **Payments:** MVP → Manual bKash/Bank with Admin approval; Future → SSLCommerz/Stripe.
- **Proctoring:** MVP → None; Future → Paid SDK.
- **Notifications:** MVP → In-app alerts; Future → Push + Email + SMS.
- **Free/Paid Access:** MVP → Chapter 1 free for everyone; Future → add trial lessons or promo codes.

---

## 6. Conclusion

This updated SRS defines the MVP version of IELTS HABIB with per-chapter deadlines, freeze/readmission rules, and ≥60% final exam requirements. It emphasizes cost-saving in the MVP while ensuring a clear upgrade path to enterprise-grade solutions. The system is designed to keep students engaged, serious, and motivated while maintaining fairness and flexibility.
