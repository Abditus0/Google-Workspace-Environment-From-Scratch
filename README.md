# Google Workspace Environment From Scratch (Domain, DNS, Security, Migration)  

## Author: Ivaylo Atanassov  

### Project Overview  

I wanted to see if I could build a full Google Workspace environment from zero — starting with a brand-new domain, setting up DNS and security, adding users, and migrating a mailbox from Microsoft 365. No clients, no shortcuts — just a clean, fully functional Workspace setup that I could use as a portfolio project.
This guide walks you through how I did it, the problems I ran into, and the practical tips I learned along the way. Think of it as a mix of a tutorial and a behind-the-scenes log of a real admin setup.  

### Requirements  

- Google Workspace account (I used the free trial)  
- Access to a domain registrar (I used Porkbun)  
- Admin access to a Microsoft 365 mailbox for migration  
- Understanding of DNS and email authentication  

### Best Practices  

Before diving in, a few habits I stick to:  

- Always backup DNS records before making changes  
- Double-check SPF/DKIM/DMARC entries — small typos break email delivery  
- Keep a monitoring mailbox for DMARC/TLS reports  
- Test everything in small steps before moving fully live  

### Step 1: Domain Registration  

I started by registering a brand new domain: example-domain.space  
What I did:  
- Purchased and verified the domain in Porkbun.  
- Added the Google Workspace TXT verification record.  
- Confirmed domain ownership in the Admin Console.  

### Step 2: DNS Configuration — Making Email Actually Work  

Once the domain was verified, I moved on to setting up all the DNS records needed to make email reliable and secure. This step is crucial: if MX, SPF, DKIM, or DMARC aren’t correct, your emails might bounce, land in spam, or be spoofed by attackers. Here’s how I tackled it:  

### MX Records — The “Mail Delivery Map”  

MX records tell the internet where emails for your domain should go. Without them, nobody can send you email.  

| Priority | Server                  | What it does            |
| -------- | ----------------------- | ----------------------- |
| 1        | aspmx.l.google.com      | Main Google mail server |
| 5        | alt1.aspmx.l.google.com | Backup server 1         |
| 5        | alt2.aspmx.l.google.com | Backup server 2         |
| 10       | alt3.aspmx.l.google.com | Backup server 3         |
| 10       | alt4.aspmx.l.google.com | Backup server 4         |

Why multiple servers?  
Google recommends multiple MX servers so if one is unreachable, email still flows. Priority numbers control which server gets tried first.  

### SPF Record — Preventing Email Spoofing  

v=spf1 include:_spf.google.com -all  
SPF (Sender Policy Framework) is like a “mail passport”: it tells other mail servers which servers are allowed to send emails for your domain.  
- include:_spf.google.com → Google is allowed to send emails for this domain  
- -all → Anything else trying to send emails fails SPF checks

Why it matters: Without SPF, spammers could send email pretending to be you. This is the first layer of email authentication.  

### DKIM — Signing Your Emails 

DKIM (DomainKeys Identified Mail) adds a cryptographic signature to every outgoing email. It proves the email really came from your domain and hasn’t been tampered with.  

- Google provides a DKIM key in the Admin Console  
- I published it in DNS under google._domainkey.example-domain.space  
- Activated DKIM signing for all users

Result: Receiving mail servers can verify your emails, which helps prevent spoofing and increases deliverability.  

### DMARC — Telling the World How to Handle Your Email  

v=DMARC1; p=quarantine; rua=mailto:dmarc-reports@example-domain.space; ruf=mailto:dmarc-reports@example-domain.space; pct=100  

DMARC ties SPF + DKIM together and tells other servers what to do if an email fails authentication:  

- p=quarantine → suspicious emails go to spam  
- rua/ruf → reports get sent to your monitoring mailbox  
- pct=100 → applies to all emails

Why it matters: DMARC protects your domain from being abused and gives you visibility into authentication failures.  

### TLS-RPT — Monitoring Encrypted Delivery  

_smtp._tls: v=TLSRPTv1; rua=mailto:dmarc-reports@example-domain.space  

TLS-RPT is optional but very useful. It tells you if other mail servers fail to deliver emails securely over TLS, so you can spot misconfigurations or delivery problems early.  

### DNS Backup — Safety First  

Before making any changes, I exported the current DNS records and saved a backup. That way, if anything went wrong, I could restore the previous state.  

Pro Tip: Always backup your DNS — small mistakes can break email for everyone.  
This step is often overlooked, but it’s the foundation of reliable email. If MX/SPF/DKIM/DMARC isn’t correct, nothing else in Workspace will matter.  

### Step 3a: User Accounts & Security  

After the domain and DNS were ready, I created the user accounts and set up the security baseline. This is where a Workspace environment starts feeling real.  
Accounts Created:  

- support@example-domain.space – for customer support or helpdesk  
- info@example-domain.space – general inquiries  
- sales@example-domain.space – sales communications  
- monitoring@example-domain.space – for DMARC/TLS reports and system notifications

Security Setup:  

- 2FA Enforcement: Two-factor authentication protects every account from unauthorized access.  
- Allowed Methods: Authenticator apps and security keys (SMS disabled for security).  
- Android Management: Advanced management enabled. Requires work profile password and auto-wipes devices inactive for 30 days.  
- iOS Management: Apple Push Certificate configured. This allows devices to enroll later — no devices were enrolled yet.  
- OAuth App Controls: Limited third-party app access to only basic info; users can request approval for additional apps.  

Why this matters: Setting security before users start sending/receiving emails ensures accounts are protected from day one, and helps prevent common mistakes like weak passwords or insecure app access.  

### Step 3b: Groups & Aliases  

After creating users, I decided to test Google Workspace Groups and email aliases to see how they behave and how they can simplify email routing.  
What I did:  
- Created a test Google Group  
  - Example: team@example-domain.space  
  - Added a couple of users to the group.  
  - Change the default Access Settings   
  - Tested sending an email to the group — all members received it.  
- Added aliases to the main admin account  
  - Example aliases: admin1@example-domain.space, admin2@example-domain.space  
  - Emails sent to either alias landed in the main admin inbox.  
Why it matters:  
- Groups: Useful for distribution lists, team collaboration, and testing email routing without creating extra accounts.  
- Aliases: Allow multiple email addresses to be routed to a single mailbox, reducing the number of actual accounts you need while keeping addresses flexible.  
Tip: Even though this was just for testing, knowing how to configure groups and aliases is essential for managing a larger Workspace setup in the future.  

### Step 4: Migration from Microsoft 365  
I migrated one admin mailbox from Microsoft 365 using Google Workspace Data Migration Tool. Here’s how I approached it:    
Process:    
- Log in to Google Workspace and search for Data Migration  
- Connect your Microsoft 365 Account  
- Selected the source mailbox (admin@old-domain.com) and destination mailbox (admin@example-domain.space).  
- Migrated emails, contacts, and calendar events.  
- Verified migration by logging into the Workspace admin account and checking for all emails and events.  
Notes:  
- Some emails initially landed in spam — this is normal for brand-new domains without established sending reputation.  
- Migrating the admin mailbox first allowed me to confirm that core functionality worked before creating or migrating additional users.  

### Step 5: Testing & Verification  
After setting up users and migration, I ran multiple checks to ensure the environment worked correctly:  
Email Deliverability:  
- Outbound: Emails sent successfully from all accounts.  
- Inbound: Emails received correctly.  
- Spam Issues: Some emails went to spam initially — this is expected for a brand-new domain. Reputation improves over 1–2 weeks of normal use.  
Account Access Verification:  
- All four accounts logged in successfully via web and mobile.  
- Admin mailbox fully migrated.  
- Why it matters: Even if DNS and migration are technically complete, testing ensures users can actually send and receive emails, and that security/authentication policies are working as intended.  

### Step 6: Known Issues  
Even in a controlled project, a few minor hiccups happened:  
- Spam filtering for new domain: Temporary. Brand-new domains have no sending history, so emails can land in spam. It resolves naturally as the domain builds reputation.  
- 2FA Temporary Disable: One user had login issues, so I temporarily disabled 2FA enforcement. Re-enabled once all accounts were working.  
- iOS Enrollment: Only the Apple Push Certificate was configured; no devices were enrolled. This allows mobile management in the future.  

### Step 7: Recommended Next Steps  
Even though the setup works, here’s what I would do next if this were a real client deployment:  
First Week:  
- Users log in and set up 2FA.  
- Admin reviews the security dashboard.  
- Test sending/receiving emails with common external contacts.  
- Monitor the monitoring@example-domain.space mailbox for DMARC and TLS reports.  
First Month:  
- Enroll mobile devices using Google Device Policy app.  
- Configure email signatures for all users.  
- Create Google Groups or Shared Drives if needed.  
- Continue monitoring deliverability and authentication reports.  

### Skills Learned  
By completing this project, I strengthened my abilities in:  

- Google Workspace Administration: From domain verification to user creation.  
- DNS & Email Authentication: MX, SPF, DKIM, DMARC, TLS-RPT — everything required for secure mail.  
- Microsoft 365 → Workspace Migration: Using Google Workspace Migration Tool efficiently for mailbox migration.  
- Security & Policy Enforcement: 2FA, device management, OAuth app controls.  
- Troubleshooting & Testing: Identifying deliverability issues, fixing user login problems, verifying authentication.  

This project isn’t just about following steps — it’s about understanding why each step matters and being able to troubleshoot when something goes wrong.  

### License  
This project documentation is released under the MIT License - feel free to use, modify, and share for educational purposes.  
