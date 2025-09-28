# API.md

> **Contract style:** Minimal, stable REST for the MVP.  
> **Auth:** JSON Web Token (JWT) via `Authorization: Bearer <token>`.  
> **Media type:** `application/json` for all requests/responses.  
> **Dates:** ISO-8601 UTC.  
> **Responses:** `{ ok: boolean, data?: any, error?: { code: string, message: string, details?: any } }`  
> **Pagination:** `?page=1&limit=20` → response includes `meta:{page,limit,total}`.  
> **Id format:** MongoDB ObjectId strings.  
> **Roles:** `student | teacher | admin`.

---

## 0) Common

### Health

`GET /health`

### Me

`GET /me`  
`PATCH /me`

---

## 1) Auth

### POST /auth/register

### POST /auth/login

### POST /auth/forgot

### POST /auth/reset

### POST /auth/logout

---

## 2) Catalog (Courses, Chapters, Contents)

### GET /courses

### GET /courses/:id

### GET /courses/:id/chapters

### GET /courses/:courseId/chapters/:chapterId/contents

### GET /contents/:contentId

### POST /courses (admin)

### PATCH /courses/:id (admin)

### POST /courses/:courseId/chapters (teacher/admin)

### POST /courses/:courseId/chapters/:chapterId/contents (teacher/admin)

### POST /courses/:courseId/chapters/:chapterId/unlock

### POST /contents/:contentId/mark-done

---

## 3) Enrollment & Payments

### POST /enrollments/request

### GET /enrollments/me

### POST /payments

### GET /payments/me

### GET /payments/pending (admin)

### POST /payments/:submissionId/review (admin)

---

## 4) Progress & Readmission

### GET /me/progress/:courseId

### POST /chapters/:chapterId/readmission

### POST /chapters/:chapterId/extend (teacher/admin, optional future)

---

## 5) Assessments & Attempts

### GET /assessments/:id

### GET /chapters/:chapterId/assessments

### POST /assessments/:id/attempts

### POST /attempts/:attemptId/answer

### POST /attempts/:attemptId/submit

### GET /attempts/:attemptId

### POST /attempts/:attemptId/finalize

### Manual / Semi-manual Grading

### GET /grading/tasks (teacher)

### POST /answers/:answerId/evaluate (teacher)

### POST /answers/:answerId/evaluate/accept (teacher)

### GET /attempts/:attemptId/evaluations

---

## 6) Leaderboards

### GET /leaderboards/chapters/:chapterId

### GET /leaderboards/courses/:courseId

---

## 7) Community & Notifications

### GET /threads/:courseId

### POST /threads/:courseId

### GET /threads/:threadId

### POST /threads/:threadId/posts

### POST /posts/:postId/vote

### GET /notifications

### PATCH /notifications/:id/read

### PATCH /notifications/read-all

---

## 8) Audit (Admin)

### GET /audit/logs (admin)

---

## 9) Errors (canonical)

- `AUTH_REQUIRED` (401)
- `FORBIDDEN` (403)
- `NOT_FOUND` (404)
- `VALIDATION_ERROR` (422)
- `CONFLICT` (409)
- `RATE_LIMITED` (429)
- `SERVER_ERROR` (500)

**Example**

```json
{
  "ok": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "chapterId is required",
    "details": { "field": "chapterId" }
  }
}
```

---

## 10) Request/Response Examples

### Enroll → Tuition Payment → Approval

1. `POST /enrollments/request`  
   `{ "courseId": "64f..." }` → `{ ok:true, data:{ enrollmentId, status:"requested" } }`
2. `POST /payments`  
   `{ "courseId":"64f...", "paymentType":"tuition", "amount":5000, "method":"bkash", "transactionId":"TX123" }`  
   → `{ ok:true, data:{ paymentSubmissionId, status:"submitted" } }`
3. (Admin) `POST /payments/:id/review` `{ "decision":"approved" }`  
   → `{ ok:true, data:{ status:"approved" } }` **(enrollment becomes `approved`)**

### Unlock Chapter 2 (gated)

- `POST /courses/:courseId/chapters/:chapterId/unlock`  
  → `{ ok:true, data:{ chapterProgressId, status:"active", deadlineAt:"..." } }`

### Finalize Final Exam

- `POST /attempts/:attemptId/finalize`  
  → `{ ok:true, data:{ status:"graded", scoreTotal:28, bands:{ overall:6.5 } } }`  
  _(auto triggers restart or completion per policy)_

---

## 11) Security Notes

- All mutation endpoints require JWT and role checks.
- Ownership: students can only access/modify their own attempts, payments, progress.
- Teachers can manage only assigned courses/chapters; Admins have full access.
- Sensitive fields (passwordHash, tokens) never returned.

---

## 12) Versioning

- Prefix future breaking changes with `/v2`.
- Current file is **v1 (MVP)**.
