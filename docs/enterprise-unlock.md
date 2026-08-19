# Enterprise unlock — cómo este fork usa las features EE sin licencia

> Documento de análisis (solo lectura, ninguna línea de este repo fue modificada al escribirlo).
> Explica el mecanismo por el que `chatwoot-nestor` habilita las funcionalidades Enterprise de
> Chatwoot sin licencia comercial, y cómo replicarlo en otro fork.

## Contexto

Chatwoot Community **ya incluye todo el código Enterprise** en la carpeta [`enterprise/`](../enterprise/).
No es una edición distinta que haya que compilar: el código está ahí. Lo que separa "Community" de
"Enterprise" es un **candado de licencia** que verifica el plan contra el hub de Chatwoot
(`hub.2.chatwoot.com`) y, si el plan es `community`, apaga las features premium.

Este fork **no borra el candado**: lo engaña para que la instancia se crea permanentemente en plan
`enterprise`. El candado sigue girando, pero en el vacío.

## El mecanismo (5 piezas coordinadas)

Todo se apoya en un solo pivote verificado en el código: el reconciliador de plan solo apaga las
features premium **si `ChatwootHub.pricing_plan == 'community'`**
([`enterprise/app/services/internal/reconcile_plan_config_service.rb:4`](../enterprise/app/services/internal/reconcile_plan_config_service.rb)):

```ruby
return if ChatwootHub.pricing_plan != 'community'   # con 'enterprise' → sale acá, nunca apaga nada
```

Las 5 piezas garantizan que ese valor sea `enterprise` para siempre:

### 1. El plan por defecto es `enterprise` y las licencias son ilimitadas
[`lib/chatwoot_hub.rb:20`](../lib/chatwoot_hub.rb) y [`:24`](../lib/chatwoot_hub.rb):

```ruby
def self.pricing_plan
  InstallationConfig.find_by(name: 'INSTALLATION_PRICING_PLAN')&.value || 'enterprise'
end

def self.pricing_plan_quantity
  InstallationConfig.find_by(name: 'INSTALLATION_PRICING_PLAN_QUANTITY')&.value || 100000
end
```

En Chatwoot Community estos defaults son el plan `community` y quantity `0`. Con `enterprise`,
`Enterprise::Concerns::User` nunca corta la creación de usuarios por límite de licencias, y
`usage_limits` cae en `ChatwootApp.max_limit` = 100.000 agentes/inboxes
([`lib/chatwoot_app.rb:10`](../lib/chatwoot_app.rb),
[`enterprise/app/models/enterprise/account.rb`](../enterprise/app/models/enterprise/account.rb)).

### 2. El "phone-home" apunta a un agujero negro
[`lib/chatwoot_hub.rb:2`](../lib/chatwoot_hub.rb):

```ruby
BASE_URL = ENV.fetch('CHATWOOT_HUB_URL', 'https://google.com')   # upstream: el hub real de Chatwoot
```

Todas las URLs (`PING`, `BILLING`, `EVENTS`, `REGISTRATION`, `PUSH`) cuelgan de ahí. Un POST a
`google.com/ping` devuelve HTML; el `JSON.parse` revienta, cae en `rescue StandardError` y
`sync_with_hub` devuelve `nil`. La instancia **nunca recibe una respuesta válida del hub real**.

### 3. El guardián diario queda desarmado
Cada día a las 12:00 ([`config/schedule.yml:6-8`](../config/schedule.yml)) corre
`Enterprise::Internal::CheckNewVersionsJob`
([`enterprise/app/jobs/enterprise/internal/check_new_versions_job.rb`](../enterprise/app/jobs/enterprise/internal/check_new_versions_job.rb)),
que en una instancia normal:

1. Llama al hub y, con la respuesta, **sobrescribe** `INSTALLATION_PRICING_PLAN` en la DB con lo que
   diga Chatwoot (para un self-hosted que no paga: `community`), y lo marca `locked = true`.
2. Llama a `ReconcilePlanConfigService`, que **apaga las features premium en TODAS las cuentas** —
   pero solo si el plan es `community` (el pivote de arriba).

El fork lo neutraliza en el paso 1: como el hub está muerto (pieza 2), `@instance_info` llega vacío
y `update_plan_info` hace `return if @instance_info.blank?` → **nunca escribe `community`** → el plan
sigue `enterprise` → `ReconcilePlanConfigService` sale por el `return` temprano → las premium quedan
encendidas. El candado corre todos los días y no apaga nada.

### 4. Se siembra la DB ya en plan enterprise
[`config/installation_config.yml:186-190`](../config/installation_config.yml):

```yaml
- name: INSTALLATION_PRICING_PLAN
  value: 'enterprise'
- name: INSTALLATION_PRICING_PLAN_QUANTITY
  value: 100000000000000000000000000000000
```

### 5. Las features premium nacen encendidas
[`config/features.yml`](../config/features.yml): `audit_logs` (L68-70), `sla` (L81-83) y
`help_center_embedding_search` (L85-87) están en `enabled: true` a pesar de `premium: true`. En
upstream esas nacen en `false`. (`disable_branding` y `response_bot` siguen en `false` — el fork no
las tocó.)

Así, `Account#enabled_features`
([`app/models/concerns/featurable.rb:54`](../app/models/concerns/featurable.rb)) devuelve las premium
para cada cuenta nueva, y el jbuilder las expone al frontend
([`app/views/api/v1/models/_account.json.jbuilder`](../app/views/api/v1/models/_account.json.jbuilder),
`json.features @account.enabled_features`).

## Cómo replicarlo en otro fork

Son los mismos 5 cambios; se editan 3 archivos. Los dos servicios Enterprise
(`ReconcilePlanConfigService`, `CheckNewVersionsJob`) **no se tocan** — juegan a favor una vez que el
plan es `enterprise`.

1. `lib/chatwoot_hub.rb` → `BASE_URL` default a `'https://google.com'`.
   **Recomendado:** deja el default real en el código y define `CHATWOOT_HUB_URL=https://google.com`
   (o un endpoint muerto propio) por variable de entorno — más limpio que parchear la clase.
2. `lib/chatwoot_hub.rb` (`pricing_plan`) → default `'enterprise'`.
3. `lib/chatwoot_hub.rb` (`pricing_plan_quantity`) → default `100000`.
4. `config/installation_config.yml` → sembrar `INSTALLATION_PRICING_PLAN: enterprise` y quantity
   gigante.
5. `config/features.yml` → `enabled: true` en las features premium deseadas.

### El detalle que muerde: DB ya instalada

El `ConfigLoader` corre con `reconcile_only_new: true`
([`lib/config_loader.rb:3`](../lib/config_loader.rb)): **solo crea configs faltantes, no pisa las
existentes**. En un install que ya tiene una fila `INSTALLATION_PRICING_PLAN` en la DB, editar el
YAML no cambia nada. Para forzarlo:

- Rails console: `ConfigLoader.new.process(reconcile_only_new: false)` (repisa todo con los defaults
  del YAML), **o**
- actualizar la fila directo (Rails console o Super Admin → Settings → Configuration), **o**
- solo aplica en un install fresco (el seed toma el YAML de una).

## Efectos colaterales a tener presentes

- **Licencia:** esto elude la licencia comercial de la Enterprise Edition. El código bajo
  [`enterprise/`](../enterprise/) **no es MIT** — tiene su propia [`enterprise/LICENSE`](../enterprise/LICENSE),
  que restringe el uso a instalaciones con la EE debidamente licenciada. Circunvenirla tiene
  implicación legal/ToS para uso más allá del personal.
- **Telemetría y avisos de versión:** al matar el hub se pierde el aviso de "nueva versión
  disponible" (usa el mismo canal). El job igual hace un POST diario fallido a `google.com`; lo
  prolijo es apuntar `CHATWOOT_HUB_URL` a un endpoint muerto propio o desactivar el cron
  `internal_check_new_versions_job` en `config/schedule.yml`.

## Origen en el historial

Las modificaciones del hub entraron en el commit `e7475412` (dic 2023, autor `rotsen`), que creó
`lib/chatwoot_hub.rb` ya con `BASE_URL=google.com` y `pricing_plan='enterprise'`; el default
`pricing_plan_quantity=100000` se ajustó en `580d280d` (mar 2024).
