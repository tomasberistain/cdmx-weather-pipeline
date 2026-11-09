# Weather data pipeline for Mexico City: Open-Meteo API → pandas → SQLite


Pipeline de extracción, limpieza y análisis de datos meteorológicos de la CDMX
utilizando la API pública de Open-Meteo.

## Sobre el flujo del pipeline

API Open-Meteo → Extracción (requests) → Limpieza (pandas) → CSV → SQLite → Consultas SQL

## La estructura de las carpetas del repositorio es:

	enki-data-engineer/
	├── data/                   * CSV y base de datos SQLite (generados al correr el pipeline)
	├── scripts/
	│   └── pipeline.py         * script principal: extracción, limpieza, carga y análisis
	├── sql/
	│   └── consultas.sql       * consultas SQL 
	├── docs/
	│   └── decisiones.md       * decisiones técnicas y mejoras propuestas
	├── requirements.txt
	└── README.md

## Sobre los requisitos

- Python 3.9+
- pip install -r requirements.txt

## Sobre la instalación

	bash
	git clone https://github.com/tomasberistain/enki-data-engineer.git
	cd enki-data-engineer
	pip install -r requirements.txt

## Uso

	bash
	python scripts/pipeline.py

El script genera automáticamente:
- data/datos_clima_cdmx.csv  
    datos limpios exportados
- data/clima_cdmx.db 
    base de datos SQLite con los mismos datos
- Resultados de las cuatro consultas SQL 
    impresos en consola
	
## Sobre las tecnologías utilizadas
- requests 
    consumo de API REST
- pandas 
    limpieza y transformación de datos
- sqlite3 
    base de datos local y consultas SQL
