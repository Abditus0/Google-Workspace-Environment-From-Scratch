# Google Workspace Environment From Scratch

A Google Workspace setup from a blank domain to a fully working email environment. New domain, DNS locked down, users created, security enforced, and a mailbox migrated over from Microsoft 365.

I wanted to see if I could build the whole thing end to end without skipping any of the parts that usually get glossed over. Most guides stop at "verify your domain and add users". The interesting work is everything around it. SPF, DKIM, DMARC, TLS-RPT, 2FA policies, device management, OAuth controls, mailbox migration. That's where this project lives.

## What I built

A new domain bought from Porkbun, verified in Google, with full DNS authentication set up. MX records pointing to all five Google mail servers. SPF locking down who can send as the domain. DKIM signing every outgoing email. DMARC tying it together with a quarantine policy. TLS-RPT for delivery failure notifications. A dedicated mailbox just for the reports.

Four user accounts created with 2FA enforced and SMS turned off. Android management with auto-wipe rules. iOS push certificate configured. OAuth locked down so users can't connect random third-party apps.

Then a real mailbox migration from Microsoft 365 to prove the whole thing works end to end. Emails, contacts, calendar events, all moved over.

## The domain

Bought `example-domain.space` from Porkbun. Added the Google Workspace TXT record to DNS, verified ownership in the Admin Console. Standard stuff, no surprises here.

Before touching anything else I exported the current DNS records and saved a backup. Small DNS mistakes can break email for everyone, and you really don't want to be staring at a broken zone with no record of what it used to look like.

## DNS and email authentication

If MX, SPF, DKIM, or DMARC aren't right, email either doesn't deliver, lands in spam, or gets spoofed by anyone who feels like it.

### MX records

| Type | Host | Priority | Value | Role |
|------|------|----------|-------|------|
| MX | @ | 1 | aspmx.l.google.com | Primary |
| MX | @ | 5 | alt1.aspmx.l.google.com | Backup 1 |
| MX | @ | 5 | alt2.aspmx.l.google.com | Backup 2 |
| MX | @ | 10 | alt3.aspmx.l.google.com | Backup 3 |
| MX | @ | 10 | alt4.aspmx.l.google.com | Backup 4 |

Five servers because if the primary is unreachable for any reason, mail still flows. Lower priority number gets tried first.

### SPF

```
v=spf1 include:_spf.google.com -all
```

Says only Google is allowed to send mail for this domain. The `-all` at the end is the important part. It means anything else trying to send as the domain hard fails. Without SPF, anyone on the internet can send email pretending to be you and there's nothing to stop it.

### DKIM

Google generates the DKIM key in the Admin Console. I published it under `google._domainkey.example-domain.space` and turned on signing for all users. Every outgoing email now gets a cryptographic signature that the receiving server can verify. If the signature doesn't match, the email gets flagged.

### DMARC

```
v=DMARC1; p=quarantine; rua=mailto:dmarc-reports@example-domain.space; ruf=mailto:dmarc-reports@example-domain.space; pct=100
```

DMARC ties SPF and DKIM together and tells receiving servers what to do when an email fails authentication. `p=quarantine` sends suspicious mail to spam. `pct=100` applies the policy to every single message. The `rua` and `ruf` addresses are where authentication reports get sent, which is why I made the dedicated monitoring mailbox.

### TLS-RPT

```
_smtp._tls: v=TLSRPTv1; rua=mailto:dmarc-reports@example-domain.space
```

Optional but worth setting up. It tells you if other servers fail to deliver mail to you over TLS. If something is wrong with encrypted delivery, you find out from the reports instead of from a user complaining their email never arrived.

## Users and security

Four accounts created:

- `support@example-domain.space` for helpdesk
- `info@example-domain.space` for general inquiries
- `sales@example-domain.space` for sales
- `monitoring@example-domain.space` for the DMARC and TLS reports

Security got locked down before anyone touched a mailbox.

**2FA enforced for everyone.** Authenticator apps and security keys only. SMS turned off because SIM swap attacks are real and SMS-based 2FA is one of the weakest forms of MFA you can use. If someone's phone number gets ported by an attacker, SMS 2FA might as well not exist.

<img width="928" height="664" alt="2FA enforcement settings" src="https://github.com/user-attachments/assets/d05bb0d2-7395-4893-ab1e-888a3f70c85b" />

**Android management on advanced.** Work profile password required, devices auto-wipe if inactive for 30 days. A lost phone with cached email and Drive access is a problem. A lost phone that wipes itself isn't.

**iOS Apple Push Certificate configured.** No devices enrolled yet but the certificate is in place so mobile management can be turned on without redoing setup.

**OAuth app access restricted.** Users can't connect random third-party apps to their Workspace account. They can request access if they need something specific. This stops the classic "I clicked allow on a sketchy app and now it has my entire mailbox" problem.

## Groups and aliases

I also tested groups and aliases to see how the routing actually behaves.

Created a Google Group at `team@example-domain.space`, added a couple of users, sent a test email. Everyone got it. Changed the access settings so external senders weren't blocked. Aliases on the admin account pointed extra addresses (`admin1@`, `admin2@`) at the main inbox.

Nothing groundbreaking, but knowing how groups and aliases route mail matters once you have more than four users to manage.

## Mailbox migration from Microsoft 365

I migrated the admin mailbox from Microsoft 365 using the Google Workspace Data Migration Tool.

Steps were straightforward. Open Data Migration in the Admin Console, connect the Microsoft 365 account, pick the source mailbox (`admin@old-domain.com`), pick the destination (`admin@example-domain.space`), kick off the migration. Emails, contacts, and calendar events all came over.

<img width="1290" height="711" alt="Microsoft 365 to Google Workspace migration" src="https://github.com/user-attachments/assets/f5849350-e02a-4145-b83d-bfb841d38a88" />

I migrated the admin account first on purpose. If something is going to break, it'll break here, and I'd rather find out before doing it for actual users.

## Testing and verification

Once everything was in place I went through a full test pass.

- MXToolbox to verify SPF, DKIM, and DMARC were all passing
- Sent test mail out from every account, confirmed delivery
- Sent mail in from external addresses, confirmed delivery
- Logged into every account via web and mobile
- Checked the monitoring mailbox for DMARC reports coming back

All four accounts working. Admin mailbox fully migrated. Authentication passing across the board.

## Problems I had to solve

**Some emails landing in spam.** Brand-new domains have no sending reputation, so even with perfect DNS the first wave of emails can get filtered. This isn't something you fix with config. It resolves on its own over a week or two of normal sending. Worth knowing in advance so you don't go chasing a problem that isn't really there.

**One user couldn't log in after 2FA enforcement.** Something went sideways during initial setup. I temporarily disabled 2FA enforcement just for that account, got the login working, then re-enabled it. The takeaway is to make sure every user can actually log in once before flipping enforcement on for the whole org. Otherwise you end up locking people out before they've ever used the account.

**iOS device management partial.** Apple Push Certificate is configured but no devices are enrolled yet. The certificate has to be in place before any iOS device can be managed, so getting it set up now means the option is there when it's needed. Without it, the first device enrollment becomes a much bigger task.

## What I learned

The thing that stuck with me most is how much of email security lives in DNS. SPF, DKIM, DMARC, TLS-RPT. None of it is in the Workspace admin panel, and getting any of it slightly wrong silently breaks delivery or leaves the domain wide open to spoofing. Most people set up MX records and call it done. That's the bare minimum.

Other practical things that came out of this:

- DMARC reports are genuinely useful once you read them. You can see who is trying to send as your domain.
- SMS 2FA is so weak that turning it off was an obvious call, but a lot of organizations still leave it on by default.
- The Google Data Migration Tool just works really well. I went in expecting issues and didn't really hit any.
- The first week of a new domain is rough on deliverability and there's not much you can do about it.

## Why I built it

I wanted to know what setting up a Workspace environment actually involves end to end. Not the "click verify domain and add users" version, the real one with DNS authentication, security policies, device management, and an mailbox migration on top.
