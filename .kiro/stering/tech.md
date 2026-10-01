# Tecnología · UNDC-Empleo

Redacta todos los documentos en español.

## Pila de referencia (decidida por la docente; no proponer alternativas sin justificación)
- **Lenguaje:** Python 3.11+.
- **Backend:** FastAPI (`apps/api`), con SQLAlchemy 2, Alembic y Pydantic v2.
- **Frontend:** portal web para estudiante, empresa y oficina (`apps/web`).
- **Modelo de lenguaje:** Gemini API (Google AI Studio, nivel gratuito), consumida por REST con `urllib`; Groq como respaldo. Las claves van en las variables de entorno `GEMINI_API_KEY` y `GROQ_API_KEY`, nunca en el código ni en el repositorio.
- **Recuperación (RAG):** corpus institucional de sílabos y reglamentos en `packages/rag`; embeddings locales con `sentence-transformers` multilingüe; índice vectorial local en ChromaDB. Cada fragmento conserva metadatos: documento, código de curso y sección.
- **Datos:** base con datos sintéticos, generada por `data/sintetica/generador.py` con semilla fija (`SEED`).
- **Acceso a datos:** servidor MCP propio en Python, construido en la semana 4 (`packages/mcp-server`), **de solo lectura**, con 9 tools y 3 resources. La única escritura permitida es registrar una postulación, y se bloquea si no hay consentimiento.
- **Control de versiones:** Git y GitHub.

## Desviaciones declaradas
| Pila del curso | Proyecto | Justificación |
|---|---|---|
| HTML + JavaScript simple | Next.js 15, React 19, Tailwind v4, Recharts | Tres portales con 15 pantallas y gráficos de indicadores para la OSE. |
| SQLite | PostgreSQL 16 con pgvector, en Docker | 24 tablas, columnas JSONB y búsqueda vectorial. |
| Backend solo por MCP | La API arma los mismos contratos con SQL propio | El SDK de MCP es incompatible con el entorno de la API. `verificar_determinismo.py` exige la misma huella y el mismo puntaje por ambos caminos. |

- Se usan tres entornos virtuales por dependencias incompatibles: `.venv` para la API y el generador, `.venv-mcp` para el MCP y el agente, y `.venv-rag` para el RAG.
- El pipeline (`packages/agent/pipeline/`) usa solo la biblioteca estándar, para que lo importen los tres entornos.

## Restricciones técnicas
- El filtro y el puntaje no usan modelo de lenguaje. La aritmética se hace con `Decimal` y redondeo explícito, para que el número sea idéntico en cualquier máquina.
- MCP se usa para datos estructurados y RAG para conocimiento no estructurado; no se mezclan.
- Toda recomendación de curso incluye código y sílabo recuperados; sin respaldo, el sistema declara que no encontró información.
- El texto del modelo pasa por un validador. Si inventa cifras o códigos, promete empleo, descarta al candidato o menciona un atributo protegido, se usa la plantilla determinista y se registra el motivo.
- Las salidas estructuradas (JSON validado con Pydantic) cubren afinidad, ranking e indicadores.
- Cada ejecución se registra en `match_registro` con las tres etapas en JSONB, la versión del pipeline y la huella SHA-256 de la entrada.
- Sin claves de modelo configuradas, el sistema funciona igual, con la plantilla determinista.
- Costo cero en desarrollo: solo servicios de nivel gratuito o ejecución local.
- Toda regla dura tiene casos ejecutables en `tests/casos-agente/`, y `scripts\verificar.ps1` debe terminar en TODO EN VERDE.
- Las soluciones deben ser simples y legibles, de nivel universitario, sin sobreingeniería: sin microservicios, sin colas de mensajes y sin frameworks de agentes.
