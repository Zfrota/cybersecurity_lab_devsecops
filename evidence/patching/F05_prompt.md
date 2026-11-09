We are now repairing a confirmed vulnerability in the supplied application.

Confirmed finding:
POST /grades/update in baseline_secure_coding_api.py (function update_grade,
lines 128-142) performs no authentication check and no role/course
authorization check before mutating the in-memory GRADES list. Any request —
authenticated or not, regardless of role — can create or overwrite any
student's grade for any course.

Reproduction (tests_security/test_grade_authorization.py, run via
`python -m pytest -v tests_security`):
- test_unauthenticated_update_rejected: sends POST /grades/update with no
  session cookie; expects 401 or 403 and the stored grade unchanged. Currently
  fails because the endpoint returns 200 and applies the update.
- test_student_cannot_update_own_grade: logs in as student "alice" and
  attempts to update her own grade; expects 403 (students may never update
  grades). Currently fails because the endpoint returns 200.
- test_professor_cannot_update_unassigned_course: logs in as "prof_srs"
  (assigned course ["SRS"]) and attempts to update a grade for course "SSD";
  expects 403. Currently fails because the endpoint returns 200.
- test_rejected_update_does_not_create_record: a denied update must not
  create a new GRADES entry.
- test_assigned_professor_can_update_grade / test_admin_can_update_any_grade
  must keep passing (legitimate access).

Required behavior:
R1 — Authentication: grade updates require an authenticated user.
R3 — Academic staff scope: professors may update grades only for their
assigned courses; in this exercise students may never update grades;
administrators may update any grade.
R4 — Complete authorization: check authorization on every request; a role
name or login alone does not establish permission for a specific course.
R7 — Error handling: unauthenticated protected requests return 401,
authenticated-but-unauthorized requests return 403; a denied update must
leave stored data unchanged, without exposing internal details.

Legitimate use cases that must keep working:
- Professor "prof_srs" (courses=["SRS"]) updating a grade for course "SRS".
- Administrator "admin1" updating a grade for any student/course.

Propose a minimal, reviewable patch addressing the root cause.
Preserve the API's legitimate functionality and relevant response contracts.
Do not delete endpoints, disable features, weaken tests, or suppress scanner
rules to hide the issue. Identify all affected paths that need the same
control.

Return a unified diff, explain the control being implemented, and identify
remaining assumptions or limitations. Suggest additional regression tests.
