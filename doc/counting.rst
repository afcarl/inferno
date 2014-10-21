Example 1 - Count Last Names
============================

The canonical map/reduce example: **count** the occurrences of words in a 
document. In this case, we'll count the occurrences of last names in a data 
file containing lines of json.

Input
-----

Here's the input data::

    diana@ubuntu:~$ cat data.txt 
    {"first":"Homer", "last":"Simpson"}
    {"first":"Manjula", "last":"Nahasapeemapetilon"}
    {"first":"Herbert", "last":"Powell"}
    {"first":"Ruth", "last":"Powell"}
    {"first":"Bart", "last":"Simpson"}
    {"first":"Apu", "last":"Nahasapeemapetilon"}
    {"first":"Marge", "last":"Simpson"}
    {"first":"Janey", "last":"Powell"}
    {"first":"Maggie", "last":"Simpson"}
    {"first":"Sanjay", "last":"Nahasapeemapetilon"}
    {"first":"Lisa", "last":"Simpson"}
    {"first":"Maggie", "last":"Términos"}

Inferno Rule
------------

And here's the Inferno rule (``inferno/example_rules/names.py``)::

    def count(parts, params):
        parts['count'] = 1
        yield parts


    InfernoRule(
        name='last_names_json',
        source_tags=['example:chunk:users'],
        map_input_stream=chunk_json_keyset_stream,
        parts_preprocess=[count],
        key_parts=['last'],
        value_parts=['count'],
    )

DDFS
----

The first step is to place the data file in 
`Disco's Distributed Filesystem <http://discoproject.org/doc/disco/howto/ddfs.html>`_ (DDFS). 
Once placed in DDFS, a file is referred to by Disco as a **blob**. 

DDFS is a tag-based filesystem. Instead of organizing files into directories, 
you **tag** a collection of blobs with a **tag_name** for lookup later.

In this case, we'll be tagging our data file as **example:chunk:users**.

.. image:: images/tag_blobs.png
   :height: 300px
   :width: 600 px
   :scale: 75 %
   :alt: tag_name -> [blob1, blob2, blob3]

Make sure `Disco <http://discoproject.org/>`_ is running::

    diana@ubuntu:~$ disco start
    Master ubuntu:8989 started

Place the input data in DDFS::

    diana@ubuntu:~$ ddfs chunk example:chunk:users ./data.txt 
    created: disco://localhost/ddfs/vol0/blob/99/data_txt-0$533-406a9-e50

Verify that the data is in DDFS::

    diana@ubuntu:~$ ddfs xcat example:chunk:users | head -2
    {"first":"Homer", "last":"Simpson"}
    {"first":"Manjula", "last":"Nahasapeemapetilon"}

Map Reduce
----------

For the purpose of this introductory example, think of an Inferno map/reduce 
job as a series of four steps, where the output of each step is used as the 
input to the next.

.. image:: images/simple_map_reduce.png
   :height: 100px
   :width: 600 px
   :align: center
   :scale: 75 %
   :alt: input -> map -> reduce -> output


**Input**

The input step of an Inferno map/reduce job is responsible for parsing and 
readying the input data for the map step.

If you're using Inferno's built in **keyset** map/reduce functionality, 
this step mostly amounts to transforming your CSV or JSON input into 
python dictionaries.

The default Inferno input reader is **chunk_csv_keyset_stream**, which is
intended for CSV data that was placed in DDFS using the ``ddfs chunk`` 
command. 

If the input data is lines of JSON, you would instead set the 
**map_input_stream** to use the **chunk_json_keyset_stream** reader in 
your Inferno rule.

The input reader will process all DDFS tags that are prefixed with the 
tag names defined in **source_tags** of your Inferno rule.

.. code-block:: python
   :emphasize-lines: 3,4

InfernoRule(
    name='last_names_json',
    source_tags=['example:chunk:users'],
    map_input_stream=chunk_json_keyset_stream,
    parts_preprocess=[count],
    key_parts=['last'],
    value_parts=['count'],
)

Example data transition during the **input** step:

.. image:: images/input.png
   :height: 600px
   :width: 800 px
   :align: center
   :scale: 60 %
   :alt: data -> input

**Map**

The map step of an Inferno map/reduce job is responsible for extracting 
the relevant key and value parts from the incoming python dictionaries and 
yielding one, none, or many of them for further processing in the reduce 
step.

Inferno's default **map_function** is the **keyset_map**. You define the 
relevant key and value parts by declaring **key_parts** and **value_parts** 
in your Inferno rule.

.. code-block:: python
   :emphasize-lines: 6,7
InfernoRule(
    name='last_names_json',
    source_tags=['example:chunk:users'],
    map_input_stream=chunk_json_keyset_stream,
    parts_preprocess=[count],
    key_parts=['last'],
    value_parts=['count'],
)

Example data transition during the **map** step:

.. image:: images/map.png
   :height: 600px
   :width: 800 px
   :align: center
   :scale: 60 %
   :alt: input -> map

**Reduce**

The reduce step of an Inferno map/reduce job is responsible for summarizing 
the results of your map/reduce query.

Inferno's default **reduce_function** is the **keyset_reduce**. It will sum
the value parts yielded by the map step, grouped by the key parts.

In this example, we're only summing one value (the ``count``). You can 
define and sum many value parts, as you'll see :doc:`here </election>` in 
the next example.

Example data transition during the **reduce** step:

.. image:: images/reduce.png
   :height: 600px
   :width: 800 px
   :align: center
   :scale: 60 %
   :alt: map -> reduce

**Output**

Unless you create and specify your own **result_processor**, Inferno 
defaults to the **keyset_result** processor which simply uses a CSV writer 
to print the results from the reduce step to standard output.

Other common result processor use cases include: populating a cache, 
persisting to a database, writing back to 
`DDFS <http://discoproject.org/doc/disco/howto/ddfs.html>`_ or 
`DiscoDB <http://discoproject.org/doc/disco/howto/discodb.html>`_, etc.

Example data transition during the **output** step:

.. image:: images/output.png
   :height: 600px
   :width: 800 px
   :align: center
   :scale: 60 %
   :alt: reduce -> output

Execution
---------

Run the last name counting job::

    diana@ubuntu:~$ inferno -i names.last_names_json
    2012-03-09 Processing tags: ['example:chunk:users']
    2012-03-09 Started job last_names_json@533:40914:c355f processing 1 blobs
    2012-03-09 Done waiting for job last_names_json@533:40914:c355f
    2012-03-09 Finished job job last_names_json@533:40914:c355f

The output::

    last,count
    Nahasapeemapetilon,3
    Powell,3
    Simpson,5
    Términos,1
