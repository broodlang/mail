# mail — guidance for Claude

Swoosh's shape for Brood: an email is a plain map, builders return new ones, and
`mail/deliver` validates then calls the adapter's `transmit`. Read the module header of
`src/mail.blsp` for the design notes (the *why*).

- **Layout.** `src/mail.blsp` is the core (builder, validation, deliver, telemetry, the
  memory and log adapters). `src/mail/local.blsp` is the mailbox + its hatch pages,
  `src/mail/mailjet.blsp` the Mailjet wire format. Another vendor is its own
  `mail-<vendor>` package with an `Adapter` impl; only a vendor everyone reaches for
  belongs in here.
- **`deliver` never raises.** `[:ok result]` / `[:error message]`, a raise in the adapter
  included. Validation runs before the adapter so no adapter checks for a sender.
- The ability op is `transmit`, not `deliver` — both would be `mail/deliver` in this
  namespace. An `impl` in another module must name the ability bare (`(impl Adapter …)`);
  qualified, it registers under a name dispatch never sees. Ability names are not module
  globals, so there is nothing to import: `(:load mail)` (or an `(:alias mail)`) is enough —
  `(:use mail :only [Adapter])` refers nothing, and `nest check` says so.
- `mail/local`: the HTML body is never inlined into the page — the iframe loads
  `<id>/html`, sandboxed by attribute and by CSP header. `stop` unregisters the name
  BEFORE the asynchronous `gen/stop`, so `pid` is nil on return; its adapter test is
  `:serial` because it owns the one keeper.
- `mail/mailjet`: field names are Mailjet's, as STRING keys; `json/decode` on the way
  back yields keyword keys. An absent field, never an empty one — Mailjet refuses
  `"HTMLPart": ""`. A 400's `Errors` array is the useful message; surface it verbatim.
- `path/extension` keeps the dot; `content-type-for` strips it.
- The telemetry test is `:isolated`: the listener is global and tests run concurrently, so
  a handler would otherwise see other tests' deliveries. Don't filter — isolate.
- Run the test files one at a time (the full suite is blocked on this machine) and
  `nest format` before committing. No trailing `!`, no abbreviations.
