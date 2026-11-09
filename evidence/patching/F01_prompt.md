We are now repairing a confirmed vulnerability in the supplied application.

Confirmed finding:
GET /files in baseline_secure_coding_api.py (function download_file, lines
144-148) builds the target path as `MATERIALS_DIR / raw_path`, where
raw_path comes directly from the unauthenticated query parameter `path`,
and passes it to flask.send_file with no normalization or containment
check. This allows both (a) reading files outside MATERIALS_DIR via `..`
segments or an absolute path, and (b) downloading any material without
being logged in at all, since the endpoint has no session check.

Reproduction (tests_security/test_file_access.py, run via
`python -m pytest -v tests_security`):
- test_unauthenticated_download_rejected: GET /files?path=notes.txt with no
  session cookie; expects 401 or 403. Currently fails, returns 200 with the
  file content.
- test_parent_traversal_blocked (parametrized): paths "../secret.txt",
  "sub/../../secret.txt", "../materials_private/x.txt" while authenticated;
  expects 400/403/404 and the outside file's content never returned.
  Currently fails, the outside file's content is returned.
- test_absolute_path_blocked: an absolute OS path to a file outside
  MATERIALS_DIR; same expectation. Currently fails because pathlib's `/`
  operator discards the left operand when the right operand is absolute.
- test_traversal_blocked_for_unauthenticated_too: unauthenticated request
  with a traversal path must never leak the outside file's content.
- test_authenticated_user_can_download_material must keep passing
  (legitimate access to files inside MATERIALS_DIR, including subfolders).

Required behavior:
R1 — Authentication: material downloads require an authenticated user.
R6 — File access: authenticated users may download teaching materials
within the materials directory; requests must not read files outside that
directory.
R7 — Error handling: unauthenticated protected requests return 401; invalid
requests return an appropriate client error without exposing internal
paths or implementation details.

Legitimate use cases that must keep working:
- Any authenticated user downloading a file that exists inside
  MATERIALS_DIR, including in a subdirectory.

Propose a minimal, reviewable patch addressing the root cause (i.e. a real
containment check, not a string prefix/startswith comparison, which the
"materials_private" sibling-directory test is specifically designed to
catch).
Preserve the API's legitimate functionality and relevant response contracts.
Do not delete endpoints, disable features, weaken tests, or suppress scanner
rules to hide the issue.

Return a unified diff, explain the control being implemented, and identify
remaining assumptions or limitations. Suggest additional regression tests.
