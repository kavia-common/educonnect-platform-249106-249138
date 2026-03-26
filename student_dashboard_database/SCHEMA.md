# Student Dashboard Database Schema (PostgreSQL)

This database supports the EduConnect student dashboard (courses/classes, assignments/tasks, grades/progress, announcements/notifications, and timetable/calendar).

> Connection info is stored in `db_connection.txt` (per container convention).

## Extensions

- `pgcrypto` is enabled to provide `gen_random_uuid()` for UUID primary keys.

## Entity Overview

### `users`
Core identity table for all roles.

Columns:
- `id` UUID PK, default `gen_random_uuid()`
- `email` unique
- `full_name`
- `role` enum-like text: `student | teacher | admin`
- `avatar_url` (optional)
- `created_at`

### `students`
Student profile, 1:1 with `users` for role `student`.

Columns:
- `id` UUID PK
- `user_id` UUID unique FK → `users(id)` (CASCADE)
- `student_number` unique (optional)
- `grade_level` int (optional)
- `created_at`

### `courses`
Course/class entity.

Columns:
- `id` UUID PK
- `code` unique (e.g., `MATH101`)
- `title`
- `description` (optional)
- `teacher_user_id` UUID FK → `users(id)` (SET NULL)
- `start_date`, `end_date` (optional)
- `created_at`

### `enrollments`
Join table between `students` and `courses`.

Columns:
- `id` UUID PK
- `course_id` UUID FK → `courses(id)` (CASCADE)
- `student_id` UUID FK → `students(id)` (CASCADE)
- `status` enum-like text: `active | dropped | completed`
- `enrolled_at`
- Unique constraint: `(course_id, student_id)`

### `assignments`
Assignments/tasks within a course.

Columns:
- `id` UUID PK
- `course_id` UUID FK → `courses(id)` (CASCADE)
- `title`
- `description` (optional)
- `due_at` timestamptz (optional)
- `max_points` numeric(6,2), default 100
- `created_at`

### `submissions`
Per-student submission state for an assignment.

Columns:
- `id` UUID PK
- `assignment_id` UUID FK → `assignments(id)` (CASCADE)
- `student_id` UUID FK → `students(id)` (CASCADE)
- `status` enum-like text: `not_submitted | submitted | graded | late`
- `submitted_at` timestamptz (optional)
- `content` (optional)
- `created_at`
- Unique constraint: `(assignment_id, student_id)`

### `grades`
Grade for a submission (1:1).

Columns:
- `id` UUID PK
- `submission_id` UUID unique FK → `submissions(id)` (CASCADE)
- `points` numeric(6,2)
- `feedback` (optional)
- `graded_by_user_id` UUID FK → `users(id)` (SET NULL)
- `graded_at`

### `announcements`
Announcements for a course.

Columns:
- `id` UUID PK
- `course_id` UUID FK → `courses(id)` (CASCADE, nullable to allow global announcements later if needed)
- `title`
- `body`
- `published_at` default `now()`
- `created_by_user_id` UUID FK → `users(id)` (SET NULL)

### `notifications`
User notifications (for announcement/task/grade events etc.).

Columns:
- `id` UUID PK
- `user_id` UUID FK → `users(id)` (CASCADE)
- `type`
- `title`
- `body` (optional)
- `is_read` boolean default false
- `created_at`

### `timetable_events`
Calendar/timetable events (lectures, exams, etc.) optionally tied to a course.

Columns:
- `id` UUID PK
- `course_id` UUID FK → `courses(id)` (CASCADE)
- `title`
- `starts_at`, `ends_at` (constraint: ends_at > starts_at)
- `location` (optional)
- `recurrence_rule` (optional, future)
- `created_at`

## Indexes (non-exhaustive)

- `students(user_id)`
- `enrollments(student_id)`, `enrollments(course_id)`
- `assignments(course_id)`
- `submissions(student_id)`
- `announcements(course_id)`
- `notifications(user_id, created_at desc)`

## Seed Data (minimal)

Seeded records (id values are generated UUIDs):

- Users:
  - `teacher1@educonnect.test` (role: `teacher`)
  - `student1@educonnect.test` (role: `student`)
- Student profile:
  - `student_number`: `S-0001`, `grade_level`: 10
- Courses:
  - `MATH101` — Mathematics 101
  - `ENG201` — English Literature
- Enrollments:
  - Student `S-0001` enrolled in both courses
- Assignments:
  - MATH101: Homework 1 (due in ~3 days)
  - ENG201: Essay Draft (due in ~7 days)
- Announcements:
  - Welcome announcement for MATH101
- Notifications:
  - One sample notification for `student1@educonnect.test`
- Timetable:
  - One lecture event for MATH101

These are intended to keep the UI/dashboard non-empty and to support upcoming backend endpoints.

## Applied changes log (Step 01.01)

All statements were executed one-at-a-time using the connection command in `db_connection.txt`:

`psql postgresql://appuser:dbuser123@localhost:5000/myapp -c "<ONE SQL STATEMENT>"`

### DDL applied

- `CREATE EXTENSION IF NOT EXISTS pgcrypto;`
- Tables created/ensured:
  - `users`
  - `students`
  - `courses`
  - `enrollments`
  - `assignments`
  - `submissions`
  - `grades`
  - `announcements`
  - `notifications`
  - `timetable_events`
- Indexes created/ensured:
  - `idx_students_user_id` on `students(user_id)`
  - `idx_enrollments_student_id` on `enrollments(student_id)`
  - `idx_enrollments_course_id` on `enrollments(course_id)`
  - `idx_assignments_course_id` on `assignments(course_id)`
  - `idx_submissions_student_id` on `submissions(student_id)`
  - `idx_announcements_course_id` on `announcements(course_id)`
  - `idx_notifications_user_created_at` on `notifications(user_id, created_at DESC)`

### Seed data applied

Seed statements were executed as individual `INSERT` statements (users/courses/enrollments use `ON CONFLICT` upserts where a suitable unique constraint exists):

- Users upserted by `email`:
  - `teacher1@educonnect.test` (Teacher One, `teacher`)
  - `student1@educonnect.test` (Student One, `student`)
- Student profile upserted by `user_id`:
  - `S-0001`, grade level `10`
- Courses upserted by `code`:
  - `MATH101`, `ENG201`
- Enrollments upserted by `(course_id, student_id)`:
  - Student `S-0001` enrolled in both courses
- Sample rows inserted:
  - One assignment per course
  - One announcement for `MATH101`
  - One notification for `student1@educonnect.test`
  - One timetable event for `MATH101`

Notes:
- If this database already had prior seed rows, non-upsert inserts (e.g., assignments/announcements/notifications/timetable_events) can add additional rows because there is no natural unique constraint on those entities.
