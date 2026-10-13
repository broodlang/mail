# mail

Compose an email and deliver it through a swappable adapter — [Swoosh](https://hexdocs.pm/swoosh),
for Brood. The builder, address parsing, validation, the delivery contract and its
telemetry, with the adapters an app reaches for first built in:

| adapter | what it does |
| --- | --- |
| `mail/local` | keeps every email in-process, with a page to read them — the mailbox preview |
| `mail/mailjet` | Mailjet's Send API v3.1 |
| `mail/memory-adapter` | keeps every email for a test to read back |
| `mail/log-adapter` | one summary line through `log`, sends nothing |

Any other vendor is the same shape in its own package: a value with an `Adapter` impl
(see *Writing an adapter*).

```lisp
(def adapter (mail/mailjet/adapter {:api-key (os/env "MAILJET_API_KEY")
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
;; → [:ok {:id 1234 :uuid "…"}]  or  [:error "mail/mailjet: 401 …"]
```

In development, the same code with `(mail/local/adapter)` puts the email in a mailbox
you read in the browser instead.

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

## The mailbox — `mail/local`

```lisp
(def mailer (mail/local/adapter))          ; starts the keeper, registered :mail/local
(mail/deliver email mailer)                ; → [:ok {:id 1}]
(mail/local/emails)                        ; newest first, each stamped :id and :at
```

In the router, behind whatever guards the operational pages:

```lisp
(get "/mailbox" (mail/local/handler {:refresh-ms 4000}))
(get "/mailbox/*path" (mail/local/handler {:refresh-ms 4000}))
```

`/mailbox` lists the inbox (newest first, refreshing itself when asked to); `/mailbox/<id>`
shows one email — every header, the HTML body as it renders, the text body, the
attachments to download. The HTML body is shown in a **sandboxed iframe from its own URL**
(`/mailbox/<id>/html`, served with `Content-Security-Policy: sandbox`), never inlined: an
email's HTML is content the app built from data. `handler` takes `:prefix` and `:title`.

The mailbox is bounded (`*max-emails*`, 200): the oldest go first. `clear` empties it,
`stop` ends the keeper. In memory; a restart forgets.

## Mailjet — `mail/mailjet`

One email, one `POST /v3.1/send`, authenticated with the account's API key and secret key.
Everything the email can carry is sent — To/Cc/Bcc, Reply-To, both bodies, headers,
attachments and inline attachments (`:cid` becomes `ContentID`); a field with nothing in
it is left out rather than sent empty.

Adapter options: `:api-key`, `:secret-key`, `:sandbox?` (validate without sending),
`:endpoint`, `:transport` (for tests). Per email, through `mail/provider-option`:
`:template-id`, `:template-language`, `:variables`, `:custom-id`, `:custom-campaign`,
`:deduplicate-campaign`, `:event-payload`, `:url-tags`, `:track-opens`, `:track-clicks`,
`:priority`, `:sandbox?` — each Mailjet's field of the same name — and `:mailjet`, a map
merged into the message as it is.

The answer is read for what it is: a 400's `Errors` naming the field refused is the text
`[:error …]` carries; a 401's flat `ErrorMessage` likewise.

## Writing an adapter

```lisp
(defmodule mail-postmark (:load mail))   ; ability names are global: `impl` names it bare

(defrecord postmark (name token))

(impl Adapter
  postmark
  (transmit [adapter email]        ; email is validated, addresses are {:name :email}
    …                              ; → [:ok {:id …}] | [:error message]; a raise is caught
    ))
```

`src/mail/mailjet.blsp` is the worked example, tests included.

## Running

```bash
nest fetch
nest test tests/mail_test.blsp     # and local_test / mailjet_test
nest format
```
