# NDP New Auth Walkthrough

Your app makes the authorization decision. It asks Keycloak to evaluate the access rules
for a user — "would these rules allow X?" — and enforces the answer itself. No role parsing
in your code, and the rules change without a redeploy.

## 0. Who does what

Three different people appear in this document, and they are not the same person.
Read this table before anything else — most confusion comes from mixing them up.

| who | when | does |
|---|---|---|
| **platform admin** (holds `ndp_admin` / `create_client`) | once, at provisioning time | creates the **client** and the **group** for a new service |
| **service/group admin** (the person handed the group) | ongoing | puts people into `admins` / `editors` / `viewers` |
| **end user** | every request | just logs in — nothing else |

**End users never create groups, and never create clients.** They do not call
`/client/create`, they do not touch the Keycloak console, and they do not need to know
that groups exist. A user is *placed* into a group by someone who already administers
it. If you are writing a UI for end users, there is no "create group" button in it.

Sections 1 and 3 below are admin work. Section 4 is the only part your app does at
runtime.

## 1. Onboard: configure a client for your app

Done **once per service, by a platform admin** — usually inside a provisioning script,
not by hand.

```bash
curl --location 'https://dev-idp.nationaldataplatform.org/api/client/create' \
--header 'Content-Type: application/json' \
--header 'Authorization: Bearer {{ADMIN-TOKEN}}' \
--data '{
  "clientName": "{{CLIENT-NAME}}",
  "groupName": "{{GROUP-NAME}}",
  "adminUsers": [],
  "editorUsers": [],
  "viewerUsers": [],
  "customPermissions": {"{{CUSTOM-PERMISSION}}": []}
}'
```

That creates:

- the **group** `{{GROUP-NAME}}`, with three **permission levels** — the subgroups
  `{{GROUP-NAME}}/admins`, `{{GROUP-NAME}}/editors`, `{{GROUP-NAME}}/viewers`
- the **client** `{{CLIENT-NAME}}`, confidential, with a service account
- **access rules** on the client, one per permission level

You get the **client secret once** — save it. You are an admin of the group.

{{ADMIN-TOKEN}} needs `ndp_admin` or `create_client`. Naming a group that already exists also
requires you to be an admin of it.

Seed the first group admin here with `adminUsers`, so the service is handed over with
someone already able to manage it. Otherwise the group ships empty and nobody but the
platform admin can add members.

## 2. What you can ask about

| you ask for | who passes |
|---|---|
| `permission=admin-area` | members of `admins` |
| `permission=editor-area` | members of `admins`, `editors` |
| `permission=viewer-area` | members of `admins`, `editors`, `viewers` |

`ndp_admin` passes everything, and the same levels on a **parent** group count too.

The parent-group rule is what makes a hierarchy like `ndp_ep/ep-123` useful: an admin of
`ndp_ep` is automatically an admin of every endpoint underneath it, without being listed
in any of them.

## 3. Put people in the right group

**Admin work, not user work.** The group already exists by this point — step 1 created
it. Nobody is creating anything here; an existing admin is only deciding who belongs at
which level.

Programmatically, your service can add users to the group with `/group/add-user`:
```bash
curl --location 'https://dev-idp.nationaldataplatform.org/api/group/add-user' \
--header 'Content-Type: application/json' \
--header 'Authorization: Bearer {{ADMIN-TOKEN}}' \
--data-raw '{
    "groupName": "ndp_ep/ep-****",
    "user": "yutian.qin@utah.edu",
    "role": "admin"
}'
```
Or simply use the Keycloak console (**Groups → {{GROUP-NAME}} → Child Group → editor → Members**).

Here `{{ADMIN-TOKEN}}` is the **group admin's** token — the person who was handed the
service. It does not have to be `ndp_admin`; being an admin of that group (or of a parent
group) is enough. That is the point: the platform admin provisions once and steps out,
and day-to-day membership is the service owner's job.

`role` is `admin`, `editor` or `viewer`. For a custom level, pass its members in
`customPermissions` at creation, or add them in the Keycloak console
(**Groups → {{GROUP-NAME}} → auditors → Members**).

Users are identified by **email** in the NDP realm.

## 4. Ask for a decision

This is the call your app makes, with the **user's** token. No client secret
involved. Keycloak evaluates the access rules you configured in step 1 and returns the
result; **your app is what allows or blocks the request.**

```bash
curl --location 'https://dev-idp.nationaldataplatform.org/realms/NDP/protocol/openid-connect/token' \
--header 'Content-Type: application/x-www-form-urlencoded' \
--header 'Authorization: Bearer {{USER-TOKEN}}' \
--data-urlencode 'grant_type=urn:ietf:params:oauth:grant-type:uma-ticket' \
--data-urlencode 'audience={{CLIENT-NAME}}' \
--data-urlencode 'permission=admin-area' \
--data-urlencode 'response_mode=decision'
```

--data-urlencode 'permission=editor-area' \
--data-urlencode 'permission=viewer-area'

| status | body | meaning |
|---|---|---|
| `200` | `{"result": true}` | allow |
| `403` | `{"error":"access_denied"}` | the rules evaluate to no |
| `401` | — | the token is invalid or expired |

>**Caveat — the token must come from the public Keycloak URL.** A token minted through the
AAI's `/api/client/login` carries `iss=http://keycloak:8080/realms/NDP`, the container's
internal address. The decision call above goes to `dev-idp.nationaldataplatform.org`, whose
issuer is the public URL, so it rejects that token with a `401` — the token is valid, it
just belongs to a different issuer. Get the token straight from
`https://dev-idp.nationaldataplatform.org/realms/NDP/protocol/openid-connect/token` and the
same call succeeds.

## Changing who has access

Move someone between permission levels, and the **next** evaluation reflects it — so does the
next decision your app makes. Nothing to redeploy, no need to refresh the token.

Every access change is a **membership** change made by a group admin. Nobody creates a
group to grant access, nobody edits code, and nobody touches the client. If the answer to
"how do I give this person access?" ever involves creating something, it is the wrong
answer.
