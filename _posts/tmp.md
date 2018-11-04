


## Avoiding spam, malware and phishing

Spam, malware and phishing is something all system administrators (and end users) have become familiar with.

Email is a powerful advertising tool and an excellent attack vector. It's estimated that around 90% of all email messages sent, are unwanted. 

The *Verizon Data Breach Investigations Report (2018)* states that:

  - more than 90% of breaches start with phishing, almost exclusively done through email.
  - the frequency of malware vectors are 92.4% email, 6.3% web and 1.3% other.

Can you prevent criminals and spammers from obtaining your email addresses? Probably not, but publishing contact information (contact lists etc.) online will definitly make sure that the addresses are harvested in no time. 

To properly deal with email spam, phishing and malware, you need multiple countermeasures.

### User awareness

There are certainly benefits to having users that are aware of the dangers of the internet and email, and who are critical when deciding which atttachments to open and which URLs to visit. User awareness training has become very popular among a lot of companies. But aware users is *not* a substitute for any of the countermeasures discussed below. It is better to look at the user as a permanent vulnerability that you need to build layers of security on top of. Why? Because no matter how much time is spent on user awareness, the risk of some of them being tricked is to great to accept. One user making one mistake can be enough for a major breach to happen. But keep in mind that user awareness is never a bad thing!


### Dealing with Spam

Spam is garbage mail, like unsolicited advertising and scam emails.

Spam can be dealt with by using a cloud anti-spam service, running an anti-spam gateway, using anti-spam software on the email servers or using anti-spam tools on the clients.

A **cloud anti-spam service** will scan incoming (and often outgoing) emails and remove or quarantine spam. 
A portal is often available for the customer, where they can get an overview, release false positives, tune the detection, whitelist and blacklist, and schedule reports.
Cloud anti-spam services will typically not require any hardware or software, but requires a change in the DNS MX record and probably some light firewall configuration.

> Many cloud anti-spam services also offer end-user digest emails where users can release (safe) blocked emails addressed to them, freeing up a lot of time for administrators.

An **anti-spam gateway** does much of the same as the service of the cloud anti-spam provider. In fact, a lot of providers base their service on well-known open source or proprietary email gateways.
The gateway comes as software, a virtual appliance or as a hardware appliance. Using a both a cloud anti-spam service and an onsite anti-spam gateway is not very common.

**Anti-spam software** for email servers is typically installed onto the email server. The installed agent can often monitor inbound, outbound and internal transport, as well as perform real-time and scheduled scanning of the storage.

**Anti-spam tools** for clients may be part of the email client, or installed as a third party software. This is the last line of defense (apart from the user's awareness, which should never be completly trusted).

Combining some of these countermeasures - *layered security* - will of course result in a more effective protection.


### Dealing with malware

Malware is malicious software like trojans, worms and viruses (and cryptoviruses).

Most of the *anti-spam* countermeasures mentioned above, will also scan for malware.

A **cloud anti-spam service** can often scan email attachments and URL's to find harmful files or sites that spread malware.

> Some cloud anti-spam services also perform URL rewrites: when the user opens a URL in an email, the contents are first analyzed in a cloud sandbox, and then the user is granted or denied access - all in a fraction of a second.
> If URLs are only checked as the email comes in, the content can be harmless and the changed to something harmful at a later time.

An **anti-spam gateway** can often perform malware scanning on the emails passing trough it, but typically just attachments.

**Anti-spam software** for email servers can often scan both email transport and email storage for malware.

**Anti-spam tools** for clients is often part of a full-fledged anti-malware solution. As the last line of defense, these solutions should contain advanced detection capabilities such as behavioral analysis (to detect zero-day malware), and have a central management for visibility, reporting and configuration. Since the client is often mobile, and therefore at times not behind a lot of layers of security, a quality anti-malware solution should be chosen.

Again, combining some of these countermeasures - *layered security* - will of course result in a more effective protection.


### Dealing with phishing

Phishing is the use of social engineering techniques to trick users into for example giving away information or downloading malware.

The presence of malicious code or URLs, the results of a lexical analysis of the contents, the reputation of the sender domain and more can help solutions remove a lot of phishing emails. Some *anti-spam* solutions have more advanced *black box* methods to block phishing email. Though, if you look at real-world results, it seems that most solutions still let the more sophisticated spear-phishing attacks (sent to a few hand-picked recipients, with spoofed domain/sender, native language and no malicious code or URLs) pass through. How can you deal with these? If the solutions in places block all obvious phishing emails, emails containing malware and malcious URLs, and even check the contents of the URLs when the users click on them, then the majority of the phishing email you will have coming are probably of a *CEO-fraud* character.

A *CEO-fraud*, also known as "Business email compromise" (BEC), is a targeted attack. Most often, a phishing email is sent by an attacker pretending to be the CEO or other C-level executive to recipient(s) that are hand-picked based on the goal of the attacker (e.g. to employees in the accounts payable department if they are asking for a money transfer, or to the HR department if they are asking for personal data). These attacks have become more and more sophisticated, with the attacker apparently spending a lot of time selecting the proper targets, spoofing information, writing a phishing email with proper jargon and so on. 

There are a few ways to avoid BEC:

  - aware users are less likely to be tricked
  - proper control procedures can prevent inproper actions being taken
  - "out of office" auto-replies may be disabled or limited to prevent attackers exploiting this knowledge
  











