---
# multilingual page pair id, this must pair with translations of this page. (This name must be unique)
lng_pair: id_blog_1
title: Conectando y descargando datos de Gaia

# post specific
# if not specified, .name will be used from _data/owner/[language].yml
#author: ""
# multiple category is not supported
category: cluster
# multiple tag entries are possible
tags: [cluster, connection]
# thumbnail image for post
img: ":post_pic2.jpg"
# disable comments on this page
comments_disable: true

# publish date
date: 2025-09-13 12:35:04 +0900

# seo
# if not specified, date will be used.
#meta_modify_date: 2021-08-11 12:35:04 +0900
# check the meta_common_description in _data/owner/[language].yml
#meta_description: ""

# optional
# please use the "image_viewer_on" below to enable image viewer for individual pages or posts (_posts/ or [language]/_posts folders).
# image viewer can be enabled or disabled for all posts using the "image_viewer_posts: true" setting in _data/conf/main.yml.
#image_viewer_on: true
# please use the "image_lazy_loader_on" below to enable image lazy loader for individual pages or posts (_posts/ or [language]/_posts folders).
# image lazy loader can be enabled or disabled for all posts using the "image_lazy_loader_posts: true" setting in _data/conf/main.yml.
#image_lazy_loader_on: true
# exclude from on site search
#on_site_search_exclude: true
# exclude from search engines
#search_engine_exclude: true
# to disable this page, simply set published: false or delete this file
#published: false
---

<!-- outline-start -->

Esta es la primera entrada en en blog, explicando como conectarse y descargar datos de Gaia.

<!-- outline-end -->

# Conectándose a Gaia

## Introducción

En esta primera entrada del blog, explicaré desde el inicio como conectarse a Gaia para descargar datos. 

He decidido realizar un proyecto que analice los datos de cualquier cúmulo abierto a partir de los datos disponibles en Gaia DR3. El análisis de cúmulos abiertos ha avanzado enormemente en los últimos años gracias a los nuevos datos proporcionados por las últimas misiones, por ejemplo Gaia. Los datos de astrometría, fotometría y espectrofotometría, permiten identificar muchas propiedades de estos cúmulos y avanzar en su conocimiento. Dedicaré alguna entrada a describir toda la ciencia que se puede realizar a partir de estos datos.

En esta primera entrada empezaré configurando un entorno de desarrollo, creando la conexión a los datos de Gaia y descargando los datos de las estrellas cercanas a un cúmulo abierto. El objetivo es familizarse con las conexión a DR3, conocer los datos disponibles, y descargar un primer conjunto de datos que posteriormente nos permita realizar análisis de membresía, cálculo de propiedades físicas del cúmulo...

Empezaré creando un cuaderno Jupyter para poder ir paso a paso viendo el detalle de cada funcionalidad, pero en el futuro planteo crear una app en Python con distintos módulos.

## Configuración del entorno

Realizaremos el desarrollo usando Python en un portátil con Linux Mint, aunque es un desarrollo que puede ser replicado y ejecutado en cualquier plataforma. Para empezar, usaré Conda para crear un entorno de desarrollo con las librerías más adecuadas para análisis astronómico.

``` bash
conda create -n cluster_env python=3.12 jupyter pandas matplotlib seaborn astropy astroquery plotly -c conda-forge

conda activate cluster_env
```

## Obtención de datos

Una vez creado y activado el entorno virtual, nos ponemos manos a la obra. 

### Datos básicos desde SIMBAD

En primer lugar, dado que en el proyecto final quiero que el nombre del cúmulo sea el dato de entrada, necesitamos consultar los datos básicos (coordenadas, tamaño...) en [SIMBAD](https://simbad.u-strasbg.fr/simbad/). 

Lo primero es importar las librerías que nos permitirán consultar SIMBAD:

``` python
import numpy as np
import pandas as pd
from astroquery.simbad import Simbad
from astropy.coordinates import SkyCoord
from astropy import units as u
```

Dentro de astroquery tenemos el módulo Simbad que nos permite conectarnos a la base de datos Simbad y obtener algunos datos a partir del nombre de un objeto. Ahora ya podemos lanzar nuestra consulta:

``` python
cluster_name='NGC 2567'

custom_simbad = Simbad()

# Reset SIMBAD to basic configuration
Simbad.reset_votable_fields()

# Add specific fields
custom_simbad.add_votable_fields('otype', 'dim', 'plx', 'pmra', 'pmdec','dim')

result = custom_simbad.query_object(cluster_name)

# Show basic information from SIMBAD
row = result[0]
ra = row['ra']
dec = row['dec']
radius = row['galdim_majaxis']/2

```

Y ya tenemos los datos necesarios para consultar en Gaia. En particular nos interesan las coordenadas (datos que ya vienen por defecto en la consulta a Simbad), y el tamaño. El tamaño nos permitirá acotar la búsqueda de estrellas en Gaia. Lo obtenemos a partir de los parámetros `galdim_minaxis` y `galdim_majaxis`. Estos dos parámetros nos dan el tamaño en arcmin del objeto consultado. Nos quedamos con el mayor de ellos para la consulta en Gaia. 

### Consulta a la base de datos DR3 de Gaia

Ahora vamos a conectarnos a la base de datos de Gaia y lanzar una consulta a partir de los parámetros obtenidos. Necesitamos importar la librería necesaria para consultar Gaia:

``` python
import getpass 
from astroquery.gaia import Gaia
max_sources=50000
```
y ahora construimos la consulta usando [ADQL](https://www.cosmos.esa.int/web/gaia-users/archive/writing-queries) (Astronomical Data Query Language), un lenguaje muy similar a SQL, con algunas extensiones que facilitan la consulta de datos astronómicos.

``` python

authenticated = False
# Request Gaia credentials 
username = input("Gaia user: ")
password = getpass.getpass("Password: ")

try:
    Gaia.login(user=username, password=password)
    print("Login successful")
    authenticated = True
    
except Exception as e:
    print(f"Login error: {e}")
    print("continue without login")


# ADQL query to obtain basic astrometric and photometric parameters from Gaia
query = f"""
SELECT TOP {max_sources}
    source_id,
    ra, dec,
    parallax, parallax_error,
    pmra, pmra_error, 
    pmdec, pmdec_error,
    phot_g_mean_mag,
    phot_bp_mean_mag, 
    phot_rp_mean_mag,
    bp_rp,
    ruwe,
    visibility_periods_used
FROM gaiadr3.gaia_source 
WHERE CONTAINS(POINT('ICRS', ra, dec), 
                CIRCLE('ICRS', {ra}, {dec}, {radius/60.0})) = 1
AND parallax IS NOT NULL
AND parallax > -5
AND phot_g_mean_mag IS NOT NULL
AND phot_g_mean_mag < 20
ORDER BY phot_g_mean_mag ASC
"""
```
En esta consulta hemos escogido algunos parámetros básicos de los posibles en Gaia y que tienen sentido para el análisis de datos de cúmulos abiertos. También hemos seleccionado como origen la tabla gaia_source en DR3, la última release de datos disponible. Ademas ya hemos añadido algunos filtros de calidad sobre los datos de Gaia:

- `paralallax IS NOT NULL`: con esto evitamos traer datos de estrellas sin paralaje, dato imprescindible para poder asignar la estrella a un cúmulo posteriormente.
- `parallax > -5`: recuperamos sólo estrellas con paralaje válido
- `phot_g_mean_mag IS NOT NULL`: estrellas con magnitud no nula
- `phot_g_mean_mag < 20`: estrellas más brillantes de magnitud 20

Y finalmente lanzamos la consulta de la forma más sencilla posible. Hemos implementado la consulta anónima por defecto, pero también solicitando el usuario/contraseña de forma segura. Puedes solicitar una cuenta gratuita en [ESAC](https://gea.esac.esa.int/archive/). La consulta con usuario permite ejecutar consultas más grandes, con mayor número de registros devueltos y mejor servicio. Con esa configuración la consulta la ejecutamos de forma asíncrona:

``` python

try:
    if authenticated:
        # Launch asynchronous query (better for big query)
        job = Gaia.launch_job_async(query)
        results = job.get_results()
    else:
        # Launch csynchronous query (anonymous)
        job = Gaia.launch_job(query)
        results = job.get_results()
        
    print(f"Number of results: {len(results)}")
    
except Exception as e:
    print(f"Error executing the query in Gaia: {e}")
    return None

```

Esta consulta devuelve un objeto `astropy.table.Table`, que podemos transformar en un Pandas Dataframe con el método to_pandas().

Podemos ejecutar el notebook para distintos objetos a partir de su nombre.

Hasta aquí esta primera entrada, muy simple para iniciarnos en la recuperación de datos de objetos en la base de datos de Gaia DR3.

En posteriores entradas reharemos este notebook más estructurado.



