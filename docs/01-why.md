# Why this exists

**The general goal is to make it easy to answer relatively simple, recurring
questions from Intune and Entra ID logs** — who changed this policy, what did
the enrollment or configuration look like before a given change, are Autopilot
deployments actually performing better now than they used to — without
turning every one of those questions into a fresh KQL query.

None of the underlying queries were invented from a blank page. Several
started as public examples or as things I'd already used ad hoc (full lineage
in [Credits & sources](10-credits.md)). What this workbook adds on top isn't
new KQL so much as a field-level view of which questions actually matter, and
centralizing queries that used to live scattered across notes and one-off
scripts into a single place — so the answers become usable by whoever needs
them, not only by whoever is comfortable writing KQL.

Concretely, it started as three separate field problems, not a design
exercise:

- **Wipe troubleshooting.**
  - "Why did this phone just wipe itself?" is a recurring support question.
    Ten failed PIN attempts on a mobile device and Intune wipes it (by
    design), but confirming that's actually what happened, on which device,
    meant digging through the Intune console one event at a time, every
    time. The Wipe / Retire / Reset tab exists to answer that in one query
    instead of a manual lookup per incident.

- **Hardening remote actions.**
  - Once every wipe/retire event was visible in one place, the same data
    answered a bigger question: who actually has the ability to trigger
    remote actions on a device, and how often are they using it. That audit
    trail is what motivates tightening who gets that capability in the
    first place, rather than only reacting after it's been used. The same
    principle extends to configuration changes more broadly: knowing who
    changed what, not only who triggered a destructive action.

- **Building the case for EPM.**
  - The LAPS tab exists for the same reason, one layer up: the frequency
    and pattern of who's recovering local administrator passwords is the
    evidence needed to justify moving from standing local admin rights
    toward an Endpoint Privilege Management approach.

The workbook grew from there, consolidating from existing queries and recurring needs.
