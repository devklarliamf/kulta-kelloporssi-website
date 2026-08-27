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

**On the day — the record changes, in this order. Order is load-bearing.**

1. `mail.kulta-kello.fi`: CNAME → **direct A `5.44.244.113`** (pin the old mail host).
2. `MX 0`: `kulta-kello.fi` → **`mail.kulta-kello.fi`**.
3. SPF TXT: `v=spf1 +a +mx +ip4:5.44.244.113 ~all` → **`v=spf1 +mx +ip4:5.44.244.113 ~all`**
   (drop `+a`, so the apex moving to Vercel does not de-authorise the mail host).
4. Apex `A` → **`216.150.1.1`** (Vercel rank-1; rank-2 `216.150.16.1`).
5. `www` stays a CNAME to the apex and follows on its own — **or** point it at
   `cname.vercel-dns.com`. Either works once step 3 of §"Now" has added it to the project.

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
