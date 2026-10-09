# CTIA Scenario Lab — golden-ticket-001

A self-contained SOC analyst training scenario built from a deterministic, ECS-compliant synthetic dataset. Investigate it with Elasticsearch + Kibana.

## Scenario narrative

An attacker who extracted the Kerberos ticket-granting-ticket signing key forged golden tickets, requested service tickets for many different service principal names from a single workstation, and used them to authenticate as Administrator on multiple internal servers that this workstation had never previously contacted.

The dataset is built from 2 log sources:

| Source | What it covers |
|---|---|
| `auth` | Windows security-style authentication events: successful and failed logons, logoffs, privilege assignment, account lockouts, Kerberos pre-auth failures, Kerberos service-ticket requests, and process creations with process name, parent and command line |
| `network` | Connection events between hosts and toward external peers: SMB, RPC, DNS queries and their answers, LDAP, HTTP/HTTPS, SSH, ICMP |

The logs are **synthetic and deterministic**: they are generated with a fixed seed, so every run produces byte-identical data. Event volume is engineered to sit in the 200–600 range expected for a realistic, noisy-but-reviewable lab.

## Contents

| File | Description |
|---|---|
| `auth.ndjson` | auth events (284 documents, NDJSON — one JSON object per line) |
| `auth.json` | Elasticsearch index template — field types for the `auth-*` index |
| `network.ndjson` | network events (136 documents, NDJSON — one JSON object per line) |
| `network.json` | Elasticsearch index template — field types for the `network-*` index |
| `rule.json` | the detection rule that fired for this scenario, already installed in the lab |
| **`guided-walkthrough.html`** | **the investigation guide — open this in a browser** |
| `setup-lab.cmd` | Windows one-click build of the whole lab, including the rule |
| `teardown.cmd` | Windows removal of everything `setup-lab.cmd` created |
| `setup-lab.sh` | the same one-click build for Linux and macOS |
| `teardown.sh` | the same removal for Linux and macOS |
| `solution.html` | **Instructor only, not in the zip.** Expected figures for Part two, for marking |
| `solution.md` | Instructor-side reference (not part of the student package) |
| `README.md` | this file |
| `summary.md` | Build notes for whoever produced the lab — not used by anyone running it |
| `Archive.zip` | The student package zipped up. Ships everything above except the two instructor files |

`guided-walkthrough.html` is the whole student guide: it is self-contained, it needs no
internet connection, and it is the only document a student needs. It is 20 cards long and
ends with the one card that matters: **Write the report, then prove it.** That card has two
parts — the written report, and the set of figures the student checked off their own screen
— and a single **COPY MY SUBMISSION** button copies both into one block of text. No answers
are printed anywhere in the guide.

## Requirements

- **Docker**, running. The setup file uses it to start Elasticsearch and Kibana; if you
  would rather not use the scripts you can point it at a stack you already have.
- A web browser, for Kibana and for the guide.
- `curl`, only if you choose the by-hand route instead of the setup file.

No Python, packages, or local servers are needed to run the lab — the data is
pre-generated and distributed as plain files, and the guide is a single self-contained
HTML file.

## Credentials

The lab runs with Elasticsearch security switched on, because a detection rule cannot be
created against a cluster that has it disabled: with `xpack.security.enabled=false`
Elasticsearch has no rule API at all and Kibana refuses the rule call, which would leave the
Alerts page permanently empty. Use this account everywhere:

```
lab_kibana  /  LabKibana001
```

In a command that is the `-u lab_kibana:LabKibana001` part; in the browser it is the
username and password on the Kibana sign-in page. It is a throwaway lab credential and
protects nothing.

## Quick start

**One-click**, if Docker is installed:

| | Windows | Linux / macOS |
|---|---|---|
| build the lab | `setup-lab.cmd` | `./setup-lab.sh` |
| remove it again | `teardown.cmd` | `./teardown.sh` |
| rebuild from scratch | `setup-lab.cmd reset` | `./setup-lab.sh reset` |
| set up without opening a browser | `setup-lab.cmd nolaunch` | `./setup-lab.sh nolaunch` |
| list the options | `setup-lab.cmd -?` | `./setup-lab.sh --help` |

They start Elasticsearch and Kibana in containers, install the templates, import both
datasets, create the data views, install the detection rule, check that the rule actually
fired, and then verify every count the walkthrough quotes. Re-running is safe: nothing is
imported twice and the rule is not installed twice.

The two pairs are twins. They do the same thing in the same order; if you change one,
change the other.

> On zsh, which is the default shell on macOS, a bare `-?` is swallowed by the shell as a
> filename glob before the script ever sees it. Use `--help`.

**By hand**, if you would rather not use the scripts — run every command from this folder:

```bash
# 1. Check Elasticsearch is up
curl -s -u lab_kibana:LabKibana001 localhost:9200/_cluster/health

# 2. Install the index templates (fixes field types)
curl -s -u lab_kibana:LabKibana001 -XPUT localhost:9200/_index_template/auth    -H 'Content-Type: application/json' --data-binary @auth.json
curl -s -u lab_kibana:LabKibana001 -XPUT localhost:9200/_index_template/network -H 'Content-Type: application/json' --data-binary @network.json

# 3. Import the data (the bulk API needs one action line + one data line per doc)
for s in auth network; do
  while read -r l; do
    printf '%s\n' '{"index":{"_index":"'$s'-2026.09.22"}}'
    printf '%s\n' "$l"
  done < $s.ndjson | curl -s -u lab_kibana:LabKibana001 -XPOST localhost:9200/_bulk -H 'Content-Type: application/x-ndjson' --data-binary @-
done
`printf '%s\n'` rather than `echo` on purpose: some log lines contain escaped quotes, and
zsh's `echo` treats a backslash as an escape character, so it quietly corrupts a document
and Elasticsearch rejects it. `printf '%s\n'` prints the line exactly as read, and behaves
identically in bash, zsh and sh.

# 4. Verify (expect the document counts from the Contents table)
curl -s -u lab_kibana:LabKibana001 'localhost:9200/_cat/indices/auth*,network*'
```

The credentials are written out in full in every command on purpose. A shell variable
holding `-u user:pass` works in bash but not in zsh, which does not split an unquoted
variable into separate words, and a student who is quietly getting 401s from a lab that is
correctly built has a bad morning.

Then sign in to Kibana as `lab_kibana` / `LabKibana001`, create a **data view** for each
index pattern (`auth-*`, `network-*`, timestamp field `@timestamp`), and open
[`guided-walkthrough.html`](./guided-walkthrough.html) in a browser. That single file is the
whole investigation, from the first search to the report they submit.

The rule in `rule.json` is a body for `POST /api/detection_engine/rules` on the Kibana
server, which is where Kibana's alerting engine picks it up:

```bash
curl -s -u lab_kibana:LabKibana001 -XPOST http://localhost:5601/api/detection_engine/rules \
  -H 'kbn-xsrf: true' -H 'Content-Type: application/json' --data-binary @rule.json
```

Once it has run, **Security → Alerts** (hamburger menu → Security → Alerts, or
`http://localhost:5601/app/security/alerts`) lists the matched events.

## How the dataset is produced

The scenario is generated by a deterministic Python pipeline: a seeded PRNG fixes the noise, the attack-chain events are emitted without any randomness, and every field is aligned to the Elastic Common Schema with explicit index mappings. `@timestamp` is ISO 8601 UTC with millisecond precision. The output is validated end to end before packaging — JSON parse, required ECS fields, mapping coverage, IP/timestamp/code types, byte-for-byte determinism across two runs, and every ground-truth assertion executed as a real query against the index.

The student package exposes only what the analyst needs: the two NDJSON datasets, the two index templates, the detection rule, one guide, and the two files that start and stop the lab. Instructor-only material is kept out of the analyst distribution by design.

The setup files also verify themselves. They refuse to report the lab as ready unless the rule has actually fired, both datasets hold the exact document counts in the table above, and every count quoted in the guide matches the live index — so a lab that built successfully is a lab whose numbers can be trusted.
