# Documentación Técnica — Arnés de Orquestación Multi-Agente

> **Proyecto:** `ejemplo-harness-subagentes` (notes-cli)
> **Objeto de este documento:** describir con detalle la configuración del
> arnés (harness), qué componentes lo forman, cómo funciona y cómo opera el
> ciclo de vida completo de una sesión de trabajo con agentes de IA.
>
> La aplicación (CLI de notas) es deliberadamente trivial. Lo importante de
> este repositorio no es *qué* hace, sino *cómo* está estructurado para que
> un agente de IA pueda trabajar sobre él de forma **auténtica, auditable y
> verificable**.

---

## Índice

1. [¿Qué es el arnés y por qué existe?](#1-qué-es-el-arnés-y-por-qu-existe)
2. [Arquitectura general: los 3 pilares](#2-arquitectura-general-los-3-pilares)
3. [Mapa completo de archivos](#3-mapa-completo-de-archivos)
4. [Componente 1: El repositorio ES el sistema](#4-componente-1-el-repositorio-es-el-sistema)
5. [Componente 2: Orquestación multi-agente](#5-componente-2-orquestación-multi-agente)
6. [Componente 3: Supervisión y verificación automáticas](#6-componente-3-supervisión-y-verificación-automáticas)
7. [Ciclo de vida de una sesión](#7-ciclo-de-vida-de-una-sesión)
8. [La aplicación bajo el arnés (src/ y tests/)](#8-la-aplicación-bajo-el-arnés-src-y-tests)
9. [Ejemplo real recorrido: feature #6 `cli_edit`](#9-ejemplo-real-recorrido-feature-6-cli_edit)
10. [Patrones de diseño del arnés](#10-patrones-de-diseño-del-arnés)
11. [Cómo ejecutar y verificar](#11-cómo-ejecutar-y-verificar)
12. [Cómo extender el arnés a un proyecto nuevo](#12-cómo-extender-el-arnés-a-un-proyecto-nuevo)
13. [Observaciones y limitaciones conocidas](#13-observaciones-y-limitaciones-conocidas)

---

## 1. ¿Qué es el arnés y por qué existe?

Un **arnés** (*harness*) en ingeniería de agentes es el conjunto de piezas
estáticas de un repositorio cuyo propósito es **encuadrar, dirigir y verificar
el trabajo de un agente de IA**, de modo que el resultado sea predecible y
auditable aunque el agente sea autónomo.

La premisa de diseño es:

> **El repositorio es el sistema operativo del agente.** Todo lo que el
> agente necesita saber (reglas, alcance, estado, criterios de calidad) vive
> en archivos versionados, no en la conversación ni en la "memoria" del modelo.

Esto aporta tres propiedades que un chat sin arnés no tiene:

| Propiedad | Significado práctico |
|-----------|----------------------|
| **Persistencia** | El estado sobrevive a reinicios, nuevas sesiones y context windows agotadas: vive en disco (`progress/`, `feature_list.json`). |
| **Auditabilidad** | Cada decisión de cada subagente queda escrita en un archivo con nombre predecible; un humano puede auditar paso a paso. |
| **Verificabilidad** | "Funciona" no se afirma, se demuestra: `init.sh` ejecuta tests reales. El arnés no se fía de lo que diga el agente. |

---

## 2. Arquitectura general: los 3 pilares

El arnés se organiza en tres pilares, tal y como se describe en `README.md`:

```
┌──────────────────────────────────────────────────────────────────────┐
│                     ARNÉS (harness)                                  │
│                                                                      │
│  PILAR 1                 PILAR 2                 PILAR 3             │
│  El repositorio ES       Orquestación             Supervisión y      │
│  el sistema              multi-agente             mejora             │
│  ─────────────           ─────────────            ─────────────      │
│  AGENTS.md               .claude/agents/          CHECKPOINTS.md     │
│  init.sh                   leader.md               init.sh           │
│  feature_list.json         implementer.md          .claude/settings  │
│  progress/                 reviewer.md               .json (hooks)   │
│  docs/                   CLAUDE.md (rol fijo)      tests/            │
│                                                                      │
│            ▼                    ▼                      ▼             │
│         EN DISCO            EN CHAT              EJECUTABLE          │
│      (estado/mapa)     (solo referencias)     (no negociable)        │
└──────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
              ┌───────────────────────────────┐
              │  src/  +  tests/  (la app)    │
              │  CLI de notas — deliberadamente│
              │  simple, sólo 3 capas         │
              └───────────────────────────────┘
```

### Pilar 1 — El repositorio ES el sistema
Archivos que componen el "mapa" y la "memoria" del proyecto: `AGENTS.md`,
`init.sh`, `feature_list.json`, `progress/`, `docs/`.

### Pilar 2 — Orquestación multi-agente
Definiciones de los subagentes con roles separados (líder / implementador /
revisor) en `.claude/agents/`, con `CLAUDE.md` fijando el rol inicial.

### Pilar 3 — Supervisión y mejora
Verificación ejecutable (`init.sh`, `tests/`) y **hooks automáticos** en
`.claude/settings.json` que el arnés ejecuta *por sí mismo* (el agente no
puede saltárselos). `CHECKPOINTS.md` define los criterios objetivos de
"estado final correcto".

---

## 3. Mapa completo de archivos

```
ejemplo-harness-subagentes/
├── AGENTS.md                    # Mapa de navegación para agentes (divulgación progresiva)
├── CLAUDE.md                    # Fuerza al modelo a actuar como "leader" al abrir sesión
├── CHECKPOINTS.md               # C1–C5: criterios objetivos de "estado correcto"
├── README.md                    # Presentación del arnés para humanos
├── init.sh                      # Verificación ejecutable del entorno (gate de arranque/cierre)
├── feature_list.json            # Alcance: features con máquina de estados
├── .gitignore
│
├── docs/                        # El contrato de calidad ("qué significa hacerlo bien")
│   ├── architecture.md          # Capas, principios, flujo de datos, qué NO hacer
│   ├── conventions.md           # Estilo, nombres, estructura de archivo, errores
│   └── verification.md          # 3 niveles de verificación + anti-patrones
│
├── progress/                    # Memoria persistente entre sesiones (anti-teléfono-descompuesto)
│   ├── current.md               # Plan vivo de la sesión (se vacía al cerrar)
│   ├── history.md               # Bitácora append-only de sesiones cerradas
│   ├── explore_*.md             # Informes de subagentes exploradores
│   ├── impl_<feature>.md        # Informe del implementer (archivos tocados + tests)
│   └── review_<feature>.md      # Veredicto del reviewer (checklist vs docs/ y CHECKPOINTS)
│
├── .claude/
│   ├── agents/
│   │   ├── leader.md            # Orquestador: descompone y coordina, NUNCA implementa
│   │   ├── implementer.md       # Trabajador: implementa EXACTAMENTE UNA feature
│   │   └── reviewer.md          # Revisor: aprueba/rechaza, NUNCA edita código
│   └── settings.json            # Hooks (PostToolUse, Stop) + permisos allow-list
│
├── src/                         # La aplicación (3 capas, stdlib puro)
│   ├── __init__.py
│   ├── storage.py               # Persistencia atómica JSON
│   ├── notes.py                 # Modelo de dominio (dataclass frozen)
│   └── cli.py                   # Interfaz argparse (6 subcomandos)
│
├── tests/                       # Verificación ejecutable (unittest)
│   ├── __init__.py
│   ├── test_storage.py
│   ├── test_notes.py
│   └── test_cli.py
│
└── documentos/                  # Documentación generada (este documento)
```

---

## 4. Componente 1: El repositorio ES el sistema

### 4.1 `AGENTS.md` — Punto de entrada y mapa (divulgación progresiva)

`AGENTS.md` **no** es una biblia de reglas. Es deliberadamente un **mapa**:
indica *qué* archivo leer y *cuándo*, para que el agente reciba la información
bajo demanda en lugar de toda de golpe. Esto reduce el ruido y el consumo de
contexto.

Contenido:

1. **Antes de empezar (obligatorio):** ejecutar `./init.sh`, leer
   `progress/current.md` y `feature_list.json`, y elegir **una** tarea
   `pending`.
2. **Tabla-mapa** del repositorio: qué contiene cada archivo y cuándo leerlo.
3. **Reglas duras (no negociables):**
   - Una sola feature a la vez.
   - No declarar `done` sin tests verdes.
   - Documentar en `progress/current.md` mientras se trabaja, no al final.
   - Dejar el repo limpio antes de cerrar.
   - Si no se sabe algo, buscar en `docs/` antes de inventar.
4. **Protocolo de elección de tarea** (5 pasos numerados).
5. **Ciclo de cierre de sesión** (`§5 lifecycle`, 5 pasos).
6. **Protocolo de bloqueo:** si algo falla de forma inesperada, documentarlo
   en `progress/current.md` y **parar** — no improvisar workarounds.

### 4.2 `feature_list.json` — El alcance con máquina de estados

Es la **lista de tareas** del proyecto y a la vez la fuente de verdad del
alcance. Cada feature tiene:

```json
{
  "id": 7,
  "name": "cli_recent",
  "title": "Comando recent",
  "description": "Lista las N notas más recientes...",
  "acceptance": [
    "`python -m src.cli recent` lista las 5 notas más recientes por defecto",
    "`--limit 10` permite cambiar el número",
    "..."
  ],
  "status": "pending"
}
```

- **`acceptance`**: criterios concretos y verificables. El implementador no
  puede salirse del scope; el revisor los usa como checklist.
- **`status`**: máquina de estados `pending → in_progress → done` (más
  `blocked` para bloqueos).
- **Reglas globales** (`"rules"`): `one_feature_at_a_time: true`,
  `require_tests_to_close: true`, y la lista de estados válidos.

**Estado actual del proyecto:** 7 features — #1 a #6 en `done`
(storage_layer, note_model, cli_add_list, cli_show_delete, cli_search,
cli_edit) y la #7 (`cli_recent`) en `pending`, lista para implementarse.

La regla "una feature a la vez" **no es un consejo, está validada por
código**: el bloque 3 de `init.sh` parsea el JSON y aborta si hay más de un
`in_progress` o un estado inválido.

### 4.3 `progress/` — Memoria persistente y trazabilidad

Es el mecanismo central de **trazabilidad** y de la **regla
anti-teléfono-descompuesto** (ver §10.2).

| Archivo | Quién lo escribe | Contenido | Ciclo de vida |
|---|---|---|---|
| `current.md` | leader (y subagentes) | Plan vivo: feature en curso, hora de inicio, plan en 3–5 bullets, bitácora de pasos, próximo paso. | Se vacía al cerrar sesión (queda solo la plantilla). |
| `history.md` | leader | Resumen append-only de cada sesión cerrada. **Nunca se editan entradas anteriores**, solo se añaden al final. | Permanente. |
| `explore_<tema>.md` | explorers | Hallazgos de investigación previa. | Permanente (referencia histórica). |
| `impl_<feature>.md` | implementer | Archivos modificados, decisiones de diseño y salida completa del último `./init.sh`. | Permanente. |
| `review_<feature>.md` | reviewer | Veredicto (`APPROVED`/`CHANGES_REQUESTED`) + checklist contra `docs/` y `CHECKPOINTS.md`. | Permanente. |

### 4.4 `docs/` — El contrato de calidad

Son los tres documentos que un revisor usa como estándar. **Si un requisito no
está en `docs/`, no es un requisito** (regla explícita de `architecture.md`).

**`docs/architecture.md` — "qué significa hacer un buen trabajo"**

| Principio | Detalle |
|---|---|
| Capas claras | Exactamente 3: `storage.py` (persistencia), `notes.py` (dominio), `cli.py` (UI). Prohibido añadir capas (servicios, repositorios, ORMs) sin razón documentada en `feature_list.json`. |
| Sin dependencias externas | Solo stdlib de Python. Si una feature requiere una dependencia, se discute primero (estado `blocked`). |
| Errores explícitos | Las funciones que pueden fallar lanzan excepciones nombradas, nunca devuelven `None`. |
| Inmutabilidad | `Note` es `@dataclass(frozen=True)`. Modificar = crear nueva instancia. |
| Atomicidad en disco | Toda escritura pasa por archivo temporal + `os.replace()`. Nunca dejar el JSON a medio escribir. |

Incluye además el diagrama de flujo de datos y una sección "Qué NO hacer"
(errores a `stderr` no a `print()`, no mezclar IO con dominio, no leer el
archivo en bucle, no inventar un sistema de configuración).

**`docs/conventions.md` — estilo y homogeneidad**

- Python 3.9+, PEP 8, líneas ≤ 100 chars, comillas dobles, f-strings.
- Tabla de nombres: `snake_case` (módulos/funciones), `PascalCase` (clases),
  `UPPER_SNAKE` (constantes), prefijo `_` (privadas).
- Estructura de archivo: docstring + `from __future__ import annotations` +
  imports stdlib + imports locales.
- Tests: un archivo por módulo, `unittest.TestCase`, `tempfile` real, nombres
  descriptivos (`test_load_returns_empty_when_file_missing`).
- Comentarios: por defecto **no**; solo para explicar un *por qué* no obvio.
- Errores: jerarquía `NoteError` → `NoteNotFound`; el CLI los captura,
  imprime a `stderr` y sale con código 1 (nunca stack traces al usuario).

**`docs/verification.md` — "el agente no dice 'funciona', lo demuestra"**

| Nivel | Qué es | Obligatoriedad |
|---|---|---|
| 1. Unitarios | Todo método público de `src/` tiene test de camino feliz + al menos un camino de error. | Obligatorio |
| 2. Integración CLI | Features de UI se verifican ejecutando el CLI real contra archivo temporal (vía `NOTES_FILE`). | Obligatorio para features de UI |
| 3. Smoke manual | Flujo end-to-end en `/tmp` antes de cerrar sesión. | Recomendado |

Anti-patrones listados: "debería funcionar" sin test; tests que solo verifican
que no lanza excepción; mocks del filesystem (usar `tempfile` real); marcar
`done` sin pasar `./init.sh`.

### 4.5 `CHECKPOINTS.md` — Evaluación por destino, no por camino

> "En sistemas multi-agente no se evalúa el camino, se evalúa el destino."

Cinco grupos de checkboxes objetivos que un juez (humano o IA) recorre:

| Checkpoint | Qué evalúa |
|---|---|
| **C1** | El arnés está completo (archivos base + docs + `init.sh` en verde). |
| **C2** | El estado es coherente (≤1 `in_progress`, `done` con tests, `current.md` limpio). |
| **C3** | El código respeta la arquitectura (solo módulos previstos, sin dependencias, sin prints de debug ni TODOs sueltos). |
| **C4** | La verificación es real (tests por módulo, `tempfile` real, todos verdes). |
| **C5** | La sesión se cerró bien (sin temporales, entrada en `history.md`, estado de features correcto). |

El revisor marca `[x]`/`[ ]` y **rechaza el cierre de sesión** si quedan boxes
vacíos en C1–C5.

---

## 5. Componente 2: Orquestación multi-agente

Los agentes están definidos en `.claude/agents/*.md` con frontmatter
(`name`, `description`, `tools`) y un protocolo operativo en el cuerpo. La
idea central es la **separación de roles**: cada agente tiene capacidades
deliberadamente recortadas para que no pueda saltarse su función.

### 5.1 `CLAUDE.md` — El rol inicial es obligatorio

Se carga automáticamente al inicio de cada sesión de Claude Code y **fija al
modelo como `leader`**. Reglas duras que impone:

- ❌ No editar `src/` ni `tests/` directamente (ni Edit, ni Write, ni Bash).
- ❌ No marcar features como `done`.
- ✅ Para código → lanzar `implementer`; para validar → lanzar `reviewer`.
- ✅ Preguntas de lectura/exploración pura → responder directamente sin
  subagentes; cambios fuera de `src/`/`tests/` (docs, `progress/`) → puede
  editar él mismo.

### 5.2 `leader.md` — El orquestador que nunca implementa

- **Herramientas:** `Read, Glob, Grep, Bash, Agent` (puede lanzar subagentes,
  pero carece de `Write`/`Edit` para código: la restricción es estructural,
  no sólo instruccional).
- **Protocolo de arranque:** leer `AGENTS.md` → `feature_list.json` +
  `progress/current.md` → ejecutar `./init.sh` (si falla, parar).
- **Descomposición:** 1 feature simple → 1 implementer; investigación
  previa → 2-3 explorers en paralelo (pregunta concreta y acotada cada uno);
  siempre 1 reviewer antes de declarar nada `done`.
- **Tabla de escalado de esfuerzo:**

  | Complejidad | Subagentes | Notas |
  |---|---|---|
  | Trivial (1 archivo) | 1 implementer | Sin explorers |
  | Media (2–3 archivos) | 1 implementer + 1 reviewer | |
  | Compleja (refactor) | 2-3 explorers → 1 implementer → 1 reviewer | |
  | Muy compleja | Dividir en sub-tareas y re-aplicar la tabla | |

- **Prohibiciones:** ❌ editar `src/`/`tests/`; ❌ marcar features `done`;
  ❌ aceptar resultados de subagentes que lleguen por chat sin referencia a
  archivo.

### 5.3 `implementer.md` — El trabajador de una sola feature

- **Herramientas:** `Read, Write, Edit, Glob, Grep, Bash` (puede escribir
  código y ejecutar verificación).
- **Protocolo (8 pasos):** leer docs → tomar *una* feature `pending` y pasarla
  a `in_progress` → anotar plan en `progress/current.md` → implementar sin
  salirse del `acceptance` → escribir tests → verificar con `./init.sh` →
  **no marcar `done` él mismo** (llamar al reviewer) → solo tras aprobación,
  marcar `done` y mover resumen a `history.md`.
- **Reglas duras:** una feature por sesión (si el cambio toca otra, se para y
  se reporta); toda escritura de código va acompañada de su test; si una
  herramienta falla inesperadamente, **no improvisar workaround** → bloquear.
- **Comunicación con el líder:** respuesta final de **una sola línea**:
  `done -> feature <id> implementada y revisada` o `blocked -> ver
  progress/current.md`. Nunca el diff completo por chat.

### 5.4 `reviewer.md` — El revisor que no edita

- **Herramientas:** `Read, Glob, Grep, Bash` (sin `Write`/`Edit`: no puede
  tocar el código, solo evaluarlo).
- **Protocolo:** leer `architecture.md` + `conventions.md` + `CHECKPOINTS.md`
  → identificar archivos modificados (según el informe del implementer en
  `progress/current.md`) → evaluar cada uno contra capas/dependencias/estilo/
  tests → ejecutar `./init.sh` (debe estar verde) → recorrer C1–C5 → emitir
  veredicto.
- **Formato del veredicto:** un único bloque markdown en
  `progress/review_<feature>.md` con `Veredicto: APPROVED |
  CHANGES_REQUESTED`, checklist y "cambios requeridos" concretos (citando
  archivos y líneas). En chat, solo una línea: `APPROVED -> ver
  progress/review.md`.
- **Reglas duras:** ❌ nunca aprobar con tests rojos ni con `init.sh` en rojo;
  ❌ nunca editar el código; ✅ feedback concreto, nada genérico.

### 5.5 Resumen de separación de poderes

| Agente | Puede escribir código | Puede marcar `done` | Puede lanzar subagentes | Verificación |
|---|---|---|---|---|
| **leader** | ❌ | ❌ | ✅ | Ejecuta `init.sh` como gate |
| **implementer** | ✅ (1 feature) | Solo tras reviewer | ❌ | `./init.sh` tras cada cambio |
| **reviewer** | ❌ | ❌ (emite veredicto) | ❌ | `./init.sh` + checklist C1–C5 |

---

## 6. Componente 3: Supervisión y verificación automáticas

### 6.1 `init.sh` — El gate ejecutable

Script de bash que el agente debe ejecutar **al comenzar la sesión** y **antes
de declarar cualquier cosa `done`**. Si falla, la sesión no avanza. Devuelve
código de salida claro y bloques `[OK]`/`[FAIL]`. Cinco bloques:

1. **Entorno:** `python3` presente y versión ≥ 3.9 (requerida por
   `dataclasses` y sintaxis de typing moderna).
2. **Archivos base:** comprueba la existencia de `AGENTS.md`,
   `feature_list.json`, `progress/current.md`, los 3 docs y `CHECKPOINTS.md`.
3. **Validación de `feature_list.json`:** parsea el JSON; **rechaza > 1
   feature en `in_progress`** y estados fuera de la lista válida
   (validación embebida en un bloque `python3 - <<'PY'`).
4. **Tests:** `python3 -m unittest discover -s tests -v` — cualquier test
   rojo marca el bloque en `[FAIL]` y eleva el exit code global.
5. **Resumen:** mensaje final y exit code 0 / 1.

### 6.2 `.claude/settings.json` — Hooks que el agente no puede saltar

Es la pieza clave de la **supervisión automática**: son disparadores que
ejecuta el propio arnés (Claude Code), no el modelo, por lo que no dependen
de la buena voluntad del agente.

```json
{
  "hooks": {
    "PostToolUse": [
      { "matcher": "Edit|Write",
        "hooks": [ { "type": "command",
          "command": "python3 -m unittest discover -s tests -q 2>&1 | tail -3" } ] }
    ],
    "Stop": [
      { "hooks": [ { "type": "command",
          "command": "./init.sh > /tmp/harness_init.log 2>&1 && echo '[harness] init.sh OK' || (echo '[harness] init.sh FALLÓ...' && tail -20 /tmp/harness_init.log)" } ] }
    ]
  },
  "permissions": {
    "allow": [
      "Bash(./init.sh)",
      "Bash(python3 -m unittest*)",
      "Bash(python3 -m src.cli*)"
    ]
  }
}
```

| Hook | Momento | Efecto |
|---|---|---|
| `PostToolUse` (matcher `Edit\|Write`) | Tras **cada** edición o escritura de archivo | Corre la suite de tests y muestra el resumen (últimas 3 líneas). El agente ve al instante si su cambio rompió algo. |
| `Stop` | Antes de **cerrar la sesión** | Fuerza `./init.sh` completo a un log (`/tmp/harness_init.log`) e imprime OK o el detalle del fallo. |
| `permissions.allow` | Siempre | Pre-aprueba los comandos seguros (`./init.sh`, unittest, `src.cli`) para reducir fricción y prompts de permiso. |

---

## 7. Ciclo de vida de una sesión

Diagrama completo del flujo de trabajo que el arnés impone:

```
┌─────────────────────────────────────────────────────────────────────────┐
│ ARRANQUE (leader)                                                       │
│  1. Leer AGENTS.md                                                      │
│  2. Leer feature_list.json + progress/current.md                        │
│  3. Ejecutar ./init.sh  ──FAIL──▶ PARAR y reportar                      │
│  4. Elegir UNA feature pending (menor id)                               │
│  5. feature_list.json: pending → in_progress; plan en current.md        │
└───────────────────────────────┬─────────────────────────────────────────┘
                                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ (Opcional) EXPLORACIÓN — 2-3 explorers en paralelo                      │
│  Cada uno: pregunta concreta → escribe progress/explore_<tema>.md       │
│  → devuelve al leader solo: "done -> progress/explore_<tema>.md"        │
└───────────────────────────────┬─────────────────────────────────────────┘
                                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ IMPLEMENTACIÓN — implementer (1 sola feature)                           │
│  6.  Implementar según acceptance + docs/                               │
│  7.  Escribir tests que validan cada criterio de acceptance             │
│  8.  Ejecutar ./init.sh  ──FAIL──▶ volver al paso 6                     │
│  9.  Escribir progress/impl_<feature>.md (archivos + diseño + output)   │
│  10. Responder al leader: "done -> feature <id> ..."                    │
└───────────────────────────────┬─────────────────────────────────────────┘
                                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ REVISIÓN — reviewer                                                     │
│  11. Checklist contra architecture.md + conventions.md + verification   │
│  12. Ejecutar ./init.sh (debe estar verde)                              │
│  13. Recorrer CHECKPOINTS.md (C1–C5)                                    │
│  14. Escribir progress/review_<feature>.md con veredicto                │
│  15. Responder: "APPROVED -> ..." o "CHANGES_REQUESTED -> ..."          │
│      CHANGES_REQUESTED ──▶ vuelve al implementer (paso 6)               │
└───────────────────────────────┬─────────────────────────────────────────┘
                                ▼ APPROVED
┌─────────────────────────────────────────────────────────────────────────┐
│ CIERRE (leader)                                                         │
│  16. feature_list.json: in_progress → done                              │
│  17. ./init.sh final (también forzado por el hook Stop)                 │
│  18. Resumen de current.md → final de history.md (append-only)          │
│  19. Vaciar current.md (dejar solo plantilla)                           │
│  20. Repo limpio: sin temporales, sin prints de debug, sin TODOs        │
└─────────────────────────────────────────────────────────────────────────┘
```

Durante **todo** el proceso, los hooks de §6.2 están activos: cada
`Edit`/`Write` dispara los tests, y el cierre de sesión dispara `init.sh`.

---

## 8. La aplicación bajo el arnés (src/ y tests/)

Aunque es secundaria respecto al arnés, la app respeta escrupulosamente las
tres capas de `architecture.md` y no usa nada fuera de la stdlib.

### 8.1 `src/storage.py` — Persistencia atómica

- Ruta por defecto `.notes.json`, sobreescribible con la variable de entorno
  `NOTES_FILE` (clave para los tests y la verificación por nivel 2).
- `load(path=None) -> list[dict]`: devuelve `[]` si el archivo no existe.
- `save(notes, path=None)`: **escritura atómica** — crea un temporal con
  `tempfile.mkstemp(prefix=".notes_", suffix=".json", dir=directory)`, escribe
  con `json.dump(..., ensure_ascii=False, indent=2)` y publica con
  `os.replace(tmp_path, path)` (atómico a nivel de sistema de ficheros). En
  caso de excepción, limpia el temporal antes de re-lanzar.

### 8.2 `src/notes.py` — Modelo de dominio

- `NoteError(Exception)` → base del dominio; `NoteNotFound(NoteError)` →
  id inexistente.
- `Note` como `@dataclass(frozen=True)` con `id`, `title`, `body`,
  `created_at`.
- `Note.new(title, body, existing)`: calcula `next_id = max(ids, default=0)+1`
  y fija `created_at` en ISO 8601 UTC con precisión de segundos.
- `to_dict()`: serialización vía `dataclasses.asdict`.

### 8.3 `src/cli.py` — Interfaz argparse

Seis subcomandos, todos con el mismo patrón: `load()` → operación en memoria →
`save()` si toca → mensaje de confirmación por stdout.

| Comando | Acción | Salida / error |
|---|---|---|
| `add <title> [--body]` | Crea nota con `Note.new` | `id=<n>` |
| `list` | Lista todas | `<id>\t<created_at>\t<title>` por línea; vacío si no hay |
| `show <id>` | Muestra una | título, fecha, cuerpo |
| `delete <id>` | Elimina una | `borrada id=<n>`; `NoteNotFound` si no existe |
| `search <query>` | Substring case-insensitive en title+body | lista coincidencias; `NoteNotFound` si ninguna |
| `edit <id> [--title] [--body]` | Actualiza campos (sin mutar la `Note` original: crea instancia nueva preservando `id`/`created_at`) | `editada id=<n>`; `NoteError` sin flags; `NoteNotFound` si id inexistente |

El handler `main()` captura `NoteError` (clase base de `NoteNotFound`),
imprime el mensaje a `sys.stderr` y devuelve exit code `1`. Los argumentos
`--title`/`--body` de `edit` usan default `None` (no `""`) para poder
distinguir "no pasado" de "cadena vacía", que es un valor legítimo.

### 8.4 `tests/` — Verificación ejecutable

22 tests en verde (suite actual). Patrón común en `TestCli`:

- `setUp`: `tempfile.TemporaryDirectory()` + `patch.object(storage,
  "DEFAULT_NOTES_PATH", self.path)` — filesystem **real** temporal, no mocks
  del filesystem (mockear solo la constante de ruta es legítimo).
- `_run(argv)` redirige stdout/stderr a `StringIO` y devuelve
  `(code, out, err)`.
- Cada comando se testea en **camino feliz y camino de fallo** (exit != 0,
  `out == ""`, mensaje en `err`).

Cobertura: `test_storage.py` (3), `test_notes.py` (4), `test_cli.py` (15).

---

## 9. Ejemplo real recorrido: feature #6 `cli_edit`

El repositorio contiene la traza completa de una sesión real, que sirve de
plantilla para entender el arnés en acción (fuentes:
`progress/history.md`, `progress/impl_cli_edit.md`,
`progress/review_cli_edit.md`).

1. **Sesión (2026-04-29), agentes:** "Claude Opus 4.7 (leader) → implementer
   → reviewer" — se ve la cadena de roles completa.
2. **Implementer** añadió `cmd_edit` + subparser en `src/cli.py` y 5 tests
   (cada flag por separado, ambos juntos, id inexistente, ausencia de flags).
   Decisiones documentadas en `progress/impl_cli_edit.md`: chequeo temprano
   de ausencia de flags, uso de `None` como default, construcción de nueva
   instancia `Note` preservando `id`/`created_at` para no violar
   inmutabilidad, y confirmación `editada id=N` siguiendo el patrón de
   `cmd_delete`.
3. **Verificación:** `./init.sh` verde con **22 tests** (17 previos + 5
   nuevos); la salida completa quedó embebida en el informe.
4. **Reviewer** produjo `progress/review_cli_edit.md` con veredicto
   **APPROVED**: los 6 criterios de `acceptance` casillados, arquitectura,
   convenciones (incluida verificación de líneas ≤ 100 chars y greps de
   TODO/debug), verificación (tempfile real, asserts concretos) y checklist
   C1–C5 completa. Detectó además una **nota menor no bloqueante**: faltaba
   la entrada de la sesión en `history.md` (que por convención se añade al
   cerrar sesión, no al cerrar feature).
5. **Cierre:** feature #6 → `done` en `feature_list.json`; entrada del
   2026-04-29 añadida al final de `history.md`.

Este es exactamente el flujo de §7, con la particularidad de que el contenido
de los informes **nunca circuló por el chat**: el líder solo recibió
referencias `done -> progress/impl_cli_edit.md` y
`APPROVED -> ver progress/review_cli_edit.md`.

---

## 10. Patrones de diseño del arnés

### 10.1 Divulgación progresiva
`AGENTS.md` es un mapa, no un reglamento. El agente recibe solo el índice y
busca el detalle bajo demanda en `docs/`. Esto escala mejor en contexto y
reduce que el agente "aplique" reglas que no tocan.

### 10.2 Anti teléfono-descompuesto
Los subagentes **escriben sus resultados en archivos** (`progress/...`) y
devuelven al líder **solo una referencia ligera** (`done ->
progress/impl_X.md`). El líder nunca ve el contenido por chat: lo lee del
disco si lo necesita. Esto elimina la degradación de información al pasar por
múltiples cadenas de agentes y deja un rastro auditable versionado.

### 10.3 Una feature a la vez
Validada por código en `init.sh` (bloque 3). Evita cambios entrelazados,
simplifica la revisión y hace que el estado sea siempre interpretable.

### 10.4 Estado en disco, no en chat
`progress/current.md` + `history.md` + `feature_list.json` sobreviven a
reinicios y a context windows agotadas. Cualquier sesión nueva puede
reconstruir el contexto leyendo archivos.

### 10.5 Verificación ejecutable (no confiar, verificar)
`init.sh` y los hooks corren los tests **reales**. El arnés asume que el
agente puede equivocarse (o exagerar) y pone la verificación fuera de su
voluntad (`PostToolUse`, `Stop`).

### 10.6 Líder-Trabajador-Revisor
El líder no implementa, el implementador no se autoaprueba, el revisor no
edita. La separación de poderes + herramientas recortadas por agente
impide los fallos típicos de un agente monolítico (autoaprobación,
deriva de scope, arreglos encubiertos).

### 10.7 Fallo duro ante lo inesperado
Tanto `AGENTS.md` como `implementer.md` prohíben inventar workarounds: si
una herramienta falla de forma inesperada, se documenta el bloqueo en
`progress/current.md` y se termina la sesión. Evita que el agente "parchee"
el entorno en vez de resolver.

---

## 11. Cómo ejecutar y verificar

```bash
# 1. Verificación completa del arnés (gate de arranque y de cierre)
./init.sh

# 2. Suite de tests directa
python3 -m unittest discover -s tests -v

# 3. Uso de la app (ruta por defecto .notes.json)
python3 -m src.cli add "comprar pan" --body "y leche"
python3 -m src.cli list
python3 -m src.cli show 1
python3 -m src.cli search "leche"
python3 -m src.cli edit 1 --title "nuevo título"
python3 -m src.cli delete 1

# 4. Demo sin ensuciar el repo (ruta vía variable de entorno)
NOTES_FILE=/tmp/demo_notes.json python3 -m src.cli add "test" --body "x"
rm /tmp/demo_notes.json
```

**Nota operativa:** `init.sh` fue creado con finales de línea CRLF (Windows).
En entornos POSIX puede fallar con errores tipo `$'\r': command not found`;
normalizar a LF con `sed -i 's/\r$//' init.sh` lo resuelve (ver §13).

**Uso del arnés con Claude Code:** abrir la sesión en la raíz del repo
(`claude`), lo que carga `CLAUDE.md` y fija el rol `leader`. Con al menos una
feature en `pending` (hoy: la #7 `cli_recent`), pedir literalmente
«implementa la siguiente feature pendiente» y observar `progress/` en el
editor: los informes de cada subagente aparecen en cuanto terminan.

---

## 12. Cómo extender el arnés a un proyecto nuevo

El arnés es portable: es un patrón, no un código acoplado a la app de notas.
Para replicarlo en otro proyecto:

1. **Copia los 7 piezas base:** `AGENTS.md`, `CLAUDE.md`, `CHECKPOINTS.md`,
   `init.sh` (adáptalo: archivos comprobados, comando de tests), y la
   estructura `docs/` (3 archivos), `progress/` (`current.md` con plantilla +
   `history.md` vacío), `.claude/agents/` (3 roles).
2. **Define el alcance:** crea `feature_list.json` con tus features reales,
   cada una con `acceptance` verificable y `status: "pending"`.
3. **Ajusta los hooks** de `.claude/settings.json` al comando de tests de tu
   stack (aquí `unittest`; en otro proyecto podría ser `pytest`, `npm test`,
   `cargo test`, etc.).
4. **Escribe `docs/architecture.md`** para tu dominio: capas permitidas,
   principios de calidad, flujo de datos, qué NO hacer. Sin esto, el revisor
   no tiene estándar.
5. **Valida el arranque:** `./init.sh` en verde antes de la primera sesión.
6. **Trabaja por features:** nunca más de una `in_progress`; toda feature
   termina con tests verdes y veredicto `APPROVED`.

---

## 13. Observaciones y limitaciones conocidas

1. **Finales de línea CRLF en `init.sh`:** el archivo contiene `\r` de
   Windows; en Linux/macOS falla al ejecutarse directamente (`env: $'bash\r':
   No such file or directory`) o vía `bash init.sh` (errores de sintaxis).
   Solución: normalizar a LF (`sed -i 's/\r$//' init.sh`) o configurar
   `core.autocrlf`/`.gitattributes` en el repo (`init.sh text eol=lf`).
2. **Referencia a un archivo inexistente:** `AGENTS.md` (§2) y `CLAUDE.md`
   mencionan `scripts/demo_orchestration.py` como demo del patrón
   anti-teléfono-descompuesto, pero ese archivo **no existe** en el árbol
   actual. Conviene crearlo o eliminar la referencia.
3. **Feature pendiente:** la #7 (`cli_recent`) está en `pending`; es el
   siguiente trabajo previsto y el caso de uso previsto para probar el arnés
   de punta a punta.
4. **`CHECKPOINTS.md` sin marcar:** los checkboxes siguen en `[ ]` (formato
   markdown editable por el revisor al emitir veredicto); no es un error, es
   el comportamiento previsto.
5. **Permisos de ejecución:** en algunos sistemas (WSL/ntfs-3g) los bit de
   ejecución pueden no persistir; ejecutar `bash init.sh` en su defecto (una
   vez normalizado el line ending).
6. **`progress/explore_*.md`:** existen informes de exploración previa
   (`explore_cli_surface.md`, `explore_note_model.md`,
   `explore_storage_layer.md`) de fases de investigación, coherentes con el
   patrón de explorers en paralelo descrito en `leader.md`.

---

*Documento generado como parte de la documentación del proyecto
`ejemplo-harness-subagentes`. Última actualización: 2026-10-21.*
