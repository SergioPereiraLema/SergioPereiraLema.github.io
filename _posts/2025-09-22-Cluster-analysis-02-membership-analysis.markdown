---
# multilingual page pair id, this must pair with translations of this page. (This name must be unique)
lng_pair: id_blog_1
title: Membership analysis of the M37 cluster

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

In this second post, I will perform a membership analysis, that is, decide which stars in a region belong to a cluster based on Gaia data.


<!-- outline-end -->

# Membership analysis

## Introduction

In this second post, we will see how to decide which stars downloaded from a region actually belong to the cluster and which are field stars.

This weekend, I took my 254 mm aperture Dobsonian telescope to the Trevinca Astronomical Centre to take advantage of the new moon. The Trevinca area (Ourense) has the best skies in Galicia and possibly in north-western Spain, and it is always worth visiting to enjoy the starry sky. The sky wasn't the best, with smoke from nearby fires and the occasional cloud getting in the way, but I was still able to enjoy quite a few objects. One of those objects that always amazes with binoculars or a telescope is the open cluster M37 (NGC 2009) in the constellation Auriga. The view is dazzling; a ‘wow’ is almost inevitable. I enjoyed it for several minutes, slowly scanning the field. Now I intend to learn more about what I saw through the eyepiece. Using Gaia data, I am going to download the stars in that field and decide which ones are actually from the cluster and which ones are background stars in the same field.

M37 is the richest cluster in the constellation Auriga, with two other very interesting members, M36 and M38. According to Wikipedia and some other sources, it is known as the 'Salt and Pepper Cluster’. According to some studies, it is approximately 4,500 light years away. The light from the stars in this cluster that I saw this weekend came out more or less when the Dolmen of Dombate, the cathedral of megalithism in north-western Spain and one of my favourite places, was being built.

### Origin and formation of open clusters

Before proceeding with the analysis, it is worth briefly explaining the origin of an open cluster. An open cluster is a group of stars that formed together from the same molecular cloud of gas and dust. They are also known as “galactic clusters” because they are found in the plane of spiral galaxies, such as our Milky Way, where star formation is most active. The stars in open clusters are usually young (< 1 billion years old) and typically consist of a few dozen to a few thousand stars. They are irregular in shape and their stars are gravitationally bound..

The process of forming an open cluster begins with a large molecular cloud of gas and dust. Within this nebula, gravity causes some areas to contract and collapse. As these regions compress, pressure and temperature increase, eventually triggering nuclear fusion reactions and the birth of new stars. The newly formed stars emit a large amount of radiation and stellar winds that push the remaining gas and dust outward. This process dissipates the original nebula, leaving behind the group of young stars that are visible as an open cluster.

Because the stars in an open cluster are not as strongly bound by gravity as in globular clusters, gravitational interaction with other stars, gas clouds, or the centre of the galaxy itself causes the cluster to disperse over time and its stars to separate.

### Membership analysis

Membership analysis seeks to separate which stars truly belong to the cluster from field stars that are only in the same line of sight by chance. Cluster stars are stars that were born together, move together, and are at the same distance. Field stars are at different distances and move randomly; they do not appear to be related to each other or to the rest of the stars in the cluster. With this in mind, we will use three parameters to perform the membership analysis:


- `pmRA` and `pmDEC` (proper motions): the stars in the cluster have very similar proper motions because they were born from the same molecular cloud and maintain similar velocities; the cluster moves as a whole through the Galaxy.
- `parallax`: the stars in the cluster are at the same distance, so they have similar parallaxes.

To perform the analysis using clustering algorithms, we can choose from several options. One that I have used in the past in business data environments is K-means. Another that is used in this type of analysis is DBSCAN. I am going to use the latter for several reasons: the main one would be that DBSCAN does not need to know in advance how many groups it has to make, but rather discovers this automatically. Since the purpose of this post is to show a possible application of Gaia data to analyse open clusters, I will not go into much more detail comparing it with other methods or improving the implementation of this algorithm (something I hope to do later). A few years ago, in a project for a company, I tried PyCaret to validate the analysis with different algorithms, fine-tune the parameters, and orchestrate the entire pipeline in just 14 lines of code. I hope to review it soon in this context.

### Description of the process

Jupyter notebook is uploaded to the [repository](https://github.com/SergioPereiraLema/01_clusterAnalysis) of the project.

## Environment configuration

I will carry out the development using Python in the same virtual environment that I created in the first post. All you need to do is add the popular **sklearn** library.


``` bash
conda activate cluster_env

pip install scikit-learn
```

## Data retrieval

### Basic data from SIMBAD

I am going to reuse part of the code I posted in the first Jupyter notebook. First, I retrieve the basic data (coordinates, size, etc.) from [SIMBAD](https://simbad.u-strasbg.fr/simbad/). With this result, we now have the data we need to query Gaia (coordinates and size). When I run the query in SIMBAD, I get:

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

I already have the coordinates and size. With that, we're going to run the query on Gaia. Keep in mind that the maximum size I'm collecting is the one returned by SIMBAD. Perhaps I should expand the search radius a little more so as not to leave any stars from the cluster out of the query.

### Querying Gaia's DR3 database

Now we are going to connect to Gaia's database and run a query based on the parameters obtained. On this occasion, I am taking the opportunity to create a function that I hope to refine and reuse later on. 

The [ADQL](https://www.cosmos.esa.int/web/gaia-users/archive/writing-queries) query already includes several quality filters:

- `paralallax IS NOT NULL`: This prevents me from downloading data on stars without parallax, which is essential for assigning the star to a cluster later on.
- `parallax > 0`: I only retrieve stars with valid parallax
- `parallax_error/parallax < 0.2`: filter only results with low parallax error
- `pmra_error IS NOT NULL` e `pmdec_error IS NOT NULL`:  stars with reported errors in their proper motions
- `pmra_error < 20` e `pmdec_error < 20`: stars with low error in proper motions 
- `ruwe < 1.4`: stars with Renormalised Unit Weight Error <1.4, which guarantees the elimination of binary stars, acceptable astrometry...
- `astrometric_excess_noise < 1`: ensures that the data is of good quality in addition to the above
- `phot_g_mean_mag IS NOT NULL`: stars with non-zero magnitude
- `phot_g_mean_mag < 20`: brightest stars of magnitude 20
- `phot_bp_mean_mag IS NOT NULL`: stars with non-zero BP magnitude
- `phot_rp_mean_mag IS NOT NULL`: stars with non-zero RP magnitude


I run a simple query, reusing the previous code. This query returns an `astropy.table.Table` object, which I transform into a Pandas Dataframe using the to_pandas() method.

The query returns ... stars

### Some basic visualisations

Now I am going to make some interesting graphs that will be useful in future analyses of the clusters. For the moment, just for illustrative purposes:

- **celestial map**
- **diagram of proper motions**
- **histogram of parallaxes**
- **colour-magnitude diagram**

I also tried converting the BP and RP parameters to RGB colours so that I could paint the actual colour of each star in these graphs and see the result.

In future posts, I will delve deeper into the information that can be extracted from each one.

### Membership analysis

Here comes the important part of the project: using clustering algorithms to determine which stars belong to the cluster. As I mentioned in the introduction, I will now only test the DBSCAN algorithm. I define a function and, following the usual steps in this kind of analysis, apply StandardScaler to *scale* the data and then apply the algorithm to the data. I used some parameters that I found in some readings, but they would require fine-tuning to obtain the best results.




