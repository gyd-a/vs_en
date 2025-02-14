Space Operation
=================

http://master_server is the master service, $db_name is the name of the created database, $space_name is the name of the created tablespace.

Create Space
------------

::
   
  curl -XPUT -H "content-type: application/json" -d'
  {
      "name": "space1",
      "partition_num": 1,
      "replica_num": 1,
      "engine": {
          "name": "gamma",
          "index_size": 70000,
          "id_type": "String",
          "retrieval_type": "IVFPQ",
          "retrieval_param": {
              "ncentroids": 256,
              "nsubvector": 32 
          }
      },
      "properties": {
          "field1": {
              "type": "keyword"
          },
          "field2": {
              "type": "integer"
          },
          "field3": {
              "type": "float",
              "index": true
          },
          "field4": {
              "type": "string",
              "array": true,
              "index": true
          },
          "field5": {
              "type": "integer",
              "index": true
          },
          "field6": {
              "type": "vector",
              "dimension": 128
          },
          "field7": {
              "type": "vector",
              "dimension": 256,
              "format": "normalization",
              "store_type": "RocksDB",
              "store_param": {
                  "cache_size": 2048,
                  "compress": {"rate":16}
              }
          }
      }
  }
  ' http://master_server/space/$db_name/_create


Parameter description:

+-------------+------------------+---------------+----------+------------------+
|field name   |field description | field type    |must      |remarks           | 
+=============+==================+===============+==========+==================+
|name         |space name        |string         |true      |                  |
+-------------+------------------+---------------+----------+------------------+
|partition_num|partition number  |int            |true      |                  |
+-------------+------------------+---------------+----------+------------------+
|replica_num  |replica number    |int            |true      |                  |
+-------------+------------------+---------------+----------+------------------+
|engine       |engine config     |json           |true      |                  |
+-------------+------------------+---------------+----------+------------------+
|properties   |schema config     |json           |true      |define space field|
+-------------+------------------+---------------+----------+------------------+

1、Space name not be empty, do not start with numbers or underscores, try not to use special characters, etc.

2、partition_num: Specify the number of tablespace data fragments. Different fragments can be distributed on different machines to avoid the resource limitation of a single machine.

3、replica_num: The number of copies is recommended to be set to 3, which means that each piece of data has two backups to ensure high availability of data. 

engine config:

+----------------+------------------------------+-----------+----------+---------------------------------------+
|field name      |field description             |field type |must      |remarks                                | 
+================+==============================+===========+==========+=======================================+
|name            |engine name                   |string     |true      |currently fixed to gamma               |
+----------------+------------------------------+-----------+----------+---------------------------------------+
|index_size      |slice index threshold         |int        |false     |                                       |
+----------------+------------------------------+-----------+----------+---------------------------------------+
|id_type         |Unique primary key type       |string     |false     |                                       |
+----------------+------------------------------+-----------+----------+---------------------------------------+
|retrieval_type  |search model                  |string     |true      |                                       |
+----------------+------------------------------+-----------+----------+---------------------------------------+
|retrieval_param |model config                  |json       |false     |                                       |
+----------------+------------------------------+-----------+----------+---------------------------------------+

1. name: now value is gamma.

2. index_size: Specify the number of records in each partition to start index creation. If not specified, no index will be created. 

3. id_type Specify primary key type, can be string or long.

4. retrieval_type search model, now support IVFPQ，HNSW，GPU，IVFFLAT，BINARYIVF，FLAT.

IVFPQ:

+---------------+-------------------------------+------------+------------+----------------------------------------+
|field name     |field description              |field type  |must        |remarks                                 |
+===============+===============================+============+============+========================================+
|ncentroids     |number of buckets for indexing |int         |false       |default 2048                            |
+---------------+-------------------------------+------------+------------+----------------------------------------+
|nsubvector     |PQ disassembler vector size    |int         |false       |default 64, must be a multiple of 4     |
+---------------+-------------------------------+------------+------------+----------------------------------------+

::
 
  "retrieval_type": "IVFPQ",
  "retrieval_param": {
      "ncentroids": 2048,
      "nsubvector": 64
  }


HNSW:

+---------------+------------------------------+------------+------------+---------------+
|field name     |field description             |field type  |must        |remarks        |
+===============+==============================+============+============+===============+
|nlinks         |Number of node neighbors      |int         |false       |default 32     |
+---------------+------------------------------+------------+------------+---------------+
|efConstruction |Composition traversal depth   |int         |false       |default 40     |
+---------------+------------------------------+------------+------------+---------------+

::

  "retrieval_type": "HNSW",
  "retrieval_param": {
      "nlinks": 32,
      "efConstruction": 40
  }

  Note: 1. Vector storage only supports MemoryOnly
        2. No training is required to create an index, and the index_size value can be greater than 0


GPU (Compiled version for GPU):

+---------------+---------------------------------+------------+------------+----------------------------------------+
|field name     |field description                |field type  |must        |remarks                                 |
+===============+=================================+============+============+========================================+
|ncentroids     |number of buckets for indexing   |int         |false       |default 2048                            |
+---------------+---------------------------------+------------+------------+----------------------------------------+
|nsubvector     |PQ disassembler vector size      |int         |false       |default 64, must be a multiple of 4     | 
+---------------+---------------------------------+------------+------------+----------------------------------------+

::
 
  "retrieval_type": "GPU",
  "retrieval_param": {
      "ncentroids": 2048,
      "nsubvector": 64
  }

IVFFLAT:

+---------------+-------------------------------+------------+------------+----------------------------------------+
|field name     |field description              |field type  |must        |remarks                                 |
+===============+===============================+============+============+========================================+
|ncentroids     |number of buckets for indexing |int         |default     |default 256                             |
+---------------+-------------------------------+------------+------------+----------------------------------------+

::
 
  "retrieval_type": "IVFFLAT",
  "retrieval_param": {
      "ncentroids": 256
  }

 Note: 1. The vector storage method only supports RocksDB  

BINARYIVF:

+---------------+-------------------------------+------------+------------+----------------------------------------+
|field name     |field description              |field type  |must        |remarks                                 |
+===============+===============================+============+============+========================================+
|ncentroids     |number of buckets for indexing |int         |default     |default 256                             |
+---------------+-------------------------------+------------+------------+----------------------------------------+

::
 
  "retrieval_type": "BINARYIVF",
  "retrieval_param": {
      "ncentroids": 256
  }
  
  Note: 1. The vector length is a multiple of 8

properties config:

1. There are four types (that is, the value of type) supported by the field defined by the table space structure: keyword, integer, float, vector (keyword is equivalent to string).

2. The keyword type fields support index and array attributes. Index defines whether to create an index, and array specifies whether to allow multiple values.

3. Integer, float type fields support the index attribute, and the fields with index set to true support the use of numeric range filtering queries.

4. Vector type fields are feature fields. Multiple feature fields are supported in a table space. The attributes supported by vector type fields are as follows:


+-------------+---------------------------+-----------+--------+------------------------------------------------------------+
|field name   |field description          |field type |must    |remarks                                                     | 
+=============+===========================+===========+========+============================================================+
|dimension    |feature dimension          |int        |true    |Value is an integral multiple of the above nsubvector value |
+-------------+---------------------------+-----------+--------+------------------------------------------------------------+
|store_type   |feature storage type       |string     |false   |support Mmap and RocksDB, default Mmap                      |
+-------------+---------------------------+-----------+--------+------------------------------------------------------------+
|store_param  |storage parameter settings |json       |false   |set the memory size of data                                 |
+-------------+---------------------------+-----------+--------+------------------------------------------------------------+
|model_id     |feature plug-in model      |string     |false   |Specify when using the feature plug-in service              |
+-------------+---------------------------+-----------+--------+------------------------------------------------------------+


5. dimension: define that type is the field of vector, and specify the dimension size of the feature.

6. store_type: raw vector storage type, there are the following three options

"MemoryOnly": Vectors are stored in the memory, and the amount of stored vectors is limited by the memory. It is suitable for scenarios where the amount of vectors on a single machine is not large (10 millions) and high performance requirements

"RocksDB": Vectors are stored in RockDB (disk), and the amount of stored vectors is limited by the size of the disk. It is suitable for scenarios where the amount of vectors on a single machine is huge (above 100 millions) and performance requirements are not high.

"Mmap": The original vector is stored in a disk file. Use the cache to improve performance. The amount of storage is limited by disk size. Applicable to the single machine data volume is huge (over 100 million), the performance requirements are not high scene.

7. store_param: storage parameters of different store_type, it contains the following two sub-parameters

cache_size: interge type, the unit is M bytes, the default is 1024. When store_type="RocksDB", it indicates the read buffer size of RocksDB. The larger the value, the better the performance of reading vector. Generally set 1024, 2048, 4096 and 6144; when store_type ="Mmap", represents the size of read buffer, generally 512, 1024, 2048 and 4096, can be set according to the actual application scenario; store_type ="MemoryOnly", cache_size is not in effect.

compress: set to {"rate":16} to compress by 50%;Default does not compress.


View Space
----------
::
  
  curl -XGET http://master_server/space/$db_name/$space_name


Delete Space
------------
::
 
  curl -XDELETE http://master_server/space/$db_name/$space_name


Modify cache size
------------
::
 
  curl -H "content-type: application/json" -XPOST -d'
  {
      "cache_models": [
          {
              "name": "table",
              "cache_size": 1024,
          },
          {
              "name": "string",
              "cache_size": 1024,
          },
          {
              "name": "field7",
              "cache_size": 1024,
          }
      ]
  }
  ' http://master_server/config/$db_name/$space_name

1. table cache size: Represents the cache size of all fixed-length scalar fields (integer, long, float, double). The default value is 512M.

2. string cache size: Represents the cache size of all variable-length scalar fields (string). The default value is 512M.

3. store_type is the vector field of Mmap that can modify the cache size.


Get cache size
------------
::
 
  curl -XGET http://master_server/config/$db_name/$space_name


1. store_type is the vector field of Mmap to view the cache size. Other storage methods for vector fields do not support viewing the cache size.
