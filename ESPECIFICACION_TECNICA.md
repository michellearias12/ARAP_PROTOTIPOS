# Especificación técnica — Sistema de digitalización de concesiones acuícolas
**Autoridad de los Recursos Acuáticos de Panamá (ARAP)**
Versión 1.0 · Mayo 2026 · Estado: Borrador para revisión

---

## Índice
1. [Contexto y alcance](#1-contexto-y-alcance)
2. [Actores y roles](#2-actores-y-roles)
3. [Fases del sistema](#3-fases-del-sistema)
4. [User stories](#4-user-stories)
5. [Contratos de API — integraciones externas](#5-contratos-de-api--integraciones-externas)
6. [Modelo de datos](#6-modelo-de-datos)
7. [Estados del expediente](#7-estados-del-expediente)
8. [Reglas de negocio críticas](#8-reglas-de-negocio-críticas)
9. [Notificaciones automáticas](#9-notificaciones-automáticas)
10. [Control de acceso (RBAC)](#10-control-de-acceso-rbac)
11. [Requisitos no funcionales](#11-requisitos-no-funcionales)
12. [Stack tecnológico recomendado](#12-stack-tecnológico-recomendado)
13. [Prototipos de referencia](#13-prototipos-de-referencia)

---

## 1. Contexto y alcance

### Trámite
Concesión para fines de acuicultura — Decreto Ejecutivo No. 11 de 5 de febrero de 1997, Ley 44 de 2006, Ley 58 de 1995.

### Institución responsable
Autoridad de los Recursos Acuáticos de Panamá (ARAP)  
Edificio La Riviera, Av. Justo Arosemena y Calle 45, Bella Vista, Ciudad de Panamá.

### Problema que resuelve
El trámite actual es 100% presencial y en papel. El ciudadano debe presentarse físicamente con 18+ documentos, los cuales son revisados manualmente por múltiples instituciones de forma secuencial. El tiempo promedio actual es de 45+ días hábiles con alta tasa de devoluciones por documentación incompleta.

### Alcance del piloto (MVP)
| Fase | Descripción | Incluido en MVP |
|---|---|---|
| Fase 1 | Portal de solicitudes ciudadano | ✅ |
| Fase 2 | Validación automática y expediente digital | ✅ |
| Fase 3 | Panel interno funcionarios ARAP | ✅ |
| Fase 3 | Portal interinstitucional (IGNTG, MiAmbiente) | ✅ |
| Fase 4 | Inspecciones con acta digital | ❌ v2 |
| Fase 5 | Firma digital y resolución electrónica | ❌ v2 |

### Fuera de alcance (v1)
- Módulo de mapa geoespacial (requiere integración con GeoServer/IGNTG)
- App móvil nativa para inspectores
- Integración con sistema de archivo físico existente
- Módulo de pagos de georeferenciación (se abona directamente al IGNTG)

---

## 2. Actores y roles

| Rol | Descripción | Institución |
|---|---|---|
| `ciudadano` | Solicitante o apoderado legal que presenta la solicitud | Externa |
| `analista_arap` | Funcionario técnico que revisa expedientes | ARAP |
| `inspector_arap` | Funcionario que realiza inspecciones de campo | ARAP |
| `director_arap` | Firma resoluciones de concesión | ARAP |
| `revisor_igntg` | Revisa coordenadas y polígonos geográficos | IGNTG |
| `revisor_ambiente` | Verifica compatibilidad ambiental con EIA | MiAmbiente |
| `admin` | Administra usuarios, roles y configuración del sistema | ARAP |

---

## 3. Fases del sistema

### Fase 1 — Portal de solicitudes
El ciudadano completa un formulario de 5 pasos y carga sus documentos.

**Pasos del formulario:**
1. Datos del solicitante (tipo de persona, cédula/RUC, representante legal, apoderado)
2. Datos del proyecto (especie, tipo de acuicultura, área, consumo de agua, coordenadas geodésicas)
3. Carga de documentos (12 documentos requeridos)
4. Pago (B/. 450.00 mínimo — tarjeta, ACH o referencia de caja)
5. Resumen y envío

**Al enviar:** el sistema genera un número de expediente (`ARAP-ACU-YYYY-NNNNN`) y dispara el flujo de la Fase 2 automáticamente.

---

### Fase 2 — Validación automática
El sistema ejecuta 6 verificaciones interinstitucionales en paralelo (asíncronas) sin intervención humana.

**Verificaciones:**
1. Cédula del representante legal → Tribunal Electoral
2. Persona jurídica activa → Registro Público
3. Paz y Salvo Nacional vigente → DGI / MEF
4. Paz y Salvo ARAP → Sistema interno ARAP
5. EIA aprobado → MiAmbiente
6. Idoneidad del apoderado → Órgano Judicial

**Resultado posible:**
- Todo correcto → expediente pasa a Fase 3 automáticamente
- Fallo en verificación → notificación automática al ciudadano con lista de subsanaciones; expediente queda en estado `subsanacion_pendiente`

---

### Fase 3 — Revisión técnica interinstitucional
Tres revisiones corren **en paralelo** desde que el expediente entra a esta fase:

| Revisión | Responsable | Documentos visibles | Acción |
|---|---|---|---|
| Técnica documental | Analista ARAP | Todos | Aprobar / Subsanar |
| Geográfica | Revisor IGNTG | Planos, coordenadas, fotos aéreas | Aprobar / Devolver |
| Ambiental | Revisor MiAmbiente | EIA, resolución EIA, plan de desarrollo | Confirmar / Solicitar aclaración |

El expediente avanza a Fase 4 (Inspección) solo cuando las **tres revisiones** tienen estado `aprobado`.

---

## 4. User stories

### Ciudadano / Apoderado legal

```
US-01
Como apoderado legal
Quiero completar la solicitud de concesión en línea
Para no tener que presentarme físicamente a las oficinas de la ARAP

Criterios de aceptación:
- El formulario valida campos obligatorios antes de avanzar de paso
- Puedo cargar PDFs de hasta 10 MB por documento
- Recibo confirmación por correo con número de expediente al enviar
- Puedo retomar el formulario si lo dejo incompleto (borrador guardado)
```

```
US-02
Como solicitante
Quiero ver el estado de mi expediente en tiempo real
Para saber en qué etapa está sin tener que llamar a la ARAP

Criterios de aceptación:
- Veo una línea de tiempo con cada etapa y su estado (completado / en curso / pendiente)
- Veo el estado de revisión de cada institución (ARAP, IGNTG, MiAmbiente) sin detalle interno
- Recibo notificación por correo y SMS cuando el estado cambia
- No puedo ver comentarios técnicos internos entre funcionarios
```

```
US-03
Como apoderado legal
Quiero recibir una lista clara de documentos faltantes cuando haya una subsanación
Para corregir exactamente lo que se necesita sin ambigüedad

Criterios de aceptación:
- El correo de subsanación lista cada documento faltante con su base legal
- Puedo cargar los documentos faltantes desde el portal sin reenviar toda la solicitud
- Tengo un plazo de 10 días hábiles para subsanar
- Si no subsano en el plazo, el expediente queda en estado `subsanacion_vencida`
```

---

### Analista ARAP

```
US-04
Como analista ARAP
Quiero ver todos los expedientes asignados a mí con su estado actual
Para priorizar mi trabajo y no perder plazos legales

Criterios de aceptación:
- La bandeja muestra: número de expediente, solicitante, estado, área, días en trámite
- Puedo filtrar por estado (nuevo, en revisión, inspección, subsanar, aprobado)
- Los expedientes que vencen en 5 días o menos se marcan visualmente en amarillo
- Los expedientes vencidos se marcan en rojo
```

```
US-05
Como analista ARAP
Quiero revisar todos los documentos de un expediente desde una sola pantalla
Para no tener que pedir archivos por correo

Criterios de aceptación:
- Puedo abrir cada PDF desde el detalle del expediente
- Veo el resultado de las verificaciones automáticas de la Fase 2
- Puedo dejar notas internas que solo ven los funcionarios ARAP
- Puedo solicitar subsanación con un formulario que especifica qué corregir
```

```
US-06
Como analista ARAP
Quiero aprobar o devolver la revisión técnica de un expediente
Para que el flujo avance sin depender de correos o llamadas

Criterios de aceptación:
- Al aprobar, el sistema notifica automáticamente a IGNTG y MiAmbiente
- Al devolver, debo completar un campo de observaciones obligatorio
- Cada acción queda registrada en el log del expediente con fecha, hora y usuario
```

---

### Revisor IGNTG

```
US-07
Como revisor del IGNTG
Quiero ver solo los documentos geográficos del expediente
Para hacer mi revisión sin acceder a información que no me corresponde

Criterios de aceptación:
- Solo veo: planos del área, planos del proyecto, descripción del polígono, fotos aéreas
- No tengo acceso a documentos legales, financieros ni al EIA
- Veo las coordenadas geodésicas declaradas por el solicitante
- El sistema me indica el plazo límite para completar mi revisión
```

```
US-08
Como revisor del IGNTG
Quiero completar una lista de verificación geográfica estructurada
Para que mi aprobación quede documentada formalmente

Criterios de aceptación:
- La lista tiene 5 ítems con base legal en cada uno
- No puedo aprobar si hay ítems sin marcar
- Puedo agregar observaciones técnicas de texto libre
- Al aprobar, la ARAP recibe notificación automática
```

---

### Revisor MiAmbiente

```
US-09
Como revisora de MiAmbiente
Quiero confirmar que el plan de desarrollo es compatible con el EIA aprobado
Sin tener que re-evaluar el EIA desde cero

Criterios de aceptación:
- Solo veo: EIA aprobado, resolución EIA, plan de desarrollo del proyecto
- Veo el número de resolución EIA y su fecha de aprobación
- La lista de verificación tiene 4 ítems enfocados en compatibilidad
- Puedo solicitar aclaración al solicitante a través del sistema
```

---

## 5. Contratos de API — integraciones externas

> **Nota para desarrolladores:** Todas las integraciones se gestionan a través del API Gateway interno. Nunca se llama a APIs externas directamente desde el frontend. Las verificaciones de Fase 2 son asíncronas — se encolan en Redis y el worker las ejecuta en paralelo.

---

### 5.1 Tribunal Electoral — Verificación de cédula

```
Estado: Disponible · Convenio AIG activo
Tipo: REST / JSON
Auth: API Key (header X-API-Key)
Timeout: 3s
Retry: 2 intentos con backoff exponencial

Request:
GET https://api.tribunal-electoral.gob.pa/v1/cedula/{cedula}
Headers:
  X-API-Key: {api_key}

Response exitoso (200):
{
  "cedula": "8-742-3156",
  "nombre": "Carlos Alberto Núñez",
  "estado": "vigente",
  "tipo": "natural"
}

Response no encontrado (404):
{
  "error": "cedula_no_encontrada",
  "message": "La cédula indicada no existe en el registro"
}

Comportamiento en fallo:
- Si timeout o error 5xx: marcar verificación como `no_pudo_verificar`
- No bloquear el expediente — escalar a revisión manual del analista
```

---

### 5.2 Registro Público — Verificación de persona jurídica

```
Estado: Disponible · Convenio AIG activo
Tipo: REST / JSON
Auth: API Key
Timeout: 3s

Request:
GET https://api.registro-publico.gob.pa/v2/empresa?ruc={ruc}
Headers:
  X-API-Key: {api_key}

Response exitoso (200):
{
  "ruc": "155-709-1-2019",
  "razon_social": "Acuícola Veraguas S.A.",
  "estado": "activa",
  "folio": "234891",
  "directores": ["Carlos Núñez", "María Pérez"],
  "representante_legal": "Carlos Núñez"
}

Response empresa inactiva (200):
{
  "ruc": "...",
  "estado": "inactiva",
  "razon": "disuelta_voluntariamente"
}

Comportamiento en fallo:
- Si estado != "activa": marcar verificación como `fallida`, notificar al ciudadano
- Si timeout: marcar como `no_pudo_verificar`, escalar a analista
```

---

### 5.3 DGI / MEF — Paz y Salvo Nacional

```
Estado: Requiere convenio con MEF (estimado: 2-3 meses)
Tipo: REST (migración desde SOAP en curso)
Auth: Certificado digital institucional
Timeout: 5s

Request:
POST https://servicios.dgi.mef.gob.pa/v1/paz-y-salvo/consultar
Headers:
  Authorization: Bearer {token_certificado}
  Content-Type: application/json
Body:
{
  "tipo_identificacion": "cedula" | "ruc",
  "identificacion": "8-742-3156"
}

Response exitoso (200):
{
  "vigente": true,
  "fecha_vencimiento": "2026-12-31",
  "deuda_pendiente": 0.00,
  "numero_pys": "PYS-2026-089234"
}

Response con deuda (200):
{
  "vigente": false,
  "deuda_pendiente": 450.75,
  "concepto": "impuesto_sobre_renta"
}

Fallback mientras convenio no está activo:
- El campo EIA queda como `verificacion_manual`
- El analista ARAP debe verificar manualmente y marcar en el sistema
```

---

### 5.4 MiAmbiente — Verificación de EIA

```
Estado: API en desarrollo (pendiente)
Fallback: Verificación manual obligatoria
Tipo: REST / JSON (spec preliminar)
Auth: API Key
Timeout: 5s

Request (cuando esté disponible):
GET https://api.miambiente.gob.pa/v1/eia/{numero_resolucion}/estado
Headers:
  X-API-Key: {api_key}

Response esperado:
{
  "numero_resolucion": "DIA-2025-0341",
  "aprobado": true,
  "fecha_aprobacion": "2025-11-14",
  "titular": "Acuícola Veraguas S.A.",
  "vigente": true,
  "condicionantes": []
}

Implementación actual (fallback):
- El ciudadano carga la resolución en PDF
- El analista ARAP verifica manualmente y marca `eia_verificado_manual` en el expediente
- Cuando la API esté disponible, se activa la verificación automática sin cambios en el frontend
```

---

### 5.5 ARAP — Paz y Salvo interno

```
Estado: Disponible (sistema interno)
Tipo: Consulta directa a base de datos ARAP
No requiere API externa

Query:
SELECT vigente, fecha_vencimiento
FROM paz_y_salvos
WHERE identificacion = $1
  AND tipo = 'arap'
  AND vigente = true
  AND fecha_vencimiento >= CURRENT_DATE

Comportamiento:
- Si no hay registro vigente: `fallida` — notificar al ciudadano
- El ciudadano puede obtener el Paz y Salvo ARAP en ventanilla o por portal interno ARAP
```

---

### 5.6 Órgano Judicial — Idoneidad de abogados

```
Estado: Disponible · Requiere convenio (proceso simple)
Tipo: REST / JSON
Auth: API Key
Timeout: 3s

Request:
GET https://api.organojudicial.gob.pa/v1/abogados/{numero_idoneidad}
Headers:
  X-API-Key: {api_key}

Response exitoso (200):
{
  "numero_idoneidad": "12345-A",
  "nombre": "Carlos Alberto Núñez",
  "estado": "activo",
  "tipo_licencia": "abogado_litigante"
}
```

---

### 5.7 Tesorería Nacional — Pasarela de pagos

```
Estado: Disponible para instituciones del Estado
Tipo: REST / JSON
Auth: Client credentials OAuth 2.0
Timeout: 10s

Flujo de pago:
1. Frontend solicita al backend una sesión de pago
2. Backend crea orden en Tesorería y devuelve `payment_url`
3. Frontend redirige al ciudadano a `payment_url`
4. Tesorería notifica al backend via webhook cuando el pago es confirmado
5. Backend actualiza el expediente y habilita el envío del formulario

Webhook de confirmación:
POST /api/webhooks/tesoreria
{
  "orden_id": "ORD-2026-00891",
  "estado": "pagado",
  "monto": 450.00,
  "referencia_interna": "solicitud_draft_id",
  "fecha_pago": "2026-05-30T09:14:22Z"
}
```

---

## 6. Modelo de datos

### Tabla `expedientes`

```sql
CREATE TABLE expedientes (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  numero          VARCHAR(25) UNIQUE NOT NULL, -- ARAP-ACU-2026-00141
  estado          VARCHAR(30) NOT NULL,         -- ver tabla de estados
  tipo_persona    VARCHAR(20) NOT NULL,         -- natural | juridica
  identificacion  VARCHAR(20) NOT NULL,         -- cedula o RUC
  razon_social    VARCHAR(200),                 -- si es juridica
  rep_nombre      VARCHAR(100) NOT NULL,
  rep_apellido    VARCHAR(100) NOT NULL,
  rep_cedula      VARCHAR(20) NOT NULL,
  rep_email       VARCHAR(150) NOT NULL,
  rep_telefono    VARCHAR(20),
  idoneidad       VARCHAR(30) NOT NULL,
  proy_nombre     VARCHAR(200) NOT NULL,
  especie         VARCHAR(100) NOT NULL,
  tipo_acuicultura VARCHAR(30) NOT NULL,
  area_ha         DECIMAL(10,2) NOT NULL,
  agua_m3_dia     INTEGER NOT NULL,
  provincia       VARCHAR(50) NOT NULL,
  distrito        VARCHAR(50) NOT NULL,
  corregimiento   VARCHAR(50),
  coord_norte     VARCHAR(30) NOT NULL,
  coord_este      VARCHAR(30) NOT NULL,
  datum           VARCHAR(20) DEFAULT 'MAGNA-SIRGAS',
  descripcion_poligono TEXT NOT NULL,
  pago_estado     VARCHAR(20) DEFAULT 'pendiente',  -- pendiente | pagado
  pago_monto      DECIMAL(8,2) DEFAULT 450.00,
  pago_referencia VARCHAR(50),
  analista_id     UUID REFERENCES usuarios(id),
  fecha_recepcion TIMESTAMPTZ DEFAULT NOW(),
  fecha_limite    DATE,                         -- 45 días hábiles desde recepción
  created_at      TIMESTAMPTZ DEFAULT NOW(),
  updated_at      TIMESTAMPTZ DEFAULT NOW()
);
```

### Tabla `documentos`

```sql
CREATE TABLE documentos (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  expediente_id   UUID REFERENCES expedientes(id) ON DELETE CASCADE,
  tipo            VARCHAR(50) NOT NULL,  -- cedula | registro | pysn | pysarap | planos_area | ...
  nombre_archivo  VARCHAR(200) NOT NULL,
  storage_key     VARCHAR(500) NOT NULL, -- ruta en object storage
  hash_sha256     VARCHAR(64) NOT NULL,  -- integridad del archivo
  tamano_bytes    INTEGER NOT NULL,
  estado          VARCHAR(20) DEFAULT 'cargado', -- cargado | aceptado | rechazado
  subido_por      UUID REFERENCES usuarios(id),
  created_at      TIMESTAMPTZ DEFAULT NOW()
);
```

### Tabla `verificaciones`

```sql
CREATE TABLE verificaciones (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  expediente_id   UUID REFERENCES expedientes(id) ON DELETE CASCADE,
  tipo            VARCHAR(30) NOT NULL,   -- cedula | registro | pysn | pysarap | eia | idoneidad
  estado          VARCHAR(30) NOT NULL,   -- pendiente | ok | fallida | no_pudo_verificar | manual
  resultado_raw   JSONB,                  -- respuesta cruda de la API externa
  mensaje         VARCHAR(300),           -- mensaje legible para el ciudadano
  fuente          VARCHAR(100),           -- nombre de la API consultada
  ejecutado_at    TIMESTAMPTZ,
  created_at      TIMESTAMPTZ DEFAULT NOW()
);
```

### Tabla `revisiones_institucionales`

```sql
CREATE TABLE revisiones_institucionales (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  expediente_id   UUID REFERENCES expedientes(id) ON DELETE CASCADE,
  institucion     VARCHAR(20) NOT NULL,  -- arap | igntg | ambiente
  estado          VARCHAR(20) DEFAULT 'pendiente', -- pendiente | aprobado | devuelto
  checklist       JSONB,                 -- items marcados del checklist
  observaciones   TEXT,
  revisado_por    UUID REFERENCES usuarios(id),
  fecha_revision  TIMESTAMPTZ,
  created_at      TIMESTAMPTZ DEFAULT NOW()
);
```

### Tabla `usuarios`

```sql
CREATE TABLE usuarios (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  nombre          VARCHAR(100) NOT NULL,
  email           VARCHAR(150) UNIQUE NOT NULL,
  rol             VARCHAR(30) NOT NULL,  -- ver sección RBAC
  institucion     VARCHAR(20) NOT NULL,  -- arap | igntg | ambiente | ciudadano
  activo          BOOLEAN DEFAULT true,
  last_login      TIMESTAMPTZ,
  created_at      TIMESTAMPTZ DEFAULT NOW()
);
```

### Tabla `log_eventos`

```sql
CREATE TABLE log_eventos (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  expediente_id   UUID REFERENCES expedientes(id),
  usuario_id      UUID REFERENCES usuarios(id),
  tipo_evento     VARCHAR(50) NOT NULL,  -- estado_cambiado | documento_cargado | nota_agregada | ...
  detalle         JSONB,
  ip_origen       INET,
  created_at      TIMESTAMPTZ DEFAULT NOW()
);
```

### Tabla `notificaciones`

```sql
CREATE TABLE notificaciones (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  expediente_id   UUID REFERENCES expedientes(id),
  destinatario    VARCHAR(150) NOT NULL,
  canal           VARCHAR(10) NOT NULL,  -- email | sms
  tipo            VARCHAR(50) NOT NULL,  -- confirmacion | subsanacion | aprobacion | ...
  asunto          VARCHAR(200),
  cuerpo          TEXT NOT NULL,
  estado          VARCHAR(20) DEFAULT 'pendiente', -- pendiente | enviado | fallido
  enviado_at      TIMESTAMPTZ,
  created_at      TIMESTAMPTZ DEFAULT NOW()
);
```

---

## 7. Estados del expediente

```
borrador
  └─► pago_pendiente
        └─► recibido
              └─► verificacion_automatica
                    ├─► subsanacion_pendiente ─► recibido (re-ingreso tras subsanación)
                    └─► revision_tecnica
                          ├─► subsanacion_tecnica ─► revision_tecnica
                          └─► inspeccion_programada
                                └─► inspeccion_realizada
                                      └─► resolucion_pendiente
                                            ├─► aprobado
                                            └─► rechazado
```

### Transiciones válidas

| Estado origen | Estado destino | Quién puede hacer la transición |
|---|---|---|
| `borrador` | `pago_pendiente` | Sistema (al completar formulario) |
| `pago_pendiente` | `recibido` | Sistema (webhook Tesorería) |
| `recibido` | `verificacion_automatica` | Sistema (automático) |
| `verificacion_automatica` | `subsanacion_pendiente` | Sistema (si falla verificación) |
| `verificacion_automatica` | `revision_tecnica` | Sistema (si todo OK) |
| `subsanacion_pendiente` | `verificacion_automatica` | Sistema (al cargar documentos subsanados) |
| `revision_tecnica` | `subsanacion_tecnica` | `analista_arap` |
| `revision_tecnica` | `inspeccion_programada` | `analista_arap` (cuando las 3 revisiones aprueban) |
| `inspeccion_programada` | `inspeccion_realizada` | `inspector_arap` |
| `inspeccion_realizada` | `resolucion_pendiente` | `analista_arap` |
| `resolucion_pendiente` | `aprobado` | `director_arap` |
| `resolucion_pendiente` | `rechazado` | `director_arap` |

---

## 8. Reglas de negocio críticas

### RN-01 — Número de expediente
- Formato: `ARAP-ACU-{YYYY}-{NNNNN}` donde NNNNN es un correlativo de 5 dígitos con padding de ceros.
- Se genera una sola vez al pasar de `pago_pendiente` a `recibido`.
- Es inmutable — no puede cambiar después de generado.

### RN-02 — Plazo legal de 45 días hábiles
- Se calcula desde `fecha_recepcion` excluyendo sábados, domingos y feriados nacionales.
- A los 40 días hábiles el sistema envía alerta interna al analista asignado.
- A los 44 días envía alerta al Director General.
- El sistema no bloquea el expediente al vencer — solo alerta.

### RN-03 — Plazo de subsanación ciudadana
- El ciudadano tiene 10 días hábiles para subsanar desde la notificación.
- Al día 9 el sistema envía recordatorio por correo.
- Al día 11 el expediente pasa a `subsanacion_vencida` y el analista debe decidir si archiva o da prórroga.

### RN-04 — Condición de avance a inspección
- El expediente solo pasa a `inspeccion_programada` cuando las 3 revisiones institucionales tienen estado `aprobado`.
- No hay excepciones a esta regla.
- El sistema verifica esta condición automáticamente cada vez que una revisión cambia de estado.

### RN-05 — Integridad documental
- Cada documento cargado debe tener su hash SHA-256 calculado y almacenado al momento del upload.
- Si el mismo archivo se descarga más adelante, el hash debe coincidir.
- Esto es requerido por la Ley 51 de 2016 (Sistema Nacional de Archivos) y la Ley 83 de 2012.

### RN-06 — Pagos
- El formulario no puede someterse sin confirmación de pago de la Tesorería Nacional.
- Los montos son fijos por ley: B/. 200 solicitud + B/. 200 documentos e inspecciones + B/. 50 ventanilla = B/. 450.
- El pago de georeferenciación se gestiona directamente con el IGNTG, fuera del sistema.

### RN-07 — Un expediente activo por solicitante por área
- Un mismo RUC/cédula no puede tener dos expedientes activos para el mismo polígono geográfico.
- La validación se hace por comparación de coordenadas centrales al momento de recepción.

---

## 9. Notificaciones automáticas

| Evento | Canal | Destinatario | Plazo de envío |
|---|---|---|---|
| Expediente recibido | Email | Ciudadano | Inmediato |
| Subsanación requerida (Fase 2) | Email + SMS | Ciudadano | Inmediato |
| Recordatorio subsanación (día 9) | Email | Ciudadano | Automático |
| Expediente en revisión técnica | Email | Ciudadano | Inmediato |
| Nuevo expediente asignado | Email | Analista ARAP | Inmediato |
| Nueva revisión solicitada | Email | Revisor IGNTG / MiAmbiente | Inmediato |
| Expediente vence en 5 días | Email | Analista ARAP | Automático |
| Expediente vence en 1 día | Email | Analista + Director | Automático |
| Subsanación técnica solicitada | Email | Ciudadano | Inmediato |
| Inspección programada | Email + SMS | Ciudadano | Inmediato |
| Resolución emitida | Email | Ciudadano | Inmediato |

### Plantillas de correo
Todas las plantillas deben incluir:
- Logo ARAP
- Número de expediente en formato destacado
- Descripción clara de la acción requerida (si aplica)
- Enlace directo al portal
- Texto de pie: "Este es un mensaje automático del sistema de gestión de concesiones de la ARAP."

---

## 10. Control de acceso (RBAC)

### Permisos por recurso

| Recurso | ciudadano | analista_arap | inspector_arap | director_arap | revisor_igntg | revisor_ambiente | admin |
|---|---|---|---|---|---|---|---|
| Ver propio expediente | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Ver todos los expedientes | ❌ | ✅ | ✅ | ✅ | ❌ | ❌ | ✅ |
| Ver documentos legales | Solo propios | ✅ | ✅ | ✅ | ❌ | ❌ | ✅ |
| Ver documentos geográficos | Solo propios | ✅ | ✅ | ✅ | ✅ | ❌ | ✅ |
| Ver EIA y plan de desarrollo | Solo propios | ✅ | ✅ | ✅ | ❌ | ✅ | ✅ |
| Ver notas internas | ❌ | ✅ | ✅ | ✅ | ❌ | ❌ | ✅ |
| Cambiar estado del expediente | ❌ | ✅ | Parcial | ✅ | ❌ | ❌ | ✅ |
| Aprobar revisión técnica | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Aprobar revisión geográfica | ❌ | ❌ | ❌ | ❌ | ✅ | ❌ | ❌ |
| Aprobar revisión ambiental | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | ❌ |
| Emitir resolución | ❌ | ❌ | ❌ | ✅ | ❌ | ❌ | ❌ |
| Gestionar usuarios | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ |

### Implementación recomendada
- Usar middleware de autorización en cada endpoint del API
- Los permisos se evalúan en backend — nunca solo en frontend
- Cada request incluye JWT con claims de `rol` e `institucion`
- Log de acceso a documentos sensibles obligatorio (Ley 6 de Transparencia)

---

## 11. Requisitos no funcionales

### Rendimiento
- Tiempo de respuesta del portal: < 2 segundos (P95)
- Carga de documentos PDF: < 5 segundos por archivo de 10 MB
- Verificaciones automáticas: todas completas en < 30 segundos
- Concurrencia esperada inicial: 50 usuarios simultáneos

### Disponibilidad
- SLA objetivo: 99.5% uptime en horario hábil (Lun–Vie 7am–6pm)
- Ventana de mantenimiento: domingos 12am–4am
- Las verificaciones asíncronas pueden continuar fuera de horario

### Seguridad
- HTTPS obligatorio en todos los endpoints
- Tokens JWT con expiración de 8 horas
- Rate limiting: 100 requests/minuto por IP en endpoints públicos
- Documentos almacenados con cifrado en reposo (AES-256)
- No almacenar datos de tarjetas — toda la transacción va a Tesorería Nacional
- Cumplimiento con Ley 81 de 2019 (Protección de Datos Personales)

### Auditoría
- Todo cambio de estado de expediente queda en `log_eventos` con usuario y timestamp
- Todo acceso a documento sensible queda registrado
- Los logs se retienen por mínimo 5 años (Ley 51 de Archivos)
- Los logs no son editables ni eliminables por ningún usuario

### Infraestructura
- Preferencia por servidores en territorio panameño o región AWS/Azure más cercana
- Alternativa: Nube del Estado (AIG) si el presupuesto lo requiere
- Backups diarios de base de datos con retención de 30 días
- Object storage para documentos con versionado activado

---

## 12. Stack tecnológico recomendado

> Esta es una recomendación, no un requisito. El equipo puede proponer alternativas justificadas.

| Capa | Tecnología recomendada | Alternativa |
|---|---|---|
| Frontend portal ciudadano | React + TypeScript | Next.js |
| Frontend panel funcionarios | React + TypeScript | Next.js |
| Backend API | Node.js + NestJS | Python + FastAPI |
| Base de datos | PostgreSQL 15+ | — |
| Cola de tareas | Redis + BullMQ | RabbitMQ |
| Object storage | MinIO (self-hosted) | AWS S3 / Azure Blob |
| API Gateway | Kong | AWS API Gateway |
| Autenticación | JWT + Panama Digital (OAuth 2.0) | Keycloak |
| Email | SendGrid | AWS SES |
| SMS | Twilio | Claro API |
| CI/CD | GitHub Actions | GitLab CI |
| Contenedores | Docker + Docker Compose | Kubernetes (v2) |

---

## 13. Prototipos de referencia

Los prototipos interactivos están publicados en:

**`https://michellearias12.github.io/ARAP_PROTOTIPOS/`**

| Prototipo | URL |
|---|---|
| Índice general | `/index.html` |
| Fase 1 — Portal ciudadano | `/fase1-portal-solicitud.html` |
| Fase 2 — Validación automática | `/fase2-validacion-automatica.html` |
| Fase 3 — Panel funcionarios ARAP | `/fase3-panel-funcionarios.html` |
| Fase 3 — Portal interinstitucional | `/fase3-portal-interinstitucional.html` |

> Los prototipos son referencias de UX/UI. El equipo de desarrollo puede proponer mejoras de usabilidad. Los flujos de negocio y reglas descritas en este documento tienen precedencia sobre el diseño visual de los prototipos.

---

## Apéndice — Documentos requeridos por el trámite

| Código | Documento | Obligatorio | Base legal |
|---|---|---|---|
| `cedula` | Cédula del representante legal (copia autenticada) | ✅ | Art. 5 num. 2 D.E. 11/1997 |
| `registro` | Cert. Registro Público con directores y representante legal | ✅ | Art. 5 D.E. 11/1997 |
| `pysn` | Paz y Salvo Nacional | ✅ | — |
| `pysarap` | Paz y Salvo ARAP | ✅ | D.E. 111 de 2018 |
| `planos_area` | Planos del área con coordenadas geodésicas IGNTG | ✅ | Art. 5 num. 3 D.E. 11/1997 |
| `descripcion_poligono` | Descripción escrita del polígono a doble espacio | ✅ | Art. 5 num. 4 D.E. 11/1997 |
| `planos_proyecto` | Planos del proyecto (estanques, canales, obras) | ✅ | Art. 5 num. 4 D.E. 11/1997 |
| `factibilidad` | Estudio de factibilidad técnico-económico | ✅ | Art. 5 num. 6 D.E. 11/1997 |
| `plan_desarrollo` | Plan de desarrollo del proyecto | ✅ | Art. 36 num. 9 Ley 44/2006 |
| `eia` | EIA aprobado por MiAmbiente | ✅ | Art. 5 num. 7 D.E. 11/1997 |
| `resolucion_eia` | Resolución de aprobación del EIA | ✅ | Ley 8 de 2015 |
| `fotografias_aereas` | Fotografías aéreas del área del proyecto | ✅ | Art. 5 D.E. 11/1997 |
| `georeferenciacion` | Comprobante de pago de georeferenciación IGNTG | ✅ | Res. J.D. 045-2012 |

---

*Documento generado como parte del piloto de digitalización — ARAP 2026.*  
*Para consultas técnicas contactar al equipo de proyecto.*
