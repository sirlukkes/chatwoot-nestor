# AI_MAP — chatwoot-nestor

> Mapa destilado para orientación rápida. Lee esto ANTES de explorar el árbol.
> **Alcance:** este repo es un fork de Chatwoot (base v3.9.0 / cwctl 2.8.0). La arquitectura general
> de Chatwoot está documentada río arriba; este mapa cubre lo que hace ÚNICO a este fork —
> principalmente el desbloqueo de la Enterprise Edition. Para el resto del sistema, aplica el
> conocimiento estándar de Chatwoot.

## 1. Subsistemas (base Chatwoot)
- **Rails app** (`app/`) — backend: modelos, controllers `api/v1`, jobs Sidekiq, servicios.
- **Frontend** (`app/javascript/`) — dashboard Vue + widget.
- **Enterprise** (`enterprise/`) — código EE (billing, SLA, audit logs, captain/response_bot). Se
  monta vía `ChatwootApp.extensions` cuando existe la carpeta ([`lib/chatwoot_app.rb`](lib/chatwoot_app.rb)).
- **Config de instalación** (`config/installation_config.yml`, `config/features.yml`) — semillas de
  `InstallationConfig` y feature flags por defecto, cargadas por `ConfigLoader`.

## 2. Flujo del gating Enterprise (lo modificado por el fork)
`ChatwootHub.pricing_plan` (lee `INSTALLATION_PRICING_PLAN` de la DB, default `enterprise`) →
`ReconcilePlanConfigService` apaga features premium **solo si el plan es `community`** →
como el plan es `enterprise` y el hub está apuntado a `google.com` (nadie escribe `community`),
nunca se apaga nada → `Account#enabled_features` devuelve las premium → jbuilder las expone al front.
**Detalle completo y cómo replicarlo:** [`docs/enterprise-unlock.md`](docs/enterprise-unlock.md).

## 3. Dónde vive cada cosa (intención → archivo)
| Para... | Abre... |
|---|---|
| Entender/replicar el desbloqueo Enterprise | [`docs/enterprise-unlock.md`](docs/enterprise-unlock.md) |
| Plan/licencia por defecto + URL del hub | [`lib/chatwoot_hub.rb`](lib/chatwoot_hub.rb) |
| Semilla de `INSTALLATION_PRICING_PLAN` / quantity | [`config/installation_config.yml`](config/installation_config.yml) (L186-190) |
| Features premium encendidas por defecto | [`config/features.yml`](config/features.yml) (audit_logs, sla, help_center_embedding_search) |
| Guardián que apaga premium (desarmado) | [`enterprise/app/services/internal/reconcile_plan_config_service.rb`](enterprise/app/services/internal/reconcile_plan_config_service.rb) |
| Job diario que reescribe el plan desde el hub | [`enterprise/app/jobs/enterprise/internal/check_new_versions_job.rb`](enterprise/app/jobs/enterprise/internal/check_new_versions_job.rb) |
| Cómo se resuelve `enabled_features` de una cuenta | [`app/models/concerns/featurable.rb`](app/models/concerns/featurable.rb) |
| Límites de agentes/inboxes | [`enterprise/app/models/enterprise/account.rb`](enterprise/app/models/enterprise/account.rb) + `ChatwootApp.max_limit` |
| Carga de config a la DB (upsert vs solo-nuevo) | [`lib/config_loader.rb`](lib/config_loader.rb) |

## 4. Gotchas / verdades no-obvias
- Features EE presentes pero apagadas en un install → el plan es `community` → el reconciliador diario
  las apaga. El fork lo evita forzando plan `enterprise` + hub muerto. Detalle: [`docs/enterprise-unlock.md`](docs/enterprise-unlock.md).
- Editar `config/installation_config.yml` NO cambia una DB ya instalada → `ConfigLoader` corre con
  `reconcile_only_new: true` (solo crea faltantes) → hay que correr `ConfigLoader.new.process(reconcile_only_new: false)` o editar la fila. Detalle: [`docs/enterprise-unlock.md`](docs/enterprise-unlock.md).
- `ChatwootHub::BASE_URL` apunta a `google.com` → mata telemetría Y el aviso de nueva versión (mismo
  canal); deja un POST diario fallido. Detalle: [`docs/enterprise-unlock.md`](docs/enterprise-unlock.md).

## 5. Infra
- **Repo:** `/Users/lukkes/Developer/chatwoot-nestor` — remotos: `origin` = sirlukkes/chatwoot-nestor, `upstream` = jmelati/chatwoot-nestor.
- **Base:** Chatwoot v3.9.0 (`VERSION_CW`), cwctl 2.8.0 (`VERSION_CWCTL`). Stack: Rails + Vue + Sidekiq + Postgres + Redis.
- **Deploy/run:** vía Docker (`docker-compose.production.yaml`) — TODO confirmar el flujo de deploy real del operador.
- **Tests:** RSpec (`bundle exec rspec`) — TODO confirmar suite usada.
- **Memoria del proyecto:** file-based (`~/.claude/projects/-Users-lukkes-Developer-chatwoot-nestor/memory/`); engram no-usado.
