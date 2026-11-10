# Job Radar: plan de ruta

> Bot que corre en mi Raspberry Pi. Recoge ofertas de empleo de muchas fuentes legales, quita los duplicados, las puntúa según mi perfil junior y me envía las mejores por Telegram.
>
> Tiene dos objetivos: encontrar trabajo y tener un proyecto de portfolio que pueda explicar de principio a fin.

---

## Regla del proyecto: aprender, no "vibecodear"

1. **Primero escribo yo** cada función, aunque sea fea. Después la IA la revisa y me la explica.
2. **No hago commit de código que no sepa explicar** en voz alta, línea por línea.
3. **Cada fase acaba con algo que funciona**, subido a GitHub y con el README actualizado.
4. **Límite de tiempo:** el bot no sustituye a aplicar. Mientras lo construyo, sigo enviando 3–5 candidaturas por semana.

---

## Stack

| Pieza | Elección | Por qué |
|---|---|---|
| Lenguaje | Python 3.11+ | Ya lo conozco; tiene buenas librerías para HTTP, RSS y emails |
| HTTP | `httpx` | Moderno, con timeouts y reintentos |
| RSS | `feedparser` | El estándar |
| Configuración | `PyYAML` + `python-dotenv` | Fuentes en YAML; claves en `.env` |
| Base de datos | SQLite (`sqlite3`) | Un solo archivo, cero mantenimiento, va perfecta en una Raspberry |
| Puntuación con IA | API de Claude (o la que prefiera) | Lee la oferta junto a mi perfil y explica por qué encaja |
| Avisos | Bot de Telegram (API HTTP) | Gratis y llega al móvil al instante |
| Emails | API de Gmail (solo lectura) | Para leer las alertas de LinkedIn, Indeed y otros |
| Programación horaria | `cron` en la Raspberry | Simple y fiable |
| Tests | `pytest` | Ya lo usé en LitCount |
| Calidad | `ruff` | Linter y formateador en uno |

## Estructura de carpetas

```
job-radar/
├── README.md
├── ROADMAP.md
├── sources.yaml          # fuentes y búsquedas
├── companies.yaml        # empresas a vigilar (fase 4)
├── profile.md            # mi perfil (el markdown completo)
├── .env.example          # nombres de las claves, sin valores
├── .gitignore            # .env, *.db, __pycache__
├── requirements.txt
├── jobradar/
│   ├── __init__.py
│   ├── main.py           # orquesta: recoger → normalizar → filtrar → puntuar → avisar
│   ├── models.py         # dataclass Job
│   ├── db.py             # SQLite: guardar, comprobar si ya existe, marcar enviada
│   ├── sources/
│   │   ├── base.py       # interfaz común: fetch() -> list[Job]
│   │   ├── adzuna.py
│   │   ├── jooble.py
│   │   ├── cido_rss.py
│   │   ├── infojobs.py
│   │   ├── gmail_alerts.py
│   │   └── ats.py        # Greenhouse, Lever, Workable…
│   ├── dedupe.py
│   ├── scoring/
│   │   ├── rules.py      # puntuación por reglas (rápida y gratis)
│   │   └── llm.py        # puntuación con IA (solo las que pasan el filtro)
│   └── notify/
│       └── telegram.py
└── tests/
    ├── fixtures/         # respuestas reales guardadas (JSON y RSS) para tests sin red
    └── test_*.py
```

## Modelo de datos

```python
@dataclass
class Job:
    source: str          # "adzuna", "cido"…
    source_id: str       # ID en la fuente original
    title: str
    company: str | None
    location: str | None
    remote: bool
    url: str
    description: str
    salary: str | None
    contract: str | None
    published_at: datetime | None
    fingerprint: str     # hash de título, empresa y ciudad normalizados, para detectar duplicados
    rule_score: int = 0
    llm_score: int | None = None
    llm_reason: str | None = None
```

Tabla `jobs` en SQLite con los mismos campos, más `first_seen`, `sent_at` y `status` (`new`, `sent`, `applied`, `discarded`).

---

## Fase 0 · Preparación (1 tarde)

**Objetivo:** tener todo listo para programar.

- [ ] Crear el repositorio `job-radar` en GitHub, público, con licencia MIT.
- [ ] Preparar la Raspberry: actualizar el sistema, instalar Python 3.11 o superior y git, y crear un `venv`.
- [ ] Crear la estructura de carpetas de arriba, el `.gitignore` y el `.env.example`.
- [ ] Crear el bot de Telegram con @BotFather y apuntar el `TELEGRAM_TOKEN` y mi `CHAT_ID`.
- [ ] Pedir las claves de API de **Adzuna** y **Jooble**. Pueden tardar, así que pídelas ya.
- [ ] Copiar mi perfil completo a `profile.md`, **sin datos sensibles**.
- [ ] A mano: escribir a la bolsa de La Salle y apuntarme a un meetup de WordPress o JavaScript.

**Terminado cuando:** `python -m jobradar.main` imprime "hola" y llega un mensaje de prueba a Telegram.

---

## Fase 1 · Versión mínima: recoger, filtrar y avisar (1 fin de semana)

**Objetivo:** recibir cada día en Telegram ofertas reales de 3 fuentes.

1. **`models.py`**: la dataclass `Job`.
2. **`sources/base.py`**: una clase base con `fetch() -> list[Job]`. Todas las fuentes la cumplen.
3. **`sources/cido_rss.py`**: leer el RSS con `feedparser` y convertir cada entrada en un `Job`. *Empezar por aquí porque no necesita clave.*
4. **`sources/adzuna.py`**: llamar a la API con cada palabra clave y ubicación, y paginar hasta 2–3 páginas.
5. **`sources/jooble.py`**: lo mismo, pero con una petición POST.
6. **`db.py`**: crear la tabla, `exists(source, source_id)` y `save(job)`.
7. **`scoring/rules.py`**: puntuación por palabras clave (tabla de abajo).
8. **`notify/telegram.py`**: enviar las 10 mejores con formato:
   `🟢 82 · Programador junior PHP · Sabadell · Adzuna` + enlace.
9. **`main.py`**: unir todo.
10. **Tests:** guardar una respuesta real de cada fuente en `tests/fixtures/` y probar que se convierte bien en `Job`.
11. **`cron`:** `0 9,18 * * * cd ~/job-radar && .venv/bin/python -m jobradar.main >> logs/run.log 2>&1`

**Terminado cuando:** llevo 3 días recibiendo el resumen y no se repiten ofertas.

### Puntuación por reglas (primera versión)

| Señal | Puntos |
|---|---|
| "junior", "sin experiencia", "prácticas", "0-1 años" | +25 |
| Cada tecnología mía que aparece (WordPress, PHP, JavaScript, WooCommerce, SEO, Flutter, Python, MySQL) | +8 (máx. +32) |
| Barcelona o área metropolitana / remoto / híbrido | +15 |
| Contrato indefinido o plaza pública C1 | +10 |
| Mencionan mentoría, formación o equipo | +10 |
| "senior", "lead", "3+ años", "5 años" | −40 |
| Carnet de conducir o vehículo obligatorio | −20 |
| Contrato mercantil o freelance | −15 |
| Stack totalmente ajeno (SAP, COBOL, .NET senior…) | −20 |

Las reglas viven en un diccionario, así se ajustan sin tocar la lógica.

---

## Fase 2 · Puntuación inteligente y duplicados (1 semana, a ratos)

**Objetivo:** que el bot entienda las ofertas, no solo busque palabras.

1. **`dedupe.py`**:
   - Normalizar el título, la empresa y la ciudad (minúsculas, sin tildes, sin "(m/f)").
   - Calcular un `fingerprint`.
   - Si ya existe, guardar la nueva fuente pero no volver a avisar.
   - Opcional: similitud con `rapidfuzz` para casi-duplicados.
2. **`scoring/llm.py`**: solo para las ofertas con `rule_score` ≥ 30, para ahorrar coste. Le envía a la IA la oferta y `profile.md`, y le pide JSON:
   ```json
   {"score": 0-100, "why_fit": "...", "gaps": "...", "pitch": "qué destacar de mi experiencia"}
   ```
   - Validar el JSON y, si falla, quedarse con la puntuación por reglas.
   - Guardar en caché por `fingerprint` para no puntuar dos veces.
   - Poner un límite diario de llamadas.
3. **Telegram mejorado:**
   - Resumen diario: las 5 mejores, plazas públicas aparte, y "X nuevas, Y descartadas".
   - Botones en línea: ✅ Aplicada · ❌ Descartar · ⏰ Luego. Actualizan el `status` en la base de datos.
4. **Tests** de la normalización y de la eliminación de duplicados con casos reales.

**Terminado cuando:** cada oferta llega con su motivo y lo que me falta, y no recibo la misma oferta dos veces aunque venga de portales distintos.

---

## Fase 3 · Más fuentes (2 semanas, una fuente cada día o dos)

**Objetivo:** cobertura máxima. Seguir el orden de `priority` en `sources.yaml`.

1. **InfoJobs API**, la fuente con más volumen en España.
2. **`gmail_alerts.py`**:
   - Crear las alertas en LinkedIn, Indeed, Tecnoempleo y otros.
   - Crear una etiqueta de Gmail "job-alerts" con un filtro automático.
   - Leer con la API de Gmail, solo con permiso de lectura, y extraer el título, la empresa y el enlace de cada email.
   - Un parser por remitente.
3. **Feeds RSS:** WordPress Jobs, We Work Remotely, ATRI y Feina Activa, si tienen.
4. **APIs de remoto:** Remotive, RemoteOK (citando la fuente) y Himalayas.
5. Antes de cada fuente con `status: verify`, comprobar sus condiciones de uso y si hay RSS o API. Si no hay vía permitida, pasarla a `manual`.
6. **Registro de salud de las fuentes:** si una fuente falla o devuelve 0 ofertas 3 días seguidos, avisar por Telegram.

**Terminado cuando:** más de 10 fuentes activas, ejecutándose estables durante una semana.

---

## Fase 4 · El mercado oculto y las candidaturas (2 semanas)

**Objetivo:** enterarme antes que nadie y aplicar mejor.

1. **`companies.yaml`**: 100–200 empresas de Barcelona (agencias web y de marketing, startups del 22@, empresas con equipo de WordPress, PHP o Flutter), cada una con su ATS y su `slug`.
2. **`sources/ats.py`**: un lector genérico para Greenhouse, Lever, Workable, Recruitee, Personio y SmartRecruiters. Avisa **solo de ofertas nuevas** de esas empresas.
3. **Borrador de candidatura:** para cada oferta de 75 o más, generar un mensaje corto y personalizado (3–4 frases) y qué viñetas del CV destacar. **Revisarlo siempre yo antes de enviarlo.**
4. **Seguimiento:** comando `/stats` en Telegram, con ofertas vistas, aplicadas, respuestas y entrevistas.

**Terminado cuando:** recibo avisos de empresas concretas el mismo día que publican, y cada oferta buena viene con un borrador.

---

## Fase 5 · Portfolio (cuando todo funcione)

- [ ] **README** completo: qué hace, diagrama de arquitectura, captura de Telegram, cómo instalarlo y decisiones técnicas (por qué SQLite, por qué reglas antes de la IA, cómo se eliminan duplicados).
- [ ] GitHub Actions: ejecutar `ruff` y `pytest` en cada push.
- [ ] Opcional: un panel web sencillo (Flask y HTML) con el historial y los filtros.
- [ ] Opcional: un `Dockerfile` para que cualquiera pueda ejecutarlo.
- [ ] Añadirlo a polgonzalez.dev y a LinkedIn como proyecto.

---

## Calendario orientativo

| Semana | Fase | Resultado |
|---|---|---|
| 1 | 0 + 1 | Avisos diarios de CIDO, Adzuna y Jooble |
| 2 | 2 | Puntuación con IA, sin duplicados, botones en Telegram |
| 3–4 | 3 | Más de 10 fuentes, incluidas las alertas de LinkedIn e Indeed |
| 5–6 | 4 | Empresas vigiladas y borradores de candidatura |
| 7 | 5 | README, CI y publicación en el portfolio |

## Seguridad y buenas prácticas

- Las claves solo en `.env`. Nunca en el código ni en GitHub.
- La Raspberry con contraseña fuerte, SSH con clave y actualizaciones automáticas.
- La API de Gmail con permiso de **solo lectura**.
- Un `User-Agent` honesto, timeouts, reintentos con espera y como máximo 1–2 consultas al día por fuente.
- Si una web pide que no se automatice, se respeta: pasa a revisión manual.

## Qué voy a poder contar en una entrevista

- Integrar APIs REST distintas detrás de **una interfaz común** (patrón adaptador).
- **Normalización y eliminación de duplicados** de datos que vienen de fuentes inconsistentes.
- Un sistema de puntuación **híbrido**: reglas baratas primero e IA solo donde aporta, con caché y límite de coste.
- Tareas programadas, registros de ejecución y alertas cuando falla una fuente.
- Tests con fixtures para no depender de la red.
- Todo funcionando 24/7 en una Raspberry Pi.
