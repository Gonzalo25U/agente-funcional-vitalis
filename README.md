# Agente Funcional — Centro Médico Vitalis

Evaluación Parcial N°2 — **Ingeniería de Soluciones con IA (ISY0101)**, Duoc UC
Encargo: *Desarrollo de un Agente Funcional*

Continuación de EP1 (router + RAG, ver `README_EP1.md`). En EP2 el flujo
fijo de EP1 se reemplaza por un **agente con uso autónomo de herramientas**,
con memoria de corto y largo plazo y re-planificación ante condiciones
cambiantes.

## Descripción del proyecto

Asistente virtual para un centro médico ficticio (Centro Médico Vitalis) que
automatiza el trabajo repetitivo de recepción: responder consultas
administrativas (horarios, convenios, valores, preparación de exámenes),
revisar disponibilidad de horas, registrar solicitudes y dejar redactados
mensajes para que una persona de recepción los revise. El agente **nunca da
consejo clínico**: ante una consulta de salud, deja registrada una
derivación a un profesional humano.

## Integrantes

- [Nombre integrante 1]
- [Nombre integrante 2]

## Stack tecnológico

| Componente | Tecnología |
|---|---|
| Framework de agente | [LangChain](https://python.langchain.com/) — `create_tool_calling_agent` + `AgentExecutor` |
| Proveedor / modelo LLM | [OpenRouter](https://openrouter.ai/) (API compatible con OpenAI), modelo `openai/gpt-6-luna`, `reasoning.effort = low` |
| Memoria de corto plazo | `ConversationSummaryBufferMemory` (LangChain), una por paciente |
| Memoria de largo plazo | Archivo JSON indexado por RUT |
| Recuperación semántica (RAG) | [ChromaDB](https://www.trychroma.com/) + `sentence-transformers` (`paraphrase-multilingual-MiniLM-L12-v2`) |
| Extracción de PDFs | `pypdf` |
| Trazabilidad | `pandas` → CSV |
| Entorno | Google Colab / Jupyter local |

## Estructura del repositorio

```
├── README.md                            # este archivo (EP2)
├── README_EP1.md                        # README de la entrega anterior
├── notebook/
│   ├── agente_funcional_vitalis.ipynb   # EP2: agente con herramientas y memoria
│   └── agente_rag_vitalis.ipynb         # EP1: router + RAG (referencia)
├── documentos/                          # PDFs oficiales indexados en el RAG (opcional)
├── diagramas/
│   ├── arquitectura_ep2.png             # diagrama de orquestación (EP2)
│   └── arquitectura_ep2.svg
├── evidencia/                           # CSV exportados por el notebook
│   ├── registro_trazabilidad_agente_vitalis.csv
│   ├── registro_solicitudes_vitalis.csv
│   └── registro_mensajes_borrador_vitalis.csv
├── informe/                             # informe técnico (máx. 5 páginas)
└── .env.example
```

## Cómo ejecutar el sistema

### Opción A — Google Colab (recomendado)

1. Sube `notebook/agente_funcional_vitalis.ipynb` a [Google Colab](https://colab.research.google.com/).
2. En el panel **Secrets** (ícono 🔑) crea, con el toggle **"Acceso al notebook"** activado:

   | Secret | Valor |
   |---|---|
   | `LLM_API_KEY` | tu API key de OpenRouter (la entregada por el docente) |
   | `LLM_BASE_URL` | `https://openrouter.ai/api/v1` |

   El modelo (`openai/gpt-6-luna`) está fijo en el código porque no es un dato
   sensible. **No pegues la API key dentro del notebook ni la subas al repositorio.**
3. Ejecuta las celdas en orden. La celda 3 abre un selector de archivos: sube
   los PDFs de `documentos/` si los tienes (opcional; sin ellos el agente usa
   la base documental simulada de horarios, convenios y FAQ).
4. La primera vez, la celda 4 descarga el modelo de embeddings (~470 MB).
5. Celda 10: escenario de planificación adaptativa. Celda 11: casos de
   prueba. Celda 12: exporta los CSV de evidencia.
6. Celda 13: conversación libre con el agente. Pide un RUT (Enter para
   entrar como anónimo; si lo escribes dentro de un mensaje, el agente
   vincula la conversación a ese RUT). El RUT `12.345.678-9` trae un
   antecedente de ejemplo en la memoria de largo plazo.

### Opción B — Local (Jupyter)

```bash
pip install langchain langchain-openai langchain-core langchain-classic python-dotenv chromadb sentence-transformers pandas pypdf jupyter
cp .env.example .env     # completar LLM_API_KEY y LLM_BASE_URL
jupyter notebook notebook/agente_funcional_vitalis.ipynb
```

La celda de subida de PDFs usa `google.colab.files`; en local se omite
automáticamente y se usa la base documental simulada.

## Arquitectura y orquestación

![Diagrama de orquestación](diagramas/arquitectura_ep2.png)

Flujo de un turno:

1. El **orquestador** (`agente_vitalis_v2`) recibe la consulta, detecta si
   el paciente mencionó su RUT (y migra la memoria anónima acumulada a ese
   RUT), y carga la memoria de corto plazo del paciente.
2. El **agente** (`AgentExecutor`) recibe el system prompt, el historial y
   la consulta. En cada paso el LLM decide si responde o llama una
   herramienta; la observación vuelve al agente, que decide el siguiente
   paso (máximo 6 iteraciones).
3. Se guarda el turno en la memoria de corto plazo y se registra la
   trazabilidad (RUT, herramientas usadas, número de pasos, respuesta).

### Herramientas (IL2.1 / IE1)

| Herramienta | Tipo | Función |
|---|---|---|
| `buscar_informacion_clinica` | Consulta | RAG sobre documentos administrativos (ChromaDB, 4 fragmentos más similares, con fuente) |
| `consultar_memoria_paciente` | Consulta | Antecedentes guardados del paciente en la memoria de largo plazo |
| `verificar_disponibilidad` | Razonamiento | Cupo simulado por profesional y día; si no hay, devuelve alternativas |
| `registrar_solicitud` | Escritura | Registra una solicitud de hora o derivación para recepción |
| `redactar_mensaje_recepcion` | Escritura | Redacta un borrador de mensaje para que recepción lo revise (no lo envía) |

El agente elige la herramienta a partir de su **docstring** (que describe
cuándo usarla y cuándo no) y de las reglas del system prompt. Para agregar
una herramienta nueva: función Python con `@tool` y docstring claro, sumarla
a la lista `herramientas`, reconstruir el agente y, si tiene dependencias de
orden con otras, agregar la regla al system prompt.

### Memoria y recuperación de contexto (IL2.2 / IE3, IE4)

- **Corto plazo:** una `ConversationSummaryBufferMemory` por RUT. Mantiene
  los últimos mensajes completos y, sobre 800 tokens, resume los turnos más
  antiguos con el LLM, de modo que el contexto no crece indefinidamente en
  conversaciones largas. Vive en RAM del proceso.
- **Largo plazo:** `memoria_largo_plazo_vitalis.json`, indexado por RUT
  normalizado (sin puntos, con guión y dígito verificador en mayúscula).
  Persiste en disco mientras exista el entorno; en Colab hay que descargarlo
  o montar Drive para conservarlo entre sesiones.
- **Recuperación semántica:** el RAG indexa los documentos como vectores en
  ChromaDB con embeddings multilingües; cada consulta trae solo los 4
  fragmentos más relevantes en vez de cargar todos los documentos.

### Planificación y toma de decisiones (IL2.3 / IE5, IE7)

Escenario de la celda 10: el paciente pide hora con el Dr. Soto el lunes,
día marcado sin cupo en la disponibilidad simulada. El agente (1) verifica
disponibilidad, (2) recibe "sin cupo" con alternativas, (3) en vez de
registrar la hora pedida, **cambia el plan**: ofrece la alternativa o redacta
un mensaje a recepción. Los casos de la celda 11 cubren las demás rutas:
consulta administrativa (RAG), consulta de memoria, consulta clínica
(derivación, sin responder), consulta de convenios y agendamiento con cupo.

Reglas de comportamiento (system prompt): 0) solo temas del centro, lo demás
se redirige con cortesía sin derivar a recepción; 1) nunca consejo clínico;
2) información administrativa siempre vía RAG; 3) consultar memoria de largo
plazo una vez al recibir el RUT; 4) verificar disponibilidad antes de
registrar una hora; 5) respuestas breves con fuente; 6) el historial de la
sesión se usa directamente, sin herramienta.

## Evidencia de pruebas

Los CSV de `evidencia/` se generan en la celda 12 y registran, por turno, la
herramienta(s) usada(s), el número de pasos y la respuesta, además de las
solicitudes y los mensajes borrador producidos por las herramientas de
escritura.

## Iteraciones y decisiones de diseño (depuración)

Problemas detectados durante el desarrollo y su corrección:

1. **Cambios de API en LangChain.** `create_tool_calling_agent`,
   `AgentExecutor` y las clases de memoria no se importan desde el mismo
   módulo según la versión instalada; se agregó una importación con
   respaldo a `langchain_classic`.
2. **Conteo de tokens con modelos no-OpenAI.** `ConversationSummaryBufferMemory`
   necesita contar tokens, y `ChatOpenAI` lanza `NotImplementedError` con
   nombres de modelo de OpenRouter. Se usa una subclase con conteo
   aproximado (~4 caracteres por token) solo para la memoria.
3. **Memoria repartida entre "anónimo" y el RUT.** Si el paciente daba su
   RUT a mitad de la conversación, el historial quedaba bajo otra clave. El
   orquestador detecta el RUT en el texto (con o sin puntos y guión) y
   migra la memoria; el identificador usa un formato canónico único.
4. **Agente derivaba todo a recepción.** Sin una regla de alcance, hasta una
   pregunta sin relación con el centro terminaba derivada. Se agregó la
   regla 0 del system prompt.
5. **Confusión entre memoria de largo plazo y la conversación actual.** Al
   recibir un RUT, el agente consultaba el JSON y, al no encontrar nada,
   decía no recordar lo ya conversado. Se aclaró en el system prompt
   (regla 3 y regla 6).

## Limitaciones conocidas

- La disponibilidad de horas es simulada; en producción sería una consulta
  al sistema de agendamiento del centro.
- La memoria de largo plazo es un JSON plano sin control de concurrencia ni
  cifrado; con datos reales de pacientes se necesitaría una base de datos con
  control de acceso.
- La memoria de largo plazo no se escribe automáticamente desde la
  conversación: hoy se alimenta con datos precargados.
- Las herramientas corren en el mismo proceso que el agente; exponerlas en
  un servidor aparte (por ejemplo, como servidor MCP o API REST) permitiría
  desplegarlas y asegurarlas por separado, a costa de latencia y de
  autenticación entre servicios.

## Referencias

- Chase, H. (2022). *LangChain* [Software]. https://github.com/langchain-ai/langchain
- LangChain. (s. f.). *LangChain documentation*. https://python.langchain.com/
- OpenRouter. (s. f.). *OpenRouter API documentation*. https://openrouter.ai/docs
- Chroma. (s. f.). *Chroma documentation*. https://docs.trychroma.com/
- Reimers, N., & Gurevych, I. (2019). Sentence-BERT: Sentence embeddings using Siamese BERT-networks. *Proceedings of EMNLP-IJCNLP 2019*. https://arxiv.org/abs/1908.10084
- Fuentes públicas indexadas en el RAG (ver `README_EP1.md`): ChileAtiende (Bono de Fonasa), Subsecretaría de Previsión Social (Comprar un bono de Fonasa), FALP (instructivo de ayuno), Clínica Bupa Antofagasta (recolección de orina).

## Declaración de uso de Inteligencia Artificial

Se utilizó Claude (Anthropic) como apoyo para estructurar el código del
agente, depurar errores técnicos (versiones de LangChain, conteo de tokens,
manejo de memoria por RUT), generar el diagrama de orquestación y redactar
este README. Las decisiones de diseño, la validación de resultados y las
reflexiones individuales de las conclusiones deben ser elaboradas por el
equipo sin apoyo de IA, conforme a la pauta de evaluación.

## Licencia

Proyecto académico para la asignatura ISY0101 — Ingeniería de Soluciones con
IA, Duoc UC. Uso educativo únicamente.
