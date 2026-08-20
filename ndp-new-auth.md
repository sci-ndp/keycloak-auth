# NDP New Auth Walkthrough

Your service asks Keycloak "may this user do X?" and gets back yes or no. No role
parsing in your code, and the rule changes without a redeploy.

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

Sections 1 and 3 below are admin work. Section 4 is the only part your service code does
at runtime.

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

This is the call your service makes per request, with the **user's** token. No
client secret involved.

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
| `403` | `{"error":"access_denied"}` | a rule said no |
| `401` | — | the token is invalid or expired |

>**Caveat — the token must come from the public Keycloak URL.** A token minted through the
AAI's `/api/client/login` carries `iss=http://keycloak:8080/realms/NDP`, the container's
internal address. The decision call above goes to `dev-idp.nationaldataplatform.org`, whose
issuer is the public URL, so it rejects that token with a `401` — the token is valid, it
just belongs to a different issuer. Get the token straight from
`https://dev-idp.nationaldataplatform.org/realms/NDP/protocol/openid-connect/token` and the
same call succeeds.

## A real use case: provisioning an NDP Endpoint

When someone registers an endpoint with `POST /ep/simple`, the
[NDP Federation API](https://github.com/sci-ndp/ndp-federation) does the Keycloak setup on
their behalf. The requester never creates a group — they asked to *register an endpoint*.

### Today: two calls

[pop.py:461-489](https://github.com/sci-ndp/ndp-federation/blob/fe4ca0d68faf9a4fd461bf739ac8d3dcef08a9ec/app/services/pop.py#L461-L489),
as the platform admin using a factory client:

1. `/client/create` with `{"clientName": "ep-{id}"}` — nothing else.
2. `/group/create` with `{"groupName": "ndp_ep/ep-{id}", "adminUsers": [userid]}`.

The group is fine. The problem is the client is created **with no group**, so it has no
resources and no policies — nothing to answer a `permission=` question. The two are
linked only by matching names in the federation's code, not in Keycloak.

### With the new `/client/create`: one call

```bash
curl --location 'https://dev-idp.nationaldataplatform.org/api/client/create' \
--header 'Content-Type: application/json' \
--header 'Authorization: Bearer {{FACTORY-TOKEN}}' \
--data '{
  "clientName": "ep-{id}",
  "groupName": "ndp_ep/ep-{id}",
  "adminUsers": ["{userid}"]
}'
```

Same client, same group, same seeded admin — plus the access rules that make section 4
work. In `keycloak.py`, fold `create_group` into the `create_client` payload and delete it.

### The roles are different, and this is the real difference

| | old `/group/create` | new `/client/create` |
|---|---|---|
| creates | **realm roles** | realm roles **+ client roles** |
| named | `group:ndp_ep/ep-{id}:admin` | `admin` on client `ep-{id}` |
| lives in | the realm, shared by everything | that one client |
| in the token | `realm_access.roles`, everywhere | `resource_access.ep-{id}.roles`, only here |

The old call gives you realm roles only — one realm-wide role per group per level, piling
up in a namespace everything shares. The new call also mints a **client role** per
permission level and maps it to that level's subgroup, so the client's own gates are
backed by roles scoped to it. Two payoffs: your service can read
`resource_access.ep-{id}.roles` straight from the token instead of asking, and the
client's token scope is confined to those roles, so its tokens stop carrying whatever else
the user happens to hold elsewhere in the realm.

**Hand-off.** The platform admin is now out of the loop. The endpoint's admin adds
colleagues with `/group/add-user` against `ndp_ep/ep-{id}`, as in section 3. An admin of
the parent `ndp_ep` still covers every endpoint under it.

### Guarding the EP's API with the policies

The EP service never reads a role from token. One dependency asks section 4 and honors the answer:

```python
DECISION_URL = f"{IDP}/realms/NDP/protocol/openid-connect/token"

def requires(permission: str):
    async def guard(request: Request):
        async with httpx.AsyncClient(timeout=5) as c:
            r = await c.post(DECISION_URL,
                headers={"Authorization": request.headers.get("authorization", "")},
                data={"grant_type": "urn:ietf:params:oauth:grant-type:uma-ticket",
                      "audience": CLIENT_ID,          # ep-{id}
                      "permission": permission,       # viewer-area / editor-area / admin-area
                      "response_mode": "decision"})
        if r.status_code != 200:
            raise HTTPException(status_code=403, detail="Not allowed")
    return Depends(guard)
```

Then each route names the level it needs:

```python
@router.get("/datasets",         dependencies=[requires("viewer-area")])
@router.post("/datasets",        dependencies=[requires("editor-area")])
@router.delete("/datasets/{id}", dependencies=[requires("admin-area")])
```

That is the whole guard. No role names, no group names, no `ENABLE_GROUP_BASED_ACCESS`
flag — it replaces the hand-parsing in ep-api's `authorization_service.py`
(`check_group_membership`, `group:{ep_uuid}:writer`, and the rest). Access changes when
someone is moved between levels, not when this code changes.

**Offline variant.** Because the new `/client/create` also mints client roles, a token
issued for this client already carries `resource_access["ep-{id}"]["roles"]`. Reading that
from the verified token gives the same answer with no network call — faster, but it goes
stale until the token is refreshed, while the decision call is always current. Use the
decision call for anything destructive.

So the three layers stay clean:

```
platform admin  →  provisions client + group        (once, one call)
EP admin        →  adds/moves members               (ongoing, self-service)
EP service      →  asks Keycloak per request        (runtime, no roles in code)
end user        →  logs in                          (that's all)
```

## Changing who has access

Move someone between permission levels, the **next** decision reflects it. Nothing to redeploy,
no need to refresh the token.

Every access change is a **membership** change made by a group admin. Nobody creates a
group to grant access, nobody edits code, and nobody touches the client. If the answer to
"how do I give this person access?" ever involves creating something, it is the wrong
answer.
