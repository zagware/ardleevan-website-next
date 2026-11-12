# Moving ardleevandogfood.co.uk to Cloudflare

This replaces the GitHub Pages plan in `DNS-CUTOVER.md`. The website moves to Cloudflare and **email stays exactly where it is** (Big Wet Fish, `mail.ardleevandogfood.co.uk` → 152.89.64.9).

Why Cloudflare rather than GitHub Pages:

- free hosting on a global CDN, with HTTPS certificates managed automatically;
- security headers and a `www` → apex redirect;
- room to add a contact form or a shop later without changing host.

The catch: Cloudflare must host the domain's **DNS** (nameservers). The steps below make sure every mail record comes across.

## Before switch-over (developer, about 15 min, no visible change)

1. Record the current zone so nothing is lost. Run this and save the output:
   ```sh
   for t in A AAAA CNAME MX TXT SRV; do for h in @ www mail webmail autodiscover autoconfig _dmarc default._domainkey; do
     n=$([ "$h" = "@" ] && echo ardleevandogfood.co.uk || echo $h.ardleevandogfood.co.uk)
     r=$(dig +short $t $n); [ -n "$r" ] && echo "$t $n -> $r"; done; done
   ```
   Also ask the owner for a screenshot of the Big Wet Fish DNS page. DKIM selectors can't be discovered with `dig`.
2. Cloudflare dashboard (Zagware account) → **Add a domain** → `ardleevandogfood.co.uk` → Free plan. Cloudflare scans the existing records. Compare the result with step 1 and add anything missing. The Worker's custom domains only attach to zones in the same account as the Worker, so the zone must live in the Zagware account.
3. Set **every mail-related record to "DNS only" (grey cloud)**: MX targets, `mail`, `webmail`, `autodiscover`, and anything pointing at 152.89.64.9. Proxying mail hosts breaks email.
4. Leave the scanned `@` A record → 152.89.64.67 and the `www` CNAME in place, both DNS only, so the old site stays up after the switch. They are removed just before the first deploy (see below). Add an `old` A record → 152.89.64.67 (DNS only) if the owner wants the old site reachable afterwards.
5. Make sure the apex has **exactly one** `v=spf1` TXT record. Two SPF records make SPF fail and mail lands in spam. Re-scans and imports can add a second copy.
6. Note the two Cloudflare nameservers it assigns. The zone is deleted if the nameservers are not switched within 28 days; re-adding it may assign a different pair.

## Switch-over (owner, about 5 min)

At Big Wet Fish (the domain's registrar), change the nameservers of `ardleevandogfood.co.uk` from `ns1/ns2/ns3.bigwetfish.co.uk` to the two Cloudflare nameservers. Keep the Big Wet Fish **hosting/email account active**: email still lives there.

The change usually takes effect within an hour, but can take up to 24 hours for `.co.uk`.

## After Cloudflare shows the zone "Active" (developer)

1. In `wrangler.jsonc`, uncomment the `routes` block for `ardleevandogfood.co.uk` and `www.ardleevandogfood.co.uk`.
2. Add repo secrets `CLOUDFLARE_API_TOKEN` and `CLOUDFLARE_ACCOUNT_ID`. The token needs Workers Scripts:Edit, Workers Routes:Edit, and Zone:Read on this zone.
3. In Cloudflare DNS, delete the `@` A record (152.89.64.67) and the `www` CNAME. Custom domains cannot be created over existing records. The site is down from here until the deploy finishes, so do steps 3 and 4 together.
4. `git tag v1.0.0 && git push --tags` runs the **Deploy (Cloudflare)** workflow: build (`--strict`), deploy, smoke test.
5. Dashboard → Rules → Redirect Rules → template **"Redirect from WWW to root"**.
6. Check the following:
   - `https://ardleevandogfood.co.uk/` shows the new site;
   - `/robots.txt` allows indexing and lists the sitemap;
   - `http://` redirects to `https://`;
   - send test emails in both directions.
7. Search Console: add the domain property and submit `https://ardleevandogfood.co.uk/sitemap.xml`.
8. Later, tidy SPF: remove `+a` (see `DNS-CUTOVER.md` step 4) and add DMARC `v=DMARC1; p=none; rua=mailto:info@ardleevandogfood.co.uk`, if not already done in the Cloudflare zone.

## Rollback

Set the nameservers back to `ns1/ns2/ns3.bigwetfish.co.uk` at Big Wet Fish. The old zone there is untouched, so WordPress and email return as they were.
