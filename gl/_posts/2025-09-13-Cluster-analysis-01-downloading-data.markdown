---
# multilingual page pair id, this must pair with translations of this page. (This name must be unique)
lng_pair: id_blog_1
title: Conectando e descargando datos de gaia

# post specific
# if not specified, .name will be used from _data/owner/[language].yml
#author: ""
# multiple category is not supported
category: cluster
# multiple tag entries are possible
tags: [cluster, connection]
# thumbnail image for post
img: ":post_pic1.png"
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

Esta é a primeira entrada no blog, explicando como conectarse e descargar datos de Gaia.

<!-- outline-end -->

# Conectándose a Gaia

## Introducción

Nesta primeira entrada do blog, explicarei dende o inicio como conectarse a Gaia para descargar datos. 

Decidín realizar un proxecto que analice os datos de calquera cúmulo aberto a partir dos datos dispoñibles en Gaia DR3. A análise de cúmulos abertos avanzou enormemente nos últimos anos gracias aos novos datos proporcionados polas últimas misións, por exemplo Gaia. Os datos de astrometría, fotometría e espectrofotometría, permiten identificar moitas propiedades destes cúmulos e avanzar no seu coñecemento. Dedicaréi algunha entrada a describir toda a ciencia que se pode realizar a partir destos datos.

Nesta entrada empezarei configurando un entorno de desenvolvemento, creando a conexión aos datos de Gaia e descargando os datos das estrelas cercanas a un cúmulo aberto. O obxectivo é familizarse coa conexión a DR3, coñecer os datos dispoñibles, e descargar un primeiro conxunto de datos que posteriormente permita realizar análise de membresía, cálculo de propiedades físicas do cúmulo...

Empezarei creando un caderno Jupyter para poder ir paso a paso vendo o detalle de cada funcionalidade, pero no futuro planteo crear unha app en Python con distintos módulos.

## Configuración do entorno

Realizaremos o desenrolo usando Python nun portátil con Linux Mint, aínda que pode ser replicado e executado en calquera plataforma. Para empezar, usaréi Conda para crear un entorno de desenvolvemento coas librerías máis axeitadas para análise astronómico.

``` bash
conda create -n cluster_env python=3.12 jupyter pandas matplotlib seaborn astropy astroquery plotly -c conda-forge

conda activate cluster_env
```

## Obtención de datos

Unha vez creado e activado o entorno virtual, poñemosnos ao choio. 

### Datos básicos dende SIMBAD

En primeiro lugar, dado que no proxecto final quero que o nome do cúmulo sexa o dato de entrada, necesitamos consultar os datos básicos (coordenadas, tamaño...) en [SIMBAD](https://simbad.u-strasbg.fr/simbad/).. 

O primeiro é importar as librerías que nos permitirán consultar SIMBAD:

``` python
import numpy as np
import pandas as pd
from astroquery.simbad import Simbad
from astropy.coordinates import SkyCoord
from astropy import units as u
```

Dentro de astroquery temos o módulo `Simbad` que permite que nos conectemos á base de datos Simbad e obter algúns datos a partir do nome dun obxecto. Agora xa podemos lanzar a nosa consulta:

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

E xa temos os datos necesarios para consultar en Gaia. En particular interesannos as coordenadas (datos que xa veñen por defecto na consulta a Simbad), e o tamaño. Este tamaño permitirá acotar a búsqueda de estrelas en Gaia. Obtémolo a partir dos parámetros `galdim_minaxis` e `galdim_majaxis`. Estes dous parámetros dannos o tamaño en arcmin do obxecto consultado. Quedamos co maior deles para a consulta en Gaia. 

### Consulta á base de datos DR3 de Gaia

Agora imos conectarnos á base de datos de Gaia e lanzar unha consulta a partir dos parámetros obtidos. Necesitamos importar a librería necesaria para consultar Gaia:

``` python
import getpass 
from astroquery.gaia import Gaia
max_sources=50000
```

E agora construimos a consulta usando [ADQL](https://www.cosmos.esa.int/web/gaia-users/archive/writing-queries) (Astronomical Data Query Language), unha linguaxe moi similar a SQL, con algunhas extensións que facilitan a consulta de datos astronómicos.

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

Nesta consulta escollemos algúns parámetros básicos dos posibles en Gaia e que teñen sentido para a análise de datos de cúmulos abertos. Tamén seleccionamos como orixe a táboa gaia_source en DR3, a última release de datos dispoñible. Ademas xa engadimos algúns filtros de calidade sobre os datos de Gaia:

- `paralallax IS NOT NULL`: con isto evitamos traer datos de estrelas sen paralaxe, dato imprescindible para poder asignar a estrela a un cúmulo posteriormente.
- `parallax > -5`: recuperamos só estrelas con paralaxe válida
- `phot_g_mean_mag IS NOT NULL`: estrelas con magnitude non nula
- `phot_g_mean_mag < 20`: estrellas máis brilantes de magnitude 20

E finalmente lanzamos a consulta da forma máis sinxela posible. Implementamos a consulta anónima por defecto, pero tamén solicitando o usuario/contrasinal de forma segura. Podes solicitar unha conta gratuíta en [ESAC](https://gea.esac.esa.int/arquive/). A consulta con usuario permite executar consultas máis grandes, con maior número de rexistros devoltos e mellor servizo. Con esa configuración a consulta executámola de forma asíncrona:


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

Esta consulta devolve un obxecto `astropy.table.Table`, que podemos transformar nun Pandas Dataframe co método to_pandas().

Podemos executar o notebook para distintos obxectos a partir do seu nome.

Ata aquí esta primeira entrada, moi simple para iniciarnos na recuperación de datos de obxectos na base de datos de Gaia DR3 e probar as conexións.

En posteriores entradas refaremos este notebook máis estruturado.



