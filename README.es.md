# Este repositorio fue reemplazado por github.com/getstratum/stratum

# Stratum — AI Governance Platform

**Gateway de IA open-source para equipos enterprise. Control, visibilidad y gobernanza sobre cada llamada a la API de IA.**

> 📖 [Read in English](./README.md)

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](./LICENSE)
[![Version](https://img.shields.io/badge/version-0.0.2--beta-violet)](./CHANGELOG.md)
[![Docker](https://img.shields.io/badge/Docker-ready-2496ED?logo=docker)](./docker-compose.yml)

---

## ¿Qué es Stratum?

Stratum se interpone entre tus equipos y cada proveedor de IA. Vos definís las reglas una vez. Cada request — de developers, usuarios de negocio y agentes automatizados — pasa por Stratum antes de llegar a OpenAI, Anthropic, Gemini o cualquier otro proveedor.

```
Developer / Agente / Usuario del Playground
          ↓
    Stratum Gateway          ← corre dentro de tu red
    (auth · política · log)
          ↓
  OpenAI / Anthropic / Gemini / Modelos locales
```

No se almacena el contenido de los prompts. Solo metadata: quién llamó qué, cuándo, cuántos tokens, a qué costo.

---

## El problema

Cuando los equipos adoptan IA de forma independiente, las organizaciones terminan con:

- **Sin visibilidad** — nadie sabe el gasto real ni quién usa qué modelo
- **Sin control** — cualquier developer puede llamar a GPT-4o sin límites
- **Sin gobernanza** — datos sensibles llegan a proveedores sin supervisión
- **Costos fragmentados** — facturas dispersas entre equipos y herramientas

Stratum es una sola capa que resuelve los cuatro.

---

## Funcionalidades principales

| Feature | Descripción |
|---------|-------------|
| **Proxy de API** | Reemplazo directo de las URLs base de OpenAI/Anthropic. Cero cambios de código para developers. |
| **Motor de políticas** | Whitelists de modelos, quotas de tokens y límites de budget enforceados en tiempo real via Redis. |
| **Dashboard** | Analytics por equipo, proveedor y modelo. Filtros de fecha. Export CSV. Sin contenido de prompts. |
| **Auth del dashboard** | Los miembros del equipo Engineering acceden al dashboard de admin. Los demás equipos van al Playground. |
| **AI Playground** | Interfaz de chat para usuarios no técnicos. Mismas políticas que la API. Adjuntos de archivos. Historial persistente. |
| **Agentes** | Registrá procesos automatizados con keys dedicadas (`aig_agt_`). El admin configura el modelo centralmente. |
| **Gestión de providers** | Agregá, habilitá o deshabilité providers de IA desde la UI — sin cambios de código. |
| **Compresión de contexto** | Un LLM local (Ollama) resume mensajes anteriores antes de enviar al proveedor. Reduce el gasto en tokens. |
| **Alertas** | Notificaciones por email y Slack cuando los equipos se acercan a los límites de budget o tokens. |
| **Observabilidad** | Timing por etapa (auth, política, proveedor). Endpoint Prometheus `/metrics`. |
| **Arquitectura split-plane** | El gateway corre dentro de la red del cliente. Los prompts nunca salen de su infraestructura. |

---

## Inicio rápido

### Prerequisitos
- Docker y Docker Compose
- API keys de al menos un proveedor (OpenAI o Anthropic)

### 1. Clonar y configurar

```bash
git clone https://github.com/matiasmospan/stratum.git
cd stratum
cp .env.example .env
```

Editá `.env` y completá:

```env
POSTGRES_PASSWORD=tu-password-seguro
OPENAI_API_KEY=sk-...
ANTHROPIC_API_KEY=sk-ant-...

# URL del gateway visible en el browser
NEXT_PUBLIC_GATEWAY_URL=http://localhost:8080

# Seguridad de sesión del dashboard
SESSION_SECRET=generá-una-string-random-larga-aquí
```

### 2. Levantar el stack

```bash
docker compose up -d --build
```

### 3. Migraciones y datos de demo

```bash
docker exec -it $(docker compose ps -q proxy) node /app/src/migrate.js
docker exec -it $(docker compose ps -q proxy) node /app/src/seed-passwords.js
```

### 4. Acceso

| Servicio | URL | Credenciales |
|---------|-----|-------------|
| Dashboard (admin) | http://localhost:3000 | `dev@acme.com` / `dev123` (equipo Engineering) |
| Playground | http://localhost:3000/prompt | `dev@acme.com` / `dev123` o `marketing@acme.com` / `mkt123` |
| Gateway API | http://localhost:8080 | API key o JWT |

---

## Hacer llamadas a la API

Solo cambiás la URL base. Todo lo demás queda igual.

```bash
# Antes (directo a OpenAI)
curl https://api.openai.com/v1/chat/completions \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -d '{"model":"gpt-4o-mini","messages":[{"role":"user","content":"Hola"}]}'

# Después (por Stratum)
curl http://localhost:8080/proxy/openai/v1/chat/completions \
  -H "Authorization: Bearer aig_sk_eng_test1234567890" \
  -d '{"model":"gpt-4o-mini","messages":[{"role":"user","content":"Hola"}]}'
```

### Llamadas de agentes — sin modelo ni proveedor hardcodeado

```python
client = openai.OpenAI(
    api_key="aig_agt_xxxx",
    base_url="http://localhost:8080/proxy/auto/v1"
)
response = client.chat.completions.create(
    model="any",   # Stratum inyecta el modelo configurado en el dashboard
    messages=[{"role": "user", "content": prompt}]
)
```

---

## Providers soportados

| Provider | Estado | Notas |
|----------|--------|-------|
| OpenAI | ✅ Activo | GPT-4o, GPT-4o-mini, o3 |
| Anthropic | ✅ Activo | Claude Sonnet, Haiku, Opus |
| Google Gemini | ✅ Activo | Gemini 3.6 Flash, 3.5 Flash, 3.5 Flash-Lite |
| Bonsai 8B (local) | ✅ Via Ollama | PrismML ternario cuantizado, ~1 GB, on-premise |
| Ollama (local) | ✅ Via Ollama | Cualquier modelo local |
| Groq | 🔧 Configurable | Agregar API key para habilitar |
| Azure OpenAI | 🔧 Configurable | Agregar endpoint y API key |
| AWS Bedrock | 🔜 Planificado | Requiere AWS Sig V4 |

Agregar un provider es una operación de datos — sin cambios de código. Ir a `/providers` en el dashboard.

---

## Modelo de acceso al dashboard

- **Equipo Engineering** → dashboard completo (`/`)
- **Otros equipos** → redirigidos al Playground (`/prompt`)
- **Playground** → siempre accesible, tiene su propio auth

Para cambiar qué equipos tienen acceso al dashboard, editá `DASHBOARD_TEAMS` en `dashboard/middleware.js`.

---

## Compresión de contexto

Cuando está habilitada en el Playground, Stratum usa un LLM local (Ollama `llama3.2:1b`) para resumir los mensajes más viejos antes de enviar al proveedor. Reduce el gasto en tokens en conversaciones largas.

Habilitá haciendo click en el toggle **⬡ Compress** en el header del Playground. El panel de compresión muestra los tokens ahorrados y una cuenta regresiva a la próxima compresión.

---

## Variables de entorno

| Variable | Requerida | Descripción |
|----------|----------|-------------|
| `POSTGRES_PASSWORD` | ✅ | Password de PostgreSQL |
| `OPENAI_API_KEY` | ✅ (si usás OpenAI) | API key de OpenAI |
| `ANTHROPIC_API_KEY` | ✅ (si usás Anthropic) | API key de Anthropic |
| `GEMINI_API_KEY` | Si usás Gemini | API key de Google AI Studio |
| `NEXT_PUBLIC_GATEWAY_URL` | ✅ | URL del gateway visible en el browser |
| `GATEWAY_INTERNAL_URL` | ✅ en Docker | URL interna del gateway (`http://proxy:8080`) |
| `SESSION_SECRET` | ✅ | Secret para firmar sesiones del dashboard |
| `JWT_SECRET` | Recomendado | Secret para firmar JWTs del proxy |
| `OLLAMA_API_KEY` | Si usás Ollama | Cualquier valor (Ollama no requiere auth) |
| `OLLAMA_MODEL` | Si usás compresión | Modelo de compresión (default: `llama3.2:1b`) |
| `SECURE_COOKIES` | En producción con HTTPS | Setear `true` para habilitar flag secure en cookies |

---

## Contribuir

Las contribuciones son bienvenidas. Por favor abrí un issue antes de hacer cambios significativos.

---

## Licencia

MIT — libre para usar, modificar y distribuir. El core open-source siempre va a ser gratuito.
