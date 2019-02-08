# Mordecai

This is a forked version of a full text geoparsing as a Python library (https://github.com/openeventdata/mordecai). Extract the place names from a piece of text, resolve them to the correct place, and return their coordinates and structured geographic information. The forked version containers docker files which are to use kibana and elastic search over the dataset. Eventually, there will be scripts to update the database with other locations and retrain the inferencing model to improve performance.

Example usage
-------------

```
>>> from mordecai import Geoparser
>>> geo = Geoparser()
>>> geo.geoparse("I traveled from Oxford to Ottawa.")

[{'country_conf': 0.96474487,
  'country_predicted': 'GBR',
  'geo': {'admin1': 'England',
   'country_code3': 'GBR',
   'feature_class': 'P',
   'feature_code': 'PPLA2',
   'geonameid': '2640729',
   'lat': '51.75222',
   'lon': '-1.25596',
   'place_name': 'Oxford'},
  'spans': [{'end': 22, 'start': 16}],
  'word': 'Oxford'},
 {'country_conf': 0.83302397,
  'country_predicted': 'CAN',
  'geo': {'admin1': 'Ontario',
   'country_code3': 'CAN',
   'feature_class': 'P',
   'feature_code': 'PPLC',
   'geonameid': '6094817',
   'lat': '45.41117',
   'lon': '-75.69812',
   'place_name': 'Ottawa'},
  'spans': [{'end': 32, 'start': 26}],
  'word': 'Ottawa'}]
```

Mordecai requires a running Elasticsearch service with Geonames in it. See
"Installation" below for instructions.


Installation and Requirements
--------------------

1. Mordecai is on PyPI and can be installed for Python 3 with pip:

```
pip install mordecai
```

However, since this is a fork, the pip module is not configued. Please run the code locally.

2. You should then download the required spaCy NLP model:

```
python -m spacy download en_core_web_lg
```

3. In order to work, Mordecai needs access to a Geonames gazetteer running in
Elasticsearch. The easiest way to set it up is by running the following

```
wget https://s3.amazonaws.com/ahalterman-geo/geonames_index.tar.gz --output-file=wget_log.txt
tar -xzf geonames_index.tar.gz
```

4. The main point of the fork is to include docker files to utilize elasticsearch and kibana. This is configured within the docker-compose.yml class. Conveniently, in order to download the containers and start them, run the Makefile

```
wget https://s3.amazonaws.com/ahalterman-geo/geonames_index_2018-06-05.tar.gz --output-file=wget_log.txt
tar -xzfgeonames_index_2018-06-05.tar.gz
make compose
```

See the [es-geonames](https://github.com/openeventdata/es-geonames) for the code used
to produce this index.

To update the index, simply shut down the old container, re-download the index
from s3, and restart the container with the new index.


Ports
-----
- http://localhost:5601 : Kibana Dashboard
- http://localhost:9200 : Elastic Search (OOS)
- http://localhost:9600 : Logstash

Data
----

In total, the Geonames database contains 11,741,135 unique coordinates for geolocations.

To extract all data from elasticsearch use curl:

```
curl -XGET "http://localhost:9200/_search" -H 'Content-Type: application/json' -d'
{
    "_source": ["asciiname", "coordinates"],
    "query": {
        "type" : {
            "value" : "geoname"
        }
    },
    "size":11741135
}' > geonames.json
```

This will limit you to 10000 objects unless you use a session id. See the jupyter notebook for extracting information from the database using the Python wrapper

Moredecai builds on top of these 11.7 million locations.

Evaluation
------
In order to evaluate location parsing, we run a series of test with a subset of 52k queries. The dimensions we focus on are case and context. The table below describes each extraction technique

- Individual (Lowercase with No Context) ex: new york
- Individual (Lowercase with Minimal Context) ex: where is new york
- World Locations List (Lowercase with No Context) ex: rome, new york, united states
- World Location List (Lowercase with Context) ex: where is rome, new york, united states
- Individual (Uppercase with No Context) ex: New York
- Individual (Uppercase with Context) ex: where is New York
- Individual (Uppercase with Random Context): random query from sample set of 20 different queries
- World Locations List (Uppercase with No Context) ex: Rome, New York, United States
- World Locations List  (Uppercase with Context) ex: Where is Rome, New York, United States?
- World Location List (Uppercase with Random Context): random query from sample set of 20 different queries

Here are the results

<p align="center">
  <img src="/images/Mordecai Location Extraction Results.png" title="Extraction Results">
</p>


Citing
------

If you use this software in academic work, please cite as 

```
@article{halterman2017mordecai,
  title={Mordecai: Full Text Geoparsing and Event Geocoding},
  author={Halterman, Andrew},
  journal={The Journal of Open Source Software},
  volume={2},
  number={9},
  year={2017},
  doi={10.21105/joss.00091}
}
```

How does it work?
-----------------

Mordecai takes in unstructured text and returns structured geographic information extracted
from it. 

- It uses [spaCy](https://github.com/explosion/spaCy/)'s named entity recognition to
  extract placenames from the text.

- It uses the [geonames](http://www.geonames.org/)
  gazetteer in an [Elasticsearch](https://www.elastic.co/products/elasticsearch) index 
  (with some custom logic) to find the potential coordinates of
  extracted place names.

- It uses neural networks implemented in [Keras](https://keras.io/) and trained on new annotated
  data labeled with [Prodigy](https://prodi.gy/) to infer the correct country and correct gazetteer entries for each
  placename. 

The training data for the two models includes copyrighted text so cannot be
shared freely, but get in touch with me if you're interested in it.

API and Configuration
---------------------

When instantiating the `Geoparser()` module, the following options can be changed:

- `es_hosts` : List of hosts where the Geonames Elasticsearch service is
    running. Defaults to `['localhost']`, which is where it runs if you're using
    the default Docker setup described above.
- `es_port` : What port the Geonames Elasticsearch service is running on.
    Defaults to `9200`, which is where the Docker setup has it
- `es_ssl` : Whether Elasticsearch requires an SSL connection.
    Defaults to `False`.
- `es_auth` : Optional HTTP auth parameters to use with ES.
    If provided, it should be a two-tuple of `(user, password)`.
- `country_confidence` : Set the country model confidence below which no
    geolocation will be returned. If it's really low, the model's probably
    wrong and will return weird results. Defaults to `0.6`. 
- `verbose` : Return all the features used in the country picking model?
    Defaults to `False`. 
- `threads`: whether to use threads to make parallel queries to the
    Elasticsearch database. Defaults to `True`, which gives a ~6x speedup.

`geoparse` is the primary endpoint and the only one that most users will need.
Other methods are primarily internal to Mordecai but may be directly useful in
some cases:

- `infer_country` take a document and attempts to infer the most probable
    country for each.
- `query_geonames` and `query_geonames_country` can be used for performing a
    search over Geonames in Elasticsearch
- methods with the `_feature` prefix are internal methods for
    calculating country picking features from text.

`batch_geoparse` takes in a list of documents and uses spaCy's `nlp.pipe`
method to process them more efficiently in the NLP step. 

Advanced users on large machines can modify the `lru_cache` parameter from 250
to 1000. This will use more memory but will increase parsing speed.

Tests
-----

Mordecai includes unit tests. To run the tests, `cd` into the
`mordecai` directory and run:

```
pytest
```

The tests require access to a running Elastic/Geonames service to
complete. The tests are currently failing on TravisCI with an unexplained
segfault but run fine locally. Mordecai has only been tested with Python 3.


Acknowledgements
----------------

An earlier verion of this software was donated to the Open Event Data Alliance
by Caerus Associates.  See [Releases](https://github.com/openeventdata/mordecai/releases) 
or the [legacy-docker](https://github.com/openeventdata/mordecai/tree/legacy-docker) branch for the
2015-2016 and the 2016-2017 production versions of Mordecai.

This work was funded in part by DARPA's XDATA program, the U.S. Army Research
Laboratory and the U.S. Army Research Office through the Minerva Initiative
under grant number W911NF-13-0332, and the National Science Foundation under
award number SBE-SMA-1539302. Any opinions, findings, and conclusions or
recommendations expressed in this material are those of the authors and do not
necessarily reflect the views of DARPA, ARO, Minerva, NSF, or the U.S.
government.


Contributing
------------

Contributions via pull requests are welcome. Please make sure that changes
pass the unit tests. Any bugs and problems can be reported
on the repo's [issues page](https://github.com/openeventdata/mordecai/issues).

