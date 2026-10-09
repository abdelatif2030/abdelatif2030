# Project handoff: FluentCRM + Amazon SES migration for melanotan-aus.com

Client: Natasha Del Bianco (store: https://melanotan-aus.com, WooCommerce, hosted on Cloudways, DNS on Cloudflare)
Contract: fixed $250. Setup, Drip migration, first sale send, rebuild of existing Drip flows. All accounts stay in the client's name.
Last updated: 2026-10-10 (session notes, no passwords or keys stored here)

> **Security:** the client pasted several passwords, a one-time login link, MFA codes and a FluentCRM license key in the chat. Do not store them in notes. At handover, tell her to change the AWS, Drip, WordPress and Cloudflare passwords, and remove your access.

---

## 0. READ FIRST: blocking issue

The existing Drip emails promote **tirzepatide, retatrutide and a tanning nasal spray** with weight-loss, muscle and tanning imagery and wording ("boosting brain power, building lean muscle, tanning smarter, dropping stubborn fat", "start your journey", "top off your stack"), while the vial labels say "research use only".

- As far as I know, tirzepatide is prescription-only, retatrutide is an unapproved investigational drug, and tanning nasal sprays like this are not approved in Australia. Advertising them to the public is restricted or illegal there (TGA rules). Verify with a lawyer; I am not giving legal advice.
- The SES production access request must describe the business accurately. It cannot call the products "research use only" while the emails promote personal use. AWS reviews content and may decline or later suspend the account.
- **Decision recorded:** the AI assistant declined to write the SES request, plan the send, or tune deliverability for this content. Continuing requires either (a) the client proves she is authorised to sell and advertise these products and the copy matches that authorisation, or (b) the product line and copy change to something lawful.
- Also check your freelance platform's rules on regulated or illegal products before continuing.

Draft message to the client is in section 8.

---

## 1. Access status (no secrets)

| System | Status | Notes |
|---|---|---|
| WordPress | Working | Login page is a custom URL `https://melanotan-aus.com/garag3/`. Username is `emailhosting`. Cookie error on first try was fixed by a clean/incognito browser. |
| Drip | Working | Account id 3064708. Dashboard link: https://www.getdrip.com/3064708/dashboard |
| AWS | Working | Account "M Aus" (832474135684), region **ap-southeast-2 (Sydney)**. Logged in as the account owner; the first link she sent led only to an AWS Builder ID, which is not an account. No limited IAM user has been created yet (contract says no shared main logins). |
| Cloudflare | Working | Zone melanotan-aus.com (Pro plan), logged in with her account. No invite to your own email yet. |
| FluentCRM Pro | Zip checked | `fluentcampaign-pro-3.2.5.zip`, genuine plugin structure, no obfuscated or suspicious code in a basic scan, licensing code intact. License key received but **not yet activated**. |
| Cloudways (cron) | Not received | Needed for the real server cron job. |

---

## 2. What is DONE

### SES (Amazon SES, Sydney)
- Domain identity **mail.melanotan-aus.com** created and **Verified**.
- Easy DKIM (RSA 2048) **Successful**; custom MAIL FROM `bounce.mail.melanotan-aus.com` **Successful**.
- Pricing plan: **Essentials** (no monthly fee). Virtual Deliverability Manager on; engagement tracking off; optimized shared delivery on; Auto Validation off; dedicated IPs off; tenant management off.
- Account still in **Sandbox** (200 emails/day, 1/sec). **Production access has NOT been requested.**
- Her email address `natashadelbianco1991@gmail.com` was added as an SES email identity; she needs to click the verification link (not confirmed).

### Cloudflare DNS (all DNS only, TTL Auto; existing 16 records untouched, now 22)
1. CNAME `kw2xuzfa36uc7ibglgvzbw2s6fvmrba5._domainkey.mail` -> `kw2xuzfa36uc7ibglgvzbw2s6fvmrba5.dkim.amazonses.com`
2. CNAME `6nqponygwk3cbxdugkmc4twgyttkd26i._domainkey.mail` -> `6nqponygwk3cbxdugkmc4twgyttkd26i.dkim.amazonses.com`
3. CNAME `7p7cud22zvctbi5cnhbheoambnn6ium7._domainkey.mail` -> `7p7cud22zvctbi5cnhbheoambnn6ium7.dkim.amazonses.com`
4. MX `bounce.mail` priority 10 -> `feedback-smtp.ap-southeast-2.amazonses.com`
5. TXT `bounce.mail` -> `v=spf1 include:amazonses.com ~all`
6. TXT `_dmarc.mail` -> `v=DMARC1; p=none;` (add `rua=mailto:<mailbox she reads>` later)

Existing records worth knowing: root DMARC is `p=reject` with a placeholder `rua=mailto:your-info@melanotan-aus.com`; root SPF is Google only; Drip/SendGrid CNAMEs (`drip.`, `s1._domainkey`, `s2._domainkey`) and a Klaviyo verification TXT exist; `mail.melanotan-aus.com` also has an A record to the web server (harmless to SES). Do not change the root records (contract).

### Drip audit (read-only, nothing changed in Drip)
- Drip sending restriction **has been lifted** (dashboard shows "fully ready for sending"). Someone answered Drip's questionnaire; ask what was said.
- Drip makes about **26% of store revenue** ($5,569 in 7 days). Run Drip and FluentCRM in parallel until flows are tested; never run two live abandoned-cart flows.
- List history: imported from Mailchimp and Klaviyo (CSV imports Sep 15 2026, bulk operation on Sep 16 touching 15,912 people, tags `Klaviyo Import - 15 September 2026`, `klaviyo-never-subscribed`, `klaviyo-unsubscribed`, `mailchimp_list:...:unsubscribed`, etc.).
- Subscribed people with `klaviyo-never-subscribed`: **0**; with `klaviyo-unsubscribed`: **0**; with the Mailchimp-unsubscribed tag: **8** (exclude them / import as Unsubscribed).
- The Sep 16 bulk operation almost certainly did not mass-subscribe people (only 928 people have a subscribe date since Sep 15, matching normal signups). Exact action not seen (do not open the bulk operation wizard).
- New signups still arrive at roughly 10 to 100 per day, with many bot-looking addresses (dotted Gmail names, clusters of corporate addresses created within minutes, e.g. Sep 30, 1:30 to 4:35 AM). Corporate "clicks" are likely security-scanner clicks.

| Group | Count |
|---|---|
| Total people | about 16,500 (16,675 on last check) |
| Placed an order, all time (all active) | 6,131 |
| Subscribed + ordered in last 12 months | 1,074 (**1,067** after excluding Mailchimp-unsubscribed) |
| Ordered in last 12 months, any status | 1,951 (so **884** recent buyers are not marked subscribed) |
| Subscribed + ordered 12 months + clicked in last 90 days | **505** |
| "Engaged 90 Days" saved segment (opens-based) | about 2.4k |

- Workflows: **Cart Abandonment** (ON, 15,951 completions, 21 orders) and **Global Welcome Series** (ON, 3 emails for non-purchasers, 10% first-order discount, exit on purchase). OFF: two "WooCommerce: Abandoned Browse", "Welcome 15 popup", "Untitled Workflow", "40% Off" onsite campaign.
- Email series "Peptides Sale Oct 2025" is marked Active but has 0 active people and $0 revenue this week. Two emails, about 10,500 recipients each, opens 28.1% / 26.7%, clicks 3.0% / 3.2%, unsubscribes 0.3% / 0.4%. Recent single sends: "Injection Pen Refill Cartridges" (Sep 30) and "Semax + Selank Stack" (Sep 27). The footer has `{{ unsubscribe_link }}` and `{{ inline_postal_address }}`; From is `info@melanotan-aus.com`.
- Custom fields: AOV, Country_Region, Last_Active, Name, Orders, Postal_Code, Region, Sign_Up, Total_Spend, Username.
- Forms: Footer Mailing List (11 submissions), Form #4 (0). Opt-in tags: `optin_checkout`, `optin_account`, `optin_thankyou`, `optin_package_qr`, `Newsletter`.

### WordPress site (seen only on the dashboard)
Plugins include WooCommerce, Jetpack, Site Kit, WPForms, Contact Form 7, Yoast, WebToffee Import/Export, Shipment Tracking, Product Bundles, FlyingPress (cache), UpdraftPlus (backup), custom "Melanotan Aus" theme plugins. About 10 plugin updates are pending (do not update as part of this job).

---

## 3. What is STILL TO DO (only if the blocking issue in section 0 is resolved)

1. Resolve section 0 with the client in writing.
2. Click her SES email verification link (ask her), then **do not** make FluentSMTP the live mailer until SES leaves sandbox (otherwise WooCommerce order emails fail).
3. Create the limited IAM users in her AWS account: one for you (`AmazonSESFullAccess` only) and one for FluentSMTP sending (send permissions only, plus an access key). Stop using the owner login.
4. **Full site backup** (UpdraftPlus, download a copy) before installing anything.
5. Install FluentCRM (free) -> FluentCRM Pro (upload zip, activate license in FluentCRM > Settings > License) -> FluentSMTP.
6. Server cron: add a real cron on Cloudways and set `DISABLE_WP_CRON` in `wp-config.php`; check FlyingPress exclusions for FluentCRM pages and REST routes.
7. Connect WooCommerce to FluentCRM; use double opt-in and CAPTCHA/honeypot on the new signup forms (bot signups).
8. Export from Drip: Subscribed people CSV **and** Unsubscribed (and never-subscribed) people CSV. Import unsubscribed people into FluentCRM with status Unsubscribed so they stay blocked. Map tags and custom fields.
9. Verify addresses with an email-verification service before the first send (typos like `hotmail.cpm` exist). The client pays for this. SES Auto Validation is off.
10. Submit the SES production access request only with an accurate description (see section 0). Typical plan to describe: start with about 505 engaged subscribed customers, then about 1,067 subscribed 12-month buyers, then grow gradually; unsubscribed and never-subscribed contacts suppressed. Ask for realistic quota (her historical campaigns reach about 10,500 people).
11. Test email passes SPF, DKIM, DMARC and lands in Gmail and Outlook; set `_dmarc.mail` reporting address.
12. Build and send the first sale email in small batches (day 1 a few hundred, then about 250 to 500 per day, widening over weeks) with compliant, factual copy.
13. Warm-up plan document, then rebuild **Cart Abandonment** and **Global Welcome Series** in FluentCRM (capture every trigger, delay, condition, subject, content, tag and exit first; confirm whether the OFF drafts are in scope). Test end to end, then switch each Drip flow off only after its FluentCRM version is confirmed.
14. Handover doc or video; remove your access; client changes all shared passwords.

Drip items still unchecked: content of the Sep 27/Sep 30 broadcasts and the Welcome Series and Cart Abandonment emails; account-level bounce/complaint rates (Analytics > Email metrics); Settings > Integrations (WooCommerce connection); Cloudways access; the Oct 9 draft bulk operation of 27 people (do not run it).

---

## 4. Open questions for the client

- Can she show authorisation to sell and advertise these products in Australia, or will the range and wording change? (blocking)
- Why did Drip restrict her sending, and what did she tell Drip in its questionnaire?
- Did she ever buy or rent a list? How did most contacts join?
- What did the Sep 16 bulk operation do, and what is the Oct 9 draft (27 people)?
- Is the "Peptides Sale Oct 2025" series still intended to run? Which flows are in scope?
- Which mailbox should receive DMARC reports (`rua`)? Is the `info@` mailbox real? Is Klaviyo still used?
- Cloudways access, or will she add the cron herself?

---

## 5. Useful links (Drip account 3064708)

- Subscribers: https://www.getdrip.com/3064708/subscribers
- Workflows: https://www.getdrip.com/3064708/workflows
- Campaigns: https://www.getdrip.com/3064708/campaigns
- Segments: https://www.getdrip.com/3064708/segments
- Tags: https://www.getdrip.com/3064708/tags
- Bulk operations (view the list only): https://www.getdrip.com/3064708/bulk_operations
- SES (Sydney): https://ap-southeast-2.console.aws.amazon.com/ses/home?region=ap-southeast-2
- Cloudflare DNS: https://dash.cloudflare.com/ (zone melanotan-aus.com)

Rules while auditing Drip: read only. Do not click Publish, Save Segment, Actions, or the bulk operation wizard; do not switch workflows on or off.

---

## 6. Technical notes

- Cloudflare entry: names go in without `.melanotan-aus.com`; all SES records must be DNS only; MX needs priority in its own box and only the hostname as the mail server; TXT content without outer quotes.
- SES recommended settings are already applied (Essentials plan, VDM on, dedicated IP and Auto Validation off).
- DMARC: root is `p=reject`; subdomain mail. has its own `p=none` record for warm-up. Send from an address aligned with the signing domain (DKIM is signed as `mail.melanotan-aus.com`).
- Opens are unreliable (Apple Mail Privacy Protection, corporate scanners). Prefer purchases and clicks inside the real-customer group.

---

## 7. Fallbacks

- If AWS declines SES: another provider (Brevo, Mailgun, SendGrid) would be needed, each with its own acceptable-use review and DNS changes. This is not a way around the section 0 issue.

---

## 8. Draft message to the client (adjust as needed)

> Hi Natasha, I've reviewed your Drip emails before sending anything from the new account. They promote tirzepatide, retatrutide and a tanning nasal spray with weight-loss, muscle and tanning imagery and wording. I can't set up and send this content through Amazon SES, because the account request has to describe the business accurately and I'm not comfortable working on marketing that could break medicines advertising rules. If you can show you're authorised to sell and advertise these products, or the range and wording change, I'm glad to continue. Otherwise I'll hand back everything done so far and remove my access. Please get legal advice about the products and advertising, and change the passwords you shared with me.
