# GNU Units backend

Routes: `POST /units/convert`, `GET /units/definitions/standard`, and
`GET /units/definitions/custom`. Request and response formats are in
[gnu-units-api.md](gnu-units-api.md).

This deployment uses GNU Units 2.22 at `/usr/bin/units`, with the installed
`/usr/share/units/definitions.units` (the distribution's units.dat equivalent)
and `/home/dungewar/.units`. Standard definitions include `currency.units`.
The definitions responses preserve full text and license notices and append
included files with public filename headings. An absent personal file means
no custom definitions; other read errors are reported. Paths are server-owned,
never selected by requests. Update these trusted files in place, atomically,
and keep included files consistent; requests read the installed definitions
fresh. File updates spanning multiple files should be made during maintenance.

Each conversion starts a fresh process with explicit definition files, an
empty stdin, no shell, and `--` before expressions. Its environment is fixed,
including UTF-8 locale and an unused home, so it cannot discover other personal
files. Shell punctuation (including `$`, backticks and semicolons) is passed
literally to GNU Units. Option-like input is also literal. Interactive-looking
input is evaluated in positional, noninteractive mode with stdin at EOF.
Control characters are rejected; unit expressions otherwise use GNU's parser.
This protects against shell and option injection; it is not a guarantee against
bugs in the native GNU Units binary. Keep GNU Units patched and run the server
as an unprivileged service account with read-only access to trusted definitions.

Limits per server worker: four running conversions, five-second deadline,
256 KiB combined output, 24 KiB JSON body, and 60 requests per minute per client
IP. Timed-out, disconnected, and excessive-output processes are killed; slots
remain occupied until the process closes. The rate-limit table is bounded.
Behind a proxy, Express currently uses the direct peer IP, so requests through
that proxy share a limit. Only configure Express trust proxy for known proxies;
never trust arbitrary forwarded headers. Multiple workers need an upstream
shared rate/concurrency policy if aggregate limits are required.

CORS allows `https://dungewar.com`; add development origins as a comma-separated
`UNITS_CORS_ORIGINS` environment variable. No credentials are enabled. Units
routing precedes the application's general CORS and body parsers so its errors
use the API contract.

Run `node --test test/units.test.js` for conversion, definition, validation,
injection, resource-limit, and HTTP checks. Integration tests require the
installed definitions/custom units above and permission to bind localhost.
Browser history remains a frontend responsibility.
