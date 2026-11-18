# Informe de Laboratorio 2: Pipeline RAG con Open WebUI y Ollama

- **Estudiante / Equipo:** Juan José Gutiérrez Lasso
- **Stack Utilizado:** qwen2.5:1.5b-instruct + nomic-embed-text + Open WebUI (v0.13.4) + Docker Compose
- **Fecha:** 18/11/2026
- **Repositorio:** https://github.com/juanjosegl/lab2-rag-openwebui

---

## 0. Despliegue y configuración (Fases 1 y 2)

| Componente | Detalle |
|---|---|
| `ollama-rag-lab` | `ollama/ollama:latest`, puerto `11434` |
| `open-webui-lab` | `ghcr.io/open-webui/open-webui:main` (v0.13.4), puerto `3000 → 8080` |
| SLM | `qwen2.5:1.5b-instruct` (986 MB) |
| Embeddings | `nomic-embed-text` (274 MB), motor **Ollama** |
| Chunking | Splitter por caracteres + markdown headers, chunk 1024 / overlap 100 |
| Vector DB | ChromaDB embebida en Open WebUI |

Evidencias: [`evidencias/docker_ps.txt`](evidencias/docker_ps.txt), [`evidencias/ollama_list.txt`](evidencias/ollama_list.txt) y la configuración de embeddings en [`evidencias/00_config_embeddings_ollama.jpg`](evidencias/00_config_embeddings_ollama.jpg).

**Ajustes respecto a la guía (y por qué):**

1. **`OLLAMA_MAX_LOADED_MODELS=1`** (la guía usa 2). La VM de Docker (Colima) tenía 3 GB de RAM. Con el SLM y el modelo de embeddings cargados a la vez, el kernel mató el proceso `llama-server` por falta de memoria (OOM) y el chat devolvía `llama-server process has terminated: signal: killed`. Con 1, Ollama carga un modelo, lo descarga y luego carga el otro (primero embebe la consulta, después genera la respuesta).
2. **`RAG_EMBEDDING_ENGINE=ollama` y `RAG_EMBEDDING_MODEL=nomic-embed-text` en el compose.** Es lo mismo que pide el paso 2.3, pero definido desde el arranque. Sin esto, Open WebUI carga en RAM el modelo por defecto `all-MiniLM-L6-v2` (sentence-transformers) y el contenedor entraba en bucle de reinicios por OOM. Aun así verifiqué y guardé el ajuste en *Admin Panel → Settings → Documents*.
3. **`Function Calling = Legacy` en el modelo** (*Admin Panel → Settings → Models → qwen2.5:1.5b-instruct → Advanced Params*). Open WebUI v0.13 usa por defecto *native tool calling*: la Knowledge Base se ofrece al modelo como una herramienta que él mismo debe decidir invocar. El SLM de 1,5B no la invocó, y su primera respuesta con `#Politicas-Innovatech-2027` fue *"Espero que estos datos te sean útiles. Te recomiendo revisar directamente con la empresa…"*, con **0 fuentes**. En modo **Legacy**, Open WebUI hace el RAG clásico: embebe la consulta, recupera los top-k chunks y los inyecta en el prompt. Desde ese cambio todas las consultas citaron fuentes. Evidencia: [`evidencias/00b_modelo_function_calling_legacy.jpg`](evidencias/00b_modelo_function_calling_legacy.jpg).

---

## 1. Comparativa: Antes del RAG (Línea Base)

**Prompt:** *"¿De cuánto es el auxilio de conectividad para teletrabajo en 2027 en Innovatech Colombia y cuál es el código interno para solicitarlo?"*, con `qwen2.5:1.5b-instruct` y sin archivos adjuntos.

**Respuesta del modelo (resumen fiel):** dijo que el auxilio "se calcula basado en las tendencias actuales de teletrabajo" y que "se puede presupuestar entre **100 a 200 dólares al mes**". Sobre el código, afirmó que "sería **"205-0568-2027"**" y recomendó verificarlo con la empresa.

**Análisis:** es una **alucinación**. El modelo no tiene acceso al documento privado, así que inventó una cifra en otra moneda (USD en lugar de COP) y un código con formato verosímil pero falso. No admitió que le faltaba la información: solo agregó una advertencia genérica al final. Esto muestra el riesgo de usar un SLM sin datos de referencia para preguntas sobre información interna. Ninguno de los dos datos coincide con la política real ($245.000 COP / `#INV-CON-2027`).

## 2. Comparativa: Después del RAG (Con Knowledge Base)

Se creó la colección **`Politicas-Innovatech-2027`** en *Workspace → Knowledge* y se indexó `politica_innovatech_2027.txt` en 2 chunks con `nomic-embed-text`. Luego se repitió exactamente el mismo prompt, adjuntando `#Politicas-Innovatech-2027`.

**Respuesta:** *"El auxilio de conectividad para teletrabajo en 2027 en Innovatech Colombia es de **$245.000 COP**. El código interno para solicitar este reembolso en el portal financiero es **#INV-CON-2027**."* (Open WebUI muestra *Retrieved 1 source* → `politica_innovatech_2027.txt`.)

**Fragmentos citados:** 2 chunks del documento. El primero, con **89,97 %** de relevancia, contiene las secciones 1 y 2 (auxilio y vacaciones). El segundo, con **79,13 %**, contiene la sección 3 (ciberseguridad).

**Batería de pruebas (Paso 4.3):**

| Tipo de prueba | Resultado obtenido | Relevancia top-1 | Veredicto |
|---|---|---|---|
| Dato exacto | $245.000 COP y `#INV-CON-2027` | 89,97 % | Correcto |
| Condicional / Rol (8 meses) | Máximo **7 días**, con aprobación del líder y de RRHH, a cargo de la **directora Laura Martínez** | 84,37 % | Correcto en los datos. Pero respondió **en inglés**, dejó etiquetas `<source id="1">` en el texto, añadió un párrafo de ciberseguridad que nadie pidió y deformó la rotación de llaves ("3-month keys every 12 months" en lugar de *trimestral*) |
| Protocolo técnico (fuga AWS) | **< 2 horas**, Slack **#sec-incidentes-urgente** o **ext. 809**, cifrado **AES-256** con rotación trimestral de llaves | 81,64 % | Correcto y completo |
| Anti-alucinación (cafetería) | "La información … no se encuentra en el contexto proporcionado. Por lo tanto, no puedo responder…" | 80,59 % | Correcto: no inventó nada |

**Análisis:** con RAG, la precisión factual pasó de 0/2 datos correctos a **4/4 pruebas correctas en el contenido**. Las respuestas reproducen literalmente las cifras, los códigos y los nombres del documento, y cada una muestra su fuente. Los errores que quedan son de **forma**, no de recuperación: el idioma, los artefactos de la plantilla de citas y el contenido extra en la prueba de vacaciones. Son limitaciones típicas de un modelo de 1,5B parámetros al seguir instrucciones largas. Se podrían reducir con un system prompt que exija responder en español y solo sobre lo preguntado, o con un modelo más grande (p. ej. `qwen2.5:3b` o `7b`).

## 3. Evaluación de Control de Alucinaciones (Fuera de Dominio)

**Prompt:** *"¿Cuál es el horario de atención de la cafetería los sábados en Innovatech?"* con `#Politicas-Innovatech-2027`.

**Respuesta:** *"Lo siento, pero la información específica sobre el horario de atención de la cafetería los sábados en Innovatech no se encuentra en el contexto proporcionado. Por lo tanto, no puedo responder a esa pregunta con base en la información disponible."*

**Análisis:** el modelo **respetó los límites del contexto recuperado**. Sin RAG había inventado datos, pero aquí reconoció que no tenía la información. Hay un matiz técnico importante: el retriever sí devolvió fragmentos (relevancia 80,59 % y 77,49 %). Con solo 2 chunks y `RELEVANCE_THRESHOLD = 0`, cualquier consulta recupera el documento completo. Lo que evitó la alucinación fue la **plantilla RAG de Open WebUI**, que indica al modelo responder solo con el contexto y decir cuando no sabe, y no un filtrado por similitud. En una base de conocimiento grande convendría:

- Definir un `RELEVANCE_THRESHOLD` (p. ej. 0,5–0,6) o activar la búsqueda híbrida (BM25 + reranker), para que las consultas fuera de dominio no traigan fragmentos irrelevantes.
- Mantener en la plantilla la instrucción explícita de *"si el contexto no contiene la respuesta, dilo"*.
- Revisar siempre las citas: la UI permite auditar qué fragmento respaldó cada respuesta.

**Conclusión general:** RAG convirtió un SLM que alucinaba en un asistente que da datos exactos y verificables sobre información privada, todo de forma local (Ollama + Open WebUI, sin APIs externas). El principal hallazgo práctico es que, en Open WebUI v0.13, los modelos pequeños necesitan el modo *Function Calling = Legacy*: con el modo nativo no invocan la herramienta de conocimiento y el RAG simplemente no ocurre.

---

## 4. Evidencias Adjuntas (Capturas de Pantalla)

- [x] **Captura 1:** respuesta del modelo sin RAG (alucinación): [`evidencias/01_captura_sin_rag.jpg`](evidencias/01_captura_sin_rag.jpg)
- [x] **Captura 2:** documento `politica_innovatech_2027.txt` indexado en *Workspace → Knowledge*: [`evidencias/02_knowledge_base_indexada.jpg`](evidencias/02_knowledge_base_indexada.jpg)
- [x] **Captura 3:** respuesta fundamentada con `#Politicas-Innovatech-2027` y citas desplegadas: [`evidencias/03_captura_con_rag_dato_exacto.jpg`](evidencias/03_captura_con_rag_dato_exacto.jpg) y [`evidencias/03b_fragmentos_citados.jpg`](evidencias/03b_fragmentos_citados.jpg)

Evidencias adicionales:

- Configuración de embeddings: [`00_config_embeddings_ollama.jpg`](evidencias/00_config_embeddings_ollama.jpg)
- Function Calling en Legacy: [`00b_modelo_function_calling_legacy.jpg`](evidencias/00b_modelo_function_calling_legacy.jpg)
- Prueba condicional: [`04_prueba_condicional_vacaciones.jpg`](evidencias/04_prueba_condicional_vacaciones.jpg)
- Protocolo técnico: [`05_prueba_protocolo_tecnico.jpg`](evidencias/05_prueba_protocolo_tecnico.jpg)
- Anti-alucinación: [`06_prueba_anti_alucinacion.jpg`](evidencias/06_prueba_anti_alucinacion.jpg)
- Transcripciones completas con fuentes y puntajes de relevancia: [`respuestas_chats.json`](evidencias/respuestas_chats.json)
