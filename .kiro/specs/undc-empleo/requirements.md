# Requirements Document

## Introduction

UNDC-Empleo es un asistente organizacional agente de empleabilidad y seguimiento a egresados para la Escuela Profesional de Ingeniería de Sistemas de la Universidad Nacional de Cañete (UNDC). El sistema traduce el perfil académico de estudiantes y egresados a competencias comparables con el mercado laboral, calcula la afinidad con vacantes concretas de forma determinista, recomienda rutas de cierre basadas en cursos con sílabo registrado y provee indicadores agregados de colocación a la Oficina de Seguimiento al Egresado, todo ello respetando el consentimiento informado y la privacidad de los datos personales (Ley N.° 29733).

---

## Glossary

- **Sistema**: el conjunto de componentes de UNDC-Empleo (API FastAPI, servidor MCP, módulo RAG, agente y portal web).
- **Pipeline**: el módulo determinista de tres etapas (filtro duro → puntaje → explicación) que calcula la afinidad entre un perfil y una vacante, sin modelo de lenguaje.
- **Filtro_Duro**: la primera etapa del Pipeline que verifica el cumplimiento de cada requisito declarado como obligatorio en la vacante.
- **Calculadora_Puntaje**: la segunda etapa del Pipeline que asigna un valor numérico de 0 a 100 usando aritmética Decimal sin modelo de lenguaje.
- **Redactor**: el componente que invoca el modelo de lenguaje únicamente para generar el texto explicativo sobre una estructura de datos ya calculada.
- **Validador**: el componente que revisa el texto producido por el Redactor e identifica si contiene cifras inventadas, códigos no existentes, promesas de empleo, descarte de candidatos o mención de atributos protegidos.
- **Plantilla_Determinista**: el texto de explicación de respaldo que el Sistema usa cuando el Validador rechaza la salida del Redactor o cuando no hay clave de modelo configurada.
- **Servidor_MCP**: el servidor de acceso a datos construido en la semana 4, de solo lectura, con 9 herramientas y 3 recursos; la única escritura permitida es registrar una postulación.
- **RAG**: el módulo de recuperación aumentada que consulta el corpus de sílabos y reglamentos institucionales de la UNDC usando ChromaDB y sentence-transformers.
- **Perfil**: el conjunto de competencias, cursos aprobados, proyectos y talleres verificados de un estudiante o egresado, almacenado en la base de datos sintética.
- **Vacante**: la oferta publicada por una empresa aliada que declara requisitos duros y deseables para un puesto de trabajo.
- **Postulación**: el registro de la intención de un estudiante o egresado de aplicar a una vacante, condicionado al consentimiento `compartir_con_empresas`.
- **Consentimiento**: el registro versionado que expresa la autorización explícita del estudiante o egresado para compartir sus datos con empresas aliadas.
- **Huella_SHA256**: el hash criptográfico SHA-256 del par (perfil_id, vacante_id) junto con el puntaje calculado, utilizado para verificar el determinismo del Pipeline.
- **Brecha**: la diferencia entre una competencia exigida por una vacante y las competencias acreditadas en el Perfil del candidato.
- **Ruta_Cierre**: la lista ordenada de cursos de la UNDC con código y sílabo registrado que el Sistema recomienda para cubrir las Brechas identificadas.
- **Taller_Verificado**: la asistencia registrada mediante el webhook de registro de talleres, usada como evidencia adicional de competencia sin modificar el CV ni la historia académica.
- **OSE**: la Oficina de Seguimiento al Egresado de la UNDC, usuaria de los indicadores agregados.
- **Cohorte**: el conjunto de egresados del mismo año de egreso.
- **Atributo_Protegido**: cualquiera de los siguientes campos: edad, género, foto, distrito, colegio, estado civil.
- **Indicador_Agregado**: cifra estadística calculada sobre una Cohorte completa que no expone datos de filas individuales.
- **Estudiante**: usuario autenticado con rol de estudiante activo de Ingeniería de Sistemas de la UNDC.
- **Egresado**: usuario autenticado con rol de egresado de Ingeniería de Sistemas de la UNDC.
- **Empresa**: usuario autenticado con rol de empresa aliada registrada en el Sistema.
- **Oficina**: usuario autenticado con rol de Oficina de Seguimiento al Egresado.
- **Webhook_Taller**: el endpoint HTTP del Sistema que recibe el evento de asistencia verificada a un taller y desencadena el recálculo del puntaje.

---

## Requirements

### Requisito 1: Cálculo determinista de afinidad perfil-vacante

**Historia de usuario:** Como estudiante, quiero conocer mi afinidad con una vacante concreta para saber qué requisitos acredito y cuáles me faltan antes de postular.

#### Criterios de aceptación

1. WHEN un Estudiante autenticado solicita la afinidad con una Vacante concreta, THE Sistema SHALL invocar el Servidor_MCP para recuperar el Perfil del estudiante y los requisitos de la Vacante antes de ejecutar el Pipeline.
2. IF el Servidor_MCP no responde en un plazo de 10 segundos al recuperar el Perfil o la Vacante, THEN THE Sistema SHALL cancelar el cálculo, SHALL informar al Estudiante que el servicio no está disponible temporalmente y SHALL no ejecutar el Pipeline con datos incompletos.
3. WHEN el Pipeline recibe un par (perfil_id, vacante_id), THE Filtro_Duro SHALL evaluar cada requisito duro de la Vacante contra las competencias del Perfil y producir una lista de coincidencias y una lista de Brechas.
4. WHEN el Filtro_Duro completa su evaluación, THE Calculadora_Puntaje SHALL calcular un puntaje entero de 0 a 100 usando aritmética con tipo `Decimal` de Python y redondeo explícito `ROUND_HALF_UP` a 4 decimales intermedios, sin invocar ningún modelo de lenguaje.
5. THE Sistema SHALL producir la misma Huella_SHA256 para el mismo par (perfil_id, vacante_id) en cualquier ejecución y en cualquier máquina.
6. WHEN el Pipeline concluye, THE Sistema SHALL registrar en `match_registro` las tres etapas en JSONB, la versión del Pipeline en formato `MAJOR.MINOR.PATCH` y la Huella_SHA256 de la entrada.
7. IF la Vacante no declara ningún requisito, THEN THE Sistema SHALL establecer el puntaje como nulo y SHALL solicitar al publicador una aclaración concreta de los requisitos, sin inventar un puntaje.
8. IF el Perfil no tiene competencias registradas, THEN THE Sistema SHALL establecer el puntaje como nulo y SHALL informar al Estudiante qué información falta en su perfil, sin inventar un número.
9. THE Sistema SHALL mostrar al Estudiante el puntaje, la lista de coincidencias, la lista de Brechas y la Ruta_Cierre en una sola respuesta.

---

### Requisito 2: Explicación redactada por el modelo de lenguaje

**Historia de usuario:** Como estudiante, quiero recibir una explicación comprensible de mi afinidad con la vacante para entender el resultado sin interpretar números.

#### Criterios de aceptación

1. WHEN el Pipeline produce la estructura de afinidad, THE Redactor SHALL generar el texto explicativo a partir únicamente de la estructura calculada, sin recalcular puntajes ni reordenar candidatos.
2. WHEN el Redactor recibe la estructura de afinidad, THE Sistema SHALL transmitir al Redactor los campos de coincidencias, Brechas y Ruta_Cierre, y SHALL omitir el nombre, el código del estudiante y cualquier Atributo_Protegido.
3. WHEN el Redactor produce un texto, THE Validador SHALL rechazar ese texto si contiene al menos una de las siguientes condiciones: cifras no presentes en la estructura de entrada, códigos de curso no registrados en el Servidor_MCP, promesas de empleo o colocación, descarte explícito del candidato, o mención de un Atributo_Protegido.
4. IF el Validador rechaza el texto del Redactor, THEN THE Sistema SHALL sustituirlo inmediatamente por la Plantilla_Determinista sin reintentos adicionales y SHALL registrar el motivo del rechazo.
5. IF no hay clave de modelo de lenguaje configurada en las variables de entorno, THEN THE Sistema SHALL generar la explicación usando la Plantilla_Determinista sin intentar invocar el modelo.
6. IF el Perfil no tiene ninguna coincidencia con la Vacante, THEN THE Sistema SHALL usar la Plantilla_Determinista indicando que no se encontraron coincidencias, en lugar de intentar listar al menos una coincidencia inexistente.
7. THE Sistema SHALL incluir en toda explicación al menos una Brecha (cuando exista) y la Ruta_Cierre correspondiente.

---

### Requisito 3: Ruta de cierre con cursos de la UNDC

**Historia de usuario:** Como estudiante, quiero recibir una ruta de cursos para cerrar mis brechas para saber qué cursos de la propia universidad pueden ayudarme a mejorar mi perfil.

#### Criterios de aceptación

1. WHEN el Pipeline identifica al menos una Brecha, THE Sistema SHALL consultar el Servidor_MCP para obtener el código de cada curso candidato cuya competencia cubra la Brecha identificada y SHALL consultar el módulo RAG para recuperar el fragmento del sílabo correspondiente a ese código.
2. THE Sistema SHALL incluir en la Ruta_Cierre únicamente los cursos que tienen código registrado en el Servidor_MCP y para los cuales el módulo RAG devuelve al menos un fragmento de sílabo.
3. IF el módulo RAG no encuentra fragmento de sílabo para un curso candidato, THEN THE Sistema SHALL excluir ese curso de la Ruta_Cierre y SHALL declarar explícitamente en la respuesta que no encontró respaldo documental para esa Brecha.
4. THE Sistema SHALL citar en cada recomendación de curso el código del curso y el fragmento de sílabo recuperado por el módulo RAG.
5. IF no se encuentra ningún curso con código y sílabo para una Brecha, THEN THE Sistema SHALL declarar que no encontró cursos con respaldo suficiente para esa Brecha y SHALL presentar la Brecha al Estudiante sin recomendación.
6. IF el Servidor_MCP no está disponible al consultar cursos candidatos, THEN THE Sistema SHALL omitir la Ruta_Cierre para esa Brecha, SHALL declarar explícitamente que el catálogo de cursos no está disponible temporalmente y SHALL no recomendar ningún curso sin verificación de código.
7. IF el módulo RAG no está disponible al consultar sílabos, THEN THE Sistema SHALL omitir la Ruta_Cierre para las Brechas afectadas, SHALL declarar explícitamente que los sílabos no están disponibles temporalmente y SHALL no recomendar ningún curso sin verificación de sílabo.

---

### Requisito 4: Gestión de consentimiento y privacidad

**Historia de usuario:** Como egresado, quiero decidir con quién se comparten mis datos para poder postular sin perder el control de mi información.

#### Criterios de aceptación

1. THE Sistema SHALL verificar la existencia del consentimiento `compartir_con_empresas` en tres momentos: al intentar postular a una vacante, al listar candidatos para una Empresa y al registrar la Postulación en el Servidor_MCP.
2. IF un Estudiante o Egresado no tiene el consentimiento `compartir_con_empresas` registrado y activo, THEN THE Sistema SHALL bloquear la Postulación, SHALL no transmitir ningún dato del perfil a la Empresa y SHALL informar al usuario qué consentimiento necesita otorgar, indicando la acción específica requerida para otorgarlo.
3. WHEN un Estudiante o Egresado otorga el consentimiento `compartir_con_empresas`, THE Sistema SHALL registrar la versión del consentimiento, la fecha y hora de otorgamiento con precisión de segundos en UTC y el medio por el que fue otorgado (interfaz web, API o importación).
4. WHEN un Estudiante o Egresado revoca el consentimiento `compartir_con_empresas`, THE Sistema SHALL marcar la revocación en un plazo máximo de 5 segundos desde la solicitud, SHALL cesar la exposición de sus datos a empresas de forma que ninguna consulta posterior a la revocación devuelva datos del perfil y SHALL conservar el registro histórico de consentimiento durante 365 días conforme a la Ley N.° 29733.
5. THE Sistema SHALL omitir de todo cálculo, visualización y exportación los Atributos_Protegidos (edad, género, foto, distrito, colegio, estado civil), de modo que ningún resultado devuelto por el sistema contenga ninguno de dichos atributos.
6. IF un CV importado contiene uno o más Atributos_Protegidos, THEN THE Sistema SHALL descartar cada campo protegido antes de almacenar cualquier dato del CV y SHALL registrar la omisión indicando el nombre de cada campo descartado.
7. THE Sistema SHALL utilizar únicamente datos sintéticos generados con semilla fija en el entorno de desarrollo, sin procesar datos personales reales.
8. WHEN el Sistema verifica la existencia del consentimiento `compartir_con_empresas` y el registro de consentimiento no está disponible en un plazo máximo de 3 segundos, THE Sistema SHALL denegar la operación solicitada, SHALL no transmitir ningún dato del perfil y SHALL informar al usuario que el consentimiento no pudo ser verificado.
9. IF el consentimiento `compartir_con_empresas` registrado corresponde a una versión anterior a la versión vigente del texto de consentimiento, THEN THE Sistema SHALL tratar dicho consentimiento como inactivo y SHALL requerir al usuario que otorgue nuevamente el consentimiento antes de permitir la Postulación.

---

### Requisito 5: Publicación de vacantes por empresas aliadas

**Historia de usuario:** Como empresa aliada, quiero publicar una vacante con requisitos duros y deseables para que el sistema pueda compararla con los perfiles de los estudiantes y egresados.

#### Criterios de aceptación

1. WHEN una Empresa autenticada envía los datos de una nueva vacante, THE Sistema SHALL almacenar la vacante con un título de puesto de entre 1 y 150 caracteres, una descripción de entre 1 y 5 000 caracteres, y al menos un requisito clasificado como "duro" o "deseable".
2. IF una Empresa publica una vacante sin declarar ningún requisito clasificado como "duro" o "deseable", THEN THE Sistema SHALL rechazar el registro de la vacante, SHALL devolver un mensaje de error indicando que se requiere al menos un requisito clasificado, y SHALL conservar los datos ingresados para que la Empresa pueda corregirlos sin reingresar la información.
3. WHILE una vacante permanece activa, THE Sistema SHALL permitir a la Empresa propietaria actualizar los requisitos de la vacante, y SHALL completar el recálculo del puntaje de afinidad para todas las Postulaciones existentes asociadas a esa vacante en un plazo máximo de 60 segundos tras confirmar la actualización.
4. THE Sistema SHALL asociar cada vacante publicada a la Empresa autenticada que la creó, de modo que solo esa Empresa pueda modificarla o eliminarla.
5. IF una Empresa autenticada intenta modificar o eliminar una vacante que no le pertenece, THEN THE Sistema SHALL rechazar la operación, SHALL devolver un mensaje de error indicando acceso no autorizado, y SHALL mantener la vacante sin cambios.

---

### Requisito 6: Ranking explicado de postulantes para la empresa

**Historia de usuario:** Como empresa aliada, quiero ver un ranking explicado de los postulantes a mi vacante para priorizar entrevistas utilizando criterios transparentes.

#### Criterios de aceptación

1. WHEN una Empresa solicita el ranking de postulantes de una Vacante, THE Sistema SHALL incluir en la lista únicamente a los candidatos que tienen el consentimiento `compartir_con_empresas` activo en el momento de la consulta.
2. THE Sistema SHALL ordenar la lista filtrada por consentimiento de mayor a menor puntaje calculado por la Calculadora_Puntaje, sin excluir a ningún candidato de la lista filtrada.
3. WHEN un candidato no cumple el Filtro_Duro de la Vacante, THE Sistema SHALL ubicarlo al final de la lista después de los candidatos que sí cumplen el filtro duro, y SHALL mostrar explícitamente cada requisito duro incumplido como motivo; entre los candidatos que no cumplen el filtro duro el orden será de mayor a menor puntaje parcial.
4. WHEN dos o más candidatos tienen puntaje nulo, THE Sistema SHALL ordenarlos por fecha de postulación ascendente dentro del grupo de puntaje nulo.
5. THE Sistema SHALL omitir del ranking y de la explicación de cada candidato todos los Atributos_Protegidos.
6. THE Sistema SHALL mostrar para cada candidato la explicación generada por el Redactor o la Plantilla_Determinista, junto con las coincidencias y Brechas.
7. THE Sistema SHALL no tomar ninguna decisión de contratación ni etiquetar a ningún candidato como aceptado, rechazado, recomendado o descartado; la decisión es siempre de una persona.
8. IF la Vacante no tiene postulantes con consentimiento activo, THEN THE Sistema SHALL devolver una lista vacía e informar a la Empresa que aún no hay candidatos que hayan autorizado compartir sus datos.

---

### Requisito 7: Registro de postulación condicionado al consentimiento

**Historia de usuario:** Como estudiante, quiero postular a una vacante de forma controlada para que el sistema registre mi interés sin exponer mis datos sin permiso.

#### Criterios de aceptación

1. WHEN un Estudiante o Egresado inicia una Postulación, THE Sistema SHALL verificar mediante el Servidor_MCP que el consentimiento `compartir_con_empresas` está activo antes de ejecutar la escritura.
2. IF el consentimiento `compartir_con_empresas` no está activo en el momento de la Postulación, THEN THE Sistema SHALL cancelar la operación de escritura, SHALL no registrar la Postulación ni almacenar ningún dato parcial y SHALL notificar al usuario con un mensaje indicando que la postulación fue cancelada por falta de consentimiento activo.
3. IF el Servidor_MCP no responde en un plazo de 10 segundos durante la verificación del consentimiento, THEN THE Sistema SHALL cancelar la Postulación, SHALL no ejecutar la escritura y SHALL notificar al usuario que la verificación de consentimiento no pudo completarse.
4. WHEN el consentimiento está activo, THE Sistema SHALL invocar la única operación de escritura permitida del Servidor_MCP para registrar la Postulación con el identificador del candidato, el identificador de la Vacante y la marca de tiempo ISO 8601 en UTC.
5. WHEN la escritura en el Servidor_MCP se completa con éxito, THE Sistema SHALL confirmar al Estudiante o Egresado que la Postulación fue registrada, mostrando el nombre de la Vacante y el nombre de la Empresa correspondiente, sin revelar datos de otros candidatos.
6. IF la operación de escritura en el Servidor_MCP falla, THEN THE Sistema SHALL realizar un rollback completo de la operación, SHALL no dejar ningún dato parcialmente registrado y SHALL notificar al usuario que la Postulación no pudo completarse debido a un error del sistema.

---

### Requisito 8: Indicadores agregados de colocación para la OSE

**Historia de usuario:** Como responsable de la OSE, quiero consultar indicadores agregados de colocación por cohorte para disponer de evidencia verificable sobre la empleabilidad de los egresados.

#### Criterios de aceptación

1. WHEN un usuario con rol Oficina consulta los indicadores de una Cohorte, THE Sistema SHALL mostrar únicamente Indicadores_Agregados calculados sobre el total de la Cohorte, sin exponer filas individuales de estudiantes o egresados.
2. WHEN un usuario con rol Oficina consulta los indicadores de una Cohorte, THE Sistema SHALL calcular y mostrar al menos los siguientes Indicadores_Agregados: número de egresados con Postulación registrada, número de egresados con al menos una Postulación activa, distribución de puntajes de afinidad en los rangos [0–24], [25–49], [50–74] y [75–100], y tasa de participación calculada como (egresados con al menos una Postulación registrada ÷ total de egresados de la Cohorte con Perfil en el Sistema) × 100 expresada con un decimal.
3. IF algún Indicador_Agregado proviene de correlaciones sembradas en el generador de datos sintéticos, THEN THE Sistema SHALL mostrar un aviso explícito que indica que el dato proviene de datos sintéticos y no refleja la realidad de la Cohorte.
4. THE Sistema SHALL impedir que el rol Oficina acceda a datos individuales de estudiantes o egresados mediante cualquier endpoint o consulta.
5. WHILE el Sistema opera en el entorno de desarrollo, identificado por la presencia de la variable de entorno `SEMILLA_DATOS_FIJA` configurada, THE Sistema SHALL basar todos los Indicadores_Agregados exclusivamente en los datos sintéticos generados con esa semilla.
6. IF una Cohorte tiene menos de 5 egresados con Perfil registrado en el Sistema, THEN THE Sistema SHALL suprimir todos los Indicadores_Agregados de esa Cohorte y SHALL informar al usuario que la Cohorte no alcanza el mínimo requerido para mostrar indicadores.

---

### Requisito 9: Registro de asistencia a talleres como evidencia de competencia

**Historia de usuario:** Como estudiante, quiero que mi asistencia verificada a un taller cuente como evidencia para acreditar competencias que no figuran en mi historia académica.

#### Criterios de aceptación

1. WHEN el Webhook_Taller recibe un evento de asistencia verificada para un Estudiante y un taller concreto, THE Sistema SHALL registrar la asistencia con el identificador del estudiante, el identificador del taller, el identificador de la competencia asociada y la marca de tiempo ISO 8601 del evento.
2. WHEN se registra una asistencia mediante el Webhook_Taller, THE Sistema SHALL recalcular el puntaje de afinidad para todas las Vacantes cuyo campo `estado` sea `activa` en el perfil del Estudiante que incluyan la competencia asociada al taller.
3. WHEN una competencia registrada por asistencia a taller tiene un `competencia_id` que coincide exactamente con el `competencia_id` de una Brecha frente a una Vacante, THE Sistema SHALL marcar esa competencia como coincidencia acreditada por asistencia en el resultado de afinidad.
4. THE Sistema SHALL no modificar el CV ni la historia académica del Estudiante al registrar un Taller_Verificado; la asistencia es evidencia adicional exclusivamente.
5. IF el Webhook_Taller recibe un evento de asistencia cuyo par (`estudiante_id`, `taller_id`) ya existe en el registro, THEN THE Sistema SHALL descartar el evento sin crear un registro duplicado y sin retornar un error al emisor.
6. THE Sistema SHALL producir la misma Huella_SHA256 para el mismo par (`perfil_id`, `vacante_id`) cuando el conjunto de entradas al cálculo — incluyendo todos los registros de Taller_Verificado vigentes para ese perfil — sea idéntico, garantizando la reproducibilidad del recálculo.
7. IF el Webhook_Taller recibe un evento con campos obligatorios ausentes o con un `estudiante_id` o `taller_id` que no existe en el Sistema, THEN THE Sistema SHALL rechazar el evento sin crear ningún registro y SHALL retornar una indicación de error que identifique el campo inválido o ausente.

---

### Requisito 10: Determinismo verificable y registro de ejecuciones

**Historia de usuario:** Como desarrollador del equipo, quiero que el Pipeline sea verificable y auditable para garantizar que el mismo par perfil-vacante siempre produce el mismo resultado.

#### Criterios de aceptación

1. THE Pipeline SHALL calcular el puntaje de afinidad usando exclusivamente aritmética con tipo `Decimal` de Python con redondeo explícito `ROUND_HALF_UP` a 4 decimales en cada operación intermedia, sin operaciones de punto flotante nativas.
2. THE Sistema SHALL calcular la Huella_SHA256 a partir de la serialización canónica UTF-8 de la cadena `perfil_id|vacante_id|puntaje` donde `puntaje` es el valor entero final de 0 a 100, y SHALL almacenarla en `match_registro`.
3. WHEN el Pipeline completa el cálculo de la afinidad, THE Sistema SHALL almacenar la Huella_SHA256 en el campo correspondiente de `match_registro` antes de retornar la respuesta al solicitante.
4. WHEN se ejecuta el script `scripts/verificar.ps1`, THE Sistema SHALL comparar la Huella_SHA256 producida por la ruta MCP con la producida por la ruta SQL directa para el mismo par (perfil_id, vacante_id) y SHALL terminar con código de error distinto de cero si las huellas difieren.
5. THE Sistema SHALL almacenar en `match_registro` para cada ejecución del Pipeline: el resultado del Filtro_Duro en JSONB, el resultado de la Calculadora_Puntaje en JSONB, el resultado del Redactor o la Plantilla_Determinista en JSONB, la versión del Pipeline en formato `MAJOR.MINOR.PATCH` y la Huella_SHA256.
6. IF el módulo RAG no encuentra sílabo para un curso candidato durante la generación de la Ruta_Cierre, THEN THE Sistema SHALL registrar el código del curso no encontrado en el campo `advertencias` de `match_registro` como un elemento de un array JSON, sin interrumpir ni cancelar el cálculo en curso.
