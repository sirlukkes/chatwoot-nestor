# Mega Stack — manual de referencia externo

`docs/mega-stack/` es una copia vendorizada de [`github.com/megaapp977/stack`](https://github.com/megaapp977/stack),
usada como fuente de consulta/manual de usuario para Mega (el producto detrás de este fork de
Chatwoot). No es código de este proyecto — es documentación de terceros, versionada directo en
este repo (sin `.git` propio ni submódulo) para tenerla offline y disponible en cualquier clon.

## Qué contiene

Guías de despliegue Docker Swarm de todo el stack de Mega (Traefik, Portainer, la app Mega,
Postgres/pgvector, Redis, MinIO, y los puentes de canal Evolution/WAHA/Wavoip/Typebot/WordPress),
troubleshooting operativo por canal (WABA, TikTok), referencias de API (colección Postman +
docs por endpoint), documentación de producto/features en dos niveles (comercial y técnico),
la guía para escribir Dashboard Scripts, SSO e integración con Google, y notas de mantenimiento
de base de datos.

Casi todo documento importante existe en ES/EN/PT-BR, pero **el sufijo de idioma no es uniforme
entre carpetas** (`*.es.md`, `*_en.md` / `*-en.md`, `*_pt_BR.md` / `*-pt-br.md` según la sección)
— listar el directorio antes de asumir el nombre exacto de un archivo.

## Cómo consultarlo

Índice navegable con todas las categorías y links directos a cada archivo:
[Mega Stack Manual (artifact)](https://claude.ai/code/artifact/1bce0c8c-5264-45c4-9373-bf7968d80ada).

Categorías, con su carpeta dentro de `docs/mega-stack/`:

| Tema | Carpeta |
|---|---|
| Despliegue e infraestructura (Docker Swarm, cada servicio) | `mega-docker/`, `base/`, `storage/`, `traefik/`, `portainer/`, `evolution/`, `waha/`, `wavoip/`, `typebot-agentbot/`, `wordpress/`, `ia-owner/` |
| Canales de mensajería (troubleshooting) | `waba/`, `tiktok/`, `idea/whatsapp_sync_reliability.md` |
| API pública (Postman + referencias) | `API-MEGA/` |
| Producto / features | `Features/` |
| Dashboard Scripts (customización sin tocar el core) | `Dashboard-Script/` |
| SSO e integración Google | `SSO/`, `google/` |
| Datos / mantenimiento de DB | `migrations/`, `Other topics/limpieza de lid en banco/` |

## Actualizar el contenido

Es una copia vendorizada, no un clon vivo — no trae cambios upstream automáticamente. Para
refrescarla: clonar `github.com/megaapp977/stack` aparte, copiar los archivos nuevos/cambiados
sobre `docs/mega-stack/` y commitear el diff normal de este repo.

Si el índice del artifact queda desactualizado tras un refresh, regenerarlo listando
`docs/mega-stack/` de nuevo — no es un documento que se mantenga a mano.
