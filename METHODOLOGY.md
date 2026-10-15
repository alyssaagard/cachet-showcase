# How I built Cachet

Alyssa I. Agard

This document explains how Cachet went from a list of requirements to a production application: how I worked, how it is designed and why, how it is secured, tested, and deployed, and what went wrong along the way. Cachet's source code is private and is not reproduced here.

## Contents

1. [The problem and the requirements](#1-the-problem-and-the-requirements)
2. [How I worked](#2-how-i-worked)
3. [Architecture](#3-architecture)
4. [Key design decisions](#4-key-design-decisions)
5. [Security](#5-security)
6. [Testing and quality](#6-testing-and-quality)
7. [Deployment and operations](#7-deployment-and-operations)
8. [What went wrong, and what I learned](#8-what-went-wrong-and-what-i-learned)
9. [Results and what comes next](#9-results-and-what-comes-next)

---

## 1. The problem and the requirements

Agard Research Associates Inc. works across several email domains, with shared mailboxes for teams and functions. Staff use Outlook on the web and on their phones. Every signature must carry the organization's identity correctly, on every device and from every mailbox, without relying on each person to configure it.

Before writing any code, I set out what administrators should be able to do:

- Build signatures from customizable templates, including templates designed from scratch.
- Offer a default set of optional details: name, title, phone number, website, a written legal disclaimer or a link to one, social and booking links (as text or uploaded icons), and an uploaded logo.
- Resize images, and reorder, show, or hide every field and image, in an editor as familiar as Outlook on the web's signature editor.
- Support a quick fill-in mode, a visual editor, and an HTML option for advanced layouts.

The real situation at Agard Research Associates Inc. added requirements I had not anticipated:

- **Signatures must be automatic everywhere**, including Outlook on iPhone and Android and shared mailboxes.
- **Replies need a different signature** from new messages, and meeting invitations may need their own.
- **Different groups need different signatures**: by email domain, by Microsoft 365 group, and occasionally by person.
- **Details must come from the directory**, so a title change in Microsoft 365 reaches the signature without retyping.
- **Nothing may depend on staff behavior**, so a server-side safety net must cover messages the add-in cannot reach.

Rather than three separate modes, the resulting editor is block-based: text lines, link lines, icons, paragraphs, spaces, dividers, a logo, and an optional custom HTML block. Together these cover the fill-in, visual, and HTML use cases in one place.

## 2. How I worked

### Ownership of the work

Cachet is the first application I have built, and I built it myself. I wrote the requirements, set the priorities, wrote the code, made every design, security, hosting, and rollout decision, tested each feature in the admin site and in Outlook, and ran every deployment from my own computer under my own Microsoft and Azure accounts.

### Working practices

- **Secrets stay out of code.** Passwords, keys, and connection strings live in one local settings file that version control ignores.
- **Commit after every milestone**, so every working state is recoverable.
- **Pure functions first.** Core logic, such as filling in details or deciding who gets which signature, was written as small, tested functions before it was connected to Microsoft's interfaces. This kept the hardest debugging inside code that could be tested in isolation.

### Planning

The work spanned many sessions between May and October 2026. A written roadmap held the current state, the next steps, and a log of decisions with their reasons, so each session started from an accurate picture. Major decisions, such as hosting on Azure or keeping Cachet proprietary, were logged along with the alternatives I rejected.

## 3. Architecture

Cachet has three parts: an admin site where templates are designed and assigned, a signature service that answers the add-in, and an Outlook add-in that runs inside each person's mail. All three are served from one container at one web address.

```mermaid
flowchart LR
  admin["Administrator"] -->|"designs and assigns templates"| site["Cachet admin site"]
  site --> db[("PostgreSQL")]
  site -->|"read-only directory sync"| graph["Microsoft Graph"]
  person["Person writing an email"] --> outlook["Outlook: web, desktop, phone"]
  outlook -->|"starts automatically"| addin["Cachet add-in"]
  addin -->|"request with a Microsoft-issued token"| api["Cachet signature service"]
  api --> db
  api -->|"email-safe signature"| addin
  addin -->|"inserts it"| outlook
  rules["Exchange Online mail rules"] -.->|"sign anything the add-in missed"| sent["Outgoing mail"]
```

**What happens when someone starts an email:**

1. Outlook starts the add-in automatically when a message, reply, forward, or meeting invitation is created, or when the From address changes.
2. The add-in obtains a sign-in token for the person from Microsoft and sends it to the signature service.
3. The service verifies that the token was issued and signed by Microsoft, is meant for Cachet, is unexpired, and comes from an organization Cachet serves.
4. It identifies the person by Microsoft account ID, selects the applicable template, and fills in their details.
5. It returns the signature as email-safe HTML and plain text, and the add-in inserts it.

**Technology:** TypeScript with Next.js and React for the admin site and service; PostgreSQL for data; Microsoft Entra ID for administrator sign-in; Microsoft Graph for the directory; the Office add-in platform for Outlook; and Azure for production hosting.

## 4. Key design decisions

### An add-in first, with server rules as the backup

Microsoft 365 offers two ways to add signatures: an Outlook add-in that inserts the signature while the person writes, or Exchange mail rules that append it after sending. Rules are simple, but the sender never sees the signature, and it lands below quoted replies. I chose the add-in as the main method because people send exactly what recipients see, with the signature where it belongs. Exchange rules remain as a safety net for cases the add-in cannot reach.

### Event-based activation, so it works on phones

Outlook add-ins can be opened by a button or started by events. Cachet uses events (new message, new meeting invitation, and From address changed), which makes insertion automatic and allows it to run in Outlook for iPhone and Android, where staff write much of their mail. Phones support new messages and From changes; meeting invitations are handled on the web and desktop. Automatic insertion requires central deployment by an administrator, which is how Cachet is rolled out.

### A block-based template model instead of free-form HTML

Free-form HTML is flexible but hard for non-technical administrators to edit, easy to break, and risky to store. Cachet's templates are structured data: lines of text and links, images, spaces, dividers, a logo, and paragraphs for text such as a disclaimer. This structure enables several features:

- **Details fill in safely.** Placeholders such as `{{title}}` are replaced with each person's values, escaped so no value can inject markup.
- **Empty details disappear cleanly.** If a person has no mobile number, the mobile item and its separator are removed rather than leaving "Mobile:" with nothing after it.
- **The output is safe.** Links are limited to web, email, and phone addresses, and custom HTML blocks are sanitized before saving.

### Email-safe rendering

Mail programs, especially Outlook for Windows, ignore most modern web layout. Cachet therefore renders signatures with tables and inline styles only, no external style sheets, and widely installed fonts with fallbacks. Uploaded WebP and SVG logos are converted to PNG, which more mail programs can display. I chose not to support custom font uploads, because mail programs do not load web fonts and the signature would look different for every recipient.

### Deciding who gets which signature

Each person's signature is resolved in a fixed order, and the first rule that applies wins:

| Order | Rule | Example |
|---|---|---|
| 1 | The person's own setting | A director who needs a booking link |
| 2 | Their Microsoft 365 groups, in an order the administrator sets | Everyone in the Fellows group |
| 3 | Their email domain's default | A second domain for a program |
| 4 | The organization default | Everyone else |

New messages and replies are resolved separately, meeting invitations follow an organization-wide setting, and "no signature" is valid at every level. Predictability matters more than flexibility: an administrator can always answer "why did this person get that signature?", and the admin site shows the reason next to each person.

### Details from the directory, with administrator overrides

Once an administrator approves read-only access, Cachet reads people, titles, departments, phone numbers, groups, and shared mailboxes from Microsoft Graph, on demand or through a scheduled sync built for a nightly job. Rooms and equipment are skipped, and departed staff are deactivated. Administrators can override or hide any synced detail, and syncing never overwrites that choice. Organizations can also add custom fields, such as a booking link or a LinkedIn address.

### One container, one address

The admin site, signature service, and add-in files all come from one container at one web address. The add-in never calls across domains, there is one certificate and one deployment, and the same build serves any organization with only its options and settings changed.

### Built for multiple organizations from the start

Although Agard Research Associates Inc. is currently the only organization using Cachet, every record belongs to an organization, and the application limits each request to the signed-in organization. Row-level security is enabled on every table, which shuts out any database account other than Cachet's own; enforcing organization boundaries inside the database is planned for the commercial version. Each Cachet site lists the Microsoft 365 organizations it serves and refuses all others. Building this in from the start means a commercial version will not require rewriting the data model.

### Ownership

Cachet is proprietary. I hold all rights to it personally, and Agard Research Associates Inc. uses it with my permission. The source code stays in a private repository; this public repository contains only the showcase.

## 5. Security

Security was built into each feature rather than added at the end:

- **Every add-in request is verified.** Requests must carry a Microsoft-issued token, and the service checks its signature, issuer, audience, permission, and expiry before looking anything up. People are identified by Microsoft account ID, not by an email address the request could claim.
- **Organizations are isolated.** Every record belongs to one organization, and every page, action, and signature request is limited to it.
- **Only listed organizations get in.** A production site with no list of allowed organizations admits no one, so a misconfiguration fails closed.
- **Least privilege.** Directory access is read-only. Cachet never holds Exchange administrator rights; the backup mail rules come from a PowerShell script that an administrator reviews and runs.
- **Safe content.** Template content is validated on save, links are limited to web, email, and phone addresses, values are escaped when filled in, and admin previews are sandboxed.
- **Abuse limits.** The signature service caps requests per network address and per mailbox each minute, well above normal use.
- **Secrets stay out of code.** Production secrets live in the hosting platform's secret store and in one local file that version control ignores; the hosting setup never prints them.

### A spoofing review that changed the design

Before enabling the Exchange backup rules, I reviewed how Cachet behaves when someone forges an email. The review found a real problem: the rules matched messages by the From address, which an outsider can forge. A forged message from outside, claiming to be from a staff member, would have received that person's genuine signature on its way to other staff, making a phishing attempt more convincing. Before any rules were created, I changed them to apply only to mail sent from inside the organization over an authenticated connection. The same review confirmed that outsiders cannot trigger the add-in at all.

## 6. Testing and quality

- **Automated tests.** About 100 tests cover the core logic: filling in details, removing empty items, resolving templates, sanitizing content, verifying add-in tokens, enforcing allowed organizations, rate limits, image storage, starter designs, and the generated Exchange script. The script runs in PowerShell against stand-ins for Exchange commands, so it is tested without touching a real mail system.
- **One workflow for every check.** Linting, type checking, tests, and production builds of the admin site and add-in run as one workflow, ready to switch on for continuous integration on GitHub.
- **Production builds tested before hosting.** The container build, the assembled server on an empty database, and image storage against Microsoft's storage emulator were all tested before the first deployment.
- **End-to-end checks in Outlook.** Each stage was confirmed by hand in Outlook on the web. A test copy of the add-in, pointed at the live site, inserted my signature into a new message before the add-in was deployed to anyone else.
- **Safe test data.** Development data can be backed up, reset, and restored with scripts that require a recent backup and the organization's typed name before deleting anything. The interface and tests use neutral example names rather than real staff.

## 7. Deployment and operations

Cachet runs in Agard Research Associates Inc.'s own Azure subscription, covered by Microsoft's nonprofit Azure grant:

- **Azure Container Apps** runs the container, using a managed identity instead of stored passwords to reach the image registry and storage.
- **Azure Container Registry** builds the container image in the cloud from source.
- **Azure Database for PostgreSQL** holds the data, with seven days of automatic backups.
- **Azure Blob Storage** keeps logos private; Cachet serves them itself.
- **A custom web address** carries a certificate that Azure manages and renews.

One setup script, run from my computer step by step, creates the whole environment: infrastructure, sign-in registration, build, app, domain, status, and logs. Every step can be rerun safely, skips what already exists, saves a log without secrets, and stops with a plain explanation when a decision is needed. Database changes apply automatically when a new version starts, and a health check confirms the service can reach its database.

The add-in is deployed through the Microsoft 365 admin center (Integrated apps), first to me alone and next to the whole organization. Before deploying it to anyone else, I verified the live site end to end with a separate test copy in my own mailbox.

## 8. What went wrong, and what I learned

Most of what I learned came from things that did not work the first time.

**A template's leftovers.** The first version of the add-in came from Microsoft's starter template and appeared in Outlook labeled "Contoso", because its manifest still carried the template's sample labels. Every generated sample name and setting has to be replaced. Later, Microsoft 365 kept serving an outdated copy of the add-in's description after two updates. The fix was to give the production add-in its own identity, separate from the development copy, so the two could never be confused.

**Sign-in behind a proxy.** The first production sign-in redirected to the container's internal address instead of the public one, because the hosting platform sits behind a proxy. Setting the site's public address explicitly fixed it, and that setting is now part of the documented configuration.

**A production outage.** On the second day of production, removing the old copy of the add-in in the Microsoft 365 admin center also deleted the sign-in registration the live site depended on. Microsoft 365 had created that registration for the add-in, and my setup had adopted it for the website. Sign-in failed for both the site and the add-in for about two and a half hours. I restored the registration from Microsoft Entra ID's deleted items and re-approved its permissions with my setup script, which restored sign-in with the same settings. I then:

- documented the lesson: Microsoft 365 creates and deletes its own registration for each custom add-in, so production sign-in must never depend on one;
- changed the setup script to stop with restore instructions when a saved registration disappears, instead of quietly creating a replacement the running site would not use;
- began moving production sign-in to a registration that only my setup script manages.

**A command that did not do what its name said.** During that move, the Azure command for creating an app registration updated the existing one instead, because it had the same name. Nothing broke, but the move had not happened. The script now renames the old registration first and refuses to continue unless it receives a genuinely new one. The broader lesson: verify the result of every administrative command, not just that it finished.

**Security findings before they mattered.** The spoofing review in [section 5](#a-spoofing-review-that-changed-the-design) caught a flaw before the feature was switched on. Reviewing each feature from an attacker's point of view before launch is now part of how I work.

Three habits came out of these experiences: test changes against something real before they reach people, verify the outcome of every change rather than trusting the command, and build each lesson into the tools so the same failure cannot recur.

## 9. Results and what comes next

**Today.** Cachet is live at Agard Research Associates Inc. Microsoft 365 is connected and synced, the organization's templates are built, and signatures are inserted automatically in Outlook on the web. The add-in is deployed through the Microsoft 365 admin center and is rolling out to more than 40 mailboxes across four email domains.

**Next for Agard Research Associates Inc.:** completing the rollout, confirming signatures on iPhone and Android, scheduling the nightly directory sync, and enabling the Exchange safety net.

**Longer term.** I plan to offer Cachet commercially through a company I am forming, listed on Microsoft's marketplace, with Gmail support for organizations on Google Workspace. The multi-organization design, the allowed-organization list, and the single container were built with that in mind.

**What it shows.** Cachet began as a practical problem at an organization I lead. Solving it required learning software development, identity and directory services, cloud hosting, and email security, none of which I had done before. The result is a production application that makes an organization's email more efficient, consistent, and compliant, along with a process for building, securing, and operating software that I can carry into any professional setting.
