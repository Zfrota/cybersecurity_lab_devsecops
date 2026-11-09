Proposed diff:

```diff
--- a/baseline_secure_coding_api.py
+++ b/baseline_secure_coding_api.py
@@
+def current_user():
+    username = flask.session.get("user")
+    return USERS.get(username) if username else None
+
+
 @app.post("/grades/update")
 def update_grade():
+    user = current_user()
+    if user is None:
+        return {"error": "authentication required"}, 401
+
     data = flask.request.get_json()
 
     student_id = data["student_id"]
     course_id = data["course_id"]
     grade = data["grade"]
 
+    if user.role == "student":
+        return {"error": "forbidden"}, 403
+    if user.role == "professor" and course_id not in (user.courses or []):
+        return {"error": "forbidden"}, 403
+    # admin: no additional restriction
+
     row = find_grade(student_id, course_id)
     if row is None:
         GRADES.append({"student_id": student_id, "course_id": course_id, "grade": grade})
     else:
         row["grade"] = grade
 
     return {"status": "ok", "student_id": student_id, "course_id": course_id, "grade": grade}
```

Control implemented:
Session-based authentication (401 if no `flask.session["user"]`) followed by
role/course authorization, checked on every request rather than relying on
login alone: students are always forbidden from updating grades, professors
are restricted to the courses listed in their `User.courses`, and admins are
unrestricted. The checks run before any mutation of `GRADES`, so a denied
request leaves stored data unchanged.

Assumptions/limitations:
- Scope limited to this finding (F05); `get_profile`/`get_grades` (BOLA) and
  grade-update input validation (R5: student existence, known course,
  numeric 0-20 range, rejecting booleans) are separate findings and are
  intentionally left untouched here.
- `current_user()` is a new shared helper; it is reused by the F01/F02 file-
  access patch.
- No change to `tests/test_api.py` — it does not exercise `/grades/update`.

Suggested additional regression tests:
- A professor attempting to update a grade for a student not yet in
  STUDENT_PROFILES/GRADES, to confirm the course-scope check still applies
  when creating a new record via the append branch.
- A request with a valid role but a missing/empty session (e.g. after
  logout) to confirm 401 rather than leaking role state from a stale client.

Reviewed and accepted as-is; applied verbatim via Edit to
baseline_secure_coding_api.py.
