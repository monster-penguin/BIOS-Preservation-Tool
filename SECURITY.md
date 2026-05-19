# Security Policy

## Scope

This project does not distribute or execute BIOS files. However, it does consume hash metadata from the upstream [Abdess/retrobios](https://github.com/Abdess/retrobios) YAML pipeline. If that upstream source were ever compromised, it is possible the tool could be made to recognise or stage a file that is malware.

## Reporting a Security Issue

If you believe the upstream YAML pipeline has introduced a hash that corresponds to malware, or you have identified any other security concern related to this tool, please **open a GitHub Issue** and label it `security`.

Include:
- The affected platform and filename
- The hash in question (MD5/SHA1/SHA256)
- Any evidence or context you have

Do not attach or link to the suspected file itself.

## What Happens Next

Security issues will be reviewed as a priority. If a malicious hash is confirmed, the affected entry will be flagged in `known_upstream_hash_issues.csv` and the upstream project will be notified.
