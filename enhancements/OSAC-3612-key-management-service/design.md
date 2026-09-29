# Key Management Service — Key Lifecycle Management

| Field       | Value |
|-------------|-------|
| Author(s)   | Dakota Crowder |
| Jira        | [OSAC-3612](https://redhat.atlassian.net/browse/OSAC-3612) |
| PRD         | [prd.md](prd.md) |
| Date        | 2026-09-22 |

# 1. Overview

This design adds tenant-owned and provider-owned `ManagedKey` resources to the Fulfillment API and performs their lifecycle operations synchronously against HashiCorp Vault Transit. The OSAC CLI exposes the same workflows. PostgreSQL holds tenant attribution, confirmed lifecycle timestamps and material-version metadata; key material exists only in Vault. Downstream resources associate through typed references to the stable logical key. See the [PRD](prd.md) for product requirements.

The first supported provider is HashiCorp Vault Transit, selected through deployment-managed backend and policy configuration. Vault ACLs enforce reversible revocation without deleting key material. A narrow internal provider contract, opaque backend coordinates, and backend-neutral version records keep Transit semantics out of the public API so KMIP and other providers can be added without replacing logical-key identities. `[User]` `[Research: Vault and OpenBao Transit]`

The PRD has no FR/NFR identifiers. This design assigns the following traceability-only anchors without changing the requirement text:

| ID | PRD requirement anchor |
|----|------------------------|
| FR-1 | Create and view tenant-owned and provider-owned logical keys through API and CLI. |
| FR-2 | Expose confirmed lifecycle state, active version, retained versions, and actionable uncertainty after failed requests. |
| FR-3 | Rotate a logical key while preserving its identity and retained material versions. |
| FR-4 | Revoke all versions, block normal use and new associations, and permit explicit authorized recovery. |
| FR-5 | Define consumer-neutral key references and reject destruction while a consumer remains attached. |
| FR-6 | Enforce Tenant Admin, Cloud Provider Admin, Tenant User, and Cloud Infrastructure Admin boundaries. |
| FR-7 | Let Cloud Provider Admins configure transparent platform backends and policies without tenant selection. |
| FR-8 | Give Cloud Infrastructure Admins read-only KMS health and availability visibility. |
| FR-9 | Return confirmed success or actionable failure, distinguish uncertain provider outcomes from committed key state, and cover API/CLI journeys. |

# 2. Goals and Non-Goals

## 2.1 Goals

- Keep the `ManagedKey` object flat, like `Secret`, and complete normal Vault lifecycle calls before returning from the API. `[User]` `[Codebase: proto/private/osac/private/v1/secret_type.proto]`
- Preserve a stable OSAC logical-key ID and backend-neutral material generations across provider-specific rotations. `[Locked: D4, D13]`
- Use the existing typed-reference and reverse-delete-guard pattern for future key consumers. Complete lifecycle calls in the request and accept the same cross-store drift risk as Secrets when a Vault effect succeeds but the database transaction fails. `[User]` `[Codebase: fulfillment-service/internal/servers/private_secrets_server.go]`
- Support HashiCorp Vault Transit first without exposing Transit paths or numeric versions publicly. `[User]`
- Validate the backend behavior required for this key usage internally; leave storage-consumer compatibility to OSAC-2389. `[User: lean public contract]` `[Research: Problem Space]`

## 2.2 Non-Goals

- This design does not add a key-management UI, automated rotation schedules, client-side cryptography, or key-material export.
- This design does not implement KMIP listeners, storage-resource bindings, re-encryption, crypto-erase orchestration, HSM integration, replication, or multi-region management. `[Locked: D5]`
- This design does not let tenants select a backend or policy and does not add provider-managed shared/default keys for tenant consumption. `[Locked: D10, D16]`
- This design does not add an audit-history product surface; ordinary resource events report committed key changes, not an audit log. `[Locked: D15]`

# 3. Motivation / Background

The current Vault-compatible integration provisions tenant namespaces and KV v2 mounts for secrets. Its `Secret` create, update, and delete handlers call Vault synchronously within the API request and accept possible drift if a later PostgreSQL commit fails. A repeated Secret update can create another internal KV version. It has no Transit client, logical-key resource, or material-version model. This design accepts the same cross-store tradeoff for key lifecycle calls. A retried Rotate can create another retained version; OSAC does not promise one version per request. Existing typed references and database delete guards supply the pattern for protecting keys used by downstream resources. `[User]` `[Codebase: fulfillment-service/internal/servers/private_secrets_server.go]`

Directly exposing Transit would couple the API to a named Vault key and its numeric versions. KMIP re-key can instead create a replacement object, while other providers use stable resources with separate material versions. OSAC therefore needs its own stable identity, key usage, lifecycle policy, and provider translation boundary. `[Research: Existing Solutions and Tools]`

HashiCorp Vault Transit is the initial provider. Its documented API has no reversible whole-key disable operation, so OSAC enforces the PRD's revocation rule through key-specific Vault ACL policies attached to every normal consumer credential. Revocation removes cryptographic permissions from existing and new consumer tokens while retaining all key versions; recovery restores permissions after authorization. This differs from the earlier research recommendation and makes credential issuance and ACL conformance release gates. `[User]` `[Research: Vault and OpenBao Transit]` [Vault Transit API](https://developer.hashicorp.com/vault/api-docs/secret/transit) [Vault policies](https://developer.hashicorp.com/vault/docs/concepts/policies)

# 4. Design

## 4.1 Architecture

The Fulfillment Service owns the public contract and persistence. Each `ManagedKeys` lifecycle request invokes a backend-neutral `KeyProvider` during the request. It locks the key row to serialize same-key lifecycle changes, calls Vault, and records the confirmed result before returning success. No key consumer is added in OSAC-3612, so there is no reverse-reference guard or durable destruction fence yet; OSAC-2389 adds those with the first concrete consumer. The initial `HashiCorpVaultTransitProvider` uses the existing Vault connection and tenant namespace/authentication infrastructure with a dedicated Transit mount and per-key consumer ACL policies. Tenant namespaces require Vault Enterprise or HCP Vault Dedicated. `[User]` [Vault namespaces](https://developer.hashicorp.com/vault/docs/enterprise/namespaces)

```mermaid
flowchart LR
    User[API or OSAC CLI] --> Public[Public Fulfillment API]
    Public --> Private[Private ManagedKeys server]
    Private --> DB[(PostgreSQL)]
    Private --> Provider[KeyProvider interface]
    Provider --> Transit[HashiCorp Vault Transit]
    Consumer[Downstream OSAC resource] --> Reference[ManagedKeyLocalReference]
    Reference --> DB
```

The handler follows the Secret request-transaction pattern: it calls Vault during the request, then commits the database change. The row lock is held while an existing key is changed. A successful RPC means the Vault effect and database write were confirmed. If Vault succeeds but the database transaction rolls back, PostgreSQL retains the old state and an operator must repair the drift; Create can leave an orphaned Vault key. A timeout or crash can likewise leave an uncertain provider effect, with no operation record, background reconciliation, or exactly-once replay guarantee. Downstream resources store the key reference in their own specifications; public key responses omit consumer details and backend coordinates. OSAC-issued consumer credentials carry only the corresponding key-specific Vault policy; no normal consumer receives a broad Transit credential or a path to bypass the revocation gate. `[User]` `[Locked: D3, D8, D14]` `[Codebase: fulfillment-service/internal/database/database_tx_interceptor.go]`

The internal provider contract expresses product intent rather than vendor verbs:

```go
type KeyProvider interface {
    Observe(ctx context.Context, key ProviderKeyRef) (ObservedKey, error)
    Create(ctx context.Context, key ProviderKeyRef, policy KeyPolicy) (ObservedKey, error)
    Rotate(ctx context.Context, key ProviderKeyRef) (ObservedKey, error)
    Revoke(ctx context.Context, key ProviderKeyRef) (ObservedKey, error)
    Recover(ctx context.Context, key ProviderKeyRef) (ObservedKey, error)
    Destroy(ctx context.Context, key ProviderKeyRef) error
    Health(ctx context.Context, backend BackendRef) (BackendHealth, error)
}
```

`ProviderKeyRef`, backend object IDs, and backend versions are private opaque values. The configured provider and `ENCRYPT_DECRYPT` policy define the required behavior, so this interface has no general capability-discovery method. Deployment validation and `Health` check the required Vault access, mount, namespace, and policy behavior; the provider conformance suite tests the full lifecycle. A future purpose or provider with optional behavior can add a specific check when a consumer needs it.

The public ManagedKeys `Delete` method calls the provider's `Destroy` operation. The internal name distinguishes permanent key-material removal from deleting only OSAC metadata.

Adapter errors carry a normalized cause (`UNAVAILABLE`, `UNAUTHORIZED`, `UNSUPPORTED`, `CONFLICT`, or `INVALID_CONFIGURATION`). A mutation error separately records whether the adapter can prove no provider effect or the outcome is uncertain, including a partial effect. `RETRYABLE` and `TERMINAL` are not mutually exclusive causes: a timeout can be retried but may already have rotated a key, while a configuration error may be fixable but cannot succeed unchanged. The lifecycle handler uses the cause and effect certainty to choose an actionable public error; it does not automatically repeat an uncertain mutation.

The public lifecycle is derived from confirmed facts, not a separate state machine:

| Confirmed result | Public fields after the final database commit |
|------------------|-----------------------------------------------|
| Create | Publish the key with generation `1` after Vault and PostgreSQL succeed; a rolled-back create is not listed and may leave an orphaned Vault object. |
| Rotate | Append every newly observed material generation; the final element of `versions` becomes the current confirmed generation, and earlier generations remain listed. |
| Revoke | Set `revocation_timestamp` after the ACL change and access denial are verified. |
| Recover | Clear `revocation_timestamp` after normal access is verified. |
| Delete | Stage removal of the OSAC record, delete the Vault key and its key-specific policy, then commit the database deletion. |

A present `revocation_timestamp` means the retained key is revoked; when absent, its last committed OSAC state is active. This timestamp records when OSAC confirmed revocation, not the exact Vault change time, and is cleared after confirmed recovery. Normal lifecycle calls return the confirmed result synchronously. Successful Delete removes the key from `Get` and `List`; its empty response and the ordinary deletion event confirm completion. There is no retained destroyed state. A failed or timed-out call reports that its Vault effect may be uncertain and leaves the last committed key state intact if the database transaction rolled back; it does not create a public interim state. `Get` and `List` return committed metadata without claiming a live Vault check. Operators verify and repair provider differences before further lifecycle changes, except that Delete can retry when the stored key's Vault object is already absent. There is no public operation history or unattended reconciliation. `[User: simplify synchronous lifecycle]` `[Locked: D8, D15, D19]`

### Transit mapping

| OSAC intent | HashiCorp Vault Transit effect | Confirmation |
|-------------|------------------------|--------------|
| Create | `POST /{mount}/keys/{opaque-name}` with `aes256-gcm96`, `exportable=false`, `allow_plaintext_backup=false`; create a key-specific consumer ACL policy | Read key and policy; latest version is 1 and configuration matches policy |
| Rotate | `POST /{mount}/keys/{opaque-name}/rotate` | Read key; latest version advanced and earlier versions remain available |
| Revoke | Replace the key-specific consumer policy's cryptographic path grants with explicit `deny` rules; retain the Transit key and every version | Read policy and verify a previously issued consumer token cannot encrypt or decrypt any retained version; only then set `revocation_timestamp` |
| Recover | Restore the key-specific consumer policy's cryptographic grants after authorized recovery | Read policy and verify a consumer token can encrypt and decrypt retained versions; only then clear `revocation_timestamp` |
| Delete | If the key exists, set `deletion_allowed=true` and call `DELETE /{mount}/keys/{opaque-name}`; remove the key-specific ACL policy | Verify that the key and policy are absent; then commit removal of the OSAC record. An already absent key or policy is accepted on retry. |

Backend names are deterministic (`osac-<managed-key-uuid>`) and never derived from tenant-provided names. The adapter treats an existing name with mismatched immutable configuration as `CONFLICT`; it never adopts or overwrites an out-of-band key.

The per-key policy covers every enabled cryptographic path for that key, including encrypt, decrypt, rewrap, data-key generation, HMAC, sign, and verify where applicable. Its `deny` rules take precedence over other ordinary ACL grants. All normal Transit credentials for a key must carry this policy, including credentials issued before revocation; credentials with root or unrestricted administrative access are excluded from normal consumer use. The API uses a separate restricted management credential to change the policy. If policy write or verification fails, `revocation_timestamp` is not changed and the RPC returns an error; an uncertain or partial effect requires verification and repair. Provider administrators inspect out-of-band policy drift through backend health and operation errors; there is no automatic repair loop. `[Locked: D8]` [Vault policies](https://developer.hashicorp.com/vault/docs/concepts/policies)

## 4.2 Data Model / Schema Changes

The editable private proto defines the public shape; `cleanapi` removes private provider fields from generated public protos. These proposed definitions use the existing `Metadata`, `google.protobuf.Timestamp`, and `cleanapi` types; omitted imports and service annotations follow the existing resource protos. The public usage follows [Google Cloud KMS's `CryptoKeyPurpose`](https://github.com/googleapis/googleapis/blob/master/google/cloud/kms/v1/resources.proto#L59). No asynchronous operation resource or interim transition field is defined. `[User]` `[Codebase: proto/private/osac/private/v1]`

```protobuf
enum ManagedKeyUsage {
  MANAGED_KEY_USAGE_UNSPECIFIED = 0;
  MANAGED_KEY_USAGE_ENCRYPT_DECRYPT = 1;
}

message ManagedKey {
  string id = 1;
  Metadata metadata = 2;
  ManagedKeyUsage usage = 3; // Immutable; ENCRYPT_DECRYPT only in this release.
  repeated ManagedKeyVersion versions = 4; // Nonempty; ascending by generation. Last is current.
  google.protobuf.Timestamp revocation_timestamp = 5; // Present only while confirmed revoked; system-owned.
  google.protobuf.Timestamp rotation_trigger = 6; // Change through Update to request rotation.
  google.protobuf.Timestamp revocation_trigger = 7; // Change through Update to request revocation.
  google.protobuf.Timestamp recovery_trigger = 8; // Change through Update to request recovery.
  KeyBackend backend = 9 [(cleanapi.field).private = true];
}

message ManagedKeyVersion {
  uint64 generation = 1;
  google.protobuf.Timestamp creation_timestamp = 2;
  string backend_object_id = 3 [(cleanapi.field).private = true];
  optional string backend_version_id = 4 [(cleanapi.field).private = true];
}

enum KeyBackend {
  option (cleanapi.enum).private = true;
  KEY_BACKEND_UNSPECIFIED = 0;
  KEY_BACKEND_VAULT_TRANSIT = 1;
}

```

`metadata.tenant` is the sole ownership signal: `system` means a provider-owned key, and any other permitted tenant means a tenant-owned key. The server uses this value to choose the Vault namespace; one deployment-managed policy applies to both ownership types. `shared` is invalid for ManagedKeys because it is visible to ordinary users and provider-managed keys for tenant consumption are out of scope. The tenant is immutable after creation. `usage=ENCRYPT_DECRYPT` means the key is for symmetric encryption and decryption; the configured policy fixes `aes256-gcm96`. It does not promise that OSAC exposes cryptographic data-plane RPCs. Every committed key has at least one `ManagedKeyVersion`; `versions` is ordered by ascending OSAC generation, and its final element is the current confirmed generation. Earlier elements remain retained, including while the key is revoked. Delete removes the logical key and every generation; no separate version-state enum or `active_version` field is needed. A successful rotation Update appends every newly observed Vault version as an OSAC generation before the database transaction commits. If the commit fails, no generation is added by `Get`; the operator repair procedure must reconcile the observed versions. Each OSAC generation has exactly one private backend material reference: the Vault key name and numeric Transit version, or a replacement object's ID for a provider without separate version IDs. OSAC owns the monotonic generation; backend IDs remain opaque, may differ from that generation, and are never exposed to consumers. A later provider requiring multiple material objects for one generation must establish that need before expanding the private mapping. Rotation mode, ACL mechanics, and destruction mode remain private provider details. No field contains key material or a client request ID. `[User: simplify version mapping]` `[Locked: D4, D8, D13, D15, D16]` `[Review: PR 319, jhernand]`

The three trigger timestamps are caller-supplied change tokens, not execution times or desired-state fields. The server stores a new value only after the corresponding Vault effect and database write are confirmed. `Get` and `List` show the last committed trigger values alongside the system-owned `revocation_timestamp`; they do not claim that Vault was rechecked. `[Review: PR 319, jhernand]`

Lifecycle preflight compares the stored private material reference with provider observation; it never assumes that an OSAC generation equals a Vault version number. `[User: simplify version mapping]`

Consumers use the same typed-reference convention as other Fulfillment resources:

```protobuf
message ManagedKeyLocalReference {
  string id = 1;
  string name = 2; // Resolve within the caller's authorized tenant scope.
}

```

`ManagedKeyLocalReference` identifies the stable logical key, never a material generation. A consumer stores it in its own resource, such as `spec.managed_key`, and the server resolves its `id` and `name` within the authorized tenant scope. The consumer's Create/Update validation rejects cross-tenant, revoked, missing, or backend-drifted keys and takes a shared lock on the key row. The same consumer change must add a reverse-reference check to `ManagedKeys.Delete`, including soft-deleted versus active-resource semantics, under an incompatible key-row lock. This matches the existing Secret reference and delete-guard pattern. No consumer-specific fields or list of consumers appear on `ManagedKey`. `[User]` `[Locked: D3, D5, D13, D14]` `[Codebase: fulfillment-service/docs/API.md]` `[Codebase: fulfillment-service/internal/database/migrations/111_add_secret_delete_protection_trigger.up.sql]`

Like `Secret`, `ManagedKey` exposes flat fields and reports confirmed Vault effects synchronously. It has no separate operation record, transition field, or exactly-once retry semantics. Create has no precommitted reservation: if Vault creates a key but PostgreSQL does not commit, the provider object is orphaned and must be found and removed through operator tooling. `[User]` `[Codebase: proto/private/osac/private/v1/secret_type.proto]`

PostgreSQL receives one additive key-resource migration:

- `managed_keys`: the standard generic-resource columns plus JSONB `data`. Provider-owned keys use the reserved `system` tenant; tenant-owned keys use their owning tenant. No separate ownership column is needed. The tenant and assigned private backend are immutable after creation.

Each concrete consumer adds a reference field and its own forward-validation and reverse-delete guard in the same change. OSAC-3612 defines the reference type but adds no consumer binding, reverse-reference check, or durable destruction fence because the first storage consumer belongs to [OSAC-2389](https://redhat.atlassian.net/browse/OSAC-2389). That consumer change must make reference admission and Delete exclusion race-safe, including the case where Vault deletion succeeds but the database commit fails. A private persisted fence may be needed then; its exact form belongs with the concrete consumer design. No generic association table or service is created. `[User]` `[Locked: D3, D5, D8]`

Connection, namespace, mount, authentication, policy, and startup-validation details remain private deployment configuration. The service validates required backend access before admitting key creation; existing request metrics and redacted logs show failures during key operations. Full lifecycle behavior is proven separately by provider conformance tests. No backend status resource is added to the public API. `[User: use existing observability]` `[Locked: D10, D17]`

## 4.3 API Changes

### ManagedKeys service

`osac.public.v1.ManagedKeys` and its private counterpart implement only the standard `Create`, `List`, `Get`, `Update`, and `Delete` methods at `/api/fulfillment/v1/managed_keys`. `Update` handles rotation, revocation, and recovery through changes to trigger fields on the key. `Delete` maps to HTTP `DELETE /api/fulfillment/v1/managed_keys/{id}` and removes both the provider key and the OSAC record, like Secret Delete. The existing CLI delete command exposes this as `osac delete managedkeys <id>`. These are additive APIs. `[Review: PR 319, jhernand]` `[Codebase: fulfillment-service/docs/API.md]` `[Codebase: fulfillment-service/internal/cmd/cli/delete/delete_cmd.go]`

- `Create` requires `metadata.name` and defaults `usage=ENCRYPT_DECRYPT`, the only supported usage in this release. A Tenant Admin may omit `metadata.tenant` to use their authorized tenant or set it to an authorized tenant. A Cloud Provider Admin must specify a tenant: `system` creates a provider-owned key, while an ordinary tenant creates a tenant-owned key under existing broad administrative access. Only Cloud Provider Admins may create in `system`; `shared` is always rejected. This explicit rule prevents the generic administrator default of `shared` from creating a broadly visible key. The server rejects caller-supplied trigger fields, `revocation_timestamp`, versions, or provider fields. It publishes the key only after Vault creation and the database result are confirmed.
- `Update` uses the standard `{ object, update_mask, lock }` request and returns the updated key. For lifecycle changes, the server requires `lock=true` and the last observed `metadata.version`. A changed `rotation_trigger`, `revocation_trigger`, or `recovery_trigger` named in `update_mask` requests that effect. A lifecycle Update must change exactly one trigger and cannot include metadata edits or another trigger in the same mask; an unchanged trigger value causes no provider effect. Create requests with triggers and malformed lifecycle Updates return `InvalidArgument`. The timestamps serve as change tokens, so the server does not require them to match its clock. Ordinary Updates may change only `metadata.display_name` and `metadata.description`. Tenant, usage, `revocation_timestamp`, versions, and provider fields remain immutable or system-owned. `[Review: PR 319, jhernand]` `[Codebase: fulfillment-service/docs/API.md]`
- Same-key lifecycle Updates serialize through the database row lock during the request. Before changing Vault, the server observes the relevant provider state and rejects a mismatch with the stored metadata using `FailedPrecondition` and an actionable repair message. Rotation and revocation require an active key; recovery requires a revoked key. After an uncertain outcome, an administrator checks the last committed key state and verifies Vault through operator tooling before another lifecycle Update; a retry of a rotation Update may create another retained version when the earlier Vault outcome is unknown. `Delete` follows the standard ID-only request shape, accepts an active or revoked key, and accepts an already absent provider key so that a failed database commit after Vault deletion can be retried. After successful Delete, the OSAC key ID no longer exists and further Updates return `NotFound`.
- `Delete` has no consumer reference to check in OSAC-3612. Beginning with OSAC-2389, it must return `FailedPrecondition/KeyInUse` while an active supported consumer resource references the key, without revealing consumer identities. `[User]` `[Locked: D3, D14]`
- Like `PrivateSecretsServer.Delete`, ManagedKeys `Delete` stages the database deletion, deletes the provider resource in the same request, and commits the database transaction only after provider success. The stored key still supplies tenant authorization, backend coordinates, and the reference guard on retry. If the Vault key or policy is already absent, Delete treats that part as complete, verifies absence, and commits the OSAC deletion. A repeat after the OSAC deletion has committed returns `NotFound`, as Secret Delete does; the provider-side retry is idempotent while the OSAC row remains. `[Codebase: fulfillment-service/internal/servers/private_secrets_server.go]`
- `Get` and `List` return tenant-filtered last-committed database metadata without a Vault call. They do not add `ManagedKeyVersion` records, repair a failed lifecycle write, or guarantee that Vault still matches the stored state. After a lost response or process crash, a caller may see stale metadata if the transaction rolled back, or `NotFound` if Delete committed. An existing row permits a Delete retry; an uncertain Rotate requires provider verification before retry. Neither method exposes backend coordinates, credentials, or key bytes.

The lifecycle Update calls Vault before returning success. A definitive failure with proof of no provider effect returns an actionable gRPC error and leaves the confirmed fields unchanged; no `FAILED` resource state is stored. If the provider effect or final PostgreSQL commit is uncertain, the server returns `Unavailable` or `DeadlineExceeded` and warns that committed metadata may differ from Vault. `Get` still returns the last committed OSAC state; provider verification uses operator tooling, and subsequent lifecycle Updates reject a detected mismatch before making another Vault change. Delete can complete a previously applied provider deletion while its OSAC row remains. Repeating the same trigger after a committed Update has no effect, but a timeout before commit may permit an overlapping retry that creates an extra rotation version. Vault is authoritative for key material, current version, and policy effects; PostgreSQL is authoritative for logical identity, tenant, and committed lifecycle metadata. `[User]` `[Review: PR 319, jhernand]`

Example rotation `Update` request, shown as JSON:

```json
{
  "object": {
    "id": "7e24b74d-7dd4-4ff1-86bb-503d37f112d3",
    "metadata": { "version": 4 },
    "rotation_trigger": "2026-09-29T12:00:00Z"
  },
  "update_mask": "rotation_trigger",
  "lock": true
}
```

Success returns the key with `versions` containing generations `1` and `2` in that order; generation `2` is current. If Vault creates version `2` but the database commit fails, `Get` still returns only committed generation `1`; it does not add generation `2` to the public key. The failed Update warns that Vault may differ, and another lifecycle Update rejects the observed mismatch until an administrator verifies Vault and repairs the mapping. A timeout before Vault's outcome is known requires the same verification; repeating the rotation Update without it can create another retained version if the original POST is still in flight. A definite no-effect failure returns a stable gRPC reason and message while the final confirmed generation remains `1`.

### Consumer reference contract

`ManagedKeyLocalReference` is defined in the key type proto for use by downstream Fulfillment resources. A consumer owns its reference field, validation, and reverse-delete guard. When such a field is added, its Create/Update path returns:

- `FailedPrecondition/KeyNotActive` for revoked keys;
- `InvalidArgument/TenantMismatch` for a consumer outside the key's tenant;
- `InvalidArgument` for an unknown or deleted key reference.

The key API has no association list or detail surface. There is no independent association CRUD service. OSAC-2389 adds the first concrete storage reference and the corresponding guards; a separate registry is considered only if a later consumer cannot store a typed reference in a Fulfillment resource. `[User]` `[Locked: D5, D14]`

## 4.4 Scalability and Performance

Key lifecycle requests are administrative control-plane operations; encryption and decryption data-plane traffic does not pass through the Fulfillment API. Each lifecycle Update observes provider state and makes the required Vault Transit or policy calls before returning success. `Get` and paginated `List` read database metadata without a Vault call. Provider concurrency is bounded per backend, and concurrent changes to the same key serialize on its database row. A timed-out Vault call may still finish after the request transaction rolls back, so a later retry can create an extra rotation version. The key-specific ACL policy adds one policy object per key and a policy update plus verification for revocation and recovery. A slow Vault call increases that RPC's latency and holds a PostgreSQL transaction open, as in the Secret flow.

Each consuming resource adds an index on its canonical key ID if needed for its reverse-reference check, following the existing Secret guard pattern. List pagination and CEL filtering follow existing generic-resource behavior.

Version metadata remains with the key. Historical material metadata grows linearly with rotations; automated rotation is out of scope, so growth follows explicit administrative operations and retries. Successful Delete removes the key and its version metadata.

## 4.5 Security Considerations

- HashiCorp Vault generates and stores all key material. Material, backups, plaintext, ciphertext payloads, and export endpoints are never represented by the OSAC API, database, CLI, logs, metrics, or events.
- Policies force `exportable=false`, `allow_plaintext_backup=false`, and disable automatic Transit rotation. Destruction temporarily enables only the provider-side deletion flag required for the selected key.
- Backend names use UUIDs rather than user input, preventing path injection and tenant-name disclosure. Endpoint, mount, namespace, and consumer-reference fields receive length and character validation before use.
- Provider calls use tenant-scoped Vault namespaces and least-privilege policies limited to the configured Transit mount. Provider-owned keys use a provider namespace inaccessible to tenant tokens. Normal consumer tokens are key-scoped and cannot modify their ACL policy; only the management identity can change revocation policy.
- Status and error messages are redacted: they may name the normalized backend and reason but not tokens, certificate data, raw provider responses, consumer identities, or key bytes.
- A consumer's reference field is validated under that resource's existing authorization and tenancy rules; possession of a key ID alone does not authorize a binding.

## 4.6 Failure Handling and Recovery

| Failure | System behavior | User-visible result |
|---------|-----------------|---------------------|
| Backend unavailable or sealed before Vault effect | The request transaction rolls back without changing the key. | The lifecycle RPC returns `Unavailable` with an actionable reason. `Get` and `List` still show last-committed metadata, not live provider availability. |
| Create times out or its database commit fails | As with Secrets, the database insert can roll back after Vault creates an object. The server does not adopt an unrecorded key on retry. An operator finds orphaned `osac-<uuid>` keys by comparing Vault names with persisted key IDs and removes them after checking that no OSAC key refers to them. | The RPC returns an error and does not claim a key was created. The caller checks List by name before retrying Create; a retry may create a new backend object. |
| Rotation times out | The request transaction rolls back. Vault may have created a version already or may create one later; the server does not record a generation without a confirmed database commit. | The RPC does not claim success. `Get` shows the last committed version; the caller verifies Vault before retrying. A later mutation rejects an observed mismatch with `FailedPrecondition`. |
| Revocation/recovery times out | The request transaction rolls back, but the Vault ACL may have changed. The operator checks the policy and an existing consumer credential before repair or retry. | The RPC reports uncertainty; `revocation_timestamp` stays at its last committed value in Get and List. A later mutation rejects an observed mismatch. |
| Deletion times out | The request transaction rolls back, but Vault may have deleted the key or policy. No key consumer exists in this feature, so there is no reference fence. | Get and List show the last committed metadata while its row remains. A retry of Delete uses that row, accepts already absent provider resources, and commits the OSAC deletion after verifying absence. |
| PostgreSQL final commit fails after Vault success | The request transaction rolls back and the provider effect may remain. Create can leave an orphaned backend object; Rotate, Revoke, or Recover can diverge from Vault; Delete leaves an OSAC row after provider deletion. | The RPC returns an error. Get and List show last-committed metadata. A later Rotate, Revoke, or Recover detects a mismatch and stops; a later Delete can complete the staged deletion. |
| Out-of-band backend or ACL mutation | Rotate, Revoke, and Recover preflight reject an unexpected provider state. Delete still enforces authorization and the consumer-reference guard, but treats an absent provider key or policy as already deleted. | Non-Delete mutations return `FailedPrecondition` with redacted corrective guidance; Get and List continue to show last-committed metadata until Delete succeeds. |

Lifecycle Updates require the standard optimistic lock; Delete follows the standard ID-only request and locks the stored key row. Vault's Rotate endpoint does not accept an idempotency key. Observing the prior version after a timeout cannot prove that an earlier POST will not finish later, so a repeated rotation trigger before a commit is explicitly at-least-once: it may create two new material versions. If Vault advances after OSAC's final write, Get and List still show the last committed OSAC version; the next lifecycle preflight detects the mismatch. An administrator must verify Vault and repair the version mapping before further lifecycle changes. No persistent operation marker survives a process crash; operator diagnostics and provider observation are the recovery path. `[User]` [Vault Transit rotate API](https://developer.hashicorp.com/vault/api-docs/secret/transit#rotate-key)

The initial recovery path is a documented, operator-only CLI procedure using the OSAC CLI and/or Vault CLI. Manual repair and possible extra retained versions after an uncertain Rotate are accepted limitations; this release has no repair RPC or automatic reconciliation. `[User]`

## 4.7 RBAC / Tenancy

| Persona | ManagedKeys | Consumer references |
|---------|-------------|---------------------|
| Tenant Admin | Create/read/update/destroy tenant-owned keys in assigned tenants | Governed by each consumer resource's permissions |
| Tenant User (ordinary client token) | No direct key-management access | Governed by each consumer resource's permissions |
| Cloud Provider Admin (`is_admin`) | Existing broad access plus provider-owned key lifecycle | Governed by each consumer resource's permissions |
| Cloud Infrastructure Admin (ordinary client token) | No direct key-management access | Governed by each consumer resource's permissions |
| Authorized downstream service | No public lifecycle access | Writes only the consumer resource fields it is authorized to manage |

OPA adds exact method rules for `ManagedKeys`, while existing tenancy logic filters tenant resources. The policy currently distinguishes administrators (`is_admin`), `tenant-admin`, `tenant-idp-manager`, and ordinary clients; it has no Cloud Infrastructure Admin predicate. ManagedKeys methods are granted to `tenant-admin` and the existing unrestricted administrators, and are omitted from client permissions. Thus Tenant Users and Cloud Infrastructure Admins with ordinary client tokens receive the same ManagedKeys denial. A Cloud Infrastructure Admin identity in an existing admin group or admin service account would inherit unrestricted ManagedKeys access, so deployments must keep that persona out of `is_admin` to satisfy the PRD's lifecycle boundary. This feature adds no infrastructure-admin realm role or KMS API grant; read-only health uses platform monitoring access. The ManagedKeys create path admits `system` only for existing administrators and rejects `shared`; it never accepts an unassigned administrator tenant default. Provider-owned keys are thus identified by `metadata.tenant=system` and do not inherit the visibility of `shared` resources. `[Codebase: fulfillment-service/internal/auth/policies/authz.rego]` `[Locked: D2, D12, D16, D17, D18]`

Cloud Infrastructure Admin identities must also lack the `tenant-admin` realm role; that role would receive the planned ManagedKeys grant. `[Locked: D17]`

Recovery uses the same lifecycle authority as revocation: Tenant Admin for a tenant-owned key and Cloud Provider Admin under existing administrative access. This release introduces no separate recovery privilege. `[User]`

## 4.8 Extensibility / Future-Proofing

The internal provider registry is keyed by immutable `backend_name`, which each key records privately. Deployment configuration accepts named backends and one shared `kms.policy` that selects a backend; only `vault-transit` is initially valid. The same policy applies to tenant-owned and provider-owned keys, while `metadata.tenant` selects their distinct Vault namespaces. The policy may change its selected backend for new keys, but existing keys keep their assigned backend. Tenants never choose either value. Adding a provider extends the closed backend-type enum and conformance suite rather than the public `ManagedKey` lifecycle. `[User: one shared configurable policy]`

The provider conformance suite checks rotation, retained-version access, revocation, recovery, and destruction for `ENCRYPT_DECRYPT` without publishing those mechanics as key fields. OSAC-2389 owns storage-consumer compatibility such as per-tenant key binding, re-encryption, and crypto-erase. If a later provider or key usage needs a client-visible distinction, add a usage value or a targeted field then; the logical key ID and version generations remain stable. `[User: lean public contract]` `[Research: Downstream KMIP readiness]`

OSAC material generations are independent of backend object IDs. A KMIP provider may therefore map each generation to a new managed object and replacement link without changing consumers' `ManagedKeyLocalReference` values or the public logical-key ID. The one-reference-per-generation mapping also covers Vault Transit, where every generation has the same backend object ID and a distinct backend version ID. `[User: simplify version mapping]` [KMIP Re-key](https://docs.oasis-open.org/kmip/kmip-spec/v2.1/kmip-spec-v2.1.html) [Vault Transit rotate](https://developer.hashicorp.com/vault/api-docs/secret/transit#rotate-key)

# 5. Interface Changes

## IC-1: ManagedKeys gRPC and REST resource

**Requirements:** FR-1, FR-2, FR-3, FR-4, FR-5, FR-6, FR-9

Adds public/private `ManagedKeys` standard CRUD with a flat `ManagedKey` object. Changes to the three lifecycle trigger fields in a locked `Update` request perform rotation, revocation, or recovery; no separate lifecycle RPCs are added. `Create`, lifecycle `Update`, and `Delete` report success only after their Vault effects and database writes are confirmed. Delete removes both the Transit key and OSAC record. On timeout, callers inspect the key before retrying; Delete can retry an already absent provider key while its OSAC row remains, but an uncommitted rotation trigger is not deduplicated. See §§4.2–4.3. `[Review: PR 319, jhernand]`

## IC-2: ManagedKey lifecycle status and Events payloads

**Requirements:** FR-2, FR-3, FR-4, FR-9

Adds backend-neutral usage, versions, three lifecycle trigger fields, and a confirmed revocation timestamp to `ManagedKey`. Ordinary Fulfillment object events carry committed changes, including deletion after successful Delete. `Get` and `List` return last-committed metadata for existing keys; provider-state mismatches are checked before lifecycle Updates. No interim transition, operation history, request IDs, key material, or backend coordinates enter the public payload. See §§4.2 and 4.6.

## IC-3: OSAC CLI key inspection commands

**Requirements:** FR-1, FR-2, FR-6

Adds `osac create key`, `osac list keys`, and `osac describe key`. The existing global `--tenant` flag selects the key's tenant; `osac --tenant system create key` is available only to Cloud Provider Admins. Describe derives active/revoked from the last committed `revocation_timestamp` value and derives the current generation from the final `versions` element. It shows `metadata.tenant`, usage, and current and retained generations. After successful Delete, Describe returns `NotFound`. The output does not claim to verify current Vault state.

## IC-4: OSAC CLI key lifecycle commands

**Requirements:** FR-3, FR-4, FR-5, FR-9

Adds `osac rotate key`, `osac revoke key`, and `osac recover key`; each CLI command reads the key, changes the corresponding trigger, and calls the standard locked `Update`. The existing generic `osac delete managedkeys <id>` command calls ManagedKeys `Delete`. Commands return when the synchronous Update or Delete completes and print the confirmed result: current revocation and the final confirmed generation for retained keys, or successful removal for Delete. On timeout they say the provider outcome may be uncertain; a Delete retry can finish deleting an existing OSAC row after provider deletion, while a repeated rotation Update may create another retained version. `--wait` is unnecessary.

## IC-5: ManagedKey local reference contract

**Requirements:** FR-4, FR-5

Adds the `ManagedKeyLocalReference { id, name }` type for downstream Fulfillment resources. Each consumer that adds this field must resolve it to an active same-tenant key, store the canonical key ID, and add a reverse-reference guard that blocks `Delete` while an active resource points to that ID. OSAC-3612 introduces no concrete consuming resource or durable deletion fence; [OSAC-2389](https://redhat.atlassian.net/browse/OSAC-2389) owns the first storage binding, its guard, and any fence required for cross-store failure safety. See §§4.2–4.3. `[User]`

## IC-6: Deployment-managed KMS backend and policy configuration

**Requirements:** FR-7

Adds Helm values and service flags for `kms.enabled`, named `kms.backends`, and one required `kms.policy` with a `backend` reference and `algorithm`. The initial closed backend type is `vault-transit`, and the only accepted algorithm is `aes256-gcm96`. The service enforces `exportable=false`, `allow_plaintext_backup=false`, and disabled automatic rotation; these safeguards cannot be relaxed by configuration. The shared policy applies to tenant-owned and provider-owned keys, with separate Vault namespaces selected by ownership rather than separate default policies. Configuration must provide tenant namespace support and an OSAC-managed, key-scoped consumer credential path so revocation cannot be bypassed by a broad Transit grant. Tenants cannot submit backend or policy fields. `[User: one shared configurable policy]`

## IC-7: Key-management authorization surface

**Requirements:** FR-6

Adds exact OPA method permissions for ManagedKeys. Tenant Admins receive tenant-owned key lifecycle methods and existing administrators retain broad access. Ordinary client tokens receive no ManagedKeys methods, regardless of whether their holder is a Tenant User or Cloud Infrastructure Admin; infrastructure administrators must not be assigned `tenant-admin` or an existing unrestricted administrator identity. `[Codebase: fulfillment-service/internal/auth/policies/authz.rego]` `[Locked: D17, D18]`

# 6. Alternatives Considered

### Expose Transit directly through the public API

This minimizes translation and implementation code, but leaks numeric material versions, ACL-based revocation details, mount paths, and vendor errors into a contract that cannot represent KMIP replacement objects cleanly. It is rejected because OSAC-2389 requires backend capability variance. `[Research: Lifecycle capability comparison]`

### Require a native whole-key disable primitive for the first provider

This would simplify revocation, but would exclude HashiCorp Vault Transit despite the PRD's Vault dependency. OSAC instead uses key-specific Vault ACL policies: every normal consumer token is subject to the policy, and terminal revocation requires verified denial of both encryption and decryption. A database-only flag is insufficient because it cannot stop direct Vault use. The extra credential and policy lifecycle is accepted to make Vault the first supported provider. `[User]` `[Locked: D8]`

### Use separate lifecycle RPCs

Separate Rotate, Revoke, and Recover RPCs would make each action explicit but expand the public service beyond standard CRUD. `ManagedKey` instead uses trigger fields in the standard `Update` request. Changing one trigger requests one lifecycle effect. The server performs that effect synchronously and commits the new trigger value and confirmed key state before returning success; no desired-versus-observed state machine or controller is introduced. An uncertain timeout still has at-least-once rotation risk, so callers must verify provider state before retrying. `Delete` remains the standard method because it removes the resource. `[Review: PR 319, jhernand]` `[Codebase: fulfillment-service/docs/API.md]`

# 7. Observability and Monitoring

Fulfillment already exports `inbound_unary_request_count{service,method,code}` and the `inbound_unary_request_duration` histogram (`_bucket`, `_sum`, and `_count` series with the same labels) on the gRPC server's Prometheus endpoint. Once ManagedKeys is registered, those metrics cover its lifecycle RPC volume, response codes, and request latency without KMS-specific duplicates. Startup KMS configuration validation and failed key operations emit redacted diagnostics. Request metrics reveal backend failures only when a key operation occurs; this release does not add periodic detection of an idle backend outage. `[Codebase: fulfillment-service/internal/metrics/grpc_metrics_interceptor.go]` `[Codebase: fulfillment-service/internal/vault/vault_health.go]` `[User: skip KMS readiness gauge]`

Deployments grant Cloud Infrastructure Admins read-only platform monitoring access to existing request metrics and redacted operational logs. This feature adds no KMS-specific alert, dashboard, or Fulfillment API monitoring role. `[User: keep observability focused on existing signals]`

Structured logs include key ID, backend name, operation, normalized reason, and duration; they exclude tenant-provided descriptions, consumer identities, tokens, provider response bodies, ciphertext, and key material.

# 8. Impact and Compatibility

The API, key-resource migration, CLI key commands, and Helm values are additive. Existing resources and clients remain compatible. `proto/private` is the source; `proto/public` and `proto/gen` are regenerated once and all consumers rebuild against the shared module. This feature defines the reference contract but adds no key-consuming resource or durable destruction fence; [OSAC-2389](https://redhat.atlassian.net/browse/OSAC-2389) adds the first reverse-reference guard and resolves the fence needed when Vault deletion succeeds but PostgreSQL does not commit. `[User]`

The provisional support target is HashiCorp Vault Enterprise 2.1.x with tenant namespaces and Transit, starting certification with 2.1.1. The exact certified patch version and any additional customer-required versions will be set by a downstream certification process using a real Vault deployment. The target can be adjusted as customer needs and expectations become clear. The bundled OpenBao backend remains an alternative for dev and CI purposes. Startup validation prevents new key creation if required namespace, mount, credential, or policy checks fail. Existing Vault KV secret behavior is unchanged. `[User]` [Vault 2.x release notes](https://developer.hashicorp.com/vault/docs/updates/release-notes) [Vault namespaces](https://developer.hashicorp.com/vault/docs/enterprise/namespaces)

There will be no Downgrade support

---

## Provenance

Authored: draft @ design 0.11.3 - 9b25062, workspace bugfix/osac-5191/mutable-userdata @ 829cb62b7
Final: revise @ design 0.11.3 - 2bd6607, workspace osac-2990/secret-docs @ 8284038be

> Context changed between draft and revise.

<!-- ai-workflow-provenance:{"schema_version":1,"provenance_kind":"session","workflow":"design","workflow_version":"0.11.3","ai_workflows":"2bd6607","source_repo":"8284038be","source_repo_branch":"osac-2990/secret-docs","commits_behind_main":0,"commits_ahead_main":3,"main_ref":"main","phases":["draft","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","revise","manual-edit","revise","revise","revise","revise","revise","revise","revise"],"authoring_modes":["manual","skill"],"context_changed":true,"origin_untracked":false} -->
