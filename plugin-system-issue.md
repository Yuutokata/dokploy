# Title

feat: plugin/addon system (manifest, event webhooks, sandboxed dashboard UI)

# Labels

enhancement (set automatically by the Feature Request template)

---

## What problem will this feature address?

Dokploy can only be extended by forking it. Today users can use templates to deploy a stack, a `custom` webhook notification, API keys, and the raw Traefik file editor. None of these let a third party add a dashboard page, react to deployment events, or ship a sidecar service that is managed together with its UI. Typical needs:

- A per-application "Metrics" or "Security scan" tab, or an extra sidebar dashboard.
- Reacting to deployment lifecycle events (scan an image, post to an internal tool).
- Shipping a helper service (Grafana, scanner) that is installed, upgraded and removed as one unit.

Changes like these currently have to land in core or live in a fork.

## Describe the solution you'd like

A deliberately small, **out-of-process** plugin system. Plugins never run inside the Dokploy server process, because that would expose the DB, SSH keys and `docker.sock` (effectively root on every server).

### 1. Manifest (`plugin.json`, schema-versioned, validated with zod)

```json
{
  "schemaVersion": 1,
  "id": "deploy-insights",
  "name": "Deploy Insights",
  "version": "1.0.0",
  "dokploy": { "minVersion": "0.x.y" },
  "service": { "compose": "docker-compose.yml", "port": 3000, "env": { "API_URL": "${dokploy.apiUrl}" } },
  "ui": [
    { "slot": "sidebar", "title": "Insights", "icon": "BarChart", "path": "/" },
    { "slot": "application-tab", "title": "Insights", "path": "/app/:applicationId" }
  ],
  "events": ["deployment.started", "deployment.succeeded", "deployment.failed"],
  "webhook": { "path": "/hooks/dokploy" },
  "permissions": ["application:read", "deployment:read"]
}
```

Both `service` and `ui` are optional (UI-only plugins can point at an external HTTPS URL; service-only plugins just consume events).

### 2. Registry

A Git-hosted index (`index.json` + one folder per plugin), the same model as Dokploy/templates. A custom registry URL is supported (HTTPS only, validated; the template `baseUrl` is currently unvalidated and is an SSRF surface). Whether this lives in a new `Dokploy/plugins` repo or in the templates repo is up to the maintainers.

### 3. Install and lifecycle (reuses the compose/template deploy flow)

- Settings → Plugins lists the registry, installed plugins, status and config.
- Install: validate the manifest, show the requested permissions for confirmation, deploy `service` as a compose service on a chosen `serverId` (main host or remote server, Swarm/overlay network `dokploy-network`), then create a scoped API key and a webhook secret.
- Enable, disable, update (version compare against `minVersion`) and uninstall (removes the service, API key and rows).
- Scoped per organization (`organizationId`).

### 4. Server hooks

A small internal event emitter called from existing lifecycle points (deployment service, application/compose created, backup finished). Delivery is signed HTTP POSTs (HMAC-SHA256 header, timestamp for replay protection, retries with backoff), modelled on the `custom` notification type. Events carry IDs, not secrets.

### 5. Permissions

A new `plugin` resource in `packages/server/src/lib/access-control.ts` (manage/read). A plugin acts through an API key limited to the permissions its manifest declares and the admin approved. It is never more privileged than that.

### 6. Dashboard UI

- The sidebar (`components/layouts/side.tsx`, currently a static `MENU`) gets a server-provided list of plugin entries.
- A generic host page `/dashboard/plugins/[pluginId]/[...slug]` and per-service tab slots render the plugin UI in `<iframe sandbox="allow-scripts allow-forms">`, with no `allow-same-origin`.
- A tiny `postMessage` protocol (context: theme, project/service IDs; a short-lived, scoped token) is published as `@dokploy/plugin-sdk`. Dokploy cookies are never exposed to plugin code.

### Delivery plan

Each step is a separate small PR and is independently useful:

1. Manifest schema + `plugin` DB table/migration + registry fetch + install/uninstall of the optional service + Settings → Plugins page.
2. Event emitter + signed webhook delivery + API-key scoping and the `plugin` permission.
3. UI slots (sidebar, application tab) + sandboxed host page + `postMessage` SDK.
4. Docs on docs.dokploy.com, the plugin authoring guide, and a reference plugin.

### Tests

vitest in `apps/dokploy/__test__` covering manifest validation, permission scoping, webhook signing/replay window, and registry URL validation. Plus manual verification of install, uninstall, an event round-trip and the iframe sandbox on a local dev setup, as CONTRIBUTING.md requires.

### Reference plugin

`deploy-insights`, a small service that stores deployment events and shows a sidebar page of recent deploys. It exercises the service, events, permissions and UI paths end to end, and doubles as the test fixture and docs example.

### Open questions for maintainers

1. Is the direction (out-of-process, manifest + events + sandboxed iframe) acceptable, or would you rather start with events/webhooks only?
2. Registry location: new `Dokploy/plugins` repo or a section in the templates repo?
3. Should this be core, or gated like the features covered by `LICENSE_PROPRIETARY.md`?
4. Cloud (`IS_CLOUD`): should plugin services be allowed there, or only UI-only/webhook plugins?
5. Any requirement on signing or verifying registry entries (e.g. checksum or publisher allowlist)?

## Describe alternatives you've considered

- **In-process plugins (Node `require`):** simplest API, but plugins would get DB, SSH and `docker.sock` access; not acceptable security-wise.
- **Templates only:** already deploys a stack but cannot add UI, receive events or declare permissions.
- **`custom` notification webhook + API key + Traefik file editor:** works for power users, but there is no packaging, install/upgrade flow, UI integration or scoping.
- **Forking:** what users do today; it diverges from upstream quickly.

## Additional context

- No code in the repo implements this yet. Relevant existing pieces: `packages/server/src/templates/github.ts`, `apps/dokploy/server/api/routers/compose.ts` (`deployTemplate`), `packages/server/src/db/schema/notification.ts` (`custom`), `packages/server/src/lib/access-control.ts`, `packages/server/src/lib/auth.ts` (API keys), `apps/dokploy/components/layouts/side.tsx`.
- Multi-server: each plugin install records a `serverId`; `remoteServersOnly` and `IS_CLOUD` are honored.
- Happy to adjust or shrink scope; I would send step 1 as a first PR once the direction is agreed.

## Will you send a PR to implement it?

Yes
