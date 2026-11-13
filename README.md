# Laboratorio de OpenSearch: instalación local, dashboard, búsqueda facetada y API REST

## 1. Objetivo

Implementar un entorno local de OpenSearch con Docker Compose, cargar datos de ejemplo, construir un dataset manual, visualizar la información en OpenSearch Dashboards, crear un dashboard con tres visualizaciones, configurar búsqueda facetada y explorar directamente el API REST mediante `curl` y Dev Tools.

## 2. Tecnologías utilizadas

- Ubuntu/Linux
- Docker 29.9.3
- Docker Compose 5.7.1
- OpenSearch 3.9.0
- OpenSearch Dashboards
- API REST de OpenSearch
- Dev Tools

## 3. Estructura sugerida del repositorio

```text
s2026q2a-opensearch-sjaimes/
├── README.md
├── Dashboard/
│   ├── Imagenes del paso a paso
│   ├── Imagenes del Dasboard y las visualizaciones
│   └── Imagenes consultas mediante Devtol y consola linux
└── Docker/
    └── docker-compose.yml
```

> No se debe subir el archivo `.env`, porque contiene la contraseña local de OpenSearch.

---

## 4. Requisitos previos

Verificar Docker y Docker Compose:

```bash
docker --version
docker compose version
docker ps
```

Si `docker ps` presenta un error de permisos, verificar la pertenencia al grupo `docker`:

```bash
id
getent group docker
```

Después de agregar el usuario al grupo `docker`, es necesario cerrar completamente la sesión de Ubuntu y volver a entrar.

Configurar el valor requerido por OpenSearch:

```bash
sudo sysctl -w vm.max_map_count=262144
echo 'vm.max_map_count=262144' | sudo tee /etc/sysctl.d/99-opensearch.conf
sudo sysctl --system
sysctl vm.max_map_count
```

Resultado esperado:

```text
vm.max_map_count = 262144
```

---

## 5. Crear el proyecto local

```bash
mkdir -p ~/opensearch-laboratorio
cd ~/opensearch-laboratorio
```

Crear el archivo `.env` con una contraseña fuerte:

```bash
printf '%s\n' 'OPENSEARCH_INITIAL_ADMIN_PASSWORD=REEMPLAZAR_CON_CONTRASENA_SEGURA' > .env
chmod 600 .env
```

La contraseña debe contener mayúsculas, minúsculas, números y caracteres especiales. Para evitar exponer credenciales, el repositorio solo debe incluir `.env.example`:

```bash
printf '%s\n' 'OPENSEARCH_INITIAL_ADMIN_PASSWORD=REEMPLAZAR_CON_CONTRASENA_SEGURA' > .env.example
printf '%s\n' '.env' '*.log' > .gitignore
```

---

## 6. Configuración de Docker Compose

Crear `docker-compose.yml`:

```yaml
services:
  opensearch:
    image: opensearchproject/opensearch:latest
    container_name: opensearch-node
    environment:
      - cluster.name=opensearch-local
      - node.name=opensearch-node
      - discovery.type=single-node
      - bootstrap.memory_lock=true
      - OPENSEARCH_JAVA_OPTS=-Xms1g -Xmx1g
      - OPENSEARCH_INITIAL_ADMIN_PASSWORD=${OPENSEARCH_INITIAL_ADMIN_PASSWORD}
    ulimits:
      memlock:
        soft: -1
        hard: -1
      nofile:
        soft: 65536
        hard: 65536
    volumes:
      - opensearch-data:/usr/share/opensearch/data
    ports:
      - "9200:9200"
      - "9600:9600"
    networks:
      - opensearch-net

  opensearch-dashboards:
    image: opensearchproject/opensearch-dashboards:latest
    container_name: opensearch-dashboards
    ports:
      - "5601:5601"
    environment:
      OPENSEARCH_HOSTS: '["https://opensearch-node:9200"]'
    depends_on:
      - opensearch
    networks:
      - opensearch-net

volumes:
  opensearch-data:

networks:
  opensearch-net:
    driver: bridge
```

Validar la configuración:

```bash
docker compose config --quiet
```

Descargar las imágenes y levantar los servicios:

```bash
docker compose pull
docker compose up -d
docker compose ps
```

Consultar logs si ocurre un error:

```bash
docker compose logs --tail=100
docker compose logs -f opensearch
docker compose logs -f opensearch-dashboards
```

---

## 7. Verificación de OpenSearch y Dashboards

En los comandos siguientes se debe reemplazar `TU_CONTRASENA` por la contraseña configurada en `.env`.

Verificar OpenSearch:

```bash
curl -k -u 'admin:TU_CONTRASENA' https://localhost:9200
```

Consultar la salud del clúster:

```bash
curl -k -u 'admin:TU_CONTRASENA' \
  'https://localhost:9200/_cluster/health?pretty'
```

Listar índices:

```bash
curl -k -u 'admin:TU_CONTRASENA' \
  'https://localhost:9200/_cat/indices?v'
```

Verificar OpenSearch Dashboards:

```bash
curl -I http://localhost:5601
```

Una respuesta `302 Found` con redirección a `/app/login` indica que Dashboards está funcionando.

Abrir en el navegador:

- URL: `http://localhost:5601`
- Usuario: `admin`
- Contraseña: la definida en `.env`

---

## 8. Cargar el dataset oficial

Desde OpenSearch Dashboards:

1. Abrir **Home**.
2. Seleccionar **Add sample data**.
3. Buscar **Sample web logs**.
4. Presionar **Add data**.
5. Abrir **View data**.

El índice esperado es:

```text
opensearch_dashboards_sample_data_logs
```

Verificarlo desde la terminal:

```bash
curl -k -u 'admin:TU_CONTRASENA' \
  'https://localhost:9200/_cat/indices/opensearch_dashboards_sample_data_logs?v'
```

---

## 9. Dataset manual original: catálogo de cursos técnicos

Para cumplir el requisito de construir y cargar un ejemplo adicional, se creó el índice `catalogo-cursos-santiago`.

### 9.1 Crear el índice y su mapping

En **OpenSearch Dashboards → Dev Tools**, ejecutar:

```text
PUT /catalogo-cursos-santiago
{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 0
  },
  "mappings": {
    "properties": {
      "curso": { "type": "text" },
      "tecnologia": { "type": "keyword" },
      "nivel": { "type": "keyword" },
      "modalidad": { "type": "keyword" },
      "ciudad": { "type": "keyword" },
      "fecha_inicio": { "type": "date" },
      "duracion_horas": { "type": "integer" },
      "cupos": { "type": "integer" },
      "precio": { "type": "double" }
    }
  }
}
```

Los campos de tipo `keyword` se utilizaron para filtros exactos, agrupaciones y facetas. Los campos numéricos permiten sumas y promedios, mientras que `fecha_inicio` permite filtros de tiempo e histogramas por fecha.

### 9.2 Cargar ocho documentos con Bulk API

En Dev Tools, ejecutar:

```text
POST /_bulk?refresh=true
{"index":{"_index":"catalogo-cursos-santiago","_id":"1"}}
{"curso":"Fundamentos de pipelines de datos","tecnologia":"Python","nivel":"Básico","modalidad":"Presencial","ciudad":"Bogotá","fecha_inicio":"2026-11-16","duracion_horas":12,"cupos":25,"precio":0}
{"index":{"_index":"catalogo-cursos-santiago","_id":"2"}}
{"curso":"Modelado analítico en la nube","tecnologia":"Snowflake","nivel":"Intermedio","modalidad":"Virtual","ciudad":"Bogotá","fecha_inicio":"2026-11-18","duracion_horas":16,"cupos":40,"precio":90000}
{"index":{"_index":"catalogo-cursos-santiago","_id":"3"}}
{"curso":"Contenedores para proyectos de datos","tecnologia":"Docker","nivel":"Básico","modalidad":"Híbrida","ciudad":"Medellín","fecha_inicio":"2026-11-20","duracion_horas":8,"cupos":30,"precio":45000}
{"index":{"_index":"catalogo-cursos-santiago","_id":"4"}}
{"curso":"Orquestación de procesos empresariales","tecnologia":"Apache Airflow","nivel":"Avanzado","modalidad":"Virtual","ciudad":"Cali","fecha_inicio":"2026-11-23","duracion_horas":20,"cupos":20,"precio":160000}
{"index":{"_index":"catalogo-cursos-santiago","_id":"5"}}
{"curso":"Transformaciones SQL reproducibles","tecnologia":"dbt","nivel":"Intermedio","modalidad":"Presencial","ciudad":"Bogotá","fecha_inicio":"2026-11-25","duracion_horas":14,"cupos":24,"precio":110000}
{"index":{"_index":"catalogo-cursos-santiago","_id":"6"}}
{"curso":"Procesamiento distribuido de grandes datos","tecnologia":"Apache Spark","nivel":"Avanzado","modalidad":"Híbrida","ciudad":"Barranquilla","fecha_inicio":"2026-11-27","duracion_horas":24,"cupos":18,"precio":210000}
{"index":{"_index":"catalogo-cursos-santiago","_id":"7"}}
{"curso":"Búsquedas y agregaciones aplicadas","tecnologia":"OpenSearch","nivel":"Intermedio","modalidad":"Presencial","ciudad":"Bogotá","fecha_inicio":"2026-11-29","duracion_horas":10,"cupos":28,"precio":70000}
{"index":{"_index":"catalogo-cursos-santiago","_id":"8"}}
{"curso":"Automatización de infraestructura de datos","tecnologia":"Terraform","nivel":"Avanzado","modalidad":"Virtual","ciudad":"Medellín","fecha_inicio":"2026-12-01","duracion_horas":18,"cupos":22,"precio":145000}
```

La respuesta debe incluir:

```json
"errors": false
```

Verificar el conteo:

```text
GET /catalogo-cursos-santiago/_count
```

Resultado esperado: `8` documentos.

---

## 10. Crear el index pattern y explorar en Discover

1. Abrir **Dashboards Management**.
2. Entrar en **Index patterns**.
3. Seleccionar **Create index pattern**.
4. Escribir `catalogo-cursos-santiago*`.
5. Seleccionar `fecha_inicio` como campo temporal.
6. Guardar.
7. Abrir **Discover**.
8. Seleccionar el index pattern creado.
9. Configurar el intervalo absoluto:
   - Inicio: `2026-11-01 00:00:00`
   - Fin: `2026-12-05 23:59:59`
10. Presionar **Refresh**.

En Discover se agregaron como columnas:

- `curso`
- `tecnologia`
- `nivel`
- `modalidad`
- `ciudad`
- `fecha_inicio`
- `cupos`
- `precio`

---

## 11. Visualizaciones

### 11.1 Cursos por tecnología

- Tipo: **Vertical bar**.
- Métrica Y: `Count`.
- Bucket X: `Terms`.
- Campo: `tecnologia`.
- Tamaño: `10`.
- Nombre: `Cursos por tecnología`.

### 11.2 Cupos disponibles por fecha

- Tipo: **Line**.
- Métrica Y: `Sum`.
- Campo: `cupos`.
- Bucket X: `Date Histogram`.
- Campo temporal: `fecha_inicio`.
- Intervalo: `Daily` o `Auto`.
- Nombre: `Cupos disponibles por fecha`.

### 11.3 Precio promedio de los cursos

- Tipo: **Metric**.
- Agregación: `Average`.
- Campo: `precio`.
- Nombre: `Precio promedio de los cursos`.
- Resultado esperado con los ocho documentos: `103750`.

---

## 12. Crear el dashboard

1. Abrir **Dashboards**.
2. Seleccionar **Create dashboard**.
3. Seleccionar **Add**.
4. Agregar las tres visualizaciones.
5. Organizar y cambiar el tamaño de los paneles.
6. Verificar que la barra DQL esté vacía para evitar filtros involuntarios.
7. Guardar como `Oferta de cursos técnicos`.

---

## 13. Búsqueda facetada en el dashboard

Las facetas configuradas fueron:

- `ciudad`
- `nivel`
- `modalidad`
- `tecnologia`
- `fecha_inicio`

### Prueba 1: filtro por ciudad

Agregar el filtro:

```text
ciudad = Bogotá
```

Resultado esperado:

- 4 cursos.
- Tecnologías: Python, Snowflake, dbt y OpenSearch.
- Precio promedio: `67500`.

### Prueba 2: combinación de facetas

Mantener el filtro anterior y agregar:

```text
nivel = Intermedio
```

La combinación representa:

```text
ciudad = Bogotá AND nivel = Intermedio
```

Resultado esperado:

- 3 cursos.
- Tecnologías: Snowflake, dbt y OpenSearch.
- Cupos totales: `92`.
- Precio promedio: `90000`.

### Prueba 3: otra combinación

```text
modalidad = Virtual AND nivel = Avanzado
```

Resultado esperado:

- 2 cursos.
- Tecnologías: Apache Airflow y Terraform.
- Cupos totales: `42`.
- Precio promedio: `152500`.

### Consultas DQL equivalentes

```text
ciudad: "Bogotá" AND nivel: "Intermedio"
```

```text
nivel: "Básico" OR nivel: "Intermedio"
```

```text
NOT modalidad: "Virtual"
```

---

## 14. Exploración del API REST con curl

La instalación usa HTTPS y un certificado de demostración. Por eso se utiliza `-k` y autenticación con el usuario `admin`.

### 14.1 Obtener un documento

```bash
curl -k -u 'admin:TU_CONTRASENA' \
  'https://localhost:9200/catalogo-cursos-santiago/_search?size=1&pretty'
```

### 14.2 Contar documentos

```bash
curl -k -u 'admin:TU_CONTRASENA' \
  'https://localhost:9200/catalogo-cursos-santiago/_count?pretty'
```

### 14.3 Consultar mapping

```bash
curl -k -u 'admin:TU_CONTRASENA' \
  'https://localhost:9200/catalogo-cursos-santiago/_mapping?pretty'
```

---

## 15. Consultas equivalentes en Dev Tools

### Obtener un documento

```text
GET /catalogo-cursos-santiago/_search
{
  "size": 1
}
```

### Contar documentos

```text
GET /catalogo-cursos-santiago/_count
```

### Consultar mapping

```text
GET /catalogo-cursos-santiago/_mapping
```


### Filtrar cursos de Bogotá

```text
GET /catalogo-cursos-santiago/_search
{
  "query": {
    "term": {
      "ciudad": "Bogotá"
    }
  }
}
```

### Combinar ciudad y nivel

```text
GET /catalogo-cursos-santiago/_search
{
  "query": {
    "bool": {
      "filter": [
        { "term": { "ciudad": "Bogotá" } },
        { "term": { "nivel": "Intermedio" } }
      ]
    }
  }
}
```

Resultado esperado: tres documentos.

---

## 16. Faceted search mediante API REST

### 16.1 Obtener varias facetas

```text
GET /catalogo-cursos-santiago/_search
{
  "size": 0,
  "aggs": {
    "por_tecnologia": {
      "terms": {
        "field": "tecnologia",
        "size": 20
      }
    },
    "por_modalidad": {
      "terms": {
        "field": "modalidad"
      }
    },
    "por_nivel": {
      "terms": {
        "field": "nivel"
      }
    },
    "por_ciudad": {
      "terms": {
        "field": "ciudad"
      }
    }
  }
}
```

`size: 0` evita devolver documentos individuales y conserva únicamente los conteos de las agregaciones.

### 16.2 Aplicar un filtro y calcular facetas

```text
GET /catalogo-cursos-santiago/_search
{
  "size": 0,
  "query": {
    "bool": {
      "filter": [
        {
          "term": {
            "ciudad": "Bogotá"
          }
        }
      ]
    }
  },
  "aggs": {
    "niveles_disponibles": {
      "terms": {
        "field": "nivel"
      }
    },
    "tecnologias_disponibles": {
      "terms": {
        "field": "tecnologia"
      }
    },
    "precio_promedio": {
      "avg": {
        "field": "precio"
      }
    }
  }
}
```

Resultados esperados:

- 4 cursos en Bogotá.
- Nivel básico: 1.
- Nivel intermedio: 3.
- Precio promedio: `67500`.

### 16.3 Consulta por rango de fechas

```text
GET /catalogo-cursos-santiago/_search
{
  "query": {
    "range": {
      "fecha_inicio": {
        "gte": "2026-11-16",
        "lte": "2026-11-25"
      }
    }
  },
  "sort": [
    {
      "fecha_inicio": {
        "order": "asc"
      }
    }
  ]
}
```

Resultado esperado: cinco cursos.

---

## Referencias

- [OpenSearch Quickstart](https://docs.opensearch.org/latest/getting-started/quickstart/)
- [Instalación de OpenSearch con Docker](https://docs.opensearch.org/latest/install-and-configure/install-opensearch/docker/)
- [OpenSearch Dashboards](https://docs.opensearch.org/latest/dashboards/quickstart/)
- [API de búsqueda](https://docs.opensearch.org/latest/api-reference/search-apis/search/)
