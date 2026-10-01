Stack y reglas técnicas · UNDC-Empleo
Decisiones vigentes
Producto: asistente organizacional agente de empleabilidad y seguimiento a egresados de la Escuela Profesional de Ingeniería de Sistemas, UNDC.
Estilo: monolito modular en FastAPI con portal renderizado en servidor (propuesto). Revisión hacia API REST + SPA si aparece un consumidor externo de la API o la interactividad supera formularios y tablas.
Orquestación del agente: flujo determinista propio en Python (ADR-001, propuesto, pendiente de firma). Alternativa de reversión antes de la semana 7: LangGraph solo con nodos deterministas. Tool calling nativo descartado (viola RNN-01 y RNN-02).
Lenguaje: Python, versión estable vigente [VERIFICAR]. Plantillas HTML: Jinja2 [VERIFICAR].
Servidor de aplicación: Uvicorn [VERIFICAR]. De XAMPP solo se usa la base de datos; Apache no sirve la aplicación.
Base de datos (temporal): MySQL/MariaDB de XAMPP [VERIFICAR cuál incluye la instalación]; motor InnoDB; collation utf8mb4_unicode_ci. Las etapas de match_registro se guardan en columnas JSON validadas porque MySQL/MariaDB no tiene JSONB (deuda aceptada en ADR-001; migración prevista a PostgreSQL). Acceso a datos: [PENDIENTE de ADR].
Servidor MCP: el propio de la semana 4, como proceso independiente. Solo lectura para perfiles, vacantes y cursos; única escritura: registrar postulación, condicionada al consentimiento compartir_con_empresas vigente.
RAG: ChromaDB con embeddings locales [VERIFICAR modelo de embeddings]. Cada fragmento conserva documento, código de curso y sección.
Modelo de lenguaje: Gemini como principal; Groq como respaldo solo ante HTTP 5xx o fallo de conexión de Gemini, nunca ante timeout. Timeout de 10 s. Sin claves o con salida rechazada: plantilla determinista.
Variables de entorno: GEMINI_API_KEY y GROQ_API_KEY (opcionales), semilla HMAC del webhook de talleres, SEED del generador sintético. Nunca se versionan; se documentan en .env.example.
Pruebas: pytest.
Control de versiones: GitHub; ramas main (estable), develop (integración), feature-<modulo> (trabajo).
Reglas técnicas obligatorias
filtro_duro y puntaje son funciones puras de Python: sin modelo de lenguaje, sin red, sin lectura de reloj ni azar.
El puntaje se calcula con Decimal y un modo de redondeo explícito declarado como constante [DEFINIR modo]; resultado entero 0–100 o nulo.
La huella SHA-256 se calcula sobre la entrada normalizada del par perfil-vacante (JSON canónico: claves ordenadas, separadores fijos). VERSION_PIPELINE es una constante semántica y se guarda en columna propia.
Orden fijo del pipeline: filtro → puntaje → ruta (MCP + RAG) → explicación. El modelo recibe la estructura ya calculada, sin nombre, código de usuario ni atributos protegidos, y solo devuelve texto.
Toda salida del modelo pasa por el Validador. Si la rechaza, se usa la plantilla y se registra el motivo. explicacion_origen siempre presente: plantilla_determinista (sin claves) o plantilla_por_fallo (fallo o timeout del modelo).
Prohibido crear columnas o campos para edad, género, foto, distrito, colegio o estado civil. Si llegan en una entrada, se descartan antes de almacenar y se registra el descarte (marca de tiempo, tipo de documento, nombres de atributos, sin valores).
Ningún candidato se elimina de un ranking: quien no cumple el filtro duro va al final con el requisito incumplido.
Un curso solo entra en la ruta de cierre con código vía MCP y sílabo vía RAG; sin ambos, se declara que no hay respaldo.
Consentimiento compartir_con_empresas verificado al postular y al listar candidatos, en la capa de servicio, no solo en la vista.
Rol oficina: solo cifras agregadas por cohorte, nunca cohortes con menos de 5 registros.
Desarrollo solo con datos sintéticos de data/sintetica/generador.py con SEED fija.
Ningún texto del sistema promete empleo, colocación ni contratación.
Convenciones
Nombres de dominio, rutas, tablas, plantillas y módulos en español.
Base de datos: snake_case y singular (match_registro, consentimiento, postulacion, asistencia_taller).
Python: módulos, funciones y variables en snake_case (PEP 8); clases en PascalCase; constantes en MAYÚSCULAS.
Rutas: kebab-case (/mi-afinidad, /vacantes/{id}/ranking); la API JSON bajo /api.
Commits en español con Conventional Commits (feat:, fix:, test:, docs:) [CONFIRMAR: el texto original se cortó aquí].