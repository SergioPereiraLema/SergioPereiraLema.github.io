---
# multilingual page pair id, this must pair with translations of this page. (This name must be unique)
lng_pair: id_blog_1
title: Connecting and downloading data from Gaia

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

This is the first blog post, explaining how to connect and download data from Gaia.

<!-- outline-end -->

# Connecting to Gaia

## Introduction

In this first blog post, I will explain from the beginning how to connect to Gaia to download data. 

I have decided to carry out a project that analyses the data of any open cluster based on the data available in Gaia DR3. The analysis of open clusters has advanced enormously in recent years thanks to new data provided by the latest missions, such as Gaia. Astrometry, photometry, and spectrophotometry data allow us to identify many properties of these clusters and advance our knowledge of them. I will devote a few posts to describing all the science that can be done with this data.

In this first post, I will start by setting up a development environment, creating a connection to Gaia data, and downloading data on stars near an open cluster. The aim is to familiarise myself with the connection to DR3, learn about the available data, and download a first set of data that will later allow us to perform membership analysis, calculate the physical properties of the cluster, etc.

I will start by creating a Jupyter notebook so that I can go step by step through the details of each functionality, but in the future I plan to create a Python app with different modules.

## Environment configuration

We will carry out the development using Python on a laptop running Linux Mint, although this development can be replicated and executed on any platform. To begin with, I will use Conda to create a development environment with the most suitable libraries for astronomical analysis.

``` bash
conda create -n cluster_env python=3.12 jupyter pandas matplotlib seaborn astropy astroquery plotly -c conda-forge

conda activate cluster_env
```

## Data collection

Once the virtual environment has been created and activated, we get down to work. 

### Basic data from SIMBAD

Firstly, given that in the final project I want the name of the cluster to be the input data, we need to consult the basic data (coordinates, size, etc.) in [SIMBAD](https://simbad.u-strasbg.fr/simbad/).

The first thing to do is to import the libraries that will allow us to consult SIMBAD:

``` python
import numpy as np
import pandas as pd
from astroquery.simbad import Simbad
from astropy.coordinates import SkyCoord
from astropy import units as u
```

Within astroquery, we have the Simbad module, which allows us to connect to the Simbad database and obtain some data based on the name of an object. Now we can run our query:

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

And now we have the necessary data to query Gaia. We are particularly interested in the coordinates (data that already comes by default in the Simbad query) and the size. The size will allow us to narrow down the search for stars in Gaia. We obtain it from the parameters `galdim_minaxis` and `galdim_majaxis`. These two parameters give us the size in arcminutes of the object queried. We use the larger of the two for the query in Gaia. 

### Query the Gaia DR3 database

Now let's connect to the Gaia database and run a query based on the parameters obtained. We need to import the library required to query Gaia:

``` python
import getpass 
from astroquery.gaia import Gaia
max_sources=50000
```
And now we build the query using [ADQL](https://www.cosmos.esa.int/web/gaia-users/archive/writing-queries)  (Astronomical Data Query Language), a language very similar to SQL, with some extensions that facilitate the querying of astronomical data.

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
In this query, we have chosen some basic parameters from those available in Gaia that are useful for analysing open cluster data. We have also selected the gaia_source table in DR3, the latest data release available, as the source. In addition, we have already added some quality filters to the Gaia data:

- `paralallax IS NOT NULL`: this prevents us from bringing in data from stars without parallax, which is essential for assigning the star to a cluster later on.
- `parallax > -5`: we only retrieve stars with valid parallax
- `phot_g_mean_mag IS NOT NULL`: stars with non-zero magnitude
- `phot_g_mean_mag < 20`: stars brighter than magnitude 20

And finally, we launched the query in the simplest way possible. We have implemented anonymous queries by default, but we also request the user name/password securely. You can request a free account at [ESAC](https://gea.esac.esa.int/archive/). Queries with a user name allow you to run larger queries, with a greater number of records returned and better service. With this configuration, we run the query asynchronously:

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

This query returns an astropy.table.Table object, which we can transform into a Pandas Dataframe using the to_pandas() method.

We can run the notebook for different objects based on their name.

That concludes this first entry, which is very simple and serves as an introduction to retrieving data from objects in the Gaia DR3 database.

In future entries, we will redo this notebook in a more structured way. 

The notebook is in the [repository](https://github.com/SergioPereiraLema/01_clusterAnalysis) of the project 

