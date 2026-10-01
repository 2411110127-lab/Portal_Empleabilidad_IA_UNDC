# Producto · UNDC-Empleo

Redacta todos los documentos (requisitos, diseño, tareas) y todas las respuestas en español.

## Propósito
Traducir el perfil académico real de los estudiantes y egresados de Ingeniería de Sistemas de la UNDC a competencias comparables con el mercado, explicar sus brechas con cursos citados por su sílabo y medir la colocación con evidencia verificable, en lugar de la encuesta anual de baja respuesta de la Oficina de Seguimiento al Egresado.

## Usuarios
- **Estudiante activo** (principal): consulta su afinidad con una vacante, sus coincidencias, sus brechas y la ruta de cursos que las cierra.
- **Egresado**: postula a vacantes, recibe una ruta de cierre frente al mercado y controla con quién se comparten sus datos.
- **Empresa aliada**: publica vacantes con requisitos duros y deseables y revisa un ranking explicado de los postulantes que dieron su consentimiento.
- **Oficina de Seguimiento al Egresado (OSE)**: consulta indicadores agregados de colocación y participación por cohorte, nunca filas individuales.

## Capacidades del producto
1. Afinidad perfil-vacante mediante un pipeline determinista de tres etapas (filtro duro, puntaje, explicación); los datos se leen por el servidor MCP.
2. Explicación redactada por el modelo sobre una estructura ya decidida, revisada por un validador y sustituida por una plantilla determinista si se rechaza.
3. Ruta de cierre con cursos citados por código (MCP) y sílabo (RAG); sin respaldo, se declara y no se recomienda nada.
4. Ranking explicado de postulantes para la empresa, **sin descartar a ninguno**.
5. Publicación de vacantes por empresas aliadas, con requisitos duros y deseables.
6. Indicadores agregados por cohorte para la OSE, con aviso cuando la correlación proviene de datos sintéticos.
7. Gestión de consentimiento versionado, revocación inmediata, derechos ARCO y retención de 365 días.
8. Registro de asistencia a talleres mediante un webhook reproducible sin hardware (IoT opcional).

## Reglas no negociables
- **El agente no decide:** el filtro y el puntaje son funciones puras de Python, sin modelo de lenguaje. El mismo par perfil-vacante produce siempre el mismo puntaje y la misma huella SHA-256.
- **El modelo solo redacta:** no calcula, no ordena y no elige cursos. Al modelo nunca le llegan el nombre ni el código del estudiante.
- **R1 · Nunca descarta:** ordena y explica; ubica al final, con el motivo, a quien no cumple el filtro duro.
- **R2 · Atributos protegidos:** edad, género, foto, distrito, colegio y estado civil no existen como columnas; si aparecen en un CV, se descartan y queda constancia.
- **R3 · Explicación completa:** todo match nombra coincidencias, brechas y ruta de cierre.
- **R4 · Cita obligatoria:** toda recomendación de curso incluye código y sílabo.
- **R5 · Sin promesas:** ninguna salida promete empleo ni colocación; la decisión de contratar es siempre de una persona.
- **R6 · Consentimiento:** ningún dato se comparte con una empresa sin el consentimiento `compartir_con_empresas` registrado; se verifica al postular, al listar candidatos y al registrar la postulación.
- **Incertidumbre:** si la vacante no declara requisitos o el perfil no tiene competencias, el puntaje es nulo y se pide la aclaración concreta; no se inventa un número.
- **Privacidad (Ley N.° 29733):** la OSE solo ve cifras agregadas; en desarrollo se usan solo datos sintéticos con semilla fija, y la correlación sembrada en el generador se declara como tal.

## Fuera de alcance (semestre 2026-II)
Decidir contrataciones o descartar candidatos, prometer empleabilidad, usar edad, género, foto, distrito, colegio o estado civil, scraping de portales de empleo, reconocimiento facial y geolocalización, extracción de CV con modelo de lenguaje, filas individuales para la OSE, integración con los sistemas académicos reales de la UNDC o uso de datos reales, despliegue en producción y hardware IoT como entregable obligatorio (los nodos ESP32 son opcionales).
