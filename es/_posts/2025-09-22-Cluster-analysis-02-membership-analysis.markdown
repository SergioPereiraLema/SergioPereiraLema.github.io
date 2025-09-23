---
# multilingual page pair id, this must pair with translations of this page. (This name must be unique)
lng_pair: id_blog_1
title: Análisis de membresía del cúmulo M37

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
date: 2025-09-22 22:00:00 +0900

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

En esta segunda entrada voy a realizar un análisis de pertenencia, es decir, decidir qué estrellas de una región pertenecen a un cúmulo a partir de los datos de Gaia.

<!-- outline-end -->

# Análisis de pertenencia

## Introducción

En esta segunda entrada veremos cómo decidir qué estrellas descargadas de una región pertenecen realmente al cúmulo y cuáles son estrellas del campo.

Este fin de semana fui con mi telescopio Dobson de 254 mm de apertura al Centro Astronómico de Trevinca aprovechando la luna nueva. Las tierras de Trevinca (Ourense) tienen los mejores cielos de Galicia y posiblemente del noroeste de España, y siempre merece la pena acercarse para disfrutar de un cielo estrellado. El cielo no era el mejor, el humo de los incendios cercanos y alguna que otra nube molestaban, pero aun así pude disfrutar de bastantes objetos. Uno de esos objetos que siempre maravilla con prismáticos o con telescopio es el cúmulo abierto M37 (NGC 2009), en la constelación de Auriga. La vista es deslumbrante; un «guau» es casi inevitable. Lo disfruté durante varios minutos, recorriendo el campo poco a poco. Ahora pretendo aprender algo más de lo que vi en el ocular. Usando los datos de Gaia, voy a descargar las estrellas de ese campo y decidir cuáles son realmente del cúmulo y cuáles son estrellas de fondo en ese mismo campo.

M37 es el cúmulo más rico de la constelación de Auriga, con otros dos miembros muy interesantes, M36 y M38. Según Wikipedia y algunas otras fuentes, se le conoce como «Cúmulo de la sal y la pimienta». Según algunos estudios, se encuentra a aproximadamente 4500 años luz de distancia. La foto que tomé este fin de semana salió más o menos cuando se estaba levantando el Dolmen de Dombate, uno de los templos del megalito del noroeste de España y uno de mis lugares favoritos.

### Origen y formación de los cúmulos abiertos

De cara al análisis, conviene explicar brevemente el origen de un cúmulo abierto. Un cúmulo abierto es un grupo de estrellas que se formaron juntas a partir de la misma nube molecular de gas y polvo. También se conocen como «cúmulos galácticos» porque se encuentran en el plano de las galaxias espirales, como nuestra Vía Láctea, donde la formación estelar es más activa. Las estrellas de los cúmulos abiertos suelen ser jóvenes (< 1.000 millones de años de edad) y suelen estar compuestos por entre unas pocas decenas y unos pocos miles de estrellas. Su forma es irregular y sus estrellas están unidas gravitacionalmente.

El proceso de formación de un cúmulo abierto comienza con una gran nube molecular de gas y polvo. Dentro de esta nebulosa, la gravedad provoca que algunas zonas se contraigan y colapsen. A medida que estas regiones se comprimen, la presión y la temperatura aumentan, lo que desencadena reacciones de fusión nuclear y el nacimiento de nuevas estrellas. Las estrellas recién formadas emiten una gran cantidad de radiación y vientos estelares que empujan el gas y el polvo restantes hacia el exterior. Este proceso disipa la nebulosa de origen, dejando atrás el grupo de estrellas jóvenes que resultan visibles como un cúmulo abierto.

Debido a que las estrellas de un cúmulo abierto no están tan fuertemente unidas por la gravedad como en los cúmulos globulares, la interacción gravitatoria con otras estrellas, nubes de gas o el propio centro de la galaxia hace que, con el tiempo, el cúmulo se disperse y sus estrellas se separen.


### Análisis de pertenencia

El análisis de pertenencia busca separar qué estrellas pertenecen realmente al cúmulo de las estrellas de campo que solo están en la misma línea de visión por casualidad. Las estrellas del cúmulo son estrellas que nacieron juntas, se mueven juntas y están a la misma distancia. Las estrellas de campo están a diferentes distancias y se mueven aleatoriamente; no parecen estar relacionadas entre sí ni con el resto de estrellas del cúmulo. Teniendo esto en cuenta, utilizaremos tres parámetros para realizar el análisis de pertenencia:

- `pmRA` y `pmDEC` (movimientos propios): las estrellas del cúmulo tienen movimientos propios muy similares porque nacieron de la misma nube molecular y mantienen velocidades similares, el cúmulo se mueve como un todo por la Galaxia.
- `paralaje`: las estrellas del cúmulo están a la misma distancia, por lo que tienen paralajes similares.

Para realizar el análisis utilizando algoritmos de *clustering*, podemos elegir entre varios. Uno que he utilizado en el pasado en entornos de datos empresariales es K-means. Otro que se utiliza en este tipo de análisis es **DBSCAN**. Voy a utilizar este último por varias razones: la principal diría que es que DBSCAN no necesita saber a priori cuántos grupos tiene que hacer, sino que lo descubre automáticamente. Dado que el propósito de esta entrada es mostrar una posible aplicación de los datos de Gaia para analizar cúmulos abiertos, no profundizaré mucho más comparando con otros métodos, ni mejorando la implementación de este algoritmo (algo que espero ir haciendo más adelante). Hace unos años, en un proyecto para una empresa, probé PyCaret para validar el análisis con distintos algoritmos, hacer el ajuste fino de los parámetros y orquestar todo el *pipeline* en sólo 14 lineas de código. Espero revisarlo próximamente en este contexto.

### Descripción del procesado

El cuaderno Jupyter está en el [repositorio](https://github.com/SergioPereiraLema/01_clusterAnalysis) del proyecto.

## Configuración del entorno

Realizaré el desarrollo usando Python en el mismo entorno virtual que creé en la primera entrada. Sólo hay que añadir la popular librería **sklearn**.


``` bash
conda activate cluster_env

pip install scikit-learn
```

## Obtención de datos

### Datos básicos desde SIMBAD

Voy a reutilizar parte del código que fijé en el primer cuaderno Jupyter. Primero, recupero los datos básicos (coordenadas, tamaño...) en [SIMBAD](https://simbad.u-strasbg.fr/simbad/). Con este resultado ya tenemos los datos necesarios para consultar en Gaia (coordenadas y tamaño). Al ejecutar la consulta en SIMBAD obtengo:

- `Name`: M  37
- `Type`: OpC
- `RA`: 88.077300º
- `Dec`: 32.543400º
- `PmRA`: 1.924 (mas/yr)
- `PmDec`: -5.648 (mas/yr)
- `galdim_majaxis`: 19.299999237060547 (arcmin)
- `galdim_minaxis`: 19.299999237060547 (arcmin)
- `Radius`: 9.649999618530273 (arcmin)
- `Parallax`: 0.666 (mas)

Ya tengo las coordenadas y el tamaño. Con eso vamos a lanzar la consulta a Gaia. Hay que tener en cuenta en este momento que el tamaño máximo que estoy recogiendo es el que devuelve SIMBAD. Quizás debería ampliar un poco más el radio de búsqueda para no dejar fuera de la consulta estrellas del cúmulo.

### Consulta a la base de datos DR3 de Gaia

Ahora vamos a conectarnos a la base de datos de Gaia y lanzar una consulta a partir de los parámetros obtenidos. En esta ocasión aprovecho para crear una función que espero ir puliendo y reutilizando más adelante. 

La consulta [ADQL](https://www.cosmos.esa.int/web/gaia-users/archive/writing-queries) ya incluye varios filtros de calidad:

- `paralallax IS NOT NULL`: con esto evito descargar datos de estrellas sin paralaje, dato imprescindible para poder asignar la estrella a un cúmulo posteriormente.
- `parallax > 0`: recupero sólo estrelas con paralaje válida
- `parallax_error/parallax < 0.2`: filtro sólo resultados con error en paralaje bajo
- `pmra_error IS NOT NULL` e `pmdec_error IS NOT NULL` : estrellas con error reportado en los movimientos propios
- `pmra_error < 20` e `pmdec_error < 20` : estrellas con bajo error en los movimientos propios 
- `ruwe < 1.4`: estrellas con Renormalised Unit Weight Error <1.4 que garantiza eliminar estrellas binarias, astrometría aceptable...
- `astrometric_excess_noise < 1`: garantiza que los datos tengan buena calidad adicional al anterior
- `phot_g_mean_mag IS NOT NULL`: estrellas con magnitud no nula
- `phot_g_mean_mag < 20`: estrellas máis brilantes de magnitud 20
- `phot_bp_mean_mag IS NOT NULL`: estrellas con magnitud BP no nula
- `phot_rp_mean_mag IS NOT NULL`: estrellas con magnitud RP no nula


Lanzo una consulta de forma sencilla, reutilizando el código anterior. Esta consulta devuelve un objeto `astropy.table.Table`, que transformo en un Pandas Dataframe con el método to_pandas().

La consulta devuelve ... estrellas.

### Algunhas visualizacións básicas

Ahora voy a hacer algunas gráficas interesantes que servirán en futuros análisis de los cúmulos. Por el momento, solo a modo ilustrativo:

- **mapa celeste**
- **diagrama de movimientos propios**
- **histograma de paralajes**
- **diagrama color-magnitud**

También probé la conversión de los parámetros BP y RP a colores RGB, de manera que pudiéra pintar el color real de cada estrella en estos gráficos y ver el resultado.

En próximas entradas profundizaré en la información que se puede extraer de cada uno.

### Análisis de pertenencia

Aquí viene una parte importante del proyecto: usar algoritmos de agrupamiento para determinar cuáles son las estrellas que pertenecen al cúmulo. Como comentaba en la introducción, ahora solo probaré el algoritmo DBSCAN. Defino una función y, siguiendo los pasos habituales, aplico StandardScaler para *escalar* los datos y luego aplico el algoritmo sobre los datos. Utilicé unos parámetros que encontré en alguna lectura, pero que requerirían un ajuste fino para obtener los mejores resultados.





