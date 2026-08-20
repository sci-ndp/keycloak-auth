# Mapping the NDP Governance Framework onto What We Built

[Privacy, Access Control, Workflow, and Lifecycle Framework](https://docs.google.com/document/d/1F4blYOliQx3TzHyfBuZMbWj2ZNI1i1vQ/edit) doc
describes governance as a federated policy graph across nine planes. This maps it onto the
model in [ndp-new-auth.md](ndp-new-auth.md): what it covers, what it can't, and what to
watch for.

## The short answer

We built one plane: access, plus the subject(user), group, and role authority the other eight
would be evaluated against. It generalizes further than it looks: any plane whose question
is coarse enough to ask at a service boundary is one gate away. What the framework mostly
needs is *state* — lifecycle phase, embargo, dependency path, output class, run identity —
and almost none of that belongs in an identity provider. So the framework sits on top of
the client-scoped model rather than replacing it, with two caveats where the shapes
differ.

## Coverage

Most planes reduce to *may this subject perform action class A at this service boundary?*,
and that is already answerable — one more named gate, created with `customPermissions` and
asked for like `viewer-area`. What is scarce is per-object granularity, obligations, and
time-or-state conditions.

| plane (framework  2) | gate expressible today | what is actually missing |
|---|---|---|
| **Access** | ✓ built and in use | — |
| **Discovery** | ✓ a `discover-area` level | which *rows* appear — per-asset filtering |
| **Metadata** | ✓ a `metadata-area` level | per-asset filtering, field-level redaction |
| **Execution** | ✓ a launch gate | separating launch from visibility (caveat 2); no run identity |
| **Result egress** | ✓ an `export-area` level | obligations — redaction, review, export caps |
| **Governance** | three-actor handoff is real | audit trail, approvals |
| **Topology** | group hierarchy is the right shape | enforced local authority (caveat 1), path-sensitivity |
| **Temporal** | ✗ | Keycloak ships a Time policy type; we are not writing one |
| **Lifecycle** | ✗ | no policy type we use reads asset state |

An app gates discovery by asking once, then listing from its own database:

```python
if allowed(user_token, "discover-area"):
    return db.list_datasets()
```

That covers "may this user browse this catalog at all."
For "which of these 10,000 datasets may they see", the app needs to filter by its own metadata and provenance tables, or create custom permission policy on keycloak client.

## Already done

Four of framework  13's recommendations are in place today:

- **The relevance check is server-side.** No service/app parses a role name; Keycloak evaluates
  the rules, the app enforces the result.
- **Rules read live state, not token state.** internally, `_upsert_group_policy` sets `groupsClaim: ""`
  on purpose, so Keycloak resolves current membership rather than trusting the presented
  token. Revocation lands on the next call with no refresh, and a token minted for one
  client still evaluates correctly against another (the property a federated topology
  needs).
- **Token scope is confined per client.** framework  10 issue
  classes 1 and 6, and the whole subject of [problem.md](problem.md).

## Caveats

Three places the shapes differ — each a decision for management, not engineering.

**1. `ndp_admin` overrides local deny.** Every gate is `AFFIRMATIVE` with `is-ndp_admin`
attached, so a realm admin passes everything everywhere. And an admin of `ndp_ep` group passes
every endpoint (`ndp_ep/ep-***`)beneath it.

**2. The protected object is a service, not an asset.** Our unit is a client and its coarse
areas; the framework's is a per-object graph. Modeling it in Keycloak means a resource and
a permission per dataset per client, evaluated on every request — the catalog maintained
twice, once in the database and once in the IdP. Keycloak decides the action class at a
service boundary; the resource server decides the object.


## What could be done next

**Add `_upsert_time_policy`.** Keycloak ships a Time policy type and `authz_service.py`
never writes one. It needs no new resource and no new subgroup — but appending it to a gate
does nothing, since gates are `AFFIRMATIVE` and membership alone would still pass. Identity
moves into an aggregate, with time as a mandatory conjunct:
```
aggregate   who-can-view       →  [in-viewers, holds-viewers, is-ndp_admin]  AFFIRMATIVE
time        opens-2026-03-01   →  notBefore 2026-03-01
permission  viewer-area-access →  [who-can-view, opens-2026-03-01]           UNANIMOUS
```

Where `is-ndp_admin` sits decides whether a hold binds the platform admin: inside the
aggregate it is bound, outside it walks through any embargo — caveat 1 as a config
choice. And it is coarse, embargoing a client's whole viewer area rather than one dataset.
Event-based transitions need a trigger outside Keycloak; recertification and expiry are
membership jobs against `/group/remove-user`, not policy features.

**Verify two assumptions before building on them:** the aggregate semantics of multiple
`permission=` parameters in one call (the hook for path-sensitive authorization,  9.2).

**Leave the rest to the catalog and federation layer:** discovery filtering, metadata
redaction, egress obligations, lifecycle, run identity, and the dependency graph. Writing
down what that layer asks Keycloak versus what it decides itself is worth more than any
single feature above.
