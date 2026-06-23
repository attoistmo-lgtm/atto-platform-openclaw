# ATTO Platform OpenClaw Deployment

Repositorio base para desplegar **OpenClaw** como plataforma de agente de IA personal en Kubernetes con FluxCD (modelo GitOps Stage 2).

OpenClaw es un gateway self-hosted que conecta modelos de lenguaje (Claude, GPT, Gemini, Ollama, etc.) con más de 20 canales de mensajería: WhatsApp, Telegram, Discord, Slack, Signal, iMessage, Microsoft Teams, Matrix, WebChat, y más.

---

## Overview

| Item | Value |
|---|---|
| Platform name | atto-platform-openclaw |
| Image | ghcr.io/attoistmo-lgtm/atto-platform-openclaw |
| Namespace | openclaw-system |
| Exposure | Ingress (Kong) + ClusterIP |
| Hostname | openclaw.attoistmo.com.mx |
| Storage | Longhorn (`longhorn-fast`) - 10Gi |
| LLM Provider | LiteLLM Gateway (interno) |
| GitOps tool | Flux CD |
| License | MIT (upstream) |

---

## Arquitectura

```
┌─────────────────────────────────────────────────────────────┐
│                    Kong Ingress (OIDC)                       │
│              openclaw.attoistmo.com.mx                       │
└────────────────────────┬────────────────────────────────────┘
                         │
                    ┌────▼────┐
                    │ Service │  ClusterIP
                    │ :3000   │  Gateway WS
                    │ :3001   │  WebChat UI
                    └────┬────┘
                         │
              ┌──────────▼──────────┐
              │    StatefulSet      │  replicas: 1
              │   openclaw-gateway  │  Recreate strategy
              └──────────┬──────────┘
                         │
          ┌──────────────┼──────────────┐
          │              │              │
    ┌─────▼─────┐  ┌─────▼─────┐  ┌────▼─────┐
    │  ConfigMap │  │  Secret   │  │   PVC    │
    │ openclaw   │  │  (SOPS)   │  │ workspace│
    │  .json     │  │  API keys │  │ 10Gi RWX │
    └───────────┘  └───────────┘  └──────────┘

Dependencias externas:
  ├── LiteLLM Gateway  → Provider de LLMs
  ├── Redis            → Caching y session state
  ├── PostgreSQL (CNPG)→ Persistencia opcional
  ├── Keycloak         → OIDC authentication
  └── Cloudflared      → Tunnels para canales
```

---

## Estructura del Repositorio

```
atto-platform-openclaw/
├── .sops.yaml                                    # Reglas de encriptación SOPS
├── README.md                                     # Esta documentación
│
├── docker/
│   └── Dockerfile                                # Containerización de OpenClaw
│
├── .github/
│   └── workflows/
│       └── build-image.yml                       # CI/CD para build de imagen
│
├── base/
│   ├── namespace.yaml                            # openclaw-system
│   └── kustomization.yaml
│
├── configmaps/
│   ├── atto-platform-openclaw-config.yaml        # openclaw.json
│   └── kustomization.yaml
│
├── secrets/
│   ├── atto-platform-openclaw-secrets.yaml       # API keys, tokens (SOPS)
│   └── kustomization.yaml
│
├── releases/
│   ├── atto-platform-openclaw-statefulset.yaml   # StatefulSet principal
│   ├── atto-platform-openclaw-service.yaml       # ClusterIP
│   ├── atto-platform-openclaw-pdb.yaml           # PodDisruptionBudget
│   ├── atto-platform-openclaw-ingress.yaml       # Ingress (Kong)
│   └── kustomization.yaml
│
├── main/
│   └── kustomization.yaml                        # Agrega base + config + secrets + releases
│
└── flux/
    ├── atto-platform-openclaw-gitrepository.yaml # Flux GitRepository
    ├── atto-platform-openclaw-kustomization.yaml # Flux Kustomization
    └── kustomization.yaml
```

---

## GitOps Flow (Stage 2)

1. **Flux GitRepository**
   - Autenticado vía HTTPS usando Git provider PAT
   - Apunta a este repositorio (branch `main`)

2. **Flux Kustomization**
   - Reconcilia desde `./main`
   - Pruning habilitado
   - Desencriptación SOPS habilitada

3. **Main Kustomization**
   - Agrega:
     - Namespace
     - ConfigMaps (openclaw.json)
     - Secrets encriptados (SOPS)
     - Releases (StatefulSet, Service, PDB, Ingress)

4. **StatefulSet**
   - Single replica (stateful)
   - Estrategia `Recreate`
   - PVC con Longhorn para workspace persistente

---

## Secrets & Encryption

- Todos los secretos se almacenan en:
  ```
  secrets/atto-platform-openclaw-secrets.yaml
  ```
- Secretos encriptados con **SOPS + PGP**
- Reglas de encriptación definidas globalmente en `.sops.yaml`
- Flux desencripta secretos en runtime usando la clave PGP almacenada en:
  ```
  flux-system/sops-gpg
  ```

**Secrets requeridos:**

| Secret Key | Descripción |
|---|---|
| `LITELLM_API_KEY` | API key para LiteLLM Gateway |
| `REDIS_PASSWORD` | Password de Redis para caching |
| `TELEGRAM_BOT_TOKEN` | Token del bot de Telegram |
| `DISCORD_BOT_TOKEN` | Token del bot de Discord |
| `SLACK_BOT_TOKEN` | Token del bot de Slack |
| `WHATSAPP_TOKEN` | Token de WhatsApp |
| `OPENCLAW_MASTER_KEY` | Master key del Gateway |

**Los secretos en plaintext en Git están prohibidos**

---

## Persistencia

- Todos los volúmenes persistentes usan:
  ```
  storageClass: longhorn-fast
  ```
- El workspace de OpenClaw (memoria, skills, logs) es persistente
- Safe para reschedulear entre nodos

---

## Ingress & Networking

- OpenClaw se expone vía Kubernetes Ingress (Kong)
- Hostname: `openclaw.attoistmo.com.mx`
- TLS termination **upstream** (Cloudflare / Cloudflared)
- Puerto 3000: Gateway WebSocket
- Puerto 3001: WebChat UI

---

## Canales Soportados

| Canal | Estado | Configuración |
|---|---|---|
| WebChat | Habilitado por defecto | `openclaw.attoistmo.com.mx` |
| Telegram | Opcional | Requiere bot token |
| Discord | Opcional | Requiere bot token |
| Slack | Opcional | Requiere bot token |
| WhatsApp | Opcional | Requiere token + Evolution API |
| Signal | Opcional | Requiere configuración adicional |
| Microsoft Teams | Opcional | Requiere app registration |
| Matrix | Opcional | Requiere homeserver |

---

## Operaciones

### Bootstrap Flux

```bash
kubectl apply -k flux
```

### Reconciliación

```bash
flux reconcile source git atto-platform-openclaw -n flux-system
flux reconcile kustomization atto-platform-openclaw -n flux-system
```

### Verificar estado

```bash
flux get kustomizations atto-platform-openclaw -n flux-system
kubectl get statefulset -n openclaw-system
kubectl get pods -n openclaw-system
kubectl logs -n openclaw-system -l app=openclaw-gateway -f
```

### Upgrade

1. Actualizar imagen o valores en ConfigMap
2. Commit y push a `main`
3. Flux reconcilia automáticamente

### Rollback

1. Revert commit en Git
2. Flux reconcilia el estado deseado

### Acceder al WebChat

```bash
kubectl port-forward -n openclaw-system svc/openclaw-gateway 3001:3001
# Abrir http://localhost:3001
```

---

## Integración con LiteLLM

OpenClaw usa LiteLLM como provider de LLMs. La configuración en `openclaw.json`:

```json
{
  "llm": {
    "provider": "openai-compatible",
    "baseUrl": "http://atto-platform-litellm-gateway.litellm-system.svc.cluster.local:4000/v1",
    "model": "gpt-4o-mini"
  }
}
```

Esto permite:
- Usar cualquier modelo configurado en LiteLLM
- Rate limiting y caching gestionados por LiteLLM
- Rotación de API keys transparente

---

## Observabilidad

### ServiceMonitor (pendiente)

```yaml
# TODO: Crear ServiceMonitor para Prometheus
# Depende de atto-platform-prometheus
```

### Logging

Los logs de OpenClaw fluyen hacia Loki vía FluentBit/FluentBit daemonset.

### Dashboards

Dashboards de Grafana para:
- Estado del Gateway
- Sesiones activas
- Uso de canales
- Latencia de respuestas

---

## Ownership

Este repositorio es un **platform repository**.

- NO debe contener código de aplicación
- NO debe incluir secretos inline
- Debe permanecer determinista y reproducible
- Los cambios deben pasar por revisión Git

### Initial commit message

```
feat(platform): add OpenClaw platform installation via Flux and Kustomize (Stage 2)

* Containerize OpenClaw with custom Dockerfile (Node.js 24 Alpine)
* Deploy as StatefulSet with single replica (Recreate strategy)
* Configure openclaw.json via ConfigMap
* Encrypt API keys and channel tokens with SOPS + PGP
* Expose via Kong Ingress with OIDC at openclaw.attoistmo.com.mx
* Integrate with LiteLLM Gateway as LLM provider
* Use Longhorn for persistent workspace storage
* Enable pruning, decryption, and declarative reconciliation
* Configure GitHub Actions for automated image builds
```
