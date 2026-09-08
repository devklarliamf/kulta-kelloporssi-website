# Kulta Kellopörssi — build status and the 4 Sep 2026 DNS cutover

All DNS/Vercel/HTTP values below **measured 2026-08-27** from this Mac against the live
zone and the Vercel API (guidance token, team `team_TSdGXKf3ZGbOBADULOAnE3lA`).

Nothing in any repo, doc, ledger or DONE-registry mentions a 4 September date. The plan
exists only verbally; this file is the first written form of it.

---

## 1. What is done

| Thing | State |
|---|---|
| Site source | `kulta-kelloporssi-website`, 3 commits, HEAD `1d226bc` (2026-07-22), clean tree, **pushed** — nothing local unpushed |
| Production build | Vercel `kulta-kelloporssi-website`, `dpl_9jmt8XH413JzoPwVmhPsg8jE5gbg` **READY** on `1d226bc` |
| Deployed == source | **Yes, byte-identical.** `kulta-kelloporssi-website.vercel.app` HTML is 34 652 bytes, identical to local `index.html` |
| Preview URL | `preview.klarsystems.com/kulta-kelloporssi/` → **200** |
| Domain attached | `kulta-kello.fi` is on the project, `verified: true` |

The website build itself is finished and shipping. Nothing on our side is half-built.

## 2. What is missing

### Blocking the cutover

1. **We do not control the DNS.** `kulta-kello.fi` nameservers are
   `ns1/ns2.webhotelli.fi`; apex A → `5.44.244.113` (not Vercel), TTL **300**.
   Registrant `Sami Salmensuo`, registered via webhotelli, expires 16.12.2026.
   No credentials for that panel are recorded anywhere on this machine.
   **Operator/owner-only.** This is the whole critical path.

2. **The Vercel attachment is a REDIRECT, not a serving alias.**
   `/v9/projects/.../domains` returns `redirect: "kulta-kelloporssi-website.vercel.app"`
   for `kulta-kello.fi`. Flip DNS with this in place and the apex 307s visitors to the
   `.vercel.app` URL instead of serving the site at `kulta-kello.fi`. **Must be cleared
   before the flip**, and it is a one-call change that needs no DNS access.

3. **`www.kulta-kello.fi` is not attached to the project at all.** Today it is a CNAME
   to the apex and follows it. After the flip it has no route and no cert — `www` breaks.
   **Must be added to the project**, also DNS-free and doable now.

4. **The mail records are self-referential — the ani.fi trap, again.**
   `MX 0 → kulta-kello.fi` (the apex itself), `mail.kulta-kello.fi` is a CNAME to the
   apex, `webmail.kulta-kello.fi` → `5.44.244.113`, and SPF is
   `v=spf1 +a +mx +ip4:5.44.244.113 ~all` — the `+a` means the apex A record *is* the
   SPF authorisation. **Change the apex A alone and every `@kulta-kello.fi` mailbox dies
   the moment the TTL expires.** Ani lost mail for ~2.5 h on exactly this.
   **Unmeasured:** whether anyone actually uses `@kulta-kello.fi` mail. Port 25 is
   blocked outbound here, so it cannot be probed from this Mac — **it has to be asked**.
   The new site carries no email address at all, only `tel:+3589645360` and the address.

5. **No certificate exists.** `/v7/certs` lists 6 for the team — ani.fi, La Lasagna,
   Roba Deli — and **zero** for `kulta-kello.fi`. The ani.fi lesson: Vercel does not
   issue one by itself; `POST /v7/certs` must be fired after DNS resolves.

### Not blocking, but open

6. **The live domain is already dead**, so there is nothing to lose by flipping:
   `https://kulta-kello.fi` fails the handshake (self-signed cert, `CN=kulta-kello.fi`,
   expired **2023-01-04**) and `http://kulta-kello.fi` returns **403**. The old
   webhotelli host serves no usable page today.
7. **Photography is stock.** `images/` is freely-licensed Unsplash per the README — no
   photographs of the actual Punavuori shop.
8. **No owner sign-off on the content is recorded** anywhere; the site has not been
   touched since 2026-07-22.
9. **No billing row.** Kulta is one of the nine unpriced tenants in the console.
   Contract + invoice were due 2026-08-27.
10. `GAP-REGISTER.md` G-007 still reads *BROKEN / operator-gated*.

---

## 3. Fastest path to 4 September

**Now, before the day — ours, no DNS needed, ~15 minutes, needs approval:**

- Clear the `redirect` on `kulta-kello.fi` in the Vercel project so the apex serves.
- Add `www.kulta-kello.fi` to the same project.

Both are safe today precisely because the domain resolves to a host that already 403s —
nothing observable changes until DNS moves.

**Now — the operator's asks, and the only real blocker:**

- Get into the webhotelli DNS for `kulta-kello.fi` (owner's panel credentials, or the
  owner sits down and makes the edits with us on the phone).
- Ask the owner one question: **does anyone send or receive mail at
  `@kulta-kello.fi`?** The answer decides whether step 1 of the flip is mandatory or
  can be skipped.

Note the standing ruling (2026-08-27): **NS delegation to a Klar-owned DNS account is
the default onboarding path** — but that account and its API token do not exist yet and
are operator-only to provision. **Do not put it on the 4 September critical path.**
Kulta flips through the owner's panel, the ani.fi way.

**ANSWERED 2026-09-03 — the operator states nobody uses `@kulta-kello.fi` mail.**
That downgrades steps 1 and 2 below from mandatory to optional. The mailbox is still
measurably alive (Exim answering, DKIM key in the zone, §5), so this is the owner's word
over a live server: if anything *is* delivered there, it stops at the flip and nothing
bounces back to us. Step 3 (dropping `+a` from SPF) stays — it is anti-spoofing hygiene
and costs one edit. The minimum flip is now **step 3 + step 4**.

**On the day — the record changes, in this order. Order is load-bearing.**

1. `mail.kulta-kello.fi`: CNAME → **`web152.webhotelli.fi`** (not a pinned A — see the
   dry run, §5: that name carries a valid auto-renewing cert and survives a server move).
2. `MX 0`: `kulta-kello.fi` → **`web152.webhotelli.fi`**.
3. SPF TXT: `v=spf1 +a +mx +ip4:5.44.244.113 ~all` → **`v=spf1 +mx +ip4:5.44.244.113 ~all`**
   (drop `+a` — see §5, this is anti-spoofing hygiene, not a delivery fix).
4. Apex `A` → **`216.150.1.1` and `216.150.16.1`** — both are rank-1; publish both.
5. `www` — **no panel edit needed.** It is a CNAME to the apex and follows automatically.
   All it needs is the Vercel project attachment from §"Now".

**There is no AAAA record on `kulta-kello.fi`** — checked; the ani.fi IPv6 trap does not
apply here.

**Immediately after the apex publishes:**

6. `POST https://api.vercel.com/v7/certs?teamId=team_TSdGXKf3ZGbOBADULOAnE3lA`
   with `{"cns":["kulta-kello.fi","www.kulta-kello.fi"]}` — do **not** wait for Vercel.
7. Force a cache MISS (unique query string) before calling the site live.

**Timing the room should be told:** TTL is **300 seconds**, not ani.fi's 4 hours — the
flip is visible in about five minutes, and the cert lands within seconds of step 6. This
cutover should take minutes, not the four hours ani.fi took. The whole cost is getting
into the panel.

---

# 4. Dry run, 2026-08-28 — what was rehearsed

Everything in §1–§3 was re-measured from scratch against the live zone and the Vercel
API. **Every value in §1 and §2 still held.** Nothing had drifted in the 24 hours.

Three scripts now exist. All three were run; none of them changes anything.

| Script | What it does | Rehearsal result |
|---|---|---|
| `~/.klar-watch/kulta-cutover-check.sh` | Reads the whole cutover state — Vercel pre-steps, every DNS record, mail liveness, cert, both hostnames end to end — and prints OK / PENDING / FAIL per line | Ran clean: **10 OK, 9 PENDING, 0 FAIL**, "NOT CUT OVER YET". Re-run it after every record change on the day |
| `~/.klar-watch/kulta-issue-cert.sh` | Fires the `POST /v7/certs` that ani.fi sat waiting an hour for | Dry run prints the exact call and sends nothing. `--send` **refuses** while DNS is still on the old host |
| `~/.klar-watch/kulta-dns-watch.sh` | Cron watcher: detects the flip, verifies site *and mail*, emails once, stops | Ran live, logged "no change", sent nothing. **NOT ARMED** — the crontab is empty; the arming command is in the file header |

The check script was also run as a **negative control** with the mail host and project
deliberately wrong. It produced 11 FAIL lines and "SOMETHING IS BROKEN". It fails when it
should, not only passes when it should.

## 5. What the dry run found that §1–§3 did not

**The mail is not hypothetical — it is live and in daily-use shape.** §2 item 4 left this
"unmeasured, it has to be asked". It is now measured. `5.44.244.113` is
`web152.webhotelli.fi` and it answers on **587 (Exim 4.99.5), 465, 143, 993, 110, 995**,
with a **valid auto-renewing Let's Encrypt certificate**, plus **2083/2087 — this is a
cPanel/WHM host**. There is also a **DKIM key** at `default._domainkey.kulta-kello.fi`.
A dead domain does not carry a signing key and a renewed mail cert.

→ **Treat the mail steps as mandatory.** Still ask the owner, but the default is now
"do them", not "skip unless he says otherwise".

**Use `web152.webhotelli.fi`, not the IP.** The mail host's certificate covers
`web152.webhotelli.fi` and `mail.web152.webhotelli.fi` — it does **not** cover
`mail.kulta-kello.fi`. Pointing the MX and `mail.` at the webhotelli hostname instead of
pinning `5.44.244.113` is both correct today and survives webhotelli moving the account
to another server. The owner's mail client is almost certainly already set to the
webhotelli hostname, which means **his client keeps working through the flip untouched**.

**The SPF reasoning in §2 was wrong in a way worth correcting.** `+a` is not what keeps
mail deliverable — `+ip4:5.44.244.113` authorises the mail host explicitly and survives
the apex moving. Dropping `+a` matters because after the flip it would authorise
*Vercel's shared anycast IP* to send mail as `kulta-kello.fi`. It is a spoofing surface,
not an outage. Do it anyway; just do not expect mail to break if it is missed.

**Four records §2 never listed, all of which survive the flip untouched** —
`ftp`, `cpanel`, `webdisk` are direct A records to the old host (not CNAMEs to the apex),
and `webmail` likewise. Nobody should "tidy" them.

**Clean ground on the things that could have blocked issuance:** no CAA record (nothing
restricts Let's Encrypt), no DNSSEC, no wildcard, no DMARC, and — reconfirmed — no AAAA.

**The panel is cPanel, so the ani.fi panel lessons transfer exactly.** Use the **per-row
edit buttons, not the bulk Save**, and treat the **SOA serial as the only proof a change
committed**. Baseline to compare against: **`2025070409`**. If it has not moved, nothing
was written, no matter what the form shows.

**`www` needs no DNS edit at all** — it is a CNAME to the apex and follows it. Correcting
§3, which offered a `cname.vercel-dns.com` edit as an alternative. If anyone does point
it explicitly, Vercel's rank-1 target for this domain is
`6c16fe7fe0c6fd70.vercel-dns-016.com` — `cname.vercel-dns.com` is only rank 2.

**The Vercel token survives the date.** It expires **2026-10-21**, so the cert call will
not fail on an expired credential the way earlier deploys did.

**The flip is effectively one-way — say this out loud before starting.** Vercel serves
`Strict-Transport-Security: max-age=63072000` (two years) on custom domains. Once anyone
loads the new site over HTTPS, their browser refuses plain HTTP for
`kulta-kello.fi` for two years. Rolling the A record back would hand those visitors the
old host's **self-signed certificate that expired 2023-01-04** — a hard TLS error, not
the old page. In practice nothing is lost, because the old host already serves only a
**403** and has no working HTTPS at all. But "we can just roll it back" is not true, and
nobody should say it in the room. *(Checked and cleared: Vercel does **not** send
`includeSubDomains` on custom domains — only on `*.vercel.app` — so `webmail`, `cpanel`
and the rest are not dragged into HSTS. That would have broken the owner's webmail.)*

## 6. Rollback record — the zone exactly as it stands before the flip

Captured 2026-08-28. If anything must be put back, this is the state to put back.

```
SOA serial            2025070409   (ns1.webhotelli.fi, alert@webhotelli.fi)
NS                    ns1.webhotelli.fi, ns2.webhotelli.fi
kulta-kello.fi        A     300   5.44.244.113
kulta-kello.fi        AAAA        (none)
kulta-kello.fi        MX 0  300   kulta-kello.fi.
kulta-kello.fi        TXT   300   "v=spf1 +a +mx +ip4:5.44.244.113 ~all"
www                   CNAME 300   kulta-kello.fi.
mail                  CNAME 300   kulta-kello.fi.
webmail               A     300   5.44.244.113
ftp                   A     300   5.44.244.113
cpanel                A     300   5.44.244.113
webdisk               A     300   5.44.244.113
default._domainkey    TXT         v=DKIM1; k=rsa; p=MIIBIjANBgkq...  (leave alone)
CAA / DS / DMARC / wildcard       (none)
```

## 7. The day itself, in order

1. `~/.klar-watch/kulta-cutover-check.sh` — baseline; expect 0 FAIL.
2. ~~Do the two Vercel pre-steps.~~ **DONE 2026-08-28**, on the operator's approval — see
   §8. Nothing is left to do here on the day.
3. Arm the watcher (command in the script header).
4. In the panel: mail records first, SPF, then the apex A last. Confirm the **SOA serial
   moved** after each write.
5. `~/.klar-watch/kulta-issue-cert.sh --send` the moment both resolvers show the new IP.
6. `~/.klar-watch/kulta-cutover-check.sh` until it prints **CUTOVER COMPLETE**.
7. `crontab -r` to remove the watcher.

**Still blocked on the owner, unchanged and unrehearsable from here:** the webhotelli
panel credentials, and confirmation of who uses `@kulta-kello.fi` mail. The dry run
narrowed the second question but cannot answer it — knowing the mailbox *works* is not
knowing who reads it.

## 8. The two Vercel pre-steps — DONE 2026-08-28

Executed on the operator's explicit approval, after the dry run. Both returned success:

- `kulta-kello.fi` now has `redirect: null` — the apex will **serve** the site rather
  than 307 visitors to the `.vercel.app` URL.
- `www.kulta-kello.fi` is attached to the project and came back `verified: true`
  immediately, so it will be on the certificate and has a route.

The check script now reads **12 OK / 7 PENDING / 0 FAIL** — both pre-step lines green.

Confirmed nothing observable changed: apex A still `5.44.244.113`, SOA still
`2025070409`, and both `kulta-kello.fi` and `www.kulta-kello.fi` still return the old
host's **403** to a visitor. Exactly as intended — these only take effect when DNS moves.

The calls that were run, for the record:

```
# clear the 307 redirect so the apex serves the site
curl -X PATCH "https://api.vercel.com/v9/projects/kulta-kelloporssi-website/domains/kulta-kello.fi?teamId=team_TSdGXKf3ZGbOBADULOAnE3lA" \
  -H "Authorization: Bearer $(cat ~/.klar-vercel-token)" -H "Content-Type: application/json" \
  -d '{"redirect":null}'

# attach www so it has a route and gets on the certificate
curl -X POST "https://api.vercel.com/v10/projects/kulta-kelloporssi-website/domains?teamId=team_TSdGXKf3ZGbOBADULOAnE3lA" \
  -H "Authorization: Bearer $(cat ~/.klar-vercel-token)" -H "Content-Type: application/json" \
  -d '{"name":"www.kulta-kello.fi"}'
```
