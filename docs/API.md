# API.md

> Minimal, stable contracts for the MVP. All endpoints return { ok: boolean, data?, error? }.

## Auth
### POST /auth/register
### POST /auth/login
### POST /auth/forgot
### POST /auth/reset
### POST /auth/logout

## Catalog
### GET /courses
### GET /courses/:id
### POST /courses (admin)
### POST /courses/:courseId/chapters/:chapterId/unlock

## Enrollment & Payments
### POST /enrollments/request
### POST /payments
### POST /payments/:submissionId/review (admin)

## Progress & Readmission
### GET /me/progress/:courseId
### POST /chapters/:chapterId/readmission

## Assessments & Attempts
### GET /assessments/:id
### POST /assessments/:id/attempts
### POST /attempts/:attemptId/finalize

## Leaderboards
### GET /leaderboards/chapters/:chapterId
### GET /leaderboards/courses/:courseId

## Community & Notifications
### GET /threads/:courseId
### POST /threads/:courseId
### POST /threads/:threadId/posts
### GET /notifications

## Audit
### GET /audit/logs (admin)
