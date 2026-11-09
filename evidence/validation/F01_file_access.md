# F01 — File-access path traversal / missing authentication: pre-patch failure evidence

CI run (security-regression job, before patching):
https://github.com/Zfrota/cybersecurity_lab_devsecops/actions/runs/42647771982
Job: https://github.com/Zfrota/cybersecurity_lab_devsecops/actions/runs/42647771982/job/129265586897

Source commit under test: `10ca903` ("tests for authentication
vulnerabilities, and also path traverssal vulnerabilities") / `032dfa4`
("Add security tests before patching"). `baseline_secure_coding_api.py` is
still the unpatched baseline at this point — `download_file()` has no
authentication check and builds the target path with an unvalidated
`MATERIALS_DIR / raw_path` concatenation.

Command: `python -m pytest -v tests_security/test_file_access.py`

## Failing assertions and observed behavior

**test_unauthenticated_download_rejected** — requirement violated: R1
(material downloads require an authenticated user).
```
    response = get(client, "notes.txt")
    assert response.status_code in (401, 403)
E   assert 200 in (401, 403)
```
Observed: a request with no session cookie still downloads the file
successfully.

**test_parent_traversal_blocked** (parametrized: `../secret.txt`,
`sub/../../secret.txt`, `../materials_private/x.txt`) — requirement
violated: R6 (requests must not read files outside the materials
directory).
```
    response = get(client, path)
    assert response.status_code in (400, 403, 404)
E   assert 200 in (400, 403, 404)
```
Observed: all three traversal variants return 200 and leak the contents of
`secret.txt` (a file outside `MATERIALS_DIR`), including the
`materials_private` sibling-directory variant, which specifically shows a
naive `startswith("materials")` style check would not have been sufficient
either.

**test_absolute_path_blocked** — requirement violated: R6.
```
    response = get(client, str(materials / "secret.txt"))
    assert response.status_code in (400, 403, 404)
E   assert 200 in (400, 403, 404)
```
Observed: supplying an absolute filesystem path as `path` also escapes
`MATERIALS_DIR`, because pathlib's `/` operator discards the left operand
entirely when the right operand is itself absolute — a second, distinct
bypass of the same control.

**test_traversal_blocked_for_unauthenticated_too** — requirement violated:
R1 and R6 together (the secret must never be disclosed, authenticated or
not).
```
    response = get(client, "../secret.txt")
    assert SECRET.encode() not in response.data
E   AssertionError: assert b'TOP-SECRET-OUTSIDE-CONTENT' not in b'TOP-SECRET-OUTSIDE-CONTENT'
```
Observed: the traversal succeeds even without logging in at all, since
there is no authentication check on this endpoint either.

## Passing (legitimate-access) test — confirms the test suite itself is sound

- `test_authenticated_user_can_download_material` — PASSED (an authenticated
  user downloading `notes.txt` and `sub/deep.txt`, both inside
  `MATERIALS_DIR`, already worked on the unpatched baseline).

This test passing alongside the five failures above shows the suite
collects and runs correctly, and the failures reflect real violations of R1
and R6, not a broken test or a missing dependency.
