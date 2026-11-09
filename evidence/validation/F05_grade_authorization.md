# F05 — Grade-update authorization: pre-patch failure evidence

CI run (security-regression job, before patching):
https://github.com/Zfrota/cybersecurity_lab_devsecops/actions/runs/42647771982
Job: https://github.com/Zfrota/cybersecurity_lab_devsecops/actions/runs/42647771982/job/129265586897

Source commit under test: `10ca903` ("tests for authentication
vulnerabilities, and also path traverssal vulnerabilities") / `032dfa4`
("Add security tests before patching"). `baseline_secure_coding_api.py` is
still the unpatched baseline at this point — `update_grade()` has no
authentication or authorization check.

Command: `python -m pytest -v tests_security/test_grade_authorization.py`

## Failing assertions and observed behavior

**test_unauthenticated_update_rejected** — requirement violated: R1
(authentication required for grade updates).
```
    response = update(client, grade=20)
    assert response.status_code in (401, 403)
E   assert 200 in (401, 403)
```
Observed: a request with no session cookie at all is accepted and the grade
is applied, instead of being rejected with 401.

**test_student_cannot_update_own_grade** — requirement violated: R3
(students may never update grades, even their own).
```
    login(client, "alice", "alicepw")
    response = update(client, 1001, "SRS", 20)
    assert response.status_code == 403
E   assert 200 == 403
```
Observed: an authenticated student can update any grade, including their
own, with no 403.

**test_professor_cannot_update_unassigned_course** — requirement violated:
R3/R4 (professors scoped to their assigned courses only; role name alone is
not sufficient authorization).
```
    login(client, "prof_srs", "profpw")
    response = update(client, 1001, "SSD", 20)
    assert response.status_code == 403
E   assert 200 == 403
```
Observed: `prof_srs` (assigned course `["SRS"]`) can update a grade for
`"SSD"`, a course they are not assigned to.

**test_rejected_update_does_not_create_record** — requirement violated: R7
(a denied update must leave stored data unchanged).
```
    login(client, "bob", "bobpw")
    update(client, 1002, "SSD", 20)
    assert api.find_grade(1002, "SSD") is None
E   AssertionError: assert {'course_id': 'SSD', 'grade': 20, 'student_id': 1002} is None
```
Observed: since the update was never rejected in the first place (no
authorization check exists), a new grade record was created for a course
`bob` has no legitimate claim to touch.

## Passing (legitimate-access) tests — confirm the test suite itself is sound

- `test_assigned_professor_can_update_grade` — PASSED (professor updating a
  grade in their own assigned course already worked, since the baseline
  endpoint accepts all requests unconditionally).
- `test_admin_can_update_any_grade` — PASSED (same reason).

These two tests already passing on the unpatched baseline, together with the
four failures above, show the failures are genuine missing-authorization
violations and not broken tests or a missing dependency: the suite collects
and runs cleanly, and only the security-relevant assertions fail.
