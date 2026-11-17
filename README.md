# VAPT CA — OWASP Juice Shop: six findings, before and after

Course-work repository for a Vulnerability Assessment and Penetration Testing CA. It holds a
local OWASP Juice Shop instance at two points in time — unpatched and patched — so that each
finding can be reproduced, then re-tested against the fix.

The target is [OWASP Juice Shop](https://github.com/juice-shop/juice-shop), an intentionally
vulnerable application published for training. All testing was authorised, local to a single
machine, against throwaway data.

## Branches

| Branch | What it is |
| --- | --- |
| `main` | This README, the collected evidence, and the six patch files. No application code. |
| `vulnerable` | Juice Shop 20.4.0-SNAPSHOT as published upstream, plus a loopback bind. The "before" baseline. |
| `patched` | The same tree with six fixes applied, one commit per finding. The "after" state. |

The whole change set is one diff:

```bash
git diff vulnerable..patched
```

Five files, 49 insertions, 17 deletions. A copy is in [`evidence/full.diff`](evidence/full.diff)
in case you would rather read it than check the repo out.

## Running it

Juice Shop's own README lists Node 22, 24 and 26 as supported; this was built and tested on
Node v26.13.0, macOS 26.6.2, arm64.

```bash
git clone https://github.com/exploreanand/vapt_lab_ca_owasp_juice_shop.git
cd vapt_lab_ca_owasp_juice_shop

git switch vulnerable      # or: git switch patched
npm install                # several minutes; also builds the Angular front end
npm start
```

Then open <http://localhost:3000>.

### Switching between the two branches

**`npm start` does not compile anything** — it runs `node build/app`, the output of a previous
build. The fixes for F1–F6 are TypeScript, so switching branches without rebuilding runs the
*other* branch's compiled code and every test gives a misleading result. After any switch:

```bash
git switch patched         # or vulnerable
npm run build:server       # must exit 0
npm start
```

`npm install` only needs re-running if `package.json` changed, which it does not here.

Two further things worth knowing before testing:

- **The database is rebuilt from scratch on every restart.** Accounts, baskets and product
  edits do not survive a restart, so create any test accounts you need after starting the
  server and finish a round of testing in one sitting.
- **The server binds to `127.0.0.1`** ([`server.ts`](https://github.com/exploreanand/vapt_lab_ca_owasp_juice_shop/blob/patched/server.ts), the `server.listen` call), so it is reachable
  only from the machine running it. This was deliberate: an intentionally vulnerable
  application should not be exposed to a shared network. If you need to reach it from another
  device, drop the `'127.0.0.1'` argument and accept that anyone on your network can reach it
  too.

## The six findings

Each row is one finding, one commit, and one pair of figures in the report. The attack steps
themselves follow the official
[Juice Shop companion guide](https://pwning.owasp-juice.shop/companion-guide/latest/part2/README.html);
what is listed here is the observable outcome that distinguishes the two branches.

| # | Finding | How to tell the branches apart | On `vulnerable` | On `patched` |
| --- | --- | --- | --- | --- |
| F1 | Mass assignment on registration | `POST /api/Users` with a `role` field added to the JSON body | Account is created with the submitted role; the administration page opens | `role` is forced to `customer`; privileged fields are stripped |
| F2 | SQL injection in product search | `GET /rest/products/search?q=` with a single quote as the term, then a `UNION` term per the guide | Raw SQLite error, and crafted terms return user records | Term is bound as a parameter; no error, no user data, normal search unaffected |
| F3 | Stored XSS + unauthenticated product update | `PUT /api/Products/:id` with no token, then a script payload in `description` | Update succeeds without a token; payload is stored and executes on page load | `401` without a token; description is sanitised with a token |
| F4 | Weak admin password, no login throttling | Log in as `admin@juice-sh.op`; then repeat failed logins | Default password `admin123` works; failed attempts are never throttled | Default password is gone; the 11th attempt in 5 minutes returns `429` |
| F5 | Unvalidated basket quantity | `PUT /api/BasketItems/:id` with a negative or fractional `quantity` | Accepted with `200`; the basket total goes negative | `400` with a validation message |
| F6 | Unauthenticated metrics exposure | `GET /metrics` with no token | `200` and the full Prometheus metrics | `401 Unauthorized` |

### Two checks that need no tooling

```bash
# F2 — normal search still works on both branches
curl -s "http://localhost:3000/rest/products/search?q=apple"

# F6 — the difference is visible in the status code alone
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:3000/metrics
#   vulnerable -> 200
#   patched    -> 401
```

## What each fix does

| Commit | File | Change |
| --- | --- | --- |
| F1 | `server.ts` | Middleware ahead of the `POST /api/Users` handler forces `role` to `customer` and deletes `deluxeToken`, `totpSecret`, `isActive` and `lastLoginIp`, so the client cannot set privileged fields. |
| F2 | `routes/search.ts` | The search term is passed as a bound parameter instead of being interpolated into the SQL string. |
| F3 | `models/product.ts`, `server.ts` | The description setter always runs `sanitizeSecure`, and `PUT /api/Products/:id` is put behind `isAuthorized()`. |
| F4 | `data/static/users.yml`, `server.ts` | The default admin password is replaced, and `/rest/user/login` is rate-limited to 10 attempts per 5 minutes, keyed on the socket address rather than a client-supplied header. |
| F5 | `routes/basketItems.ts` | `quantityCheck` rejects anything that is not an integer of at least 1 with `400`. |
| F6 | `server.ts` | `/metrics` requires a valid admin JWT. |

### F6 needed three commits, and that is the interesting part of this exercise

The first F6 patch applied cleanly, compiled without error, and **did not fix the
vulnerability**. `/metrics` still answered `200` with full metrics to an unauthenticated
client.

Juice Shop 20.4.0 registers `GET /metrics` twice: once inside `configureApp()` and once at
module scope. The module-scope registration is evaluated at import time, so Express matches it
first, and the guard added to the other route was never reached. The giveaway was that a
deliberately malformed token also returned `200` — a working guard would have rejected it.

The fix hoists the guard to module scope and applies it to both registrations. A third commit
handles a malformed token: `jws.verify` throws on input that is not a well-formed JWT, which
reached the Express error handler and rendered a `500` page echoing the submitted token, so any
unusable token is now treated as unauthorised and answered with `401`.

A patch that applies, builds green, and leaves the vulnerability open is the kind of result
worth verifying rather than assuming.

## Evidence

| File | Contents |
| --- | --- |
| [`evidence/commit-log.txt`](evidence/commit-log.txt) | The nine commits in `vulnerable..patched` |
| [`evidence/diff-stat.txt`](evidence/diff-stat.txt) | Files touched, insertions and deletions |
| [`evidence/full.diff`](evidence/full.diff) | The complete change set |
| [`evidence/environment.txt`](evidence/environment.txt) | OS, architecture, Node version, Juice Shop version, baseline commit |
| [`patches/`](patches/) | The six patch files, applied in numeric order |

The commit log carries nine commits for six findings. Three are corrections, kept as separate
commits so the reasoning stays visible: `chore: coerce port to number for loopback bind` fixed
a `TS2769` type error introduced by the loopback change, and the two extra F6 commits are
described above.

Screenshots for each finding, before and after, are in the report document rather than this
repository.

## A note on the committed admin password

`data/static/users.yml` on the `patched` branch contains the replacement admin passphrase in
plaintext, and it appears in `evidence/full.diff`. This is deliberate — F4 is *about* that
value, and the diff has to show it being replaced — but it is worth being explicit about what
it is and is not:

- It was generated randomly with `openssl rand -base64 18` for this exercise alone.
- It guards the admin account of an intentionally vulnerable training application, bound to
  loopback, holding data that is regenerated from a fixture on every restart.
- It is not reused anywhere.

The residual issue is real nonetheless, and the fix as written is incomplete for anything
beyond a lab: a credential still lives in a file in version control. A production fix would
read it from an environment variable or a secret manager and keep it out of the repository
entirely. F4 as patched here removes a *published default* password, which is the finding being
addressed; it does not make credential handling in this application correct.

## Attribution

OWASP Juice Shop is copyright Bjoern Kimminich and the OWASP Juice Shop contributors, released
under the MIT licence — see [`LICENSE`](LICENSE). The `vulnerable` branch is unmodified upstream
source at commit `266448a` apart from the loopback bind; upstream history is not included, as
this was a shallow import for course work rather than a fork for contribution. The six patches
and this write-up are course-work additions.
