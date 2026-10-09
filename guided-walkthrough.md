# Guided Walkthrough — SOC Investigation (Absolute Beginner)

This guide walks you through the whole investigation one click at a time. If you follow
every step literally, you cannot fail. Do your own report before asking your instructor
for the answer key.

---

## Step 1 — Words you need to know

- **Alert** — a rule in the security tool that fires when something looks dangerous. It is a *hint*, not a *fact*. Investigators confirm or dismiss it.
- **Index / data view** — a named collection of log records with the same shape. We have `auth-*` (login and Kerberos events) and `network-*` (traffic events). In Kibana this is called a **data view** (older versions call it an index pattern).
- **NDJSON** — the text format the log data is stored in: one JSON object per line. Our log files are `auth.ndjson` and `network.ndjson`. "NDJSON" just means "Newline-Delimited JSON".
- **Index template** — a configuration that tells Elasticsearch which *type* every field has (that an IP field is an IP, that `event.code` is a number, that `service.name` is an exact keyword, etc.) before any data arrives. Our templates are `auth.json` and `network.json`. If you skip this step, Elasticsearch guesses and searches can silently break.
- **Elasticsearch** — the database that stores and searches the logs. It answers URLs like `http://localhost:9200`.
- **Kibana** — the web interface on top of Elasticsearch where you will do the investigation. It lives at `http://localhost:5601`.
- **Discover** — the Kibana screen where you type a search and read the matching log lines. We will spend all our time here.
- **KQL** — the query language you type into Kibana's search bar. `field: value` means "field equals value", and `and` joins two conditions.
- **IOC (Indicator of Compromise)** — a piece of evidence that something bad happened, e.g. an IP address, an account, a host. The workstation in this alert is an IOC.
- **Active Directory (AD)** — the system that holds all the accounts, the servers and the rules about who may touch what, in a Windows organisation. It is the thing being attacked here.
- **Domain controller** — the server that runs Active Directory's authentication service. When a computer wants to prove who someone is, it asks the domain controller.
- **Kerberos** — the protocol Windows uses to prove identity in an AD network. It works by handing out **tickets** instead of checking a password at every door.
- **Ticket** — a temporary, signed permission slip. Two kinds matter here:
  - the **TGT** (ticket-granting ticket) — your proof of identity, held by your own machine;
  - the **service ticket** — a slip for *one particular service*, handed out on request.
- **SPN (service principal name)** — the name of one particular service, written like `CIFS/finance-share.corp.local`. The first part is the *kind* of access (CIFS = file share, HTTP = web, HOST = computer account), the second is *which* machine. When a machine wants a service ticket, it names the SPN it wants. This is the field `service.name`.
- **Event 4769** — the Windows record written every time a service ticket is requested. In a real AD network this is one of the **most common records there is**, because every machine requests tickets all day long. So 4769 by itself tells you nothing; a *burst* of them from one machine tells you plenty.
- **Golden ticket** — a **forged TGT**. Normally only a domain controller can issue one. But the TGT is signed with a secret key belonging to a special built-in account called **`krbtgt`**, the domain's own ticket-signing key. If an attacker extracts that key — usually by dumping memory from a machine they already administrate — they can **mint a ticket for any user they like**, with whatever group memberships they like, valid for as long as they like. No password, no MFA prompt, no failed logon. The forged ticket simply *is* the credential, so nothing ever looks wrong. This is the attack in this lab.
- **Mimikatz** — the best-known tool for stealing Kerberos secrets (tickets, hashes and the `krbtgt` key) out of a running Windows machine's memory. It is what makes golden tickets practical.
- **"No prior baseline"** — the phrase used when a machine does something to a server it has **never** had any relationship with. No normal user logs into a server they do not work with, so it is strong evidence on its own.
- **Two-password reset (domain recovery)** — the standard response to a stolen `krbtgt` key. You change the domain admin password **twice**: the first change still produces a ticket signed with the *old* key, and only the second produces one signed with the new key. Doing it once does not fix anything.
- **False positive** — the alert fired but nothing bad actually happened.
- **True positive** — the alert fired and something bad actually did happen. Your job is to decide which one this is.

---

## Step 2 — What files you need

These **seven files** are the ones you need. They sit in your lab folder alongside
the setup scripts, a `README.md` and a browser version of this guide — you can
ignore those. Your instructor pre-generated the log data with a deterministic
generator, so you never need to regenerate or verify anything:

| File | What it is |
|---|---|
| `auth.ndjson` (284 documents) | The account and Kerberos log data: logons, failed logons, logoffs, lockouts, Kerberos service-ticket requests, process creations |
| `network.ndjson` (136 documents) | The network traffic log data: SMB, RPC, DNS, LDAP, HTTP/HTTPS, SSH, ICMP |
| `auth.json` | Index template that fixes the field types for the `auth-2026.09.22` index |
| `network.json` | Index template that fixes the field types for the `network-2026.09.22` index |
| `rule.json` | The detection rule that fired. If you used the setup file it is already installed; either way you only *read* it — you never run it yourself |
| `guided-walkthrough.md` | This guide |
| `instructions.md` | The shorter companion brief |

- **What to do:** keep all seven files together in one folder, open a terminal, and go
  into that folder. Every command below is run from there.
- **Type:** `ls`
- **What you should see:** the seven files above, plus others you can ignore — a `README.md`,
  the setup and teardown files, a browser version of this guide, and an answer key you are
  not to open. If any of the **seven** are missing, say so before you start.
- **Why this matters:** these two `.ndjson` files are the evidence dataset. Nothing in
  Kibana will appear until they are loaded into Elasticsearch (next step).

---

## Step 3 — Load the logs into Elasticsearch

> **Do this step only if you did *not* start the lab with the setup file.**
> If you already ran `setup-lab.cmd` (Windows) or `./setup-lab.sh` (macOS / Linux), that
> script has loaded the data, created the data views and installed the detection rule for
> you — **skip the rest of this step, then go to Step 4 to sign in** (you still need that) and
> on to Step 5. Running the import below as well would load every document a
> *second* time, giving you 568 documents instead of 284, and then every count in this
> guide would be wrong by a factor of two. The script refuses to do this to you; the manual
> commands below do not, so this is the one step worth reading twice.

This lab needs a real Elastic Stack (Elasticsearch + Kibana), and it has security
switched on. That means every command needs a login, and the browser will ask you for one
too. Use this account everywhere:

```
lab_kibana  /  LabKibana001
```

It is a throwaway lab account. It protects nothing, and you may type it anywhere.

Make sure Elasticsearch is running first:

- **What to do:** in the terminal, from your lab folder, run:
- **Type:** `curl -s -u lab_kibana:LabKibana001 localhost:9200/_cluster/health`
- **What you should see:** a JSON line with `"status":"green"` or `"status":"yellow"`.
  If it says you are not authenticated, the `-u lab_kibana:LabKibana001` part is missing.
  If the command fails to connect, start your Elasticsearch service first.
- **Why this matters:** the log data has nowhere to go until Elasticsearch answers.

**A1 — Install the two index templates** (mandatory: it fixes the field types so queries
behave correctly). The templates are in the same folder as the data:

- **Type:**
```bash
curl -s -u lab_kibana:LabKibana001 -XPUT localhost:9200/_index_template/auth    -H 'Content-Type: application/json' --data-binary @auth.json
curl -s -u lab_kibana:LabKibana001 -XPUT localhost:9200/_index_template/network -H 'Content-Type: application/json' --data-binary @network.json
```
- **What you should see:** both return `{"acknowledged":true}`.
- **Why this matters:** every field in the log is now typed (IPs as IPs, `event.code` as a
  number, `user.name` / `host.name` / `service.name` as exact keywords, `message` as
  text). Without this, Elasticsearch would guess, and searches like
  `event.code: 4769 and source.ip: "10.0.4.10"` could return wrong results.

**A2 — Import the data** (the bulk API needs one "action" line + one data line per
document, so we add the action line on the fly):

- **Type:**
```bash
for s in auth network; do
  while read -r l; do
    printf '%s\n' '{"index":{"_index":"'$s'-2026.09.22"}}'
    printf '%s\n' "$l"
  done < $s.ndjson | curl -s -u lab_kibana:LabKibana001 -XPOST localhost:9200/_bulk -H 'Content-Type: application/x-ndjson' --data-binary @-
done
```
- **What you should see:** two JSON responses from the bulk API, each ending in `"errors":false`.
- **Why this matters:** this copies every line of each `.ndjson` file into its own index
  (`auth-2026.09.22` and `network-2026.09.22`). The patterns you will use in Kibana are
  `auth-*` and `network-*`.
`printf '%s\n'` rather than `echo` on purpose: some log lines contain escaped quotes, and
zsh's `echo` treats a backslash as an escape character, so it quietly corrupts a document
and Elasticsearch rejects it. `printf '%s\n'` prints the line exactly as read, and behaves
identically in bash, zsh and sh.

- **Verify it worked**, then type — note the single quotation marks around the URL, they
  are required:
```bash
curl -s -u lab_kibana:LabKibana001 'localhost:9200/_cat/indices/auth*,network*'
```
- **What you should see:** both index names, with `284` and `136` documents. (Without the
  quotes your shell tries to expand the `*` and the command fails with
  `no matches found` — that is a shell quirk, not a broken lab.)
- **Continue with Step 4.**

---

## Step 4 — Sign in to Kibana and create the two data views

Kibana will not show any data until you sign in and tell it which indices exist.

- **What to do:** if you started the lab with the setup file, your browser is already open on
  the alerts page — carry straight on to Step 5. Otherwise open Kibana yourself.
- **Type in the address bar:** `http://localhost:5601`
- **What you should see:** a **sign-in** screen, because this lab's Kibana has security switched on.
- **Type:** in **Username** type `lab_kibana`, in **Password** type `LabKibana001`, then click **Log in**.
- **What you should see:** the Kibana home screen. You only do this once; the browser remembers you.
- **Type/click:** click the **hamburger menu** (☰) in the top-left corner, then **Stack Management**, then **Data Views**, then the **Create data view** button.
- **Type:** in **Name** type `auth`, in **Index pattern** type `auth-*`, and under **Timestamp field** choose `@timestamp`. Click **Save data view to Kibana**.
- **Repeat** for a second data view: **Name** `network`, **Index pattern** `network-*`, **Timestamp field** `@timestamp`, save it.
- **What you should see:** two data views in the list, `auth` and `network`. (If you used the setup
  file they are already there and this whole step can be skipped.)
- **Why this matters:** without these, the "Select the data view" step later in this guide has nothing to select, and every query would return *"no results"*. Getting this right now avoids hours of confusion later.

---

## Step 5 — Look at the alerts that started this

Before you search for anything, look at what the monitoring system already caught. This
is the alert your instructor was given, and you are about to work out whether it was
right.

- **What to do:** open the alerts screen.
- **Type in the address bar:** `http://localhost:5601/app/security/alerts`
- **If the table is empty, it is the time picker and not a broken lab.** An alert is stamped
  with the moment it was *raised* — minutes ago, because the rule runs every minute — and the
  range this page starts on (**Today**, meaning from midnight) stops covering it once you cross
  midnight. Click the time picker at the top right, switch to the **Relative** tab, and set it to
  **1** unit **Days**. Some versions offer a ready-made **Last 24 hours** button there instead;
  that does the same job. The alerts come straight back.
- **What you should see:** a table of alerts. Every row is one Kerberos service-ticket
  request that the rule in `rule.json` matched, one alert per request rather than one
  summary line. The severity is **critical**.
- **Type/click:** in the first column, called **Actions**, click the **first small icon** on
  any row — the two arrows pointing outwards. A panel opens on the right showing that one
  alert. Read its **Alert reason** (it names the source address and the account) and the
  **Highlighted fields**. For the raw log record underneath, click the **Table** tab at the
  top of the panel: that is where `event.code` (4769), `source.ip` and `service.name` live.
- **What you should notice:** all thirty came from **one address**, `10.0.4.10`, and all
  thirty were drawn for **one identity**, `Administrator`. And although there are thirty of
  them, they only ask for **ten different services** — three requests each. That is the
  shape of the attack handed to you: one machine, one administrative identity, and a
  deliberate sweep across ten different services. You have not run a single query yet.
- **One thing to be careful about:** the times on these rows are all within the same
  instant, because they are the time the *rule ran*, not the time the tickets were
  requested. The real event times are on 22 September; the **Table** tab will show them.
- **Type/click:** now open **Security → Rules** in the **hamburger menu** (☰). This is the
  page of detection rules (older builds call it "Detection rules"). Our rule is the only
  one you have, so it is under **Custom rules**; click the rule named "Burst of Kerberos
  service-ticket requests from a single workstation" to open it.
- **What you should see:** the rule, its severity, and the query it runs. Read the
  `query` line. It is the same filter you are about to type by hand.
- **Why this matters:** the alert count and the count you get from your own query have to
  be the same number. If they are not, one of the two is wrong, and that is worth knowing
  before you write a report. The rule also tells you which two fields matter: the event
  code and the source address.

---

## Step 6 — Open Discover

- **What to do:** open the Discover search screen.
- **Type/click:** click the **hamburger menu** (three horizontal lines ☰) in the top-left corner, then click **Discover** (it lives under "Analytics" in newer versions).
- **What you should see:** a search bar at the top, a field list on the left, and a list of documents below.
- **Why this matters:** Discover is where you will type your queries and read the answers.

---

## Step 7 — Select the auth data view

- **What to do:** tell Kibana which index to look at.
- **Type/click:** at the top of Discover, click the dropdown that shows the current index. In the list, choose **`auth`** (the data view you created in Step 4).
- **What you should see:** the index name at the top now reads `auth`, and the left sidebar shows fields like `event.code`, `event.action`, `user.name`, `host.name`, `source.ip`, `service.name`.
- **Why this matters:** the logon and Kerberos evidence all lives in `auth`. Wrong index → wrong answer. You will switch to `network` once, in Step 13.

---

## Step 8 — Set the time range

- **What to do:** make sure you are looking at the correct day.
- **Type/click:** click the **time picker** at the top right (it says something like "Last 15 minutes"). Click the **Absolute** tab. Set **From** to `Sep 22, 2026 @ 00:00:00.000` and **To** to `Sep 22, 2026 @ 23:59:59.999`. Click **Apply**.
- **What you should see:** the search field at the top now shows `@timestamp` ranging across the whole of September 22, 2026.
- **Why this matters:** all the relevant events happened on that day, in UTC. A too-narrow time window silently hides them. Use the whole day — you do not yet know the hour, and this activity happens in the evening.

---

## Step 9 — First query: every service-ticket request that day

- **What to do:** start broad, the way you would in a real triage.
- **Type (into the Discover search bar):** `event.code: 4769`
- **What the query means:**
  - `event.code: 4769` — keep only events of type "a Kerberos service ticket was requested". This is *every* ticket request on your estate for the day.
- **Type/click:** click the **field list icon** (the little table/columns button above the results) and add **`@timestamp`**, **`user.name`**, **`source.ip`**, and **`service.name`** as columns.
- **What you should see:** a **large** list, and a hit count in the top right. **Write that number down** — it is not your answer, and it is deliberately big.
- **What you should notice:** the `source.ip` column shows **many different machines**, and the `user.name` column shows **many different users**. Some of them repeat the same `service.name` over and over. That is exactly what normal Kerberos traffic looks like: every machine asking for the tickets it needs, all day, in no particular pattern.
- **Why this matters:** this step exists to kill a very common bad instinct. A student sees "Kerberos ticket anomaly" in an alert, queries `event.code: 4769`, gets a huge number, and either panics or dismisses it. **The event code is not the finding.** In any real AD network 4769 is background noise. What you are hunting for is a *shape* — one machine, a tiny time window, many different services. You cannot see that until you narrow to a source.

---

## Step 10 — Second query: the ticket requests from the alert's workstation

- **What to do:** now narrow to the one workstation the alert named. This is the same query the shorter companion brief (`instructions.md`) gives you as its single one-shot query.
- **Type:** `event.code: 4769 and source.ip: "10.0.4.10"`
- **What the query means, clause by clause:**
  - `event.code: 4769` — keep only "a Kerberos service ticket was requested" events.
  - `source.ip: "10.0.4.10"` — keep only the ones from the workstation named in the alert. Type the value exactly, including the quotation marks.
  - `and` — both must be true.
- **Type/click:** keep your `@timestamp`, `user.name`, `source.ip` and `service.name` columns, and sort by `@timestamp` ascending.
- **What you should see:** a much shorter list, and a **new, much smaller hit count**. Read it and **write the number down**.
- **What you should notice — this is the finding:**
  - the `@timestamp` column: the requests are **seconds apart** and the whole run takes **well under a minute**. Look at the first and last timestamps and work out the span. **Write both down.** No human being needs a file share, a web app, a print server and a domain controller in one minute; a script does, because a script is enumerating.
  - the `service.name` column: it shows a **wide spread of different values** — file shares, web services, computer accounts, the domain controller. **Count how many distinct values you can see** and write that number down too. This is the "how wide was the sweep" number.
  - the `user.name` column: read it carefully. The tickets are being drawn for an **administrative identity**, not for a named employee. That is a serious clue on its own — an ordinary user requests tickets for their own handful of services, not a domain-wide sweep.
  - the `source.ip` column is constant, and these records are written on the **requesting** machine (check the `host.name` column too — it is the same workstation).
- **What a suspicious answer looks like versus a benign one:** *suspicious* is "dozens of requests from one machine, inside one minute, across many distinct service names, for an administrative identity". *Benign* is "requests spread across the day, each user asking for the one or two services they actually use". Note that some service names here will also appear in Step 9's list — that is normal and it is why you cannot identify the attacker by service name alone.
- **Why this matters:** this is the technical signature of a forged ticket. A stolen `krbtgt` key lets an attacker ask for a ticket for *anything*, so the natural next move is to ask for everything and see what sticks. The burst and the spread are the evidence; record both.

---

## Step 11 — Third query: the forged sessions on the target servers

- **What to do:** now find where that administrative identity actually got in.
- **Type:** `event.code: 4624 and user.name: "Administrator" and source.ip: "10.0.4.10"`
- **What the query means, clause by clause:**
  - `event.code: 4624` — keep only "an account was **successfully** logged on" events. Note this is a different code from Step 9 and 10: those were ticket *requests*, this is an actual *session*. You need both halves of the story.
  - `user.name: "Administrator"` — keep only the ones for the domain administrator account.
  - `source.ip: "10.0.4.10"` — the same workstation again.
  - `and` — all three must be true.
- **Type/click:** add **`host.name`**, **`logon.type`**, and **`@timestamp`** as columns, and sort by `@timestamp` ascending.
- **What you should see:** a small result set. Read the hit count and **write down every distinct value you see in the `host.name` column** — one line per server.
- **What you should notice:**
  - **several different servers**, minutes apart.
  - `logon.type` is `3` on every row — a **network** logon, so these are remote sessions, not someone sitting at those machines.
  - the `@timestamp` values come **after** the ticket burst in Step 10, never during it. Tickets first, then sessions. That order is the attack.
- **A trap to avoid here:** if you drop the `event.code: 4624` clause and search on the account name and address alone, you will pick up the ticket requests from Step 10 as well, because those were *also* requested for this same identity. The event code is what separates "asked for a ticket" from "got a session". Always keep it.
- **Why this matters:** these are the sessions the forged tickets paid for. The count and the list of servers are the scope of your incident — every one of those machines is now a system an attacker held domain-admin rights on, whether or not they did anything visible on it.

---

## Step 12 — Fourth query: everything that workstation did all day

- **What to do:** check what this machine was doing *before* the attack, to see whether these logons are normal.
- **Type:** `event.code: 4624 and source.ip: "10.0.4.10"`
- **What the query means, clause by clause:**
  - `event.code: 4624` — successful logons only, from any account.
  - `source.ip: "10.0.4.10"` — from that one workstation.
  - This is Step 11 with the account filter removed, so you see the workstation's **entire** day.
- **Type/click:** add **`user.name`**, **`host.name`**, and **`@timestamp`** as columns, and sort by `@timestamp` ascending.
- **What you should see:** a **very short** list — a handful of rows for the whole day. Read the count and **write down each row's `user.name` and `host.name`**.
- **What you should notice — this is your "no prior baseline" evidence:**
  - the **earliest** row is an **ordinary service account** (a `svc_…` name) logging in early in the day to a server it has a routine relationship with. That is normal background activity, and it is the *baseline*: this is what this workstation legitimately does.
  - every other row is the **administrator account**, and **all of them are at the tail of the day**, in the burst window from Step 11.
  - there is **no** history here of this workstation administering those servers at any other time. A workstation that suddenly holds domain-admin sessions on several servers it has never touched before is the definition of a compromise.
- **Why this matters:** this one query is what separates "an administrator was busy tonight" from "these sessions were forged". Real administrators log in to the same servers repeatedly and from their own workstation. A machine whose *entire* administrative history fits inside a few minutes is a machine that was stolen, not a machine that was used.

---

## Step 13 — Fifth query: the file server that was read

- **What to do:** switch to the network traffic index and find what the forged session actually did.
- **Type/click:** click the data-view dropdown at the top of the page (currently `auth`) and choose **`network`**.
- **Type:** `network.protocol: "smb" and source.ip: "10.0.4.10"`
- **What the query means, clause by clause:**
  - `network.protocol: "smb"` — keep only SMB traffic. SMB is Windows file sharing, and it always uses **port 445**.
  - `source.ip: "10.0.4.10"` — only connections *originating* from that same workstation.
  - `and` — both must be true.
- **Type/click:** add **`@timestamp`**, **`destination.ip`**, **`destination.port`**, and **`network.bytes`** as columns.
- **What you should see:** a short list of SMB connections. Read the hit count, and **write down the `destination.ip` and `destination.port`** of the one at the tail of the day.
- **What you should notice, and this is the closing of the loop:** read the destination's address, then look back at the `service.name` values from Step 10. The host you just found should appear in that list as a **CIFS (file-share) service name**. If it does, the story is complete: the attacker requested a ticket for the file server, then used the forged session to read it. **Write down the byte count and the timestamp too.**
- **Why this matters:** this is why the incident is critical and not merely suspicious. Ticket activity proves capability; an actual data transfer proves intent and gives you the scope of the breach for the notification and legal side of the response.

---

## Step 14 — Sixth query: confirm the one-shot query from the companion brief

- **What to do:** re-run your Step 10 query to confirm your own finding, exactly as the shorter brief writes it.
- **Type/click:** switch back to the **`auth`** data view.
- **Type:** `event.code: 4769 and source.ip: "10.0.4.10"`
- **What you should see:** the **same number** you counted in Step 10. That is your answer to the brief's single query.
- **Why this matters:** this is your primary piece of evidence, and re-running the brief's exact query confirms the two documents describe the same set of events. If the number changed, you edited the query by accident — go back to Step 10.

---

## Step 15 — Build the timeline

- **What to do:** open a blank note (paper or text file) and list your findings oldest → newest. Use the timestamps exactly as Kibana shows them.
- **Type:** one line per event, in this order:
  1. (time) — the workstation's ordinary service-account logon (the baseline)
  2. (time) — the first service-ticket request … and (time) the last one, seconds apart (note the total number of requests, how many distinct service names, and the whole burst's length)
  3. (time) — the first administrator logon, to the first server
  4. (time) — the administrator logon to the second server
  5. (time) — the administrator logon to the third server
  6. (time) — the SMB read of the file server
- **What you should see:** a quiet day, then a tight cluster of activity at the end of it, in strict order: requests → sessions → file read. Write down how much idle time separated the baseline logon from the burst — that gap is the time the attacker spent preparing.
- **Why this matters:** a timeline **is** your evidence of cause and effect. The order is the whole argument: you cannot claim these sessions were forged unless the ticket requests came *first*.

---

## Step 16 — Write the report

Use this fill-in-the-blank template. Each section has an example sentence.

### Summary
*Fill in:* what happened, from where, on how many servers, verdict.
> Example: "In the evening of 22 September 2026 the workstation at 10.0.4.10 issued a burst of Kerberos service-ticket requests covering many distinct service principal names in under a minute, then used the domain Administrator identity to log in to several servers it had never previously contacted, and finally read an internal file server over SMB. Assessed as a true positive golden-ticket compromise of the domain's Kerberos signing key."

### Timeline
*Fill in:* your rows from Step 15, oldest first.

### Evidence
*Fill in:* each query you ran and what it proved.
> Example: "`event.code: 4769` showed the day's total ticket-request noise spread across many machines and users, establishing that the event code alone is not an anomaly. `event.code: 4769 and source.ip: \"10.0.4.10\"` proved a single workstation issued a tightly packed burst of ticket requests spanning many distinct service names for an administrative identity. `event.code: 4624 and user.name: \"Administrator\" and source.ip: \"10.0.4.10\"` proved that identity then opened network sessions on multiple servers. `event.code: 4624 and source.ip: \"10.0.4.10\"` proved the workstation's only prior activity was a single routine service-account logon, so it had no history with those servers. `network.protocol: \"smb\" and source.ip: \"10.0.4.10\"` proved the forged session read an internal file server, whose CIFS service name appeared in the earlier ticket burst."

### Verdict
*Fill in:* True Positive or False Positive, and why.
> Example: "TRUE POSITIVE — a single workstation requesting Kerberos service tickets for a wide set of services in under a minute, immediately followed by domain-Administrator network sessions on servers with no prior relationship to that machine, and then a file read, is a coherent forged-ticket → privilege-assertion → lateral-movement → data-access chain."

### Recommendations
*Fill in:* at least four concrete actions, and be sure to include the domain recovery.
> Example: "Isolate the workstation at 10.0.4.10 and preserve it for memory forensics, since the krbtgt key was almost certainly read from it or from a host it reached. Treat the domain as fully compromised: perform the two-password reset of the domain administrator account to rotate the krbtgt signing key, invalidate every existing Kerberos ticket, and re-image every privileged host including all domain controllers. Reset the credentials of every account whose session appears in the timeline, starting with the service account on the baseline logon. Review access to the file server that was read and begin breach-notification analysis. Harden detection long-term: alert on service-ticket bursts from a single source, on any new administrative logon to a server, and on ticket requests for unusually many distinct SPNs; restrict the account that can retrieve the krbtgt key and monitor all 4662 ticket-decryption events."

When you are done, submit your report. Your instructor will compare your findings
against the answer key.
