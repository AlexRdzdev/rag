Instalar las siguientes librerias:

```python
!pip install -q transformers==4.41.2  
!pip install -q sentence-transformers==2.2.2  
!pip install -q chromadb==0.4.20
```

Importamos pandas

```python
import pandas as pd
```

Leemos el dataset utilizando Pandas, por cuestiones de no estresar demasiada memoria se utiliza un maximo de 1000 Registros

```python
news = pd.read_csv('labelled_newscatcher_dataset.csv', sep=';')  
MAX_NEWS = 1000  
DOCUMENT="title"  
TOPIC="topic"
```

```python
news["id"] = news.index  
news.head()  
subset_news = news.head(MAX_NEWS)
```

```python
import chromadb
```

Se inicializa el cliente de Chomadb apuntando al directorio actual

```python
chroma_client = chromadb.PersistentClient(path="")
```

```python
from datetime import datetime
```

Se crea una colección de ChromaDB:

```python
collection_name = "news_collection_" + str(int(datetime.now().timestamp()))  
if len(chroma_client.list_collections()) > 0 and collection_name in [chroma_client.list_collections()[0].name]:  
        chroma_client.delete_collection(name=collection_name)  

collection = chroma_client.create_collection(name=collection_name)

```

Se agrega el subset de noticias a la coleccion de Chromadb

```python
collection.add(  
    documents=subset_news[DOCUMENT].tolist(),  
    metadatas=[{TOPIC: topic} for topic in subset_news[TOPIC].tolist()],  
    ids=[f"id{x}" for x in range(MAX_NEWS)],  
)
```

Lista de preguntas y contextos

```python
questions = [  
    ("technology", "What are the latest trends in technology advancements?"),  
    ("economy", "How has the economy been affected by recent global events?"),  
    ("climate", "What are the most recent developments in climate change research?"),  
    ("artificial intelligence", "What are the key findings in the field of AI this year?"),  
    ("politics", "How is politics influencing social media trends?"),  
    ("space", "What are the latest scientific discoveries in space exploration?"),  
    ("healthcare", "What are the emerging trends in the healthcare industry?"),  
    ("entertainment", "How is the entertainment industry adapting to new digital platforms?"),  
    ("education", "What are the biggest challenges facing the education sector today?"),  
    ("transportation", "What recent innovations are shaping the future of transportation?")  
]
```

Script para ejecutar las preguntas a los 3 modelos distintos incluyendo contexto en el prompt:

```python
import time
import psutil
import pandas as pd
from transformers import pipeline, AutoModelForSeq2SeqLM, AutoModelForCausalLM, AutoTokenizer
from tqdm import tqdm  # Para mostrar progreso

# Definir los modelos a evaluar con sus tipos correspondientes
models = {
    "dolly": ("databricks/dolly-v2-3b", AutoModelForCausalLM),  # Dolly usa modelo causal
    "t5": ("t5-small", AutoModelForSeq2SeqLM),  # T5 usa modelo seq2seq
    "tinyllama": ("TinyLlama/TinyLlama-1.1B-Chat-v1.0", AutoModelForCausalLM)  # TinyLlama también es causal
}

# Nombre del archivo CSV para almacenar los resultados progresivamente
output_csv = "llm_model_comparison_con_contexto.csv"

# Inicializar el archivo CSV con encabezados
df_init = pd.DataFrame(columns=["Model", "Topic", "Question", "Answer", "Execution Time (s)",
                                "Memory Used (MB)", "Model Size (MB)"])
df_init.to_csv(output_csv, index=False)

# Proceso de evaluación con barra de progreso
for model_name, (model_path, model_class) in tqdm(models.items(), desc="Evaluando modelos"):
    print(f"\nCargando modelo: {model_name}")

    # Cargar modelo y tokenizer
    tokenizer = AutoTokenizer.from_pretrained(model_path)
    model = model_class.from_pretrained(model_path)
    pipe = pipeline("text-generation" if model_class == AutoModelForCausalLM else "text2text-generation",
                    model=model, tokenizer=tokenizer)

    # Calcular tamaño del modelo en MB
    model_size = sum(p.numel() for p in model.parameters()) * 4 / (1024 ** 2)

    for topic, question in tqdm(questions, desc=f"Procesando preguntas con {model_name}", leave=False):
        results = collection.query(query_texts=[topic], n_results=10)
        context = " ".join([f"#{str(i)}" for i in results["documents"][0]])

        prompt_template = f"""
        Relevant context: {context}
        Considering the relevant context, answer the question.
        Question: {question}
        Answer: """

        # Medición del uso de memoria antes
        mem_before = psutil.virtual_memory().used / (1024 ** 2)
        start_time = time.time()

        # Generar respuesta del modelo con max_new_tokens
        try:
            lm_response = pipe(prompt_template, max_new_tokens=100, do_sample=True, temperature=0.7)
            answer = lm_response[0]['generated_text'] if lm_response else "No response"
        except Exception as e:
            answer = f"Error: {str(e)}"

        print(answer)

        end_time = time.time()
        mem_after = psutil.virtual_memory().used / (1024 ** 2)

        # Calcular métricas
        execution_time = end_time - start_time
        memory_used = mem_after - mem_before

        # Almacenar resultados en un diccionario
        result_data = {
            "Model": model_name,
            "Topic": topic,
            "Question": question,
            "Answer": answer,
            "Execution Time (s)": execution_time,
            "Memory Used (MB)": memory_used,
            "Model Size (MB)": model_size
        }

        # Guardar el resultado en el archivo CSV inmediatamente
        df_temp = pd.DataFrame([result_data])
        df_temp.to_csv(output_csv, mode='a', header=False, index=False)

        print(f"Guardado: Modelo {model_name}, Tópico {topic}")

print(f"Todas las métricas han sido guardadas en '{output_csv}'. Puedes revisarlas en cualquier momento.")

```

Script para ejecutar las preguntas a los 3 modelos distintos sin incluir contexto en el prompt:

```python
import time
import psutil
import pandas as pd
from transformers import pipeline, AutoModelForSeq2SeqLM, AutoModelForCausalLM, AutoTokenizer
from tqdm import tqdm  # Para mostrar progreso

# Definir los modelos a evaluar con sus tipos correspondientes
models = {
    "dolly": ("databricks/dolly-v2-3b", AutoModelForCausalLM),  # Dolly usa modelo causal
    "t5": ("t5-small", AutoModelForSeq2SeqLM),  # T5 usa modelo seq2seq
    "tinyllama": ("TinyLlama/TinyLlama-1.1B-Chat-v1.0", AutoModelForCausalLM)  # TinyLlama también es causal
}

# Nombre del archivo CSV para almacenar los resultados progresivamente
output_csv = "llm_model_comparison_sin_contexto.csv"

# Inicializar el archivo CSV con encabezados
df_init = pd.DataFrame(columns=["Model", "Topic", "Question", "Answer", "Execution Time (s)",
                                "Memory Used (MB)", "Model Size (MB)"])
df_init.to_csv(output_csv, index=False)

# Proceso de evaluación con barra de progreso
for model_name, (model_path, model_class) in tqdm(models.items(), desc="Evaluando modelos"):
    print(f"\nCargando modelo: {model_name}")

    # Cargar modelo y tokenizer
    tokenizer = AutoTokenizer.from_pretrained(model_path)
    model = model_class.from_pretrained(model_path)
    pipe = pipeline("text-generation" if model_class == AutoModelForCausalLM else "text2text-generation",
                    model=model, tokenizer=tokenizer)

    # Calcular tamaño del modelo en MB
    model_size = sum(p.numel() for p in model.parameters()) * 4 / (1024 ** 2)

    for topic, question in tqdm(questions, desc=f"Procesando preguntas con {model_name}", leave=False):

        prompt_template = f"""
        answer the question.
        Question: {question}
        Answer: """

        # Medición del uso de memoria antes
        mem_before = psutil.virtual_memory().used / (1024 ** 2)
        start_time = time.time()

        # Generar respuesta del modelo con max_new_tokens
        try:
            lm_response = pipe(prompt_template, max_new_tokens=100, do_sample=True, temperature=0.7)
            answer = lm_response[0]['generated_text'] if lm_response else "No response"
        except Exception as e:
            answer = f"Error: {str(e)}"

        print(answer)

        end_time = time.time()
        mem_after = psutil.virtual_memory().used / (1024 ** 2)

        # Calcular métricas
        execution_time = end_time - start_time
        memory_used = mem_after - mem_before

        # Almacenar resultados en un diccionario
        result_data = {
            "Model": model_name,
            "Topic": topic,
            "Question": question,
            "Answer": answer,
            "Execution Time (s)": execution_time,
            "Memory Used (MB)": memory_used,
            "Model Size (MB)": model_size
        }

        # Guardar el resultado en el archivo CSV inmediatamente
        df_temp = pd.DataFrame([result_data])
        df_temp.to_csv(output_csv, mode='a', header=False, index=False)

        print(f"Guardado: Modelo {model_name}, Tópico {topic}")

print(f"Todas las métricas han sido guardadas en '{output_csv}'. Puedes revisarlas en cualquier momento.")

```
