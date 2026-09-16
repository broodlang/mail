# mail

Compose an email and deliver it through a swappable adapter — [Swoosh](https://hexdocs.pm/swoosh),
for Brood. This package is the builder, the address parsing, validation, the delivery
contract and its telemetry, plus the two adapters an app needs before it has a vendor.
A vendor is its own package with an `Adapter` impl:

- [mail-mailjet](https://github.com/broodlang/mail-mailjet) — Mailjet's Send API v3.1.
- [mail-local](https://github.com/broodlang/mail-local) — keeps every email in-process,
  with a hatch page to read them (Swoosh's mailbox preview).

```lisp
(def adapter (mail-mailjet/adapter {:api-key (os/env "MAILJET_API_KEY")
                                    :secret-key (os/env "MAILJET_SECRET_KEY")}))

(-> (mail/new)
    (mail/from "Hive <noreply@brood.fly.dev>")
    (mail/to "ada@example.com")
    (mail/cc ["Bob" "bob@example.com"])
    (mail/subject "Your token")
    (mail/text-body "Here it is: …")
    (mail/html-body "<p>Here it is: …</p>")
    (mail/attachment "reports/today.pdf")
    (mail/deliver adapter))
;; → [:ok {:id "…"}]  or  [:error "mail-mailjet: 401 …"]
```

## The email

An email is a plain map — `{:from :to :cc :bcc :reply-to :subject :text :html :headers
:attachments :provider-options}` — and every builder returns a new one, so `->` threads
them and a test can build one by hand. `(mail/new :from … :to … :subject … :text …)` takes
the common fields as a plist.

Addresses are normalised at the edge: `"ada@x.io"`, `"Ada <ada@x.io>"`, `["Ada" "ada@x.io"]`
and `{:name "Ada" :email "ada@x.io"}` all become `{:name :email}`, and `to`/`cc`/`bcc` take
one or a vector of them and accumulate.

`attachment` takes a path (read now, named by its basename) or `{:filename :data
:content-type :disposition :cid}`; the content type is inferred from the name when absent.
`header` sets a custom header; `provider-option` sets something only one adapter reads.

## Delivering

`(mail/deliver email adapter)` validates — a sender, at least one recipient, a body — then
calls the adapter and answers `[:ok result]` or `[:error message]`. It never raises: an
adapter's raise is caught and reported the same way. Delivery is synchronous; send it off
the request path with `spawn` if the caller should not wait.

Every delivery emits `[:mail :deliver :stop]` (or `:exception`) through `telemetry` with
`:duration` and `{:adapter :recipients :subject :result}`, so a monitoring package such as
[observe](https://github.com/broodlang/observe) sees every send with no instrumentation.

## Adapters

```lisp
(defability Adapter
  (transmit [self email] :-> any))   ; [:ok {:id …}] | [:error message]
```

Shipped here:

- `(mail/memory-adapter :notify pid)` — keeps every email for `mail/delivered` to read
  back (`mail/clear`, `mail/stop`), and sends each one to `pid` as `[:mail email]`. For tests.
- `(mail/log-adapter :level :info)` — writes one summary line through `log` and sends
  nothing. For development, or a deploy without credentials yet.

Writing one: a `defrecord` holding the credentials and a `transmit` that turns the
normalised email into the vendor's request. `mail-mailjet` is the worked example, tests
included.

## Running

```bash
nest fetch
nest test tests/mail_test.blsp
nest format
```
