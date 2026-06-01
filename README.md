# PRIVIA — Agente Multi-LLM de Auditoría de Privacidad

> **Proyecto final del curso _Simulación basada en Agentes_** · Junio 2026
>
> Integrantes: De la Fuente Sanhueza, Gonzalo · Jiménez Mercado, Miguel · Rivera Orellana, Carolina · Rodríguez Parco, Julissa

---

## 🧭 Índice rápido

| Para… | Sección |
|---|---|
| Entender por qué existe PRIVIA | [§1 Caso de negocio](#1-caso-de-negocio) |
| Ver la arquitectura del sistema | [§2 Arquitectura](#2-arquitectura) |
| Saber por qué cada decisión técnica | [§3 Decisiones de diseño](#3-decisiones-de-diseño) |
| Correr el notebook | [§4 Cómo correrlo](#4-cómo-correrlo) |
| Ver qué encontramos al iterar | [§5 Hallazgos del proceso](#5-hallazgos-del-proceso) |
| Diferencias con la arquitectura del informe | [§6 Académico vs. productivo](#6-alcance-académico-vs-arquitectura-productiva) |
| Referencia técnica (veredictos, Redis, credenciales) | [§7 Referencia](#7-referencia) |
| Mirar la estructura del repo | [§8 Estructura del repositorio](#8-estructura-del-repositorio) |

---

## 1. Caso de negocio

La **Ley 21.719** de Protección y Tratamiento de Datos Personales entra en vigencia en **diciembre 2026** en Chile. Las multas van desde 5.000 UTM (infracción leve) hasta **20.000 UTM** (gravísima). El contexto inmediato:

- **54%** de empresas chilenas no tiene protección de datos adecuada; las implementaciones típicas toman 12–18 meses.
- **37%** tiene políticas para gestionar el uso no controlado de IA sobre datos corporativos.
- En 2025 se registraron **8,8 billones de intentos de ataque** en Chile.

La adopción acelerada de IA generativa agrava todos estos riesgos. Detectar problemas de privacidad **en etapa de diseño** reduce hasta un **80%** el costo de remediación frente a detectarlos en producción.

> Cifras tomadas del informe del proyecto (Sección "Caso de negocio"). Las fuentes originales —reportes sectoriales de adopción de IA, registros públicos de incidentes en Chile 2025 y la Ley 21.719— se detallan ahí.

**PRIVIA** es un agente multi-LLM que actúa como auditor senior de privacidad: recibe la descripción técnica de una arquitectura (APIs, bases de datos, soluciones con IA), recupera evidencia normativa desde un índice vectorial Redis, identifica PII en el catálogo de datos y entrega un **veredicto estructurado** (`OK | ISSUES | CORRECTED | INCOMPLETE`) con citas trazables a la Ley 21.719, NIST CSWP 40 y la política interna del banco.

---

## 2. Arquitectura

Pipeline secuencial con **LangGraph**, 5 nodos + un fallback `incompleto`:

```
INPUT (descripción de arquitectura)
  ↓
[0] Sanitizer        → bloquea/scrubea PII real (RUT, email, IP, tarjeta, teléfono CL, pasaporte)
  ↓
[1] Orquestador (GPT-4o)
                    → clasifica: legal | technical | complex | incomplete | validation_only
  ↓             ↘
[2] Workers      [incompleto] → solicita antecedentes y termina
   ├─ Query expansion (gpt-4o-mini)  → reescribe la consulta como conceptos jurídico-técnicos
   ├─ tool_search_normativa          → KNN coseno sobre Redis (top-K=5, umbral 0.45)
   └─ tool_query_catalog             → Data Catalog simulado (12 campos representativos)
  ↓
[3] Auditor (GPT-4o)       → redacta el reporte preliminar SOLO con evidencia recuperada
  ↓
[4] Fiscalizador (GPT-4o)  → valida citas, PII residual, consistencia y formato
  ↓
END  →  audit_status: OK | ISSUES | CORRECTED  +  AUDIT_LOG en memoria
```

**Componentes y modelos:**

| Componente | Motor | Rol |
|---|---|---|
| Sanitizer | Regex puro | Detecta/redacta PII real antes del LLM |
| Orquestador | GPT-4o | Clasifica la consulta y decide qué workers invocar |
| Query expansion | GPT-4o-mini | Reescribe la consulta para mejorar el match RAG |
| Worker RAG | text-embedding-3-small (1536 dims) | KNN coseno sobre 23 chunks normativos |
| Worker Catalog | Lookup local | Sensibilidad, PII, dueño y ubicación de cada campo |
| Auditor | GPT-4o | Redacta el reporte con la evidencia recuperada |
| Fiscalizador | GPT-4o | QA independiente del reporte preliminar |

**Reglas duras (forzadas en código, no solo en prompt):**

- `rag_hits == 0` → `evidence_gap = True` → **ISSUES** obligatorio.
- `retrieval_score < 0.45` → `weak_citation` → degrada a **ISSUES** aunque el LLM diga OK.
- PII detectada en el input → scrubbing + `pii_scrubbed = True` antes de tocar el LLM.
- PII **real** (regex de valores, no categorías) detectada en el reporte → **ISSUES**.
- El fiscalizador **respeta el veredicto del auditor como piso**: si el auditor escribió `VEREDICTO PRELIMINAR: ISSUES`, el final no puede ser OK.

---

## 3. Decisiones de diseño

**Por qué multi-LLM y no un solo modelo.** Auditar privacidad no es chatear: requiere clasificar, buscar evidencia, redactar y *fiscalizar*. Separar responsabilidades por agente reduce alucinaciones (el auditor solo cita lo que el RAG recuperó) y permite usar modelos más baratos donde no se necesita razonamiento profundo (`gpt-4o-mini` para query expansion). En el roadmap del informe queda habilitado el escalado a agentes adicionales (revisión contractual, ciberseguridad, evaluación de impacto) sin reescribir los existentes.

**Por qué Redis como vector store.** El informe contempla migrar a Redis local en contenedor para garantizar residencia chilena de datos bajo Ley 21.719. La capa de KNN coseno es 100% código del proyecto (no usa `RediSearch` ni extensiones), portable sin cambios entre Redis Cloud y Redis on-prem.

**Por qué query expansion antes del KNN.** El corpus normativo es texto jurídico denso; las consultas reales son descripciones técnicas de arquitecturas. Estilísticamente lejanas en el espacio de embeddings → scores bajos. La query expansion reescribe la consulta como conceptos jurídicos (`datos biométricos; tratamiento de datos sensibles; transferencia internacional…`) y sube los scores ~0.15. Costo marginal: ~$0.0003 por consulta.

**Por qué reglas duras en código además del prompt.** Un prompt puede decir "no apruebes sin evidencia" y aun así el LLM aprueba. El código fuerza `audit_status = 'ISSUES'` cuando `evidence_gap` o `weak_citations > 0` o hay PII real. El prompt es defensa primaria; el código es defensa final.

**Por qué prompts en archivos `.md` separados.** Los 4 system prompts viven en `prompts/*.md` para poder versionarlos en git como contenido (no como JSON escapado dentro del notebook), revisarlos en GitHub, y editarlos sin reabrir Jupyter. El loader del notebook los carga con `Path().read_text()` y aplica `.format(SCORE_UMBRAL=...)` cuando hay placeholders.

**Por qué el fiscalizador como agente independiente.** Si auditor y fiscalizador son el mismo modelo, comparten sesgos. La arquitectura productiva del informe usa Claude Sonnet para fiscalizar (proveedor distinto a GPT-4o). La versión académica usa GPT-4o en ambos para simplificar credenciales, declarándolo como trade-off (ver §6).

---

## 4. Cómo correrlo

### 4.1 Opción A — Google Colab (recomendado)

1. Sube el `.ipynb` a Colab.
2. En el panel **🔑 Secrets** (ícono lateral izquierdo) crea los 4 secrets:

   | Secret name | Ejemplo |
   |---|---|
   | `OPENAI_API_KEY` | `sk-proj-...` |
   | `REDIS_HOST` | `redis-12345.redislabs.com` |
   | `REDIS_PORT` | `14159` |
   | `REDIS_PASSWORD` | (cadena entregada por Redis Cloud) |

3. Ejecuta las celdas en orden. El **setup de corpus** detecta Colab y, si `/content/corpus/` o `/content/prompts/` están vacíos, te pide subir los archivos con un diálogo.
4. La indexación corre **una sola vez** (~30–60 s). Si `normativa:index` ya tiene chunks, omite.

> Redis Cloud gratuito → https://redis.io/try-free/ → crea una BD → copia *Endpoint* y *Default user password*.

### 4.2 Opción B — Local

```bash
cd privia/
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env       # completar OPENAI_API_KEY y REDIS_*
jupyter notebook PRIVIA_Agente_Auditoria_Redis_v1.ipynb
```

La celda de credenciales carga automáticamente `.env` vía `python-dotenv` cuando corre fuera de Colab.

### 4.3 Casos de prueba incluidos

| Caso | Input | Veredicto esperado | Por qué |
|---|---|---|---|
| 1 — Biometría en cloud | Autenticación biométrica, vectores en AWS US, scoring IA, sin DPIA | **ISSUES** | Transferencia internacional + datos sensibles sin base legal documentada |
| 2 — Logs con PII en el prompt | Texto con RUT, email e IP reales | **OK** + `pii_scrubbed=True` | El sanitizador redacta PII antes de llegar al LLM; el sistema cumple (RBAC, AES-256, DPO) |
| 3 — Consulta vaga | `"¿Es nuestro sistema legal?"` | **INCOMPLETE** | El orquestador detecta falta de antecedentes y aborta sin invocar workers |

---

## 5. Hallazgos del proceso

La implementación inicial **siempre devolvía ISSUES** — el sistema era robusto (no aprobaba nada sin evidencia) pero inservible (jamás aprobaba nada). Cinco iteraciones empíricas, todas basadas en outputs reales:

| Fix | Síntoma observado | Causa | Solución | Métrica antes → después |
|---|---|---|---|---|
| **1** | Todos los `retrieval_score < 0.60` → todo era `weak_citation` | Umbral 0.60 sin calibrar al corpus real | Bajar a `SCORE_UMBRAL = 0.45` | Caso 1: **5 weak → 0 weak** |
| **2** | Scores topaban en ~0.59 incluso para queries relevantes | Texto jurídico denso vs descripción técnica = lejos en embedding | Query expansion con `gpt-4o-mini` antes del KNN | Top-score Caso 1: **0.59 → 0.72** |
| **3** | Fiscalizador marcaba `pii_leak` por mencionar "RUT" como categoría | Prompt no distinguía valor real (`12345678-9`) de nombre de campo (`usuarios.rut`) | Regex programático sobre el reporte + prompt explicando la diferencia | Falsos positivos PII en Caso 2: **eliminados** |
| **4** | Auditor decía ISSUES, fiscalizador lo subía a OK borrando los hallazgos | El fiscalizador validaba citas y PII pero no leía el veredicto del auditor | Parsear `VEREDICTO PRELIMINAR` y usarlo como piso del `audit_status` | Caso 1 preserva ISSUES legítimo |
| **5** | Fiscalizador alucinaba `weak_citation` aunque el código contaba 0 | `PROMPT_FISCALIZADOR` hardcodeaba `< 0.60` desactualizado | Convertir el prompt a f-string con `{SCORE_UMBRAL}` + instrucción explícita: "confía en el conteo programático" | Coherencia código ↔ prompt |

**Aprendizajes que cruzan los 5 fixes:**

1. **Las reglas duras en prompt son insuficientes.** Cada vez que confiamos solo en lenguaje natural, el LLM encontró formas de violarlas. Las reglas críticas tienen que estar en código.
2. **Los embeddings genéricos no alcanzan para texto normativo chileno.** Query expansion barata compensa donde un modelo de embeddings más caro hubiera sido la opción obvia y costosa.
3. **El fiscalizador debe respetar al auditor, no rehacerlo.** Su rol es QA, no segunda opinión sustantiva.
4. **Los parámetros se propagan o se desincronizan.** Bajar un umbral en código sin tocar el prompt creó alucinaciones; parametrizar vía `{SCORE_UMBRAL}` lo cerró.

---

## 6. Alcance académico vs. arquitectura productiva

El **informe** describe una arquitectura de referencia productiva. El **notebook** implementa una versión académica que conserva la lógica esencial (multi-LLM, RAG, fiscalizador independiente, reglas duras) pero simplifica la infraestructura para que corra en Colab/local sin levantar servicios externos.

| # | Productivo (informe) | Académico (notebook) | Justificación |
|---|---|---|---|
| 1 | Fiscalizador en **Claude Sonnet** (proveedor distinto al auditor) | Fiscalizador en **GPT-4o** | Evita exigir API key de Anthropic. Trade-off: misma familia de modelo en auditor y fiscalizador → posibles sesgos compartidos. |
| 2 | Trazabilidad en **PostgreSQL** (`audit_interactions` con hash del request) | Lista en memoria `AUDIT_LOG` | Colab no garantiza persistencia. La estructura ya es JSON serializable, migrable sin tocar el pipeline. |
| 3 | **Redis local** en Docker (residencia CL) | **Redis Cloud** Azure US | Decisión de demo. El informe explica la ruta de migración. |
| 4 | Workers expuestos como **MCP server** stdio | Funciones Python invocadas directamente desde el grafo | El transporte `direct` está permitido por el informe para prototipos. |
| 5 | **FastAPI** + endpoint `/audit` | Llamada directa a `auditar_arquitectura(...)` | El input ya entra estructurado al `StateGraph`. |
| 6 | Data Catalog corporativo vía **SQL** | Lista hardcoded de 12 entradas representativas | El contrato (`tool_query_catalog(keywords) -> dict`) es idéntico. |
| 7 | Sanitizer con políticas parametrizables | Sanitizer con 6 patrones fijos | Cubre el set PII relevante para los casos de prueba. |

Las reglas duras (`evidence_gap`, `weak_citation`, `pii_scrubbed`, piso del veredicto) están implementadas idénticas a las del diseño productivo.

---

## 7. Referencia

### 7.1 Veredictos

| Veredicto | Cuándo aparece |
|---|---|
| `OK` | El auditor no encontró riesgos sustantivos, hay evidencia normativa con `score ≥ 0.45`, sin PII expuesta, y el fiscalizador validó |
| `ISSUES` | `evidence_gap`, `weak_citation`, PII real, inconsistencia técnica, **o** el auditor identificó al menos un riesgo sustantivo (el fiscalizador no puede borrar esto) |
| `CORRECTED` | El fiscalizador detectó errores menores en el reporte y los corrigió directamente |
| `INCOMPLETE` | Consulta sin antecedentes técnicos suficientes — el orquestador aborta sin invocar workers |

### 7.2 Claves Redis generadas por el sistema

```
normativa:index          → SET con todos los MD5 de los chunks indexados
normativa:doc:<md5>      → JSON con document_name, document_type, article, section,
                            content, document_uri, criticality, topic, source_owner,
                            updated_at, embed_model, embed_dims, indexed_at
normativa:emb:<md5>      → bytes float32 del vector (1536 dims)
```

Comandos de mantenimiento desde el notebook:

```python
ver_estado_redis()              # estado actual del índice
indexar_normativa()             # solo indexa si normativa:index está vacío
indexar_normativa(forzar=True)  # re-indexa todo (gasta tokens de embedding)
REDIS_CLIENT.delete('normativa:index')  # limpia el índice (requiere re-indexar)
```

### 7.3 Política de credenciales

- Las credenciales **nunca** se escriben en el notebook ni en el repo. La celda de credenciales las resuelve en orden: **Colab Secrets → variables de entorno → `getpass`**.
- `.env` está en `.gitignore`; solo `.env.example` (con placeholders) se versiona.
- Si una API key cae en un commit o en un chat se considera comprometida: revocar en https://platform.openai.com/api-keys y rotar la password de Redis desde el panel de Redis Cloud.

---

## 8. Estructura del repositorio

```
privia/
├── PRIVIA_Agente_Auditoria_Redis_v1.ipynb   # Notebook ejecutable (Colab o local)
├── corpus/                                  # Base normativa indexada en Redis
│   ├── ley_21719.json                       #   10 chunks
│   ├── nist_cswp40.json                     #    8 chunks
│   └── politica_banco_general.json          #    5 chunks
├── prompts/                                 # System prompts de cada agente (editables)
│   ├── orquestador.md
│   ├── query_expansion.md
│   ├── auditor.md
│   └── fiscalizador.md                      # Template — {SCORE_UMBRAL} se interpola al cargar
├── requirements.txt                         # Dependencias Python para correr local
├── .env.example                             # Plantilla de credenciales (.env real está gitignored)
├── .gitignore
└── README.md
```

El corpus suma **23 chunks** validados contra el schema `NormativaChunk` (Pydantic) que define el notebook. Los 4 prompts y los 3 JSON del corpus pueden editarse sin tocar código: el notebook los carga al inicio de la sesión.
