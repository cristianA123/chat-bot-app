# Plan de Implementacion: WhatsApp AI Sales Agent SaaS

> Objetivo: Plataforma multi-tenant donde cada empresa cliente tiene su propio agente de IA
> configurado para responder WhatsApp, cerrar ventas y escalar a humanos cuando sea necesario.

---

## Resumen de decisiones de arquitectura

| Decision | Eleccion | Por que |
|---|---|---|
| Estructura de repo | Monorepo (1 repo, 2 paquetes) | Un solo `git clone`, un solo Docker Compose, mas facil para un solo dev |
| Backend | Python + FastAPI + LangGraph | Mejor ecosistema de IA, LangGraph es el estandar para agentes con estado |
| Frontend | Next.js 14 + Tailwind + shadcn/ui | Dashboard con SSR, facil de desplegar en Vercel |
| Base de datos | PostgreSQL | Conversaciones, tenants, configuraciones |
| Base de conocimiento | ChromaDB | RAG por cliente, coleccion separada por tenant |
| Agente IA | GPT-4o / Claude Sonnet | LLM configurable por tenant |
| Workers async | ARQ + Redis | Mas simple que Celery, el webhook debe responder en menos de 3 segundos |
| WhatsApp QA | Twilio Sandbox | Gratis, aprobacion inmediata, sin documentos |
| WhatsApp produccion | Gupshup o Meta Cloud API | Mas barato a escala, migrar cambiando solo credenciales |
| Multi-tenancy | Row-level con `tenant_id` | Sin schemas separados por tenant, mas simple para empezar |

---

## Estructura de carpetas del proyecto

```
whatsapp-saas/
├── backend/
│   ├── app/
│   │   ├── main.py
│   │   ├── api/
│   │   │   ├── deps.py
│   │   │   ├── auth.py
│   │   │   ├── tenants.py
│   │   │   ├── conversations.py
│   │   │   ├── knowledge.py
│   │   │   └── analytics.py
│   │   ├── webhooks/
│   │   │   ├── router.py
│   │   │   └── dispatcher.py
│   │   ├── providers/
│   │   │   ├── base.py          ← contrato comun para todos los proveedores
│   │   │   ├── twilio.py
│   │   │   ├── gupshup.py
│   │   │   ├── meta.py
│   │   │   └── factory.py
│   │   ├── agents/
│   │   │   ├── graph.py
│   │   │   ├── state.py
│   │   │   ├── nodes.py
│   │   │   ├── tools.py
│   │   │   ├── prompts.py
│   │   │   └── runner.py
│   │   ├── rag/
│   │   │   ├── client.py
│   │   │   ├── ingestion.py
│   │   │   └── retriever.py
│   │   ├── conversations/
│   │   │   ├── models.py
│   │   │   ├── crud.py
│   │   │   └── schemas.py
│   │   ├── tenants/
│   │   │   ├── models.py
│   │   │   ├── crud.py
│   │   │   └── schemas.py
│   │   └── core/
│   │       ├── config.py
│   │       ├── database.py
│   │       ├── security.py
│   │       └── exceptions.py
│   ├── workers/
│   │   ├── main.py
│   │   └── tasks/
│   │       ├── process_message.py
│   │       └── ingest_document.py
│   ├── migrations/
│   │   └── versions/
│   ├── tests/
│   ├── alembic.ini
│   ├── pyproject.toml
│   └── Dockerfile
├── frontend/
│   ├── src/
│   │   ├── app/
│   │   │   ├── (auth)/login/page.tsx
│   │   │   ├── (dashboard)/
│   │   │   │   ├── layout.tsx
│   │   │   │   ├── page.tsx
│   │   │   │   ├── conversations/
│   │   │   │   │   ├── page.tsx
│   │   │   │   │   └── [id]/page.tsx
│   │   │   │   ├── knowledge-base/page.tsx
│   │   │   │   ├── agent-config/page.tsx
│   │   │   │   └── settings/page.tsx
│   │   ├── components/
│   │   │   ├── ui/
│   │   │   ├── conversations/
│   │   │   ├── knowledge/
│   │   │   └── dashboard/
│   │   ├── lib/
│   │   │   ├── api.ts
│   │   │   └── auth.ts
│   │   └── types/index.ts
│   ├── package.json
│   └── Dockerfile
├── infra/
│   ├── docker-compose.yml
│   ├── docker-compose.prod.yml
│   └── nginx/nginx.conf
├── scripts/
│   ├── seed_dev_tenant.py
│   └── create_superadmin.py
└── .env.example
```

---

## Base de datos: tablas y esquema completo

```sql
-- Empresa cliente
CREATE TABLE tenants (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name        VARCHAR(255) NOT NULL,
  slug        VARCHAR(100) UNIQUE NOT NULL,
  plan        VARCHAR(50) DEFAULT 'trial',
  is_active   BOOLEAN DEFAULT true,
  created_at  TIMESTAMPTZ DEFAULT now()
);

-- Configuracion del agente por tenant
CREATE TABLE tenant_configs (
  id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id         UUID UNIQUE REFERENCES tenants(id),
  whatsapp_number   VARCHAR(30),              -- E.164: +521234567890
  provider          VARCHAR(30) DEFAULT 'twilio', -- 'twilio' | 'gupshup' | 'meta'
  provider_config   JSONB,                    -- API keys encriptadas con Fernet
  agent_name        VARCHAR(100) DEFAULT 'Asistente',
  agent_persona     VARCHAR(50) DEFAULT 'friendly',
  system_prompt     TEXT,
  language          VARCHAR(10) DEFAULT 'es',
  handoff_keywords  TEXT[],                   -- palabras que escalan a humano
  llm_model         VARCHAR(50) DEFAULT 'gpt-4o',
  updated_at        TIMESTAMPTZ DEFAULT now()
);

-- Contactos (clientes del tenant)
CREATE TABLE contacts (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id     UUID REFERENCES tenants(id),
  phone_number  VARCHAR(30) NOT NULL,
  display_name  VARCHAR(255),
  metadata      JSONB,                        -- datos capturados del lead
  created_at    TIMESTAMPTZ DEFAULT now(),
  UNIQUE (tenant_id, phone_number)
);

-- Hilo de conversacion
CREATE TABLE conversations (
  id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id             UUID REFERENCES tenants(id),
  contact_id            UUID REFERENCES contacts(id),
  status                VARCHAR(30) DEFAULT 'active', -- active | resolved | handed_off
  langgraph_thread_id   VARCHAR(100),                 -- clave para el checkpointer
  started_at            TIMESTAMPTZ DEFAULT now(),
  last_message_at       TIMESTAMPTZ,
  resolved_at           TIMESTAMPTZ
);

-- Mensajes individuales
CREATE TABLE messages (
  id                   UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  conversation_id      UUID REFERENCES conversations(id),
  tenant_id            UUID REFERENCES tenants(id),  -- desnormalizado para queries rapidas
  role                 VARCHAR(20),  -- user | assistant | system | tool
  content              TEXT,
  provider_message_id  VARCHAR(255),                 -- ID externo de Twilio/Meta
  metadata             JSONB,                        -- adjuntos, tool calls
  sent_at              TIMESTAMPTZ DEFAULT now()
);

-- Documentos subidos por el tenant (catalogo, FAQ, etc.)
CREATE TABLE knowledge_documents (
  id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id      UUID REFERENCES tenants(id),
  filename       VARCHAR(500),
  file_type      VARCHAR(50),
  status         VARCHAR(30) DEFAULT 'pending', -- pending | processing | ready | failed
  chunk_count    INT,
  error_message  TEXT,
  uploaded_at    TIMESTAMPTZ DEFAULT now(),
  processed_at   TIMESTAMPTZ
);

-- Usuarios con acceso al dashboard
CREATE TABLE users (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id        UUID REFERENCES tenants(id),
  email            VARCHAR(255) UNIQUE NOT NULL,
  hashed_password  VARCHAR(255),
  role             VARCHAR(30) DEFAULT 'admin',  -- admin | viewer
  created_at       TIMESTAMPTZ DEFAULT now()
);

-- Indices
CREATE INDEX ON conversations(tenant_id);
CREATE INDEX ON messages(conversation_id);
CREATE INDEX ON messages(tenant_id);
CREATE INDEX ON contacts(tenant_id);
CREATE INDEX ON knowledge_documents(tenant_id);
```

---

## Flujo principal: como llega un mensaje y el agente responde

```
1. Cliente escribe en WhatsApp
2. Twilio/Meta recibe el mensaje
3. Twilio/Meta llama POST /webhook/inbound en tu servidor
4. El webhook valida la firma HMAC del proveedor
5. El webhook parsea el payload con el Provider correcto → InboundMessage
6. El webhook busca el tenant por numero de telefono destino
7. El webhook encola tarea process_message en ARQ (async)
8. El webhook responde 200 OK inmediatamente (< 1 segundo)
9. El worker ARQ toma la tarea:
   a. Carga o crea la conversacion en PostgreSQL
   b. Guarda el mensaje del usuario
   c. Carga el LangGraph thread (con historial del checkpointer)
   d. Ejecuta el agente:
      - LLM decide si responder directo o buscar en la base de conocimiento
      - Si busca: RAG en ChromaDB con coleccion del tenant
      - LLM genera respuesta con el contexto
      - Si detecta keyword de handoff: cambia status a handed_off
   e. Guarda la respuesta del agente en messages
   f. Llama al Provider para enviar el mensaje de vuelta
```

---

## Paso a paso de implementacion

### PASO 1 — Infraestructura base
**Objetivo:** Entorno local funcionando con un solo comando.

**Tareas:**
- [ ] Crear el repositorio con la estructura de carpetas completa
- [ ] Crear `infra/docker-compose.yml` con los servicios:
  - `postgres` (imagen: postgres:16)
  - `redis` (imagen: redis:7-alpine)
  - `chromadb` (imagen: chromadb/chroma)
  - `backend` (FastAPI, hot-reload)
  - `worker` (ARQ worker)
  - `frontend` (Next.js dev server)
- [ ] Crear `.env.example` con todas las variables necesarias:
  ```
  DATABASE_URL=postgresql+asyncpg://user:pass@postgres:5432/whatsapp_saas
  REDIS_URL=redis://redis:6379
  CHROMADB_HOST=chromadb
  CHROMADB_PORT=8000
  SECRET_KEY=...
  OPENAI_API_KEY=...
  TWILIO_ACCOUNT_SID=...
  TWILIO_AUTH_TOKEN=...
  TWILIO_WHATSAPP_NUMBER=whatsapp:+14155238886
  ```
- [ ] Crear `backend/app/core/config.py` con Pydantic BaseSettings
- [ ] Crear `backend/app/core/database.py` con SQLAlchemy async engine
- [ ] Configurar Alembic para migraciones

**Verificacion:** `docker-compose up` levanta todos los servicios sin errores.

---

### PASO 2 — Modelos y migracion inicial
**Objetivo:** Todas las tablas creadas en PostgreSQL con una migracion de Alembic.

**Tareas:**
- [ ] Crear modelos SQLAlchemy en `tenants/models.py` y `conversations/models.py`
- [ ] Crear primera migracion con Alembic: `alembic revision --autogenerate -m "initial"`
- [ ] Aplicar migracion: `alembic upgrade head`
- [ ] Crear `scripts/seed_dev_tenant.py`:
  - Crea un tenant de prueba con slug `demo`
  - Asigna numero de Twilio Sandbox
  - Crea un usuario admin con email/password para el dashboard

**Verificacion:** Script de seed corre sin errores, las tablas aparecen en PostgreSQL.

---

### PASO 3 — Capa de providers (abstraccion WhatsApp)
**Objetivo:** El codigo nunca habla directamente con Twilio o Meta, solo con la abstraccion.

**Tareas:**
- [ ] Crear `providers/base.py`:
  ```python
  from dataclasses import dataclass
  from abc import ABC, abstractmethod

  @dataclass
  class InboundMessage:
      tenant_phone: str        # numero que recibio el mensaje
      from_phone: str          # numero del cliente
      body: str
      provider_message_id: str
      media_urls: list[str]
      raw: dict                # payload original para debugging

  class BaseProvider(ABC):
      @abstractmethod
      def parse_inbound(self, raw: dict) -> InboundMessage: ...

      @abstractmethod
      async def send_message(self, to: str, body: str) -> str: ...

      @abstractmethod
      def validate_signature(self, request) -> bool: ...
  ```
- [ ] Crear `providers/twilio.py` implementando `BaseProvider`
- [ ] Crear `providers/factory.py`: `get_provider(name: str) -> BaseProvider`

**Verificacion:** Enviar un mensaje desde el Sandbox de Twilio y ver el `InboundMessage` en los logs.

---

### PASO 4 — Webhook de entrada
**Objetivo:** Recibir mensajes de WhatsApp y enrutarlos al tenant correcto.

**Tareas:**
- [ ] Crear `webhooks/router.py` con el endpoint `POST /webhook/inbound`:
  ```
  1. Detectar proveedor por headers
  2. Validar firma HMAC
  3. Parsear payload → InboundMessage
  4. Buscar tenant por InboundMessage.tenant_phone
  5. Si no existe el tenant: return 200 (silencioso)
  6. Encolar tarea ARQ: process_message(tenant_id, inbound_msg)
  7. return 200 OK
  ```
- [ ] Agregar endpoint `GET /webhook/inbound` para el challenge de verificacion de Meta
- [ ] Exponer el webhook con ngrok para que Twilio pueda llamarlo localmente:
  `ngrok http 8000`

**Verificacion:** Mensaje desde WhatsApp → log del webhook → tarea encolada en Redis.

---

### PASO 5 — Agente basico con LangGraph
**Objetivo:** El agente recibe un mensaje y responde usando el LLM. Sin RAG todavia.

**Tareas:**
- [ ] Crear `agents/state.py`:
  ```python
  from typing import TypedDict, Annotated
  from langgraph.graph.message import add_messages

  class AgentState(TypedDict):
      messages: Annotated[list, add_messages]
      tenant_id: str
      contact_phone: str
      conversation_id: str
      status: str   # active | handed_off
  ```
- [ ] Crear `agents/prompts.py`: funcion que construye el system prompt con el config del tenant:
  ```
  Eres {agent_name}, asistente de {business_name}.
  Tu tono es {persona}. Respondes en {language}.
  Si el cliente menciona: {handoff_keywords}, escala al equipo humano.
  ```
- [ ] Crear `agents/graph.py`: LangGraph con un solo nodo `call_llm`
- [ ] Configurar `PostgresSaver` como checkpointer de LangGraph
- [ ] Crear `agents/runner.py`: funcion `run_agent(tenant_config, conversation_id, message)`
- [ ] Crear `workers/tasks/process_message.py`:
  - Crea o carga la conversacion
  - Guarda el mensaje entrante en `messages`
  - Llama `run_agent`
  - Guarda la respuesta en `messages`
  - Llama `provider.send_message` para enviar la respuesta

**Verificacion:** Escribir en WhatsApp → el agente responde con GPT-4o.

---

### PASO 6 — Worker async con ARQ
**Objetivo:** El webhook responde en menos de 1 segundo, el agente corre en background.

**Tareas:**
- [ ] Instalar ARQ: `pip install arq`
- [ ] Crear `workers/main.py` con la configuracion del worker ARQ
- [ ] Mover la llamada al agente del webhook al worker
- [ ] Agregar el servicio `worker` al `docker-compose.yml`

**Verificacion:** El webhook retorna 200 inmediatamente, la respuesta llega al WhatsApp 3-10 segundos despues.

---

### PASO 7 — RAG: base de conocimiento por tenant
**Objetivo:** El agente puede buscar en el catalogo o FAQ del cliente antes de responder.

**Tareas:**
- [ ] Crear `rag/client.py`: conexion singleton a ChromaDB
- [ ] Crear `rag/ingestion.py`:
  - Parsear PDF con `pypdf`
  - Parsear DOCX con `python-docx`
  - Parsear CSV con pandas
  - Dividir en chunks de ~500 tokens con `langchain.text_splitter`
  - Generar embeddings con OpenAI `text-embedding-3-small`
  - Guardar en ChromaDB en coleccion `tenant_{tenant_id}`
- [ ] Crear `rag/retriever.py`: `query(tenant_id, text, k=5) -> list[Chunk]`
- [ ] Crear `workers/tasks/ingest_document.py`: tarea async que procesa un documento
- [ ] Agregar herramienta RAG al agente en `agents/tools.py`:
  ```python
  @tool
  def buscar_en_catalogo(query: str) -> str:
      """Busca informacion sobre productos, precios o servicios del negocio."""
      chunks = retriever.query(tenant_id, query)
      return "\n".join(chunks)
  ```
- [ ] Conectar la herramienta al grafo de LangGraph con un nodo condicional

**Verificacion:** Subir un PDF con precios → preguntar por un precio en WhatsApp → el agente responde con el dato correcto.

---

### PASO 8 — Persistencia completa de conversaciones
**Objetivo:** Todas las conversaciones quedan guardadas y el agente recuerda el historial.

**Tareas:**
- [ ] Completar `conversations/crud.py`:
  - `get_or_create_contact(tenant_id, phone)`
  - `get_or_create_conversation(tenant_id, contact_id)`
  - `create_message(conversation_id, role, content, provider_message_id)`
  - `update_conversation_status(conversation_id, status)`
- [ ] Verificar que `langgraph_thread_id` se genera en la primera interaccion y se reutiliza en las siguientes
- [ ] Agregar logica de handoff: si el agente detecta las `handoff_keywords`, cambiar `conversation.status = 'handed_off'` y notificar

**Verificacion:** Cortar Docker, reiniciar, escribir de nuevo → el agente recuerda la conversacion anterior.

---

### PASO 9 — API REST para el dashboard
**Objetivo:** Endpoints que el frontend Next.js consume.

**Endpoints a implementar:**

```
POST   /api/auth/login                     → JWT token
GET    /api/auth/me                         → usuario actual

GET    /api/conversations                   → lista paginada
GET    /api/conversations/{id}              → detalle con mensajes
PATCH  /api/conversations/{id}/status       → resolver o tomar control

GET    /api/knowledge-base                  → lista de documentos
POST   /api/knowledge-base/upload           → subir archivo
DELETE /api/knowledge-base/{id}             → eliminar documento

GET    /api/agent-config                    → config actual del tenant
PATCH  /api/agent-config                    → actualizar config

GET    /api/analytics/summary               → metricas del dashboard
```

**Tareas:**
- [ ] Crear `api/deps.py`: `get_current_user`, `get_db`, `get_tenant` como dependencias de FastAPI
- [ ] Implementar autenticacion JWT en `core/security.py`
- [ ] Implementar cada router listado arriba
- [ ] Agregar CORS para permitir peticiones del frontend

---

### PASO 10 — Dashboard Next.js
**Objetivo:** Interface para que cada cliente gestione su agente.

**Paginas a construir en orden:**

1. **`/login`** — formulario email + password, guarda JWT en cookie httpOnly
2. **`/dashboard`** — 4 metricas (conversaciones hoy, leads, handoffs, tiempo de respuesta) + grafico de 7 dias
3. **`/conversations`** — tabla paginada con filtro por status
4. **`/conversations/[id]`** — chat UI de solo lectura, boton "Tomar control"
5. **`/knowledge-base`** — drag & drop para subir archivos, tabla con estado de procesamiento
6. **`/agent-config`** — formulario con nombre, tono, prompt, keywords de escalacion, modelo LLM
7. **`/settings`** — numero WhatsApp, rotar API keys

**Tareas:**
- [ ] Inicializar Next.js 14 con App Router y Tailwind
- [ ] Instalar shadcn/ui: `npx shadcn@latest init`
- [ ] Crear `lib/api.ts`: wrapper tipado de fetch con manejo de errores y JWT
- [ ] Crear `types/index.ts`: tipos TypeScript que reflejan los schemas del backend
- [ ] Construir cada pagina en el orden listado

---

### PASO 11 — Segundo proveedor (Gupshup o Meta directo)
**Objetivo:** Migrar a produccion cambiando solo las credenciales del tenant.

**Tareas:**
- [ ] Crear `providers/gupshup.py` implementando `BaseProvider`
- [ ] Crear `providers/meta.py` implementando `BaseProvider`
- [ ] Actualizar `factory.py` para resolver el proveedor desde `tenant_configs.provider`
- [ ] Documentar el proceso de registro en Gupshup y Meta for Developers

**Verificacion:** Cambiar `provider` del tenant de `twilio` a `gupshup` en la DB → los mensajes siguen funcionando sin tocar el codigo del agente.

---

## Tecnologias y dependencias

### Backend (Python)
```toml
[tool.poetry.dependencies]
python = "^3.11"
fastapi = "^0.115"
uvicorn = "^0.30"
sqlalchemy = {extras = ["asyncio"], version = "^2.0"}
asyncpg = "^0.29"
alembic = "^1.13"
pydantic-settings = "^2.0"
langgraph = "^0.4"
langchain = "^0.3"
langchain-openai = "^0.2"
langgraph-checkpoint-postgres = "^2.0"
chromadb = "^0.5"
arq = "^0.26"
redis = "^5.0"
twilio = "^9.0"
pypdf = "^4.0"
python-docx = "^1.1"
cryptography = "^42.0"
python-jose = "^3.3"
passlib = "^1.7"
python-multipart = "^0.0.9"
```

### Frontend (Node.js)
```json
{
  "dependencies": {
    "next": "14",
    "react": "^18",
    "react-dom": "^18",
    "tailwindcss": "^3",
    "@shadcn/ui": "latest",
    "recharts": "^2",
    "react-dropzone": "^14",
    "jose": "^5",
    "swr": "^2"
  }
}
```

---

## Variables de entorno (.env.example)

```bash
# Base de datos
DATABASE_URL=postgresql+asyncpg://postgres:postgres@localhost:5432/whatsapp_saas
REDIS_URL=redis://localhost:6379

# ChromaDB
CHROMADB_HOST=localhost
CHROMADB_PORT=8000

# Seguridad
SECRET_KEY=genera-una-clave-segura-de-32-chars
ACCESS_TOKEN_EXPIRE_MINUTES=60

# LLM
OPENAI_API_KEY=sk-...

# Twilio (desarrollo / QA)
TWILIO_ACCOUNT_SID=ACxxx
TWILIO_AUTH_TOKEN=xxx
TWILIO_WHATSAPP_NUMBER=whatsapp:+14155238886

# Gupshup (produccion - agregar cuando tengas cliente real)
GUPSHUP_API_KEY=
GUPSHUP_APP_ID=

# Meta Cloud API (produccion directa - opcional)
META_PHONE_NUMBER_ID=
META_ACCESS_TOKEN=
META_VERIFY_TOKEN=
```

---

## Costos estimados en produccion

### Por cliente (mensual)
| Item | Costo estimado |
|---|---|
| VPS o Railway (backend + worker) | $10 - $20 USD |
| PostgreSQL managed (Supabase/Neon) | $0 - $25 USD |
| ChromaDB (self-hosted en VPS) | Incluido en VPS |
| Redis (Upstash) | $0 - $10 USD |
| OpenAI GPT-4o (1000 conversaciones/mes) | ~$15 - $40 USD |
| Gupshup fee | $0.001/mensaje |
| Meta mensaje de servicio (24h window) | Gratis |
| **Total infraestructura** | **~$25 - $95 USD/mes** |

### Precio sugerido al cliente
- Plan Starter: $99 USD/mes (hasta 500 conversaciones)
- Plan Growth: $199 USD/mes (hasta 2,000 conversaciones)
- Plan Pro: $399 USD/mes (ilimitado + analiticas avanzadas)

**Margen estimado:** 60-75% desde el primer cliente.

---

## Hoja de ruta de versiones

### v0.1 — MVP interno (Pasos 1-8)
- [ ] Agente responde en WhatsApp con RAG
- [ ] Un tenant hardcodeado para demos

### v0.2 — Dashboard basico (Pasos 9-10)
- [ ] El cliente ve sus conversaciones
- [ ] El cliente sube su catalogo
- [ ] El cliente configura el agente

### v0.3 — Multi-tenant real
- [ ] Registro de nuevos clientes
- [ ] Onboarding automatizado
- [ ] Cada cliente tiene su numero configurado

### v0.4 — Produccion
- [ ] Migracion a Gupshup o Meta directo
- [ ] Facturacion (Stripe)
- [ ] Metricas avanzadas

### v1.0 — Producto maduro
- [ ] Multi-idioma
- [ ] Campanas outbound (mensajes proactivos)
- [ ] Integracion con CRMs (HubSpot, Pipedrive)
- [ ] API publica para integraciones de terceros
