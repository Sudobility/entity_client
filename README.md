# @sudobility/entity_client

React client library for entity/organization management with TanStack Query v5 hooks for CRUD operations on entities, members, and invitations.

## Installation

This package is published to **GitHub Packages** under the `@sudobility` scope.
Consumers must point the scope at that registry (e.g. in a project `.npmrc`):

```
@sudobility:registry=https://npm.pkg.github.com
```

Then install:

```bash
bun add @sudobility/entity_client
```

### Peer dependencies

`react` (18 or 19), `@tanstack/react-query` (v5), and `@sudobility/types` must be
installed by the host app.

## Usage

```typescript
import {
  EntityClient,
  useEntities,
  useCreateEntity,
  useEntityMembers,
  useMyInvitations,
  useAcceptInvitation,
  CurrentEntityProvider,
  useCurrentEntity,
} from '@sudobility/entity_client';

const client = new EntityClient({
  baseUrl: 'https://api.example.com/api/v1',
  networkClient: myNetworkClient,
});

// In components
const { data: entities } = useEntities(client);
const createEntity = useCreateEntity(client);
```

## API

### Client

| Export | Description |
|--------|-------------|
| `EntityClient` | HTTP client wrapping authenticated fetch calls to the entity API |
| `EntityClientConfig` | Config: `{ baseUrl, networkClient }` |

### Entity Hooks

| Hook | Description |
|------|-------------|
| `useEntities(client)` | List all entities for current user |
| `useEntity(client, slug)` | Get single entity by slug |
| `useCreateEntity(client)` | Create organization (invalidates list) |
| `useUpdateEntity(client)` | Update entity (invalidates detail + list) |
| `useDeleteEntity(client)` | Delete entity (invalidates list) |

### Member Hooks

| Hook | Description |
|------|-------------|
| `useEntityMembers(client, slug)` | List members of an entity |
| `useUpdateMemberRole(client)` | Update member role |
| `useRemoveMember(client)` | Remove member from entity |

### Invitation Hooks

| Hook | Description |
|------|-------------|
| `useMyInvitations(client)` | List pending invitations for current user |
| `useEntityInvitations(client, slug)` | List invitations for an entity |
| `useCreateInvitation(client)` | Create invitation |
| `useCancelInvitation(client)` | Cancel invitation |
| `useRenewInvitation(client)` | Renew an expired invitation |
| `useAcceptInvitation(client)` | Accept invitation by token |
| `useDeclineInvitation(client)` | Decline invitation by token |

### API Key Hooks

| Hook | Description |
|------|-------------|
| `useEntityApiKeys(client, slug)` | List an entity's API keys (secrets never returned) |
| `useCreateApiKey(client)` | Create an API key (plaintext key returned once) |
| `useUpdateApiKey(client)` | Rename a key or toggle active state |
| `useRevokeApiKey(client)` | Permanently revoke a key |

### Context

| Export | Description |
|--------|-------------|
| `CurrentEntityProvider` | Manages workspace/entity selection with per-user localStorage persistence (renders its own `EntityUserProvider`) |
| `useCurrentEntity()` | Access current entity context (throws if outside provider) |
| `useCurrentEntityOptional()` | Access current entity context (returns null if outside) |
| `EntityUserProvider` | Names the signed-in user id for the query hooks; only needed if you use the hooks without `CurrentEntityProvider` |
| `useEntityUserId(explicit?)` | Resolve the user id the query hooks cache under |

Every query key is scoped to the signed-in user id, so one account's cached data
can never be served to another. Hooks read that id from the nearest
`EntityUserProvider`; without one they cache under a shared `null` user.

### Error Types

`EntityErrorCode` (enum) and `EntityApiError` describe typed API error responses.

### Query Key Factories

`entityKeys`, `memberKeys`, `invitationKeys`, `apiKeyKeys` -- for cache management.
Every factory function takes the user id as its first argument.

## Development

```bash
bun run build          # Compile TypeScript to dist/
bun run build:watch    # Watch mode
bun run typecheck      # Type-check (no emit)
bun run lint           # ESLint
bun run lint:fix       # ESLint with auto-fix
bun run test:unit      # Run tests once (vitest)
bun run test:watch     # Run tests in watch mode
bun run format         # Prettier format
```

## License

BUSL-1.1
