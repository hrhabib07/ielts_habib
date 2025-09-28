# DATA.md

This document defines the **MongoDB-first data model** for **IELTS HABIB**. It is copy‑paste friendly and complete.

---

## 0) Conventions (apply to all)

- **IDs:** `ObjectId`. Foreign keys named `xxxId`.
- **Timestamps:** `createdAt`, `updatedAt` on every doc (set via middleware).
- **Soft delete:** `isDeleted` where relevant (use **partial unique** indexes).
- **Enums:** enforce with Mongo **JSON Schema** validators.
- **RBAC:** enforce server-side using `users.role` + ownership checks.

---

## 1) Auth & Users

### `users`

```json
{
  "_id": "...",
  "role": "student|teacher|admin",
  "email": "a@b.com",
  "emailVerified": true,
  "phoneNumber": "01xxxxxxxxx",
  "phoneVerified": false,
  "passwordHash": "...",
  "dateOfBirth": "2003-05-10",
  "emergencyContact": { "name": "", "relation": "", "phone": "" },
  "address": {
    "line1": "",
    "city": "",
    "district": "",
    "postCode": "",
    "country": "BD"
  },
  "status": "active|blocked|pending",
  "isDeleted": false,
  "mfaEnabled": false,

  /* student-only */
  "targetScore": {
    "overall": 7.0,
    "reading": 7.0,
    "listening": 7.0,
    "writing": 6.5,
    "speaking": 6.5
  },

  /* teacher-only */
  "qualifications": ["CELTA"],
  "bio": "",

  "createdAt": "...",
  "updatedAt": "..."
}
```

**Indexes**

```js
db.users.createIndex(
  { email: 1 },
  { unique: true, partialFilterExpression: { isDeleted: { $ne: true } } }
);
db.users.createIndex({ role: 1, status: 1 });
```

---

## 2) Course Catalog

### `courses`

```json
{
  "_id": "...",
  "name": "IELTS Advanced",
  "difficulty": "beginner|intermediate|advanced",
  "description": "",
  "teacherIds": ["..."],
  "status": "draft|published|archived",
  "accessRules": {
    "freeChapterNumber": 1,
    "downloadAllowed": false
  },
  "createdAt": "...",
  "updatedAt": "..."
}
```

**Indexes**

```js
db.courses.createIndex({ status: 1 });
db.courses.createIndex({ teacherIds: 1 });
```

### `chapters`

```json
{
  "_id": "...",
  "courseId": "...",
  "chapterNumber": 1,
  "title": "Reading Basics",
  "deadlineSpec": {
    "idealDurationHours": 3,
    "allowedTimeMultiple": 3
  },
  "completionPolicy": {
    "requireAllRequiredContents": true,
    "finalExamPassingPct": 60
  },
  "createdAt": "...",
  "updatedAt": "..."
}
```

**Indexes**

```js
db.chapters.createIndex({ courseId: 1, chapterNumber: 1 }, { unique: true });
db.chapters.createIndex({ courseId: 1 });
```

### `contents` (lesson items)

```json
{
  "_id": "...",
  "courseId": "...",
  "chapterId": "...",
  "contentNumber": 1,
  "type": "video|pdf|article|quiz|exam|audio|assignment|live",
  "title": "Skimming & Scanning",
  "required": true,
  "durationSec": 900,
  "resource": { "url": "...", "files": [] },
  "nextContentId": null,
  "prevContentId": null,
  "createdAt": "...",
  "updatedAt": "..."
}
```

**Index**

```js
db.contents.createIndex({ courseId: 1, chapterId: 1, contentNumber: 1 });
```

---

## 3) Enrollment & Access

### `enrollments` (source of truth for paid access)

```json
{
  "_id": "...",
  "studentId": "...",
  "courseId": "...",
  "status": "requested|approved|rejected|completed|dropped",
  "lastActivityAt": "...",
  "streakCount": 5,
  "streakUpdatedAt": "...",
  "bands": {
    "overall": 6.0,
    "reading": 6.5,
    "listening": 6.5,
    "writing": 5.5,
    "speaking": 5.5
  },
  "createdAt": "...",
  "updatedAt": "..."
}
```

**Indexes**

```js
db.enrollments.createIndex({ studentId: 1, courseId: 1 }, { unique: true });
db.enrollments.createIndex({ courseId: 1, status: 1 });
db.enrollments.createIndex({ studentId: 1, status: 1 });
```

**Access logic**

- Free access to Chapter `courses.accessRules.freeChapterNumber` even without enrollment.
- Any other chapter requires `enrollments.status === 'approved'`.

---

## 4) Per-Chapter Lifecycle (Deadlines, Freeze, Restart)

### `chapterProgress` (per student × chapter)

```json
{
  "_id": "...",
  "studentId": "...",
  "courseId": "...",
  "chapterId": "...",

  "status": "locked|active|frozen|completed",
  "unlockedAt": "ISODate",
  "deadlineAt": "ISODate",
  "completedAt": null,

  "requiredContentDonePct": 0,
  "requiredContentDoneFlags": [],

  "finalExam": {
    "assessmentId": null,
    "bestScorePct": null,
    "lastAttemptId": null,
    "passed": false
  },

  "extensions": [],
  "readmission": { "count": 0, "lastAt": null },

  "createdAt": "...",
  "updatedAt": "..."
}
```

**Indexes**

```js
db.chapterProgress.createIndex(
  { studentId: 1, courseId: 1, chapterId: 1 },
  { unique: true }
);
db.chapterProgress.createIndex({ studentId: 1, courseId: 1, status: 1 });
db.chapterProgress.createIndex({ courseId: 1, chapterId: 1, status: 1 });
```

**Rules to enforce in backend**

- **Unlock chapter:** allowed if it’s Chapter 1 or previous chapter’s `status='completed'` and (for non-free chapters) enrollment is approved. On unlock: set `status='active'`, `unlockedAt=now`, `deadlineAt = now + (idealDurationHours * allowedTimeMultiple)`.
- **Freeze chapter:** job checks `now > deadlineAt` for `status='active'` → set `status='frozen'`.
- **Readmission approved:** set `status='active'`, increment `readmission.count`, `readmission.lastAt=now`, and reset `deadlineAt` from zero using the same formula (fresh countdown).
- **Final exam < 60%:** on finalize, reset progress — set `requiredContentDonePct=0`, clear `requiredContentDoneFlags`, set `finalExam.passed=false`, `finalExam.bestScorePct=null`, set `unlockedAt=now`, and reset `deadlineAt` from zero (fresh deadline).
- **Complete chapter:** allowed when `requiredContentDonePct==100` and `finalExam.passed==true` → set `status='completed'`, `completedAt=now`, and unlock next chapter.

---

## 5) Payments (MVP Manual)

### `paymentSubmissions`

```json
{
  "_id": "...",
  "studentId": "...",
  "courseId": "...",
  "chapterId": null,
  "paymentType": "tuition|readmission",
  "amount": 0,
  "method": "bkash|bank",
  "transactionId": "TX123",
  "senderInfo": { "name": "", "phone": "" },
  "screenshotUrl": "https://...",
  "status": "submitted|approved|rejected",
  "reviewedBy": null,
  "reviewedAt": null,
  "submittedAt": "...",
  "createdAt": "...",
  "updatedAt": "..."
}
```

**Indexes**

```js
db.paymentSubmissions.createIndex({ courseId: 1, status: 1, submittedAt: -1 });
db.paymentSubmissions.createIndex({
  studentId: 1,
  courseId: 1,
  submittedAt: -1,
});
db.paymentSubmissions.createIndex({ transactionId: 1 });
db.paymentSubmissions.createIndex({
  studentId: 1,
  courseId: 1,
  chapterId: 1,
  paymentType: 1,
  status: 1,
});
```

**Flows**

- **Tuition (enroll):** on approval → upsert `enrollments` to `approved`.
- **Readmission:** on approval → reactivate corresponding `chapterProgress` and reset deadline.

---

## 6) Assessments

### `assessments`

```json
{
  "_id": "...",
  "courseId": "...",
  "chapterId": "...",
  "type": "quiz|exam",
  "isFinalExam": false,
  "title": "Chapter 1 Final Exam",
  "timeLimitSec": 1200,
  "attemptLimit": 2,
  "settings": {
    "randomizeQuestions": true,
    "randomizeOptions": true,
    "startAt": null,
    "endAt": null,
    "showReview": "never|afterSubmit|afterDeadline"
  },
  "ieltsSkills": ["reading"],
  "createdBy": "teacherId",
  "createdAt": "...",
  "updatedAt": "..."
}
```

**Indexes**

```js
db.assessments.createIndex({ courseId: 1, chapterId: 1, type: 1 });
db.assessments.createIndex({ chapterId: 1, isFinalExam: 1 });
```

### `questions`

```json
{
  "_id": "...",
  "assessmentId": "...",
  "type": "mcq|tf|match|fill|short|essay|speaking",
  "stem": "Which paragraph mentions ...?",
  "options": ["A", "B", "C", "D"],
  "answerKey": { "correct": "B" },
  "tags": ["matching headings", "section3"],
  "points": 1
}
```

**Index**

```js
db.questions.createIndex({ assessmentId: 1 });
```

---

## 7) Attempts, Answers, Manual/Semi-Manual Grading

### `attempts`

```json
{
  "_id": "...",
  "assessmentId": "...",
  "courseId": "...",
  "chapterId": "...",
  "studentId": "...",
  "status": "started|submitted|grading|graded",
  "startedAt": "...",
  "submittedAt": "...",
  "gradedAt": "...",
  "timeSpentSec": 1180,
  "autoScore": 24,
  "manualScore": 6,
  "scoreTotal": 30,
  "bands": { "overall": 6.5, "reading": 7.0 }
}
```

**Indexes**

```js
db.attempts.createIndex({ studentId: 1, assessmentId: 1, createdAt: -1 });
db.attempts.createIndex({ courseId: 1, chapterId: 1, status: 1 });
```

### `answers`

```json
{
  "_id": "...",
  "attemptId": "...",
  "questionId": "...",
  "response": "B",
  "isCorrect": true,
  "score": 1,
  "gradingStatus": "pending|ai_suggested|teacher_required|finalized",
  "finalEvaluationId": "...",
  "timeSpentSec": 22,
  "createdAt": "...",
  "updatedAt": "..."
}
```

**Indexes**

```js
db.answers.createIndex({ attemptId: 1, questionId: 1 }, { unique: true });
db.answers.createIndex({ gradingStatus: 1 });
```

### `artifacts` (essay/audio/etc.)

```json
{
  "_id": "...",
  "attemptId": "...",
  "questionId": "...",
  "type": "text|audio|file",
  "text": "Student essay...",
  "audio": { "url": "...", "durationSec": 265 },
  "files": [{ "name": "task2.pdf", "url": "..." }],
  "createdAt": "...",
  "updatedAt": "..."
}
```

**Index**

```js
db.artifacts.createIndex({ attemptId: 1, questionId: 1 });
```

### `evaluations` (manual & semi-manual unified)

```json
{
  "_id": "...",
  "attemptId": "...",
  "answerId": "...",
  "questionId": "...",

  "source": "ai|teacher",
  "status": "suggested|submitted|finalized",

  "rubricId": null,
  "rubricScores": [
    { "criterionId": "taskResponse", "score": 6.0 },
    { "criterionId": "cohesion", "score": 6.5 },
    { "criterionId": "lexical", "score": 6.0 },
    { "criterionId": "grammar", "score": 5.5 }
  ],
  "score": 6.0,
  "band": 6.0,

  "match": {
    "isMatch": true,
    "confidence": 0.92,
    "method": "keyword|semantic|regex",
    "details": []
  },

  "feedback": "Clear position; work on cohesion.",
  "suggestedBy": "rule-engine|gpt-xx",
  "teacherId": "…",

  "decision": "pending|accepted|rejected|edited",
  "decidedBy": "…",
  "decidedAt": "...",

  "createdAt": "...",
  "updatedAt": "..."
}
```

**Indexes**

```js
db.evaluations.createIndex({ answerId: 1, source: 1 });
db.evaluations.createIndex({ attemptId: 1, questionId: 1, status: 1 });
```

### `gradingTasks` (teacher queue)

```json
{
  "_id": "...",
  "attemptId": "...",
  "answerId": "...",
  "questionId": "...",
  "assigneeId": "teacherId|null",
  "priority": "normal|high",
  "status": "open|in_review|done",
  "dueAt": null,
  "createdAt": "...",
  "updatedAt": "..."
}
```

**Index**

```js
db.gradingTasks.createIndex({ assigneeId: 1, status: 1, priority: 1 });
```

**Critical business hook (final exam rule)**

- On `attempts.status → graded` for a final exam:
  - Compute `pct = scoreTotal / maxScore * 100`.
  - Update `chapterProgress.finalExam` (bestScorePct, passed, lastAttemptId).
  - If `pct < finalExamPassingPct (60)` → restart chapter: reset progress flags + reset deadline from now.

---

## 8) Leaderboards (Chapter & Course)

### `chapterLeaderboards`

```json
{
  "_id": "...",
  "courseId": "...",
  "chapterId": "...",
  "period": "allTime",
  "rankings": [{ "studentId": "...", "points": 240, "rank": 1 }],
  "generatedAt": "...",
  "scoringVersion": 1
}
```

**Index**

```js
db.chapterLeaderboards.createIndex(
  { courseId: 1, chapterId: 1, period: 1 },
  { unique: true }
);
```

### `courseLeaderboards`

```json
{
  "_id": "...",
  "courseId": "...",
  "period": "allTime",
  "rankings": [{ "studentId": "...", "points": 980, "rank": 1 }],
  "breakdown": { "chapterIds": ["..."], "students": { "S1": { "ch1": 240 } } },
  "generatedAt": "...",
  "scoringVersion": 1
}
```

**Index**

```js
db.courseLeaderboards.createIndex({ courseId: 1, period: 1 }, { unique: true });
```

> Rebuild chapter LB when a chapter final/attempt is graded; roll up to course LB.

---

## 9) Community & Notifications (MVP)

### `threads`

```json
{
  "_id": "...",
  "courseId": "...",
  "chapterId": "...",
  "title": "...",
  "authorId": "...",
  "createdAt": "...",
  "updatedAt": "..."
}
```

### `posts`

```json
{
  "_id": "...",
  "threadId": "...",
  "authorId": "...",
  "content": "...",
  "parentPostId": null,
  "votes": { "up": 0, "down": 0 },
  "createdAt": "...",
  "updatedAt": "..."
}
```

**Indexes**

```js
db.posts.createIndex({ threadId: 1, createdAt: -1 });
db.posts.createIndex({ authorId: 1, createdAt: -1 });
```

### `notifications`

```json
{
  "_id": "...",
  "userId": "...",
  "type": "grade_released|enrollment_approved|deadline_soon|frozen|readmission_approved|announcement",
  "title": "Your Writing Task graded",
  "body": "Band: 6.0. See feedback.",
  "isRead": false,
  "createdAt": "...",
  "updatedAt": "..."
}
```

**Index**

```js
db.notifications.createIndex({ userId: 1, isRead: 1, createdAt: -1 });
```

---

## 10) Audit

### `auditLogs`

```json
{
  "_id": "...",
  "actorId": "...",
  "action": "GRADE_UPDATE|ROLE_CHANGE|PAYMENT_REVIEW|CHAPTER_FREEZE|CHAPTER_READMISSION|CHAPTER_RESTART",
  "entity": {
    "collection": "answers|chapterProgress|paymentSubmissions|...",
    "id": "..."
  },
  "meta": { "from": {}, "to": {} },
  "createdAt": "..."
}
```

**Indexes**

```js
db.auditLogs.createIndex({ action: 1, createdAt: -1 });
db.auditLogs.createIndex({ actorId: 1, createdAt: -1 });
```

---

## Validators (examples)

### `chapterProgress`

```js
db.runCommand({
  collMod: "chapterProgress",
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: [
        "studentId",
        "courseId",
        "chapterId",
        "status",
        "unlockedAt",
        "deadlineAt",
        "createdAt",
        "updatedAt",
      ],
      properties: {
        status: { enum: ["locked", "active", "frozen", "completed"] },
        requiredContentDonePct: {
          bsonType: ["double", "int"],
          minimum: 0,
          maximum: 100,
        },
      },
    },
  },
});
```

### `paymentSubmissions`

```js
db.runCommand({
  collMod: "paymentSubmissions",
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: [
        "studentId",
        "courseId",
        "paymentType",
        "method",
        "transactionId",
        "status",
        "submittedAt",
        "createdAt",
        "updatedAt",
      ],
      properties: {
        paymentType: { enum: ["tuition", "readmission"] },
        method: { enum: ["bkash", "bank"] },
        status: { enum: ["submitted", "approved", "rejected"] },
      },
    },
  },
});
```

---

## Key Backend Rules (succinct)

**Unlock Chapter**

- Preconditions: enrollment (approved) for non-free chapter, previous chapter completed.
- Actions: create/update `chapterProgress` to `active`; set `unlockedAt=now`; `deadlineAt = now + (idealDurationHours * allowedTimeMultiple)`.

**Start New Content / Attempt**

- Allow only if `chapterProgress.status='active'` and `now <= deadlineAt`.
- If `status='frozen'`: allow **read-only** for unlocked items; block new progress.

**Deadline Job (hourly/daily)**

- For `chapterProgress.status='active' && deadlineAt < now` → set `status='frozen'`, notify student.

**Readmission (2% fee)**

- Student submits `paymentSubmissions { paymentType:'readmission', chapterId }`.
- Admin approves → set `chapterProgress.status='active'`, increment `readmission.count`, `readmission.lastAt=now`, **reset** `deadlineAt = now + (idealDurationHours * allowedTimeMultiple)`.

**Final Exam Rule (≥60%)**

- On final exam graded: compute `%`.
- If `< 60%` → **restart**: zero progress flags and reset deadline from now; keep `chapterProgress.status='active'`.
- If `≥ 60%` **and** required contents done → mark `completed`, unlock next chapter.

**Enrollment Completion**

- When last chapter completed → optional set `enrollments.status='completed'`.
