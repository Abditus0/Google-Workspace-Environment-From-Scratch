# Google Workspace Environment From Scratch (Domain, DNS, Security, Migration)  

## Project Overview  

I wanted to see if I could build a full Google Workspace setup from scratch, starting with a brand-new domain, setting up DNS and security, adding users, and moving a mailbox from Microsoft 365. No clients, no shortcuts, just a clean, fully working Workspace setup that I could use as a portfolio project.

In this guide, I’ll walk through how I did everything, the problems I ran into, and the useful things I learned along the way. It’s basically a mix of a tutorial and a behind-the-scenes look at a real admin setup.  

## Requirements  

- Google Workspace account (I used the free trial)  
- Access to a domain registrar (I used Porkbun)  
- Admin access to a Microsoft 365 mailbox for migration  
- Understanding of DNS and email authentication  

## Best Practices  

Before getting into it, here are a few habits I always stick to:

- I always back up DNS records before making any changes  
- I double-check SPF, DKIM, and DMARC entries. Even a small typo can break email delivery  
- I keep a separate mailbox just for DMARC and TLS reports  
- I test everything step by step before going fully live   

## Step 1: Domain Registration  

I started by registering a brand new domain: example-domain.space  
What I did:  
- Purchased and verified the domain in Porkbun.  
- Added the Google Workspace TXT verification record.  
- Confirmed domain ownership in the Admin Console.  

## Step 2: DNS Configuration (Making Email Actually Work)  

Once the domain was verified, I moved on to setting up all the DNS records to make email actually work and stay secure. This part is super important because if MX, SPF, DKIM, or DMARC aren’t set up right, emails can bounce, end up in spam, or even get spoofed. Here’s how I handled it:  

## MX Records (The “Mail Delivery Map”)  

MX records tell the internet where emails for your domain should go. Without them, nobody can send you email.  

Type | Host |Priority | Value                   | What it does            |
---- | ---- |-------- | ----------------------- | ----------------------- |
MX   | @    | 1       | aspmx.l.google.com      | Main Google mail server |
MX   | @    | 5       | alt1.aspmx.l.google.com | Backup server 1         |
MX   | @    | 5       | alt2.aspmx.l.google.com | Backup server 2         |
MX   | @    | 10      | alt3.aspmx.l.google.com | Backup server 3         |
MX   | @    | 10      | alt4.aspmx.l.google.com | Backup server 4         |

Why multiple servers?  
Google recommends multiple MX servers so if one is unreachable, email still flows. Priority numbers control which server gets tried first.  

## SPF Record (Preventing Email Spoofing)  

v=spf1 include:_spf.google.com -all  
SPF (Sender Policy Framework) is like a “mail passport”: it tells other mail servers which servers are allowed to send emails for your domain.  
- include:_spf.google.com → Google is allowed to send emails for this domain  
- -all → Anything else trying to send emails fails SPF checks

It's important because without SPF, spammers could send email pretending to be you. This is the first layer of email authentication.  

## DKIM (Signing Your Emails) 

DKIM (DomainKeys Identified Mail) adds a cryptographic signature to every outgoing email. It proves the email really came from your domain and hasn’t been changed.  

- Google provides a DKIM key in the Admin Console  
- I published it in DNS under google._domainkey.example-domain.space  
- Activated DKIM signing for all users

That helps receiving mail servers can check that your emails are legit, which helps stop spoofing and makes sure your emails actually get delivered. 

## DMARC (Telling the World How to Handle Your Email)  

v=DMARC1; p=quarantine; rua=mailto:dmarc-reports@example-domain.space; ruf=mailto:dmarc-reports@example-domain.space; pct=100  

DMARC ties SPF + DKIM together and tells other servers what to do if an email fails authentication:  

- p=quarantine → suspicious emails go to spam  
- rua/ruf → reports get sent to your monitoring mailbox  
- pct=100 → applies to all emails

Important because DMARC protects your domain from being abused and gives you visibility into authentication failures.  

## TLS-RPT (Monitoring Encrypted Delivery)  

_smtp._tls: v=TLSRPTv1; rua=mailto:dmarc-reports@example-domain.space  

TLS-RPT is optional but very useful. It tells you if other mail servers fail to deliver emails securely over TLS, so you can spot misconfigurations or delivery problems early.  

## DNS Backup (Safety First)  

Before making any changes, I exported the current DNS records and saved a backup. That way, if anything went wrong, I could restore the previous state.  

Always backup your DNS. Small mistakes can break email for everyone.  
This step is often overlooked, but it’s the foundation of reliable email. If MX/SPF/DKIM/DMARC isn’t correct, nothing else in Workspace will matter.  

## Step 3a: User Accounts & Security  

Once the domain and DNS were set up, I created the user accounts and put in the basic security settings. This is the point where a Workspace environment starts to feel real.  
Accounts Created:  

- support@example-domain.space – for customer support or helpdesk  
- info@example-domain.space – general inquiries  
- sales@example-domain.space – sales communications  
- monitoring@example-domain.space – for DMARC/TLS reports and system notifications

Security Setup:  

- 2FA Enforcement: Two-factor authentication protects every account from unauthorized access.

  <img width="928" height="664" alt="520867655-81038c2d-2d18-4430-ba92-8f34aa5f5b2f" src="https://github.com/user-attachments/assets/d05bb0d2-7395-4893-ab1e-888a3f70c85b" />



- Allowed Methods: Authenticator apps and security keys (SMS disabled because SMS-based 2FA is vulnerable to SIM swapping attacks).   
- Android Management: Advanced management enabled. Requires work profile password and auto-wipes devices inactive for 30 days.  
- iOS Management: Apple Push Certificate configured. This allows devices to enroll later. No devices were enrolled yet.  
- OAuth App Controls: Limited third-party app access to only basic info; users can request approval for additional apps.  

Setting up security before users start sending or receiving emails makes sure their accounts are protected from day one. It also helps avoid common mistakes like weak passwords or letting insecure apps connect.  

## Step 3b: Groups & Aliases  

After creating users, I decided to test Google Workspace Groups and email aliases to see how they behave and how they can simplify email routing.  
What I did:  
- Created a test Google Group  
  - Example: team@example-domain.space  
  - Added a couple of users to the group.  
  - Change the default Access Settings   
  - Tested sending an email to the group. All members received it.  
- Added aliases to the main admin account  
  - Example aliases: admin1@example-domain.space, admin2@example-domain.space  
  - Emails sent to either alias landed in the main admin inbox.  
 
Groups are useful for distribution lists, team collaboration, and testing email routing without creating extra accounts.  
Aliases allow multiple email addresses to be routed to a single mailbox, reducing the number of actual accounts you need while keeping addresses flexible.  
Even though this was just for testing, knowing how to configure groups and aliases is important for managing a larger Workspace setup in the future.  

## Step 4: Migration from Microsoft 365  
I migrated one admin mailbox from Microsoft 365 using Google Workspace Data Migration Tool. Here’s how:       
- Log in to Google Workspace and search for Data Migration  
- Connect your Microsoft 365 Account  
- Selected the source mailbox (admin@old-domain.com) and destination mailbox (admin@example-domain.space).  
- Migrated emails, contacts, and calendar events.  
- Verified migration by logging into the Workspace admin account and checking for all emails and events.


  <img width="1290" height="711" alt="520867988-893b4da3-3437-487f-aa03-6a69e28c3500" src="https://github.com/user-attachments/assets/f5849350-e02a-4145-b83d-bfb841d38a88" />



Notes:  
- Some emails initially landed in spam. This is normal for brand-new domains without established sending reputation.  
- Migrating the admin mailbox first allowed me to confirm that core functionality worked before creating or migrating additional users.  

## Step 5: Testing & Verification  
After setting up users and migration, I ran multiple checks to ensure the environment worked correctly:  
- Verified SPF/DKIM/DMARC using MXToolbox. (https://mxtoolbox.com/SuperTool.aspx#) Type your domain.  
- Outbound: Emails sent successfully from all accounts.  
- Inbound: Emails received correctly.  
- Spam Issues: Some emails went to spam initially. This is expected for a brand-new domain. Reputation improves over 1–2 weeks of normal use.  
- All four accounts logged in successfully via web and mobile.  
- Admin mailbox fully migrated.  
- Even if DNS and migration are technically complete, testing ensures users can actually send and receive emails, and that security/authentication policies are working as intended.  

## Step 6: Known Issues  
Even in a controlled project, a few minor hiccups happened:  
- Spam filtering for new domain: Temporary. Brand-new domains have no sending history, so emails can land in spam. It resolves naturally as the domain builds reputation.  
- 2FA Temporary Disable: One user had login issues, so I temporarily disabled 2FA enforcement. Re-enabled once all accounts were working.  
- iOS Enrollment: Only the Apple Push Certificate was configured; no devices were enrolled. This allows mobile management in the future.  

## Step 7: Next Steps  
Even though the setup works, here’s what I would do next if this were a real client deployment:  
First Week:  
- Users log in and set up 2FA.  
- Admin reviews the security dashboard.  
- Test sending/receiving emails with common external contacts.  
- Check the monitoring@example-domain.space mailbox for DMARC and TLS reports.  
First Month:  
- Enroll mobile devices using Google Device Policy app.  
- Configure email signatures for all users.  
- Create Google Groups or Shared Drives if needed.  
- Continue checking deliverability and authentication reports.  

## What I Learned  
Doing this project helped me get better at:  

- Google Workspace Administration: From domain verification to user creation.  
- DNS & Email Authentication: MX, SPF, DKIM, DMARC, TLS-RPT. Everything required for secure mail.  
- Microsoft 365 → Workspace Migration: Using Google Workspace Migration Tool without issues for mailbox migration.  
- Security & Policy Enforcement: 2FA, device management, OAuth app controls.  
- Troubleshooting & Testing: Deliverability issues, fixing user login problems, verifying authentication.  
 

## License  
This project documentation is released under the MIT License. feel free to use, modify, and share for educational purposes.  
