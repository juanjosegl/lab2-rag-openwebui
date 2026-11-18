# Lab 2 — Pipeline RAG Documental con Open WebUI y Ollama

RAG 100 % local: **Ollama** (`qwen2.5:1.5b-instruct` + `nomic-embed-text`) como motor de inferencia y **Open WebUI** como orquestador y gestor documental. El lab compara el comportamiento del SLM *antes* y *después* de conectarlo a una base de conocimiento vectorial.

📄 **Informe entregable:** [`Entregable_Lab2_RAG_Juan_Jose_Gutierrez_Lasso.md`](Entregable_Lab2_RAG_Juan_Jose_Gutierrez_Lasso.md)

## Estructura

```
.
├── docker-compose.yml                 # Ollama + Open WebUI
├── politica_innovatech_2027.txt       # Documento privado del caso de estudio
├── Entregable_Lab2_RAG_*.md           # Informe técnico
└── evidencias/                        # Capturas, docker ps, ollama list y transcripciones
```

## Reproducir

```bash
docker compose up -d
docker exec -it ollama-rag-lab ollama pull qwen2.5:1.5b-instruct
docker exec -it ollama-rag-lab ollama pull nomic-embed-text
```

1. Abre http://localhost:3000 y crea la cuenta administradora (el primer usuario es admin).
2. *Admin Panel → Settings → Documents*: Embedding Model Engine = **Ollama**, modelo **`nomic-embed-text`** (ya viene precargado por variables de entorno).
3. *Admin Panel → Settings → Models → qwen2.5:1.5b-instruct → Advanced Params*: **Function Calling = Legacy** (ver la nota abajo).
4. *Workspace → Knowledge → +*: crea `Politicas-Innovatech-2027` y sube `politica_innovatech_2027.txt`.
5. En un chat nuevo escribe `#` y selecciona la colección antes de enviar el prompt.

## Notas técnicas

- **`Function Calling = Legacy`**: en Open WebUI v0.13 el modo por defecto (*native*) expone la Knowledge Base como una herramienta que el modelo debe invocar. Un SLM de 1,5B no la invoca y responde sin fuentes. En *Legacy*, Open WebUI recupera los chunks y los inyecta en el prompt (RAG clásico).
- **`OLLAMA_MAX_LOADED_MODELS=1`** y embeddings vía Ollama desde el arranque: son necesarios en máquinas con poca RAM para Docker (en este caso, 3 GB). Sin ellos, el kernel mata los procesos por falta de memoria (OOM).
