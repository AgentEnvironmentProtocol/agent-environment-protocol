# 09 — Configuration & Passthrough Mount Manifests

This document formalizes the declarative **AEP Configuration File (`aep.config.json`)** for configuring AEP servers, client runtimes, transport listeners, security policies, reusable downstream connections, and **passthrough mounts**.

Related documents: [00 — Overview](./00-overview.md) | [02 — Tree Nodes & Data Model](./02-tree-nodes.md) | [03 — Passthrough & Composition](./03-passthrough-and-composition.md) | [05 — Transport & Interoperability](./05-transport-and-interop.md) | [06 — Profiles, Sessions & Vocabularies](./06-profiles-sessions-and-vocabularies.md) | [08 — Security, Authentication & Privilege Elevation](./08-security-and-elevation.md)

---

## 1. Scope, File Formats & Discovery Order

### 1.1 Canonical JSON & Lossless Format Equivalence

- The canonical configuration format is **JSON** (`aep.config.json`, UTF-8, validated against the Draft 2020-12 JSON Schema in Section 4).
- Compliant tools and runtimes MAY additionally load **JSONC** (JSON with comments/trailing commas), **YAML** (`aep.config.yaml` / `aep.config.yml`), or **TOML** (`aep.config.toml`) provided they deserialize losslessly into the exact same canonical object model defined by the JSON Schema.

### 1.2 Deterministic Discovery Precedence

When no explicit configuration path is supplied, AEP CLI and SDK loaders MUST resolve configuration files in the following precedence order (first match wins):

1. Explicit CLI flag (`--config <path>`) or environment variable (`AEP_CONFIG`).
2. Current working directory / project workspace (`./aep.config.json`, `./aep.config.yaml`, `./aep.config.toml`).
3. User configuration directory (`~/.config/aep/aep.config.json` on POSIX; `%APPDATA%\aep\aep.config.json` on Windows).
4. System-wide configuration directory (`/etc/aep/aep.config.json` on POSIX; `%ProgramData%\aep\aep.config.json` on Windows).

---

## 2. Secret Interpolation & Redaction

Configuration files MUST NOT force operators to hardcode plaintext secrets:

- **Interpolation Syntax**: String values inside `auth`, `tls`, `headers`, `env`, or `authPropagation` fields MAY use:
  - `${ENV:VAR_NAME}` — resolves from the process environment variable `VAR_NAME`.
  - `${FILE:/path/to/secret}` — resolves by reading UTF-8 file contents with trailing `\r` / `\n` trimmed.
- **Strict Failure on Missing Secrets**: If a referenced environment variable is unset or a referenced secret file cannot be read, startup or connection initialization MUST fail immediately with a descriptive configuration error before binding listeners or opening mounts.
- **Mandatory Redaction**: Resolved secrets (tokens, API keys, private key passphrases, TLS keys) MUST NEVER be echoed into `Head`, `node.properties`, `node.state`, `node.meta`, diagnostic logs, or `ErrorResponse.data` payloads.

---

## 3. Normative Configuration Sections

### 3.1 Top-Level Root (`aep.config.json`)

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `schemaVersion` | `"1.0"` | Yes | Configuration schema version. |
| `metadata` | `{ name?, environment?, description? }` | No | Human-readable deployment descriptor. |
| `server` | `ServerConfig` | No | Present when configuring an AEP server or passthrough aggregator. |
| `connections` | `Record<string, ConnectionConfig>` | No | Named reusable outbound connections to upstream or downstream AEP providers. |
| `mounts` | `MountConfig[]` | No | Ordered list of passthrough or client-composite subtree mounts referencing `connections`. |
| `client` | `ClientConfig` | No | Present when configuring an AEP client or agent runtime. |

### 3.2 `server` Section

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `info` | `{ name, version, description? }` | Yes | Advertised in `Head["aep.serverInfo"]`. |
| `profile` | `"aep-core-static" \| "aep-core-streaming" \| string` | No | Default conformance profile. |
| `capabilities` | `CapabilitiesObject` | No | Server capability limits and supported rendering/channel modes ([06 — Profiles, Sessions & Vocabularies](./06-profiles-sessions-and-vocabularies.md)). |
| `listeners` | `ListenerConfig[]` | Yes | Transport endpoints exposed by this server (`quic`, `wss`, `ws`, `unix`, `named-pipe`, `stdio`). |
| `security` | `ServerSecurityConfig` | No | Allowed `authModes`, TLS server cert/key/trustRoots, JWT/OIDC verification, elevation policy, and `allowInsecureTransport`. |

### 3.3 `connections` Section

Each entry under `connections.<id>` defines a reusable outbound transport target:

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `transport` | `"quic" \| "wss" \| "ws" \| "unix" \| "named-pipe" \| "stdio" \| "memory"` | Yes | Transport binding used to reach the provider. |
| `uri` | `string` | Conditional | Required for single-endpoint `quic`, `wss`, `ws` (e.g., `"quic://robot-1.local:7443"`, `"wss://api.internal/aep"`). |
| `uris` | `string[]` | Conditional | Optional list of backend endpoint URIs for load-balanced / redundant `quic` or `wss` pools (alternative to `uri`). |
| `loadBalancing` | `{ strategy: "failover" \| "round-robin" \| "least-loaded" \| "consistent-hash", healthCheckIntervalMs?: number }` | No | Load-balancing and failover policy when `uris` lists multiple backend endpoints. |
| `path` | `string` | Conditional | Required for `unix` (`/run/aep/term.sock`) or `named-pipe` (`\\\\.\\pipe\\aep-term`). |
| `spawn` | `{ command: string, args?: string[], cwd?: string, env?: Record<string, string> }` | Conditional | Required for spawned `stdio` provider subprocesses. |
| `tls` | `{ caFile?, certFile?, keyFile?, serverName?, pinSha256?: string[], insecureSkipVerify?: boolean }` | No | TLS 1.3 / mTLS / certificate pinning configuration. |
| `auth` | `{ mode: AuthMode, token?, apiKey?, keyId?, privateKeyFile? }` | No | Outbound application-layer authentication configuration. |
| `reconnect` | `{ enabled?: boolean, initialDelayMs?: number, maxDelayMs?: number, multiplier?: number, maxAttempts?: number }` | No | Exponential backoff reconnect policy for resilient long-lived mounts. |
| `allowInsecureTransport` | `boolean` | No | Defaults to `false`. Must be `true` to permit non-loopback `ws://`. |

### 3.4 `mounts` Section

Each entry in `mounts[]` binds a `connection` onto an `anchorPath` in the local server or client composite `Body` tree:

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `id` | `Segment` | Yes | Unique mount identifier used for diagnostics, `refId` isolation namespaces, and `meta["aep.mount"]`. |
| `anchorPath` | `Path` | Yes | Target parent `Path` in the host tree (e.g., `"/robot"`, `"/workstation/ide/layout/term"`, `"/services/billing"`). |
| `connection` | `string` | Yes | Key referencing an entry in `connections`. |
| `remotePath` | `Path` | No | Subtree root on the downstream provider to mount (defaults to `"/"`, mounting the provider's `Body`). |
| `role` | `"controller" \| "observer"` | No | Session role requested from the downstream provider (defaults to `"controller"` for server mounts; clamped by upstream session role). |
| `lazy` | `boolean` | No | When `true`, exposes `hasChildren = true` at `anchorPath` and defers downstream `tree.get` until `node.loadChildren`. |
| `required` | `boolean` | No | When `true`, host startup fails if initial mount handshake fails; when `false` (default), unavailable mounts isolate cleanly with `MOUNT_UNAVAILABLE`. |
| `timeoutMs` | `PositiveInteger` | No | Per-call deadline for forwarded requests to this mount (`MOUNT_TIMEOUT`). |
| `authPropagation` | `{ strategy: "none" \| "static" \| "token-exchange" \| "passthrough", audience?: string, scopes?: string[], tokenEndpoint?: string }` | No | Credential propagation policy across the mount boundary ([08 — Security](./08-security-and-elevation.md#6-passthrough-security-credential-propagation--mount-isolation)). Defaults to `"none"`. |
| `allowElevation` | `boolean` | No | Defaults to `false`. Controls whether step-up elevation challenges may cross this mount boundary. |
| `stripMetaPrefixes` | `string[]` | No | List of internal `meta` key prefixes stripped before projecting downstream nodes upstream. |

**Startup & Attach Validation**: Hosts MUST validate every mount against the grammar containment rules ([03 — Passthrough & Composition](./03-passthrough-and-composition.md#3-grammar-containment-at-mount-points)) and cycle-detection rules (`MOUNT_CYCLE`). Mounts violating containment MUST fail validation with `INVALID_MESSAGE`.

### 3.5 `client` Section

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `info` | `{ name, version }` | No | Advertised in `session.initialize` `clientInfo`. |
| `defaultRole` | `"controller" \| "observer"` | No | Preferred session role (defaults to `"controller"`). |
| `allowDowngradeToObserver` | `boolean` | No | Whether client accepts downgrade when a `single-controller` session is occupied. |
| `rendering` | `{ preferredMode?: "renderMinimal" \| "renderExpress" \| "renderFull", minimalSynthesizer?: "client" \| "server" \| "auto", format?: "image/svg+xml" \| "image/png" \| "text/x-aep-layout", annotatePaths?: boolean }` | No | Client rendering preferences ([02 — Tree Nodes & Data Model](./02-tree-nodes.md#6-multi-tier-surface-rendering-renderminimal-renderexpress-renderfull)). |

---

## 4. Normative JSON Schema (Draft 2020-12)

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://aep.dev/schemas/aep.config.v1.0.json",
  "title": "Agent Environment Protocol (AEP) Configuration",
  "type": "object",
  "required": ["schemaVersion"],
  "additionalProperties": false,
  "properties": {
    "schemaVersion": {
      "type": "string",
      "const": "1.0"
    },
    "metadata": {
      "type": "object",
      "additionalProperties": false,
      "properties": {
        "name": { "type": "string" },
        "environment": { "type": "string" },
        "description": { "type": "string" }
      }
    },
    "server": {
      "type": "object",
      "required": ["info", "listeners"],
      "additionalProperties": false,
      "properties": {
        "info": {
          "type": "object",
          "required": ["name", "version"],
          "additionalProperties": false,
          "properties": {
            "name": { "type": "string" },
            "version": { "type": "string" },
            "description": { "type": "string" }
          }
        },
        "profile": {
          "type": "string"
        },
        "capabilities": {
          "type": "object"
        },
        "listeners": {
          "type": "array",
          "minItems": 1,
          "items": {
            "type": "object",
            "required": ["transport"],
            "additionalProperties": false,
            "properties": {
              "id": { "type": "string" },
              "transport": {
                "type": "string",
                "enum": ["quic", "wss", "ws", "unix", "named-pipe", "stdio"]
              },
              "address": { "type": "string" },
              "path": { "type": "string" },
              "alpn": {
                "type": "array",
                "items": { "type": "string" }
              },
              "allowInsecureTransport": {
                "type": "boolean",
                "default": false
              },
              "tls": { "$ref": "#/$defs/tlsServerConfig" }
            }
          }
        },
        "security": {
          "type": "object",
          "additionalProperties": false,
          "properties": {
            "authModes": {
              "type": "array",
              "items": { "$ref": "#/$defs/authMode" }
            },
            "allowUnauthenticatedLoopback": {
              "type": "boolean",
              "default": false
            },
            "jwt": {
              "type": "object",
              "additionalProperties": false,
              "properties": {
                "issuer": { "type": "string" },
                "audience": { "type": "string" },
                "jwksUri": { "type": "string" },
                "publicKeyFile": { "type": "string" }
              }
            },
            "elevation": {
              "type": "object",
              "additionalProperties": false,
              "properties": {
                "enabled": { "type": "boolean", "default": true },
                "maxDurationMs": { "type": "integer", "minimum": 1000 },
                "defaultAllowedFactors": {
                  "type": "array",
                  "items": {
                    "type": "string",
                    "enum": ["human-approval", "totp", "passkey", "token", "policy-attestation"]
                  }
                },
                "auditLogPath": { "type": "string" }
              }
            }
          }
        }
      }
    },
    "connections": {
      "type": "object",
      "additionalProperties": { "$ref": "#/$defs/connectionConfig" }
    },
    "mounts": {
      "type": "array",
      "items": { "$ref": "#/$defs/mountConfig" }
    },
    "client": {
      "type": "object",
      "additionalProperties": false,
      "properties": {
        "info": {
          "type": "object",
          "required": ["name", "version"],
          "additionalProperties": false,
          "properties": {
            "name": { "type": "string" },
            "version": { "type": "string" }
          }
        },
        "defaultRole": {
          "type": "string",
          "enum": ["controller", "observer"],
          "default": "controller"
        },
        "allowDowngradeToObserver": {
          "type": "boolean",
          "default": true
        },
        "rendering": {
          "type": "object",
          "additionalProperties": false,
          "properties": {
            "preferredMode": {
              "type": "string",
              "enum": ["renderMinimal", "renderExpress", "renderFull"],
              "default": "renderMinimal"
            },
            "minimalSynthesizer": {
              "type": "string",
              "enum": ["client", "server", "auto"],
              "default": "auto"
            },
            "format": {
              "type": "string",
              "enum": ["image/svg+xml", "image/png", "text/x-aep-layout"]
            },
            "annotatePaths": {
              "type": "boolean",
              "default": true
            }
          }
        }
      }
    }
  },
  "$defs": {
    "authMode": {
      "type": "string",
      "enum": ["unauthenticated", "mtls", "os-peer", "bearer", "api-key", "challenge-response"]
    },
    "path": {
      "type": "string",
      "pattern": "^(/|(/[A-Za-z0-9._-]+)+)$"
    },
    "tlsServerConfig": {
      "type": "object",
      "additionalProperties": false,
      "properties": {
        "certFile": { "type": "string" },
        "keyFile": { "type": "string" },
        "clientCaFile": { "type": "string" },
        "requireClientCert": { "type": "boolean", "default": false }
      }
    },
    "tlsClientConfig": {
      "type": "object",
      "additionalProperties": false,
      "properties": {
        "caFile": { "type": "string" },
        "certFile": { "type": "string" },
        "keyFile": { "type": "string" },
        "serverName": { "type": "string" },
        "pinSha256": {
          "type": "array",
          "items": { "type": "string" }
        },
        "insecureSkipVerify": { "type": "boolean", "default": false }
      }
    },
    "connectionConfig": {
      "type": "object",
      "required": ["transport"],
      "additionalProperties": false,
      "properties": {
        "transport": {
          "type": "string",
          "enum": ["quic", "wss", "ws", "unix", "named-pipe", "stdio", "memory"]
        },
        "uri": { "type": "string" },
        "uris": {
          "type": "array",
          "minItems": 1,
          "items": { "type": "string" }
        },
        "loadBalancing": {
          "type": "object",
          "additionalProperties": false,
          "properties": {
            "strategy": {
              "type": "string",
              "enum": ["failover", "round-robin", "least-loaded", "consistent-hash"],
              "default": "failover"
            },
            "healthCheckIntervalMs": { "type": "integer", "minimum": 100 }
          }
        },
        "path": { "type": "string" },
        "spawn": {
          "type": "object",
          "required": ["command"],
          "additionalProperties": false,
          "properties": {
            "command": { "type": "string" },
            "args": { "type": "array", "items": { "type": "string" } },
            "cwd": { "type": "string" },
            "env": {
              "type": "object",
              "additionalProperties": { "type": "string" }
            }
          }
        },
        "tls": { "$ref": "#/$defs/tlsClientConfig" },
        "auth": {
          "type": "object",
          "required": ["mode"],
          "additionalProperties": false,
          "properties": {
            "mode": { "$ref": "#/$defs/authMode" },
            "token": { "type": "string" },
            "apiKey": { "type": "string" },
            "keyId": { "type": "string" },
            "privateKeyFile": { "type": "string" }
          }
        },
        "allowInsecureTransport": { "type": "boolean", "default": false },
        "reconnect": {
          "type": "object",
          "additionalProperties": false,
          "properties": {
            "enabled": { "type": "boolean", "default": true },
            "initialDelayMs": { "type": "integer", "minimum": 50 },
            "maxDelayMs": { "type": "integer", "minimum": 100 },
            "multiplier": { "type": "number", "minimum": 1 },
            "maxAttempts": { "type": "integer", "minimum": 0 }
          }
        }
      }
    },
    "mountConfig": {
      "type": "object",
      "required": ["id", "anchorPath", "connection"],
      "additionalProperties": false,
      "properties": {
        "id": {
          "type": "string",
          "pattern": "^[A-Za-z0-9._-]+$"
        },
        "anchorPath": { "$ref": "#/$defs/path" },
        "connection": { "type": "string" },
        "remotePath": { "$ref": "#/$defs/path", "default": "/" },
        "role": {
          "type": "string",
          "enum": ["controller", "observer"],
          "default": "controller"
        },
        "lazy": { "type": "boolean", "default": false },
        "required": { "type": "boolean", "default": false },
        "timeoutMs": { "type": "integer", "minimum": 1 },
        "authPropagation": {
          "type": "object",
          "required": ["strategy"],
          "additionalProperties": false,
          "properties": {
            "strategy": {
              "type": "string",
              "enum": ["none", "static", "token-exchange", "passthrough"],
              "default": "none"
            },
            "audience": { "type": "string" },
            "scopes": { "type": "array", "items": { "type": "string" } },
            "tokenEndpoint": { "type": "string" }
          }
        },
        "allowElevation": { "type": "boolean", "default": false },
        "stripMetaPrefixes": {
          "type": "array",
          "items": { "type": "string" }
        }
      }
    }
  }
}
```

---

## 5. Complete Configuration Examples

### 5.1 Example 1 — Workstation & Headless API Aggregator Server

An aggregator server exposing QUIC (`ALPN: "aep/1"`, TLS 1.3 + `bearer` / `mtls`) and local POSIX Unix Domain Socket (`os-peer`) listeners, mounting a local terminal provider over spawned `stdio` and a headless billing `Api` provider over QUIC with `token-exchange`:

```json
{
  "schemaVersion": "1.0",
  "metadata": {
    "name": "workstation-aggregator",
    "environment": "production"
  },
  "server": {
    "info": {
      "name": "aep-workstation-host",
      "version": "1.0.0"
    },
    "profile": "aep-core-streaming",
    "listeners": [
      {
        "id": "quic-primary",
        "transport": "quic",
        "address": "0.0.0.0:7443",
        "alpn": ["aep/1"],
        "tls": {
          "certFile": "/etc/aep/certs/server.crt",
          "keyFile": "/etc/aep/certs/server.key",
          "clientCaFile": "/etc/aep/certs/clients-ca.crt",
          "requireClientCert": false
        }
      },
      {
        "id": "local-ipc",
        "transport": "unix",
        "path": "/run/aep/workstation.sock"
      }
    ],
    "security": {
      "authModes": ["mtls", "bearer", "os-peer"],
      "jwt": {
        "issuer": "https://auth.example.internal",
        "audience": "aep://workstation-aggregator",
        "jwksUri": "https://auth.example.internal/.well-known/jwks.json"
      },
      "elevation": {
        "enabled": true,
        "maxDurationMs": 300000,
        "defaultAllowedFactors": ["human-approval", "passkey"],
        "auditLogPath": "/var/log/aep/elevation-audit.jsonl"
      }
    }
  },
  "connections": {
    "local-terminal": {
      "transport": "stdio",
      "spawn": {
        "command": "aep-terminal-provider",
        "args": ["--shell", "/bin/bash"],
        "env": {
          "TERM": "xterm-256color"
        }
      }
    },
    "billing-api-upstream": {
      "transport": "quic",
      "uri": "quic://billing-aep.internal:7443",
      "tls": {
        "caFile": "/etc/aep/certs/internal-ca.crt",
        "certFile": "/etc/aep/certs/aggregator-client.crt",
        "keyFile": "/etc/aep/certs/aggregator-client.key"
      },
      "auth": {
        "mode": "mtls"
      },
      "reconnect": {
        "enabled": true,
        "initialDelayMs": 250,
        "maxDelayMs": 10000,
        "multiplier": 2
      }
    }
  },
  "mounts": [
    {
      "id": "ide-term",
      "anchorPath": "/workstation/ide/layout/term",
      "connection": "local-terminal",
      "remotePath": "/",
      "role": "controller",
      "timeoutMs": 5000,
      "authPropagation": { "strategy": "none" },
      "allowElevation": false,
      "stripMetaPrefixes": ["internal."]
    },
    {
      "id": "billing-api",
      "anchorPath": "/services/billing",
      "connection": "billing-api-upstream",
      "remotePath": "/billing",
      "role": "controller",
      "lazy": true,
      "timeoutMs": 8000,
      "authPropagation": {
        "strategy": "token-exchange",
        "audience": "aep://billing-api",
        "scopes": ["billing.read", "billing.refund.execute"],
        "tokenEndpoint": "https://auth.example.internal/oauth/token"
      },
      "allowElevation": true
    }
  ]
}
```

### 5.2 Example 2 — Embodied Robot + Cloud Workspace Agent Client

An autonomous agent client assembling a unified "body of senses" from a robotic platform (over QUIC with pinned TLS + bearer token) and a cloud workspace aggregator (over `wss://`), configured to synthesize lightweight `renderMinimal` views client-side with path callouts:

```json
{
  "schemaVersion": "1.0",
  "metadata": {
    "name": "field-agent-runtime",
    "environment": "staging"
  },
  "connections": {
    "robot-rig": {
      "transport": "quic",
      "uri": "quic://robot-04.fleet.internal:7443",
      "tls": {
        "caFile": "${FILE:/etc/aep/fleet-ca-path}",
        "pinSha256": ["47DEQpj8HBSa+/TImW+5JCeuQeRkm5NMpJWZG3hSuFU="]
      },
      "auth": {
        "mode": "bearer",
        "token": "${ENV:AEP_ROBOT_TOKEN}"
      },
      "reconnect": {
        "enabled": true,
        "initialDelayMs": 200,
        "maxDelayMs": 5000,
        "multiplier": 1.5
      }
    },
    "cloud-workspace": {
      "transport": "wss",
      "uri": "wss://workspace.example.internal/aep",
      "auth": {
        "mode": "bearer",
        "token": "${ENV:AEP_WORKSPACE_TOKEN}"
      }
    }
  },
  "mounts": [
    {
      "id": "robot",
      "anchorPath": "/robot",
      "connection": "robot-rig",
      "remotePath": "/",
      "role": "controller",
      "required": true,
      "timeoutMs": 3000,
      "allowElevation": true
    },
    {
      "id": "workspace",
      "anchorPath": "/workspace",
      "connection": "cloud-workspace",
      "remotePath": "/",
      "role": "controller",
      "lazy": true,
      "timeoutMs": 10000,
      "allowElevation": true
    }
  ],
  "client": {
    "info": {
      "name": "autonomous-operator-agent",
      "version": "2.1.0"
    },
    "defaultRole": "controller",
    "allowDowngradeToObserver": false,
    "rendering": {
      "preferredMode": "renderMinimal",
      "minimalSynthesizer": "auto",
      "format": "image/svg+xml",
      "annotatePaths": true
    }
  }
}
```
