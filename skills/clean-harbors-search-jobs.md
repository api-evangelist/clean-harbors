---
name: clean-harbors-search-jobs
description: >-
  Search and read Clean Harbors' 1,000+ open positions through the anonymous Job Query API
  the company publishes at careers.cleanharbors.com, and hand a candidate the right apply
  link.
generated: '2026-09-05'
method: generated
source: >-
  Grounded in llms/clean-harbors-llms.txt (published verbatim by Clean Harbors at
  https://careers.cleanharbors.com/llms.txt), in the tool manifest saved at
  mcp/clean-harbors-careers-jobs-manifest.json, and in live calls executed against the
  endpoint on 2026-09-05. Every tool name and parameter below was returned by the provider;
  none is invented.
api: Clean Harbors Careers Job Query API
base: https://careers.cleanharbors.com/api/mcp/jobs
operations:
  - search_jobs
  - get_job
  - list_departments
  - list_locations
---

# Search Clean Harbors jobs

Clean Harbors publishes an unauthenticated JSON API over its job postings. There is no key,
no sign-up and no quota. Every call is an HTTP `GET` against one endpoint, and the operation
is chosen with a `tool` query parameter.

    https://careers.cleanharbors.com/api/mcp/jobs?tool=<operation>

Calling the endpoint with no parameters returns a self-describing manifest of the four
operations. Start there if you are unsure what is available.

Despite the `/api/mcp/` path, **this is not an MCP server**. A JSON-RPC `POST` of
`tools/list` returns `405 Method Not Allowed`. Use plain `GET`.

## Steps

1. **Find matching postings.**

       GET /api/mcp/jobs?tool=search_jobs&search=driver&location=Texas&pageSize=20

   Optional parameters: `search`, `department`, `employmentType`, `location`, `page`,
   `pageSize`. The response is
   `{ tool, results[], totalCount, page, pageSize, summary }`. Each result carries
   `requisitionId`, `title`, `department`, `location`, `employmentType`, `datePosted`,
   `applyUrl` and a truncated `description`.

2. **Page through the rest.** Pagination is by page number only — there is no cursor.
   `page` defaults to `1` and `pageSize` to `10`. Divide `totalCount` by your `pageSize` to
   know how many pages to walk. A bare `search_jobs` returned `totalCount: 1023` on
   2026-09-05.

3. **Read the full posting.**

       GET /api/mcp/jobs?tool=get_job&jobId=165235

   `jobId` is the `requisitionId` from step 1. The response is `{ tool, result }`. The full
   `description` is **HTML**, not plain text — strip or render the tags before showing it to
   a person.

4. **Hand over the apply link.** Do not try to submit an application through this API; it is
   read-only. `applyUrl` leaves the careers site entirely and lands in Oracle Fusion HCM
   (`epyc.fa.us2.oraclecloud.com`, candidate site `CLH-us`). Give the candidate that URL.

## Narrowing by department or location

`list_locations` returns locations grouped by state and country with job counts, and works
as documented.

`list_departments` returns `{ id, name, jobCount }` — but **every record sets `name` equal
to the opaque `id`** (for example `687e8e451c9a05008f6ad30f`), so it cannot actually tell you
what a department is called. Do not route a user through it. Get real department names from
the `department` field on `search_jobs` results instead ("Driver – Bulk", "Driver-National"),
then filter with `search=` on those words.

## Errors and limits

An unknown `tool` returns HTTP `400` with `{"error":"Unknown tool \"…\". Available: …"}`.
That is the entire error vocabulary; there is no problem-details envelope.

No rate limit is documented and no `RateLimit-*` or `Retry-After` header is returned. The
host is behind Cloudflare, so an undisclosed edge limit may exist — pace yourself
conservatively and back off on any `429` or `5xx` rather than assuming a budget.
