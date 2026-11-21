# Local DynamoDB Setup

## Overview

The project uses [DynamoDB Local](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/DynamoDBLocal.html) as a containerised stand-in for DynamoDB during local development. A bootstrap script reads the CloudFormation template and creates all required tables against the local instance.

## Containers involved

| Container | Image | Role |
|---|---|---|
| `dynamodb-local` | `amazon/dynamodb-local` | Local DynamoDB instance on port 8000 |
| `server` | `spirit-of-kiro-server` | App server; runs the bootstrap script |

## Bootstrapping tables

The bootstrap script reads `server/iac/dynamodb.yml` and creates the six tables the server needs: Items, Inventory, Location, Users, Usernames, and Persona.

```bash
podman exec server mkdir -p /app/server/iac && \
podman cp scripts/bootstrap-local-dynamodb.js server:/app/ && \
podman cp server/iac/dynamodb.yml server:/app/server/iac/ && \
podman exec server bun run /app/bootstrap-local-dynamodb.js
```

The script connects to DynamoDB at `DYNAMODB_ENDPOINT` (set to `http://dynamodb-local:8000` in the `server` container). Tables that already exist are skipped without error.

## Known issues and fixes

### 1. YAML warnings for `!Ref` and `!GetAtt`

**Symptom:** The bootstrap script emits `YAMLWarning: Unresolved tag: !Ref` / `!GetAtt` for every entry in the `Outputs` section of the CloudFormation template.

**Cause:** The `yaml` npm package does not know CloudFormation intrinsic function tags by default.

**Fix:** `scripts/bootstrap-local-dynamodb.js` registers these tags as custom resolvers when calling `parse()`, so they are silently accepted instead of warned about. The `Outputs` section is not used by the script anyway — only `Resources` is read.

---

### 2. `dynamodb-local` data directory not writable (tables hang on create)

**Symptom:** The bootstrap script hangs indefinitely on `🚀 Creating table...` and never prints `✅ Created table`. The `dynamodb-local` container logs show a repeating error:

```
com.almworks.sqlite4java.SQLiteException: [14] unable to open database file
SQLiteQueue[shared-local-instance.db]: stopped abnormally, reincarnating in 3000ms
```

**Cause:** Rootless Podman maps the host user (uid 1000) to `root` (uid 0) inside the container's user namespace. A bind-mounted directory owned by the host user therefore appears owned by `root` inside the container. The `dynamodblocal` process (uid 1000 inside the container) cannot write to it, so SQLite cannot open its database file. DynamoDB Local responds to reads (returning HTTP 400) but hangs on any write operation.

**Fix:** Use `podman unshare` to set ownership within the correct user namespace:

```bash
podman unshare chown 1000:1000 ./docker/dynamodb
podman restart dynamodb-local
```

`podman unshare` runs the command inside the same user namespace Podman uses, so uid 1000 inside the container maps to the right subordinate uid on the host. After the restart, `dynamodb-local` can open its SQLite database and table creation succeeds.

**This needs to be run once after cloning the repo**, or any time the `docker/dynamodb` directory is recreated. Add it to your project setup script or `Makefile` to avoid repeating this manually:

```bash
podman unshare chown 1000:1000 ./docker/dynamodb
```

---

### 3. SDK credential hang (background context)

The `server` container has `AWS_CONTAINER_CREDENTIALS_RELATIVE_URI=/creds` set, which tells the AWS SDK to fetch credentials from the ECS local endpoints container. For the bootstrap script this is irrelevant because DynamoDB Local accepts any credentials — but if that endpoint is unreachable, the SDK will hang waiting for credentials before it even sends the DynamoDB request. This manifests identically to issue #2.

If you suspect credential fetching is the cause rather than the SQLite issue, test with:

```bash
podman exec server bun -e "fetch('http://169.254.170.2/creds').then(r=>console.log(r.status)).catch(e=>console.log(e.message))"
```

A hang here confirms the ECS metadata endpoint is unreachable. Issue #2 (SQLite) is the more common cause during initial setup.
