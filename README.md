# Weather data pipeline for Mexico City: Open-Meteo API → pandas → SQLite

Pipeline de extracción, limpieza y análisis de datos meteorológicos de la CDMX
utilizando la API pública de [Open-Meteo](https://open-meteo.com/).

## Flujo del pipeline

```
API Open-Meteo → Extracción (requests) → Limpieza (pandas) → CSV → SQLite → Consultas SQL
```

## Estructura del repositorio

```
cdmx-weather-pipeline/
├── data/                      ← CSV y base de datos SQLite (generados al correr el pipeline)
├── scripts/
│   └── pipeline.py            ← script principal: extracción, limpieza, carga y análisis
├── sql/
│   └── consultas.sql          ← consultas SQL
├── docs/
│   └── decisiones_tecnicas.md ← decisiones técnicas y mejoras propuestas
├── requirements.txt
└── README.md
```

## Requisitos

- Python 3.9+

## Instalación

```bash
git clone https://github.com/tomasberistain/cdmx-weather-pipeline.git
cd cdmx-weather-pipeline
pip install -r requirements.txt
```

## Uso

```bash
python scripts/pipeline.py
```

El script genera automáticamente:

- `data/datos_clima_cdmx.csv` — datos limpios exportados
- `data/clima_cdmx.db` — base de datos SQLite con los mismos datos
- Los resultados de las cuatro consultas de `sql/consultas.sql`, impresos en consola

Los archivos de `data/` no se versionan; se regeneran en cada ejecución.

## Decisiones técnicas

Las decisiones de diseño y las mejoras propuestas están documentadas en
[`docs/decisiones_tecnicas.md`](docs/decisiones_tecnicas.md).

## Tecnologías

- [requests](https://requests.readthedocs.io/) — consumo de la API REST
- [pandas](https://pandas.pydata.org/) — limpieza y transformación de datos
- [sqlite3](https://docs.python.org/3/library/sqlite3.html) — base de datos local y consultas SQL
