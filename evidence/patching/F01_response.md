Proposed diff:

```diff
--- a/baseline_secure_coding_api.py
+++ b/baseline_secure_coding_api.py
@@
 @app.get("/files")
 def download_file():
-    raw_path = flask.request.args.get("path", "")
-    target = MATERIALS_DIR / raw_path
-    return flask.send_file(target)
+    if current_user() is None:
+        return {"error": "authentication required"}, 401
+
+    raw_path = flask.request.args.get("path", "")
+    base = MATERIALS_DIR.resolve()
+    target = (base / raw_path).resolve()
+
+    try:
+        target.relative_to(base)
+    except ValueError:
+        return {"error": "file not found"}, 404
+
+    if not target.is_file():
+        return {"error": "file not found"}, 404
+
+    return flask.send_file(target)
```

Control implemented:
Session-based authentication (401 if unauthenticated, reusing the
`current_user()` helper added for F05), followed by a real path-containment
check: both `base` and `target` are resolved to absolute, normalized paths,
and `Path.relative_to()` is used to verify `target` actually sits inside
`base`. This closes both `..`-style traversal and the absolute-path bypass
(pathlib's `/` operator silently discards the left operand when the right
operand is itself absolute), and avoids the classic `startswith()` bug that
a sibling directory like `materials_private/` would defeat.

Assumptions/limitations:
- Scope limited to this finding (F01 path traversal, F02 missing
  authentication on the same endpoint); no other endpoint is touched.
- Returns a generic 404 for both "outside the materials directory" and
  "file does not exist inside it," so the response does not disclose
  whether a given out-of-bounds path exists (R7: no internal path
  disclosure).
- No change to `tests/test_api.py` — it does not exercise `/files`.

Suggested additional regression tests:
- A symlink inside MATERIALS_DIR pointing outside it, to confirm
  `resolve()` + `relative_to()` still rejects the escape.
- A request for a directory path instead of a file, to confirm the
  `is_file()` check returns 404 rather than letting `send_file` error out.

Reviewed and accepted as-is; applied verbatim via Edit to
baseline_secure_coding_api.py.
