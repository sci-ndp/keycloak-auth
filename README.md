# Keycloak Across NDP: Current Integration & the New Authorization Model

**Purpose:** (1) summarize how NDP services currently use the Keycloak-based identity stack and what user data flows through it, and (2) introduce the new client-scoped authorization model that addresses the risks that picture exposes.

---

## Part 1 — Current State of Keycloak Integration Across NDP Services

### 1.1 The stack

Identity and access for NDP runs on a self-hosted stack, all maintained by the AAI (Authentication & Authorization Infrastructure) team:

```mermaid
flowchart LR
    subgraph AAI["AAI Stack (self-hosted)"]
        KC["Keycloak 26\n(Identity Provider,\nrealm: NDP)"]
        PG[("PostgreSQL\n(Keycloak's data store)")]
        API["AAI API\n(custom Flask service)"]
        UI["Admin Console UI\n(custom, talks only to AAI API)"]
        KC --- PG
        API -->|Keycloak Admin REST| KC
        UI -->|group/role/user mgmt| API
    end
    FE["ndp-frontend\n(one Keycloak client per CNDP deployment)"]
    WA["ndp-workspaces-api"]
    JH["ndp-jupyterhub"]
    EP["ndp-ep (ep-api)"]
    Other["Other NDP services\n(each its own Keycloak client)"]

    FE -->|OIDC login, token refresh| KC
    WA -->|verifies bearer tokens| KC
    JH -->|OIDC login, token refresh| KC
    EP -->|"optional OIDC login, /user/login proxy"| KC
    EP -.->|"bearer token → /information"| API
    Other -->|OIDC / bearer tokens| KC
    FE -.->|group/user admin calls| API
    WA -.->|group/user admin calls| API
    JH -.->|forwards user's token| WA
```

- **Keycloak 26** is the identity provider (IdP) for the single `NDP` realm — every NDP-affiliated service authenticates users against it.
- **PostgreSQL** is Keycloak's backing store (users, groups, roles, sessions).
- **AAI API** is a custom FastAPI service that sits in front of Keycloak's Admin REST API. Instead of handing product teams direct Keycloak admin credentials, they call this API, which enforces a group-based permission model on their behalf (see below) and is the only thing with standing admin access to Keycloak.
- **Admin Console UI** is a custom front-end for the AAI API, used to manage fast user lookups, group lookups, whitelist management, without touching Keycloak's own admin console.

**How permissions are modeled today:** access is organized around Keycloak **groups** (e.g. `ndp_ep/ep-123`, each group automatically gets three realm roles named `group:<path>:admin`, `group:<path>:editor`, `group:<path>:viewer`; `jhub_user`). The AAI API checks these role names to decide whether a caller can manage a given group. This is the mechanism that lets teams self-serve group/role management without a central admin doing it by hand.

**What every client currently receives:** regardless of which of the above a service actually needs, Keycloak's default token mappers put the **user's full group membership (every group, platform-wide) and full realm role list (every role, platform-wide)** into every access token it issues, for every client. This is the pattern that motivates Part 2.

### 1.2 How individual services use it

Two service teams documented their own integration in detail; `ndp-jupyterhub` and `ndp-ep` are summarized here directly from their repos. All four are condensed as illustrative examples of the same underlying pattern.

#### ndp-frontend

- **How it authenticates:** standard OIDC Authorization Code + PKCE (via NextAuth), one dedicated Keycloak client per CNDP deployment, each with its own client ID/secret.
- **What it retrieves from Keycloak:** at login, the user's `sub`, `name`, `email`, `preferred_username`, and a custom onboarding flag. On every permission check, it reads `realm_access.roles` (**all** realm roles the user holds, not just ones relevant to this app) and a `groups` claim (**every** group path the user belongs to) directly out of the access token.
- **Where that data lives:** identity claims, access token, and refresh token are held in an encrypted, server-side session cookie (4-hour lifetime); the decoded roles/groups are additionally cached in the browser's `localStorage` for the same window.
- **Admin operations:** group creation, membership changes, and per-deployment client provisioning go through the AAI API using a service-account credential, not the logged-in user's own token.
- **Sensitive info exposed to this client:** real name, email, and — critically — the user's complete group and role footprint across the entire NDP platform, even though a given CNDP deployment only ever needs to check membership in a small, fixed set of groups/roles relevant to it.

#### ndp-workspaces-api

- **How it authenticates:** verifies the bearer JWT issued by Keycloak on every request (no login flow of its own) and decodes it — `sub` becomes the user's canonical `keycloak_id`, alongside username, email, name, `realm_access.roles`, and `groups`, all taken as-is from the token.
- **Where that data lives:** `keycloak_id` is stored as the identity/ownership key on essentially every table in the workspaces database (workspaces, classrooms, data challenges, projects, sites, bookmarks, catalog entries, alerts, user profiles, and endpoint access requests) — roughly a dozen tables and ~35 API schemas carry it directly. For endpoint access requests and user profiles specifically, the user's **name and email are additionally copied out of the token and stored in the database in plaintext**, independent of Keycloak.
- **Admin operations:** has its own wrapper around the Keycloak Admin REST API for group/user lookups and membership management (service-to-service, not end-user tokens).
- **Sensitive info exposed to this service:** the same platform-wide roles/groups as above, trusted directly from the token with no independent check of whether they're relevant to this service — plus a persisted copy of the user's name and email that outlives the login session entirely.

#### ndp-jupyterhub

- **How it authenticates:** standard OIDC Authorization Code flow via JupyterHub's `GenericOAuthenticator` (subclassed as `MyAuthenticator`), talking directly to Keycloak's `/realms/NDP/protocol/openid-connect/*` endpoints, requesting only the `openid` and `profile` scopes (not `email`). One dedicated Keycloak client per deployment, with `client_id`/`client_secret` delivered by NDP admins and stored as a Kubernetes secret.
- **What it retrieves from Keycloak:** after login, it calls Keycloak's `/userinfo` endpoint (passing the access token as Bearer auth) rather than decoding any claim out of the token itself. From that response it reads `preferred_username` (used directly as the JupyterHub username) and a `groups` field, which it checks against a single allow-listed login group (e.g. `jhub_user`) — anyone not in that group is denied login outright; a separate static list of admin emails grants JupyterHub-admin privileges, independent of any Keycloak role. It does not consume `realm_access.roles` or the full `groups` claim the way ndp-frontend/ndp-workspaces-api do. Note that the access token it presents to `/userinfo` doesn't itself need to carry a `groups` claim — Keycloak only requires the token to have been granted the `openid` scope, and builds the `/userinfo` response (including `groups`) fresh from the realm's configured mappers, independent of what claims happen to be embedded in the token.
- **Where that data lives:** the OAuth `access_token` and `refresh_token` are persisted as JupyterHub "auth state" in the Hub's own database, and are also injected as **plaintext environment variables (`ACCESS_TOKEN`, `REFRESH_TOKEN`) inside every spawned single-user notebook pod** — so the user's live token pair is directly readable by any code the user runs in their own notebook.
- **Admin operations:** none — ndp-jupyterhub never calls the AAI API or Keycloak's Admin REST API. It only performs the end-user OIDC login/refresh directly against Keycloak (refreshing the access token itself via the token endpoint, roughly daily or before each spawn), then reuses the user's own access token to call `ndp-workspaces-api` on their behalf (listing their workspaces/entities, provisioning and updating their PVCs).
- **Sensitive info exposed to this service:** username and login-group membership, plus — via the access/refresh token pair copied into every notebook pod's environment — whatever those tokens are valid for, for as long as they remain valid, directly exposed to user-run code inside the notebook.

#### ndp-ep (ep-api + ndp-ep-py)

- **How it authenticates:** ep-api never decodes or verifies a JWT itself — every request's bearer token is POSTed as-is to the AAI API's `/information` endpoint (`AUTH_API_URL`), and whatever `{sub, username, roles, groups}` comes back is trusted directly; there's no local signature check. A caller can arrive at that token three ways: pasting one obtained elsewhere; calling ep-api's own `POST /user/login`, which proxies a username/password straight to the AAI's `/user/login` (Keycloak Resource Owner Password Credentials) and hands back the raw access token; or, if the deployment opts in (`OIDC_ENABLED`, off by default), a standard OIDC Authorization Code + PKCE flow against a dedicated **public** Keycloak client, added specifically for users who authenticate through a federated identity provider (CILogon, EarthScope, ORCID) and so have no realm password to type into the second option. A companion Python SDK, `ndp-ep-py`, does no authentication of its own — it just carries a caller-supplied token as a Bearer header into every ep-api call. A dev-only `TEST_TOKEN` bypass also exists that skips the AAI call entirely; it's documented as required to be left blank in production.
- **What it retrieves from Keycloak (via the AAI, not directly):** the same unfiltered `sub`, `username`, `roles` (every realm role) and `groups` (every group) that ndp-frontend/ndp-workspaces-api read directly out of the token — ep-api just gets them relayed through the AAI's live lookup instead. It layers its own three-tier permission model (`viewer`/`writer`/`admin`) on top by pattern-matching six specific role names — `ndp_{tier}` (platform-wide) or `group:{endpoint-uuid}:{tier}` (per-endpoint) — out of that same full, unfiltered `roles` list; every other role the user holds rides along unused.
- **Where that data lives:** ep-api doesn't persist tokens or claims — the AAI lookup happens fresh on every request. It does derive and store one thing from the user's `sub`: a truncated SHA-256 hash (`ndp_creator_md5` / `ndp_user_id`) stamped onto every catalog resource the user creates, used as a pseudonymous "creator" identifier so downstream consumers (e.g. Search) can filter by creator without the raw `sub`, name, or email ever being exposed on the resource itself.
- **Admin operations:** a thin wrapper (`aai_client.py`) calls the same AAI endpoints documented in §1.1 (`/group/add-user`, `/role/assign`, `/group/members`) to back its access-request approval workflow — always using the requesting admin's own bearer token, never a standing service-account credential.
- **Sensitive info exposed to this service:** the same platform-wide, unfiltered roles and groups every other client receives; plus, whenever the username/password login path is used, ep-api's own `/user/login` endpoint sees the user's raw Keycloak password in transit (proxied straight through to the AAI, not stored) before it ever reaches Keycloak.

### 1.3 The pattern

`ndp-frontend`, `ndp-workspaces-api`, and `ndp-ep` (and, by construction of the current token mappers, every other NDP client that reads roles/groups the same way):

- Every client gets the user's **entire** group membership and **entire** realm role list, not just what pertains to it.
- Clients trust that content directly, with no per-client filtering happening at the Keycloak layer.
- At least one service persists identity fields (name, email) from the token into its own database, so exposure isn't limited to the lifetime of a session.

This "put everything in the token, let each app sort out what it needs" design is the shared root cause behind the two problems Part 2 addresses.

---

## Part 2 — The New Authorization Model

### 2.1 The problem

**Over-broad tokens.** Because every client receives the same all-encompassing token, a token leaked or stolen from *any one* client — a browser session, a log line, a misconfigured proxy — discloses far more than "this person is logged in to this app." It discloses the user's complete group membership and role footprint across NDP: every program, dataset, or admin capability they have anywhere on the platform, whether or not it has anything to do with the app the token was actually issued for.

**Token size / infrastructure risk.** The same design means token size grows with the *total* number of groups and roles a user holds anywhere on NDP, not with what a given client needs. For users who accumulate many group memberships (e.g. instructors, admins, or endpoint owners), this can produce large `Authorization` headers and cookies. We've built tooling internally to measure this (summing role-name and group-path lengths per user across the realm), and the risk is concrete: reverse proxies commonly reject requests whose headers exceed a fixed size limit, which would surface as unexplained login/authorization failures for exactly the users with the broadest legitimate access.

Both problems trace back to the same root cause: **token content is scoped to the user, not to the client asking for it.**

### 2.2 The new model

We've been developing an authorization model that scopes both the *content* and the *validity* of a token to the specific client it was issued for:

1. **Client-scoped authorization via Group policies.** Each client gets its own Keycloak Authorization Services configuration with a **Group policy**: a rule that checks whether the user belongs to the specific group(s) that client actually cares about. Instead of a client reading a firehose of every group the user is in and figuring out relevance itself, the relevance check moves server-side, into Keycloak, scoped per client.

2. **Client-bound tokens.** A token issued through client A's login is only valid for authorizing against client A — **even if the same user also belongs to groups tied to client B.** A token stolen from client A can't be replayed against client B's API, regardless of what that user is otherwise entitled to.

Together, these mean a client's tokens shrink to contain only what that client needs, and a compromised token's blast radius is limited to the one client it was issued for — instead of, as today, exposing the user's entire cross-platform footprint and being potentially replayable anywhere that footprint would grant access.

### 2.3 How this would affect NDP services

Using the services profiled in Part 1 as concrete examples of what changes:

- **ndp-frontend** currently reads `realm_access.roles` and the full `groups` array straight off the token for its permission checks, including a CMS-driven pattern where arbitrary additional groups can be referenced per page. Under the new model, ndp-frontend's Keycloak client would need its own Group policy that explicitly covers every group/role it currently depends on (`professor`, `catalog_admin`, `ckan_admin`, `jhub_user`, the per-deployment admin group, the `ndp_ep/*` composite groups, and any CMS-supplied ones) — anything not enumerated there would no longer show up implicitly just because it happened to be in the token.
- **ndp-workspaces-api** currently trusts whatever roles/groups arrive in the bearer token for nearly every route via `Depends(get_user_info)`. It would move to the same audience-restricted validation, and — by design — would stop seeing groups/roles that belong to other clients, even for the same user. Any workflow that currently relies on that (e.g., cross-service checks) would need to go through the AAI API's service-account path instead, which isn't affected by this change since it's a trusted backend call, not an end-user token.
- **Data already persisted outside Keycloak** (e.g., workspaces-api's stored `keycloak_name`/`keycloak_email`) is a related but separate exposure surface — this model reduces what's *in the token*, not what a service has already chosen to copy into its own database.

**Suggested next step:** treat Part 1 of this document as the template — before switching any given client over, audit exactly what claims/groups/roles it currently reads (as done here for ndp-frontend and ndp-workspaces-api), so its new Group policy is defined to match reality rather than guessed at. A phased rollout, client by client, starting with whichever service carries the most sensitive exposure today, avoids breaking checks that currently work only because the old token happened to contain everything.
