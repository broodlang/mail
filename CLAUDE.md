# mail — guidance for Claude

Swoosh's shape for Brood: an email is a plain map, builders return new ones, and
`mail/deliver` validates then calls the adapter's `transmit`. Read the module header of
`src/mail.blsp` for the design notes (the *why*).

- **No vendor code here.** A vendor is its own `mail-<vendor>` package with an `Adapter`
  impl; this package stays the builder, validation, telemetry and the two local adapters.
- **`deliver` never raises.** `[:ok result]` / `[:error message]`, a raise in the adapter
  included. Validation runs before the adapter so no adapter checks for a sender.
- The ability op is `transmit`, not `deliver` — both would be `mail/deliver` in this
  namespace. An `impl` in another module must name the ability bare
  (`(:use mail :only [Adapter])`); qualified, it registers under a name dispatch never sees.
- `path/extension` keeps the dot; `content-type-for` strips it.
- The telemetry test is `:isolated`: the listener is global and tests run concurrently, so
  a handler would otherwise see other tests' deliveries. Don't filter — isolate.
- Run the one test file (the full suite is blocked on this machine) and `nest format`
  before committing. No trailing `!`, no abbreviations.
