# IABD-5075 · Laboratorio Hadoop

Despliegue de un clúster **Apache Hadoop** multi-nodo con **Docker Compose**, revisión de cómo HDFS guarda los bloques, simulación de la caída de un nodo y ejecución de un trabajo **MapReduce** (WordCount) sobre **YARN**.

**Autor:** José Raimundo Nieves Rosa
**Módulo:** 5074/5075 · Laboratorio (AEE)

---

## Arquitectura desplegada

Seis contenedores con la imagen `apache/hadoop:3` (Hadoop 3.3.6), conectados a una red interna de Docker llamada `hadoop-net`:

| Contenedor        | Rol                                  | Puerto publicado |
|-------------------|--------------------------------------|------------------|
| `namenode`        | Maestro de HDFS (guarda los metadatos) | `9870` (web)     |
| `datanode1`       | Trabajador de HDFS (guarda bloques)  | —                |
| `datanode2`       | Trabajador de HDFS (guarda bloques)  | —                |
| `datanode3`       | Trabajador de HDFS (guarda bloques)  | —                |
| `resourcemanager` | Maestro de YARN (reparte recursos)   | `8088` (web)     |
| `nodemanager`     | Trabajador de YARN (ejecuta tareas)  | —                |

### Parámetros principales (`docker/config`)

| Parámetro | Valor | Para qué sirve |
|-----------|-------|----------------|
| `dfs.blocksize` | 67108864 (64 MiB) | Tamaño máximo de cada bloque en HDFS |
| `dfs.replication` | 2 | Número de copias de cada bloque |
| `mapreduce.framework.name` | yarn | MapReduce se ejecuta sobre YARN |
| `yarn.nodemanager.resource.memory-mb` | 2048 | Memoria que ofrece el NodeManager |
| `yarn.nodemanager.resource.cpu-vcores` | 2 | CPUs virtuales que ofrece el NodeManager |

---

## Requisitos

- Docker Desktop (o Docker Engine + Docker Compose v2) en marcha
- Unos 6 GB de RAM libres
- Git

## Cómo arrancarlo

```bash
git clone https://github.com/jrnr-campuscamara/IABD-5075-LabHadoop-Nieves-Rosa.git
cd IABD-5075-LabHadoop-Nieves-Rosa/docker
docker compose up -d
docker compose ps
```

Interfaces web:

- NameNode (HDFS): <http://localhost:9870>
- ResourceManager (YARN): <http://localhost:8088>

Entrar en el maestro y comprobar que los 3 DataNodes están registrados:

```bash
docker exec -it namenode bash
hdfs dfsadmin -report
```

### Ejecutar el WordCount

Con solo 2048 MB en el NodeManager, hay que bajar la memoria del ApplicationMaster a 1024 MB. Si no, el trabajo se queda parado en `map 0%`, porque el AM ocupa toda la memoria (ver el informe):

```bash
hadoop jar $HADOOP_HOME/share/hadoop/mapreduce/hadoop-mapreduce-examples-*.jar wordcount \
  -Dyarn.app.mapreduce.am.resource.mb=1024 \
  /datos/wordcount/input /datos/wordcount/output
```

### Parar el clúster

```bash
docker compose down -v
```

---

## Estructura del repositorio

```
IABD-5075-LabHadoop-Nieves-Rosa/
├── .gitignore
├── README.md
├── docker/
│   ├── config
│   └── docker-compose.yml
├── evidencias/
│   ├── 01_namenode_ui_9870.png
│   ├── 02_hdfs_fsck_bloques.png
│   ├── 03_simulacion_fallo_datanode2.png
│   └── 04_yarn_wordcount_8088.png
└── informe/
    └── Nieves_Rosa_InformeLabHadoop.pdf
```

## Resumen de resultados

- **Bloques:** el fichero de 150 MiB se divide en 3 bloques (64 + 64 + 22 MiB). Con 2 copias de cada uno, son 6 bloques físicos en el clúster.
- **Tolerancia a fallos:** con `datanode2` apagado, el fichero se sigue leyendo entero (157 286 400 bytes). Unos 10,5 minutos después, HDFS marca el nodo como *Dead* y vuelve a copiar sus bloques a los otros nodos.
- **MapReduce:** el WordCount termina como `FINISHED / SUCCEEDED` y produce 21 palabras distintas en `part-r-00000`.

El análisis completo está en [`informe/Nieves_Rosa_InformeLabHadoop.pdf`](informe/Nieves_Rosa_InformeLabHadoop.pdf).
