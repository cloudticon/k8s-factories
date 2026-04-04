# k8s-factories

`k8s-factories` is a small helper layer for creating single-container Kubernetes workloads with less boilerplate and a cleaner separation between workload creation and traffic exposure.

It provides two main helpers:

- `webApp()` for `Deployment` + `Service` (+ optional `HPA`, PVC creation, and inline expose)
- `expose()` for routing (`VirtualService`/`Ingress`) with optional TLS artifacts

## Import

```typescript
import { webApp, expose, env, vol } from "github.com/cloudticon/k8s@master";
```

All public exports are re-exported from `index.ct`.

## Quick start

```typescript
import { webApp } from "github.com/cloudticon/k8s@master";

webApp({
  name: "node",
  image: "node:20-alpine",
  port: 3000,
  expose: {
    type: "istio",
    host: Values.host,
  },
});
```

This creates a `Deployment`, a `Service`, and an Istio `VirtualService` (using default shared gateway unless `tls` is configured).

## API Reference: `webApp(config)`

Creates:

- `Deployment`
- `Service`
- optional `HorizontalPodAutoscaler` when `hpa` is set
- optional `PersistentVolumeClaim` resources when `vol.pvc()` is used
- optional routing by calling `expose()` internally when `expose` is set

Returns:

```typescript
type WebAppResult = {
  deploy: AnyRecord;
  svc: AnyRecord;
};
```

### `WebAppConfig`

| Field | Type | Default | Description |
|---|---|---|---|
| `name` | `string` | required | Base resource name (`Deployment`, `Service`, and related objects). |
| `image` | `string` | required | Container image. |
| `port` | `number` | required | Container and service port. |
| `replicas` | `number` | `1` | Initial deployment replica count. |
| `tty` | `boolean` | `false` | Enables TTY on container. |
| `command` | `string[]` | none | Container entrypoint override. |
| `args` | `string[]` | none | Container args override. |
| `env` | `Record<string, Primitive \| EnvSource>` | none | Environment variables (plain values or `env.*` references). |
| `volumes` | `Record<string, string \| VolumeSource>` | none | Volume map where key is mount path. |
| `resources` | `Resources` | none | Container CPU/memory requests and limits. |
| `probes` | `string \| ProbesConfig` | none | Liveness/readiness/startup probe configuration. |
| `hpa` | `HPAConfig` | none | HPA min/max and optional CPU/memory targets. |
| `labels` | `Record<string, string>` | `{}` | Extra labels for workload/service metadata. |
| `annotations` | `Record<string, string>` | `{}` | Deployment metadata/template annotations. |
| `imagePullSecrets` | `string[]` | none | Pull secret names for pod spec. |
| `serviceType` | `"ClusterIP" \| "NodePort" \| "LoadBalancer"` | cluster default (`ClusterIP`) | Service type override. |
| `expose` | `InlineExposeConfig` | none | Inline shortcut for single-route expose. |
| `deploymentOverrides` | `Record<string, unknown>` | `{}` | Raw override merged into deployment payload. |
| `serviceOverrides` | `Record<string, unknown>` | `{}` | Raw override merged into service payload. |

Notes:

- `app: <name>` label is always set and merged with custom labels.
- If `hpa` is provided, `replicas` is the initial value and HPA controls scaling afterward.
- Inline `expose` maps one route with `routePrefix` (default `/`) to the created service.

## API Reference: `expose(config)`

`expose()` is a standalone routing helper. It accepts destinations from:

- `webApp()` return value (`WebAppResult`)
- direct `service()` return value (`ServiceResult`)

### Core routing types

```typescript
type Route = {
  prefix: string;
  destination: WebAppResult | ServiceResult;
};
```

`Route.destination` is auto-resolved to service host/port.

### TLS type (Istio)

```typescript
type TLSConfig = {
  issuer: string;
  secretName?: string;      // default: `${name}-tls`
  certificateName?: string; // default: `${name}-cert`
};
```

When `tls` is set for Istio exposure, helper creates:

- `Certificate` in namespace `istio-system`
- dedicated `Gateway` with HTTP redirect and HTTPS server
- `VirtualService` bound to that gateway

### `IstioExposeConfig`

| Field | Type | Default | Description |
|---|---|---|---|
| `name` | `string` | required | Resource name for routing artifacts. |
| `type` | `"istio"` | required | Selects Istio routing mode. |
| `host` | `string` | required | External host. |
| `gateway` | `string` | `"default/default"` | Shared gateway reference when `tls` is not set. |
| `tls` | `TLSConfig` | none | Enables cert + dedicated gateway generation. |
| `routes` | `Route[]` | required | Prefix-based route list. |

### `IngressExposeConfig`

| Field | Type | Default | Description |
|---|---|---|---|
| `name` | `string` | required | Ingress name. |
| `type` | `"ingress"` | required | Selects Kubernetes Ingress mode. |
| `host` | `string` | required | Host rule. |
| `ingressClassName` | `string` | none | Optional ingress class. |
| `tls` | `{ secretName: string }` | none | TLS block for ingress. |
| `annotations` | `Record<string, string>` | none | Ingress annotations. |
| `routes` | `Route[]` | required | Prefix-based route list. |

## `env.*` helpers

Helpers for environment variable values that come from Kubernetes objects.

### `env.secret(secretName, key)`

Returns `EnvSource` pointing to `valueFrom.secretKeyRef`.

### `env.configMap(configMapName, key)`

Returns `EnvSource` pointing to `valueFrom.configMapKeyRef`.

### Example

```typescript
webApp({
  name: "api",
  image: "api:latest",
  port: 8080,
  env: {
    NODE_ENV: "production",
    DB_PASSWORD: env.secret("db-credentials", "password"),
    APP_CONFIG: env.configMap("app-config", "json"),
  },
});
```

## `vol.*` helpers

Volume helpers use mount-path keyed object API:

```typescript
volumes: {
  "/data": "existing-pvc",            // string = existing PVC
  "/storage": vol.pvc({ size: "10Gi" }),
  "/cache": vol.emptyDir(),
  "/config": vol.configMap("app-config"),
  "/secrets": vol.secret("app-secrets"),
}
```

### `PVCOpts`

```typescript
type PVCOpts = {
  size: string;
  storageClass?: string;
  accessMode?: string; // default: "ReadWriteOnce"
};
```

### `vol.pvc(opts)`

- declares a mount backed by a new generated PVC
- generated PVC name format: `${appName}-${sanitizedMountPath}`

### `vol.emptyDir(opts?)`

- declares `emptyDir` volume
- `opts.sizeLimit` is optional

### `vol.configMap(name)`

- declares volume from config map

### `vol.secret(secretName)`

- declares volume from secret

## Probes

`probes` supports three usage modes.

### 1) String shorthand

```typescript
probes: "/health"
```

Creates both liveness and readiness probes using the app port.

### 2) Config with shared path

```typescript
probes: { path: "/health" }
```

Also creates liveness + readiness with defaults.

### 3) Full control

```typescript
probes: {
  liveness: { path: "/healthz", initialDelaySeconds: 30, periodSeconds: 10 },
  readiness: { path: "/ready", periodSeconds: 5 },
  startup: { path: "/started", failureThreshold: 30, periodSeconds: 2 },
}
```

Types:

```typescript
type ProbeDetail = {
  path?: string;
  port?: number;
  initialDelaySeconds?: number;
  periodSeconds?: number;
  timeoutSeconds?: number;
  failureThreshold?: number;
};

type ProbesConfig = {
  path?: string;
  port?: number;
  liveness?: ProbeDetail | false;
  readiness?: ProbeDetail | false;
  startup?: ProbeDetail | false;
};
```

## HPA

```typescript
type HPAConfig = {
  min: number;
  max: number;
  cpu?: number;
  memory?: number;
};
```

Usage:

```typescript
hpa: { min: 2, max: 10, cpu: 80 }
hpa: { min: 1, max: 5, cpu: 70, memory: 80 }
```

This generates metrics for CPU and/or memory utilization target.

## Labels and annotations

- `labels` are applied to deployment template metadata and service metadata.
- `annotations` are applied to deployment metadata and pod template metadata.
- `app: <name>` is always present.

## End-to-end examples

### 1) Simple Istio service

```typescript
import { webApp } from "github.com/cloudticon/k8s@master";

webApp({
  name: "node",
  image: "node:20-alpine",
  port: 3000,
  expose: {
    type: "istio",
    host: "app.example.com",
  },
});
```

### 2) Multi-service routing with TLS

```typescript
import { webApp, expose } from "github.com/cloudticon/k8s@master";

const api = webApp({ name: "api", image: "api:latest", port: 8080 });
const frontend = webApp({ name: "frontend", image: "frontend:latest", port: 3000 });
const admin = webApp({ name: "admin", image: "admin:latest", port: 3000 });

expose({
  name: "my-app",
  type: "istio",
  host: "app.example.com",
  tls: { issuer: "letsencrypt-prod" },
  routes: [
    { prefix: "/api", destination: api },
    { prefix: "/admin", destination: admin },
    { prefix: "/", destination: frontend },
  ],
});
```

### 3) Service with volumes, env secrets, probes, and HPA

```typescript
import { env, vol, webApp } from "github.com/cloudticon/k8s@master";

webApp({
  name: "worker",
  image: "worker:latest",
  port: 8080,
  env: {
    NODE_ENV: "production",
    DB_PASSWORD: env.secret("db-credentials", "password"),
  },
  volumes: {
    "/data": "existing-pvc",
    "/storage": vol.pvc({ size: "10Gi", storageClass: "ssd" }),
    "/cache": vol.emptyDir(),
    "/secrets": vol.secret("app-secrets"),
  },
  probes: {
    readiness: { path: "/ready", periodSeconds: 5 },
    liveness: { path: "/healthz", initialDelaySeconds: 20 },
  },
  hpa: { min: 2, max: 8, cpu: 75, memory: 80 },
});
```

### 4) Ingress exposure instead of Istio

```typescript
import { expose, webApp } from "github.com/cloudticon/k8s@master";

const app = webApp({
  name: "node",
  image: "node:20-alpine",
  port: 3000,
});

expose({
  name: "node",
  type: "ingress",
  host: "app.example.com",
  ingressClassName: "nginx",
  routes: [{ prefix: "/", destination: app }],
});
```

### 5) Internal service without expose

```typescript
import { webApp } from "github.com/cloudticon/k8s@master";

webApp({
  name: "redis",
  image: "redis:7-alpine",
  port: 6379,
});
```

## File layout (`.ct`)

- `helpers.ct` - public exports
- `webapp.ct` - workload helper
- `expose.ct` - routing helper
- `env.ct` - env helper namespace
- `volumes.ct` - volume helper namespace + conversion
- `probes.ct` - probe conversion logic
- `hpa.ct` - HPA metrics helper
- `types.ct` - shared types
