# NDP Keycloak Authorization

| document | what it is | read it if you… |
|---|---|---|
| [problem.md](problem.md) | How NDP services use Keycloak today, what each one exposes, and the client-scoped model that fixes it | want the background, or want to know why adopt this new approach |
| [ndp-new-auth.md](ndp-new-auth.md) | Walkthrough of the new model: create a client, put people in groups, ask Keycloak for a decision | are integrating a service and want the calls to make |

## The idea in three lines

- Today every token carries the user's **entire** group and role footprint, and each
  service picks through it. That over-exposes users and makes tokens grow without bound.
- Instead, each client gets its own access rules, and your service asks Keycloak
  *"may this user do X?"* — one call, yes or no.
- Access changes by moving someone between `admins` / `editors` / `viewers`. No redeploy,
  no role parsing in your code.
