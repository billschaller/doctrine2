The Second Level Cache
======================

.. note::

    The second level cache functionality is marked as experimental for now. It
    is a very complex feature and we cannot guarantee yet that it works stable
    in all cases.

The Second Level Cache (SLC) is designed to reduce the number of queries your
application has to make against your database.

When the SLC is turned on, the cache will be queried for entity data before your
database. If an entity is not found in the cache, a database query will be fired
to fetch the entity data. The entity data will then be stored in the cache, so
that the next time the same entity is requested, no database query will be
necessary.

There are multiple caching modes available via the core SLC implementation. In
general, using the read-only mode is less complex, and will help you to avoid
some consistency issues that can arise when using the read-write strategies.

When planning an implementation using the SLC, keep in mind that the SLC cache
is not aware of changes to persistent data made outside of Doctrine. Caches can,
however, be configured to regularly expire cached data.

Cache Regions
-------------

The SLC does not store instances of entities; instead it caches only an array
containing the entity identifier and property values.

The cache keys for each item stored are built canonically from a hash of the
entity name, collection name, or query text. Additionally, each cache key is
prefixed with a region name.

Each entity class, collection association and query can be assigned a region.
The region name can be configured for entities and collections in the mapping
data for the entity or property. A query object can be assigned a region name
using the query API.

If a region is not assigned for an entity, collection, or query, the default
region will be used.

.. _reference-second-level-cache-regions:

Default Implementations
-----------------------

``Doctrine\ORM\Cache\Region\DefaultRegion`` Is the default implementation.
This is a simple cache region which is compatible with all doctrine-cache
drivers but does not support locking.

``Doctrine\ORM\Cache\Region`` and ``Doctrine\ORM\Cache\ConcurrentRegion``
Define contracts that should be implemented by cache providers.

Implementing one of these contracts will allow you to provide your own
cache implementation that can take advantage of a specific cache driver.

If you want to support locking for ``READ_WRITE`` strategies you should
implement ``Doctrine\ORM\Cache\ConcurrentRegion`` instead of
``Doctrine\ORM\Cache\Region``.

Cache Region Interface
~~~~~~~~~~~~~~~~~~~~~~

``Doctrine\ORM\Cache\Region``

Defines a contract for a non-concurrently managed data region.

A ``Doctrine\ORM\Cache\Region`` is designed to store data without considering concurrency.
This type of region is used for read-only data.

`See API Doc <http://www.doctrine-project.org/api/orm/2.5/class-Doctrine.ORM.Cache.Region.html/>`_.

Concurrent cache region
~~~~~~~~~~~~~~~~~~~~~~~

``Doctrine\ORM\Cache\ConcurrentRegion``

Defines a contract for a concurrently managed data region.

A ``Doctrine\ORM\Cache\ConcurrentRegion`` is designed to store concurrently managed data.
By default, Doctrine provides a very simple implementation based on file locks ``Doctrine\ORM\Cache\Region\FileLockRegion``.

If you want to use a ``READ_WRITE`` cache, you should consider providing your own cache region.


`See API Doc <http://www.doctrine-project.org/api/orm/2.5/class-Doctrine.ORM.Cache.ConcurrentRegion.html/>`_.

Timestamp Region
~~~~~~~~~~~~~~~~

``Doctrine\ORM\Cache\TimestampRegion``

Tracks the timestamps of the most recent updates to particular entity.

`See API Doc <http://www.doctrine-project.org/api/orm/2.5/class-Doctrine.ORM.Cache.TimestampRegion.html/>`_.

Data Storage
~~~~~~~~~~~~

.. todo::
Clean up this section...

When caching collection and queries only identifiers are stored.
The entity values will be stored in its own region

Something like below for an entity region:

.. note::

    The following data structures represents now the cache will looks like, this is not actual cached data.

.. code-block:: php

    <?php
    [
      'region_name:entity_1_hash' => ['id'=> 1, 'name' => 'FooBar', 'associationName'=>null],
      'region_name:entity_2_hash' => ['id'=> 2, 'name' => 'Foo', 'associationName'=>['id'=>11]],
      'region_name:entity_3_hash' => ['id'=> 3, 'name' => 'Bar', 'associationName'=>['id'=>22]]
    ];


If the entity holds a collection that also needs to be cached.
An collection region could look something like :

.. code-block:: php

    <?php
    [
      'region_name:entity_1_coll_assoc_name_hash' => ['ownerId'=> 1, 'list' => [1, 2, 3]],
      'region_name:entity_2_coll_assoc_name_hash' => ['ownerId'=> 2, 'list' => [2, 3]],
      'region_name:entity_3_coll_assoc_name_hash' => ['ownerId'=> 3, 'list' => [2, 4]]
    ];

A query region might be something like :

.. code-block:: php

    <?php
    [
      'region_name:query_1_hash' => ['list' => [1, 2, 3]],
      'region_name:query_2_hash' => ['list' => [2, 3]],
      'region_name:query_3_hash' => ['list' => [2, 4]]
    ];



.. _reference-second-level-cache-mode:

Caching mode
------------

* ``READ_ONLY`` (DEFAULT)

  * Can do reads, inserts and deletes, cannot perform updates or employ any locks.
  * Useful for data that is read frequently but never updated.
  * Best performer.
  * It is Simple.

* ``NONSTRICT_READ_WRITE``

  * Read Write Cache doesn’t employ any locks but can do reads, inserts, updates and deletes.
  * Good if the application needs to update data rarely.
    

* ``READ_WRITE``

  * Read Write cache employs locks before update/delete.
  * Use if data needs to be updated.
  * Slowest strategy.
  * To use it a the cache region implementation must support locking.


Built-in cached persisters
~~~~~~~~~~~~~~~~~~~~~~~~~~

Cached persisters are responsible to access cache regions.

    +-----------------------+-------------------------------------------------------------------------------+
    | Cache Usage           | Persister                                                                     |
    +=======================+===============================================================================+
    | READ_ONLY             | Doctrine\\ORM\\Cache\\Persister\\ReadOnlyCachedEntityPersister                |
    +-----------------------+-------------------------------------------------------------------------------+
    | READ_WRITE            | Doctrine\\ORM\\Cache\\Persister\\ReadWriteCachedEntityPersister               |
    +-----------------------+-------------------------------------------------------------------------------+
    | NONSTRICT_READ_WRITE  | Doctrine\\ORM\\Cache\\Persister\\NonStrictReadWriteCachedEntityPersister      |
    +-----------------------+-------------------------------------------------------------------------------+
    | READ_ONLY             | Doctrine\\ORM\\Cache\\Persister\\ReadOnlyCachedCollectionPersister            |
    +-----------------------+-------------------------------------------------------------------------------+
    | READ_WRITE            | Doctrine\\ORM\\Cache\\Persister\\ReadWriteCachedCollectionPersister           |
    +-----------------------+-------------------------------------------------------------------------------+
    | NONSTRICT_READ_WRITE  | Doctrine\\ORM\\Cache\\Persister\\NonStrictReadWriteCacheCollectionPersister   |
    +-----------------------+-------------------------------------------------------------------------------+

Configuration
-------------
Doctrine allows you to specify configurations and some points of extension for the second-level-cache


Enable Second Level Cache
~~~~~~~~~~~~~~~~~~~~~~~~~

To enable the second-level-cache, you should provide a cache factory
``\Doctrine\ORM\Cache\DefaultCacheFactory`` is the default implementation.

.. code-block:: php

    <?php
    /* @var $config \Doctrine\ORM\Cache\RegionsConfiguration */
    /* @var $cache \Doctrine\Common\Cache\Cache */

    $factory = new \Doctrine\ORM\Cache\DefaultCacheFactory($config, $cache);

    // Enable second-level-cache
    $config->setSecondLevelCacheEnabled();

    // Cache factory
    $config->getSecondLevelCacheConfiguration()
        ->setCacheFactory($factory);


Cache Factory
~~~~~~~~~~~~~

Cache Factory is the main point of extension.

It allows you to provide a specific implementation of the following components :

* ``QueryCache`` Store and retrieve query cache results.
* ``CachedEntityPersister`` Store and retrieve entity results.
* ``CachedCollectionPersister`` Store and retrieve query results.
* ``EntityHydrator``  Transform an entity into a cache entry and cache entry into entities
* ``CollectionHydrator`` Transform a collection into a cache entry and cache entry into collection

`See API Doc <http://www.doctrine-project.org/api/orm/2.5/class-Doctrine.ORM.Cache.DefaultCacheFactory.html/>`_.

Region Lifetime
~~~~~~~~~~~~~~~

To specify a default lifetime for all regions or specify a different lifetime for a specific region.

.. code-block:: php

    <?php
    /* @var $config \Doctrine\ORM\Configuration */
    /* @var $cacheConfig \Doctrine\ORM\Configuration */
    $cacheConfig  =  $config->getSecondLevelCacheConfiguration();
    $regionConfig =  $cacheConfig->getRegionsConfiguration();

    // Cache Region lifetime
    $regionConfig->setLifetime('my_entity_region', 3600);   // Time to live for a specific region; In seconds
    $regionConfig->setDefaultLifetime(7200);                // Default time to live; In seconds


Cache Log
~~~~~~~~~
By providing a cache logger you should be able to get information about all cache operations such as hits, misses and puts.

``\Doctrine\ORM\Cache\Logging\StatisticsCacheLogger`` is a built-in implementation that provides basic statistics.

 .. code-block:: php

    <?php
    /* @var $config \Doctrine\ORM\Configuration */
    $logger = new \Doctrine\ORM\Cache\Logging\StatisticsCacheLogger();

    // Cache logger
    $config->setSecondLevelCacheEnabled(true);
    $config->getSecondLevelCacheConfiguration()
        ->setCacheLogger($logger);


    // Collect cache statistics

    // Get the number of entries successfully retrieved from a specific region.
    $logger->getRegionHitCount('my_entity_region');

    // Get the number of cached entries *not* found in a specific region.
    $logger->getRegionMissCount('my_entity_region');

    // Get the number of cacheable entries put in cache.
    $logger->getRegionPutCount('my_entity_region');

    // Get the total number of put in all regions.
    $logger->getPutCount();

    // Get the total number of entries successfully retrieved from all regions.
    $logger->getHitCount();

    // Get the total number of cached entries *not* found in all regions.
    $logger->getMissCount();

If you want to get more information you should implement ``\Doctrine\ORM\Cache\Logging\CacheLogger``.
and collect all information you want.

`See API Doc <http://www.doctrine-project.org/api/orm/2.5/class-Doctrine.ORM.Cache.CacheLogger.html/>`_.


Entity cache definition
-----------------------
* Entity cache configuration allows you to define the caching strategy and region for an entity.

  * ``usage`` Specifies the caching strategy: ``READ_ONLY``, ``NONSTRICT_READ_WRITE``, ``READ_WRITE``. see :ref:`reference-second-level-cache-mode`
  * ``region`` Optional value that specifies the name of the second level cache region.


.. configuration-block::

    .. code-block:: php

        <?php
        /**
         * @Entity
         * @Cache(usage="READ_ONLY", region="my_entity_region")
         */
        class Country
        {
            /**
             * @Id
             * @GeneratedValue
             * @Column(type="integer")
             */
            protected $id;

            /**
             * @Column(unique=true)
             */
            protected $name;

            // other properties and methods
        }

    .. code-block:: xml

        <?xml version="1.0" encoding="utf-8"?>
        <doctrine-mapping xmlns="http://doctrine-project.org/schemas/orm/doctrine-mapping" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://doctrine-project.org/schemas/orm/doctrine-mapping http://doctrine-project.org/schemas/orm/doctrine-mapping.xsd">
          <entity name="Country">
            <cache usage="READ_ONLY" region="my_entity_region" />
            <id name="id" type="integer" column="id">
              <generator strategy="IDENTITY"/>
            </id>
            <field name="name" type="string" column="name"/>
          </entity>
        </doctrine-mapping>

    .. code-block:: yaml

        Country:
          type: entity
          cache:
            usage : READ_ONLY
            region : my_entity_region
          id:
            id:
              type: integer
              id: true
              generator:
                strategy: IDENTITY
          fields:
            name:
              type: string


Association cache definition
----------------------------
The most common use case is to cache entities. But we can also cache relationships.
It caches the primary keys of association and cache each element will be cached into its region.


.. configuration-block::

    .. code-block:: php

        <?php
        /**
         * @Entity
         * @Cache("NONSTRICT_READ_WRITE")
         */
        class State
        {
            /**
             * @Id
             * @GeneratedValue
             * @Column(type="integer")
             */
            protected $id;

            /**
             * @Column(unique=true)
             */
            protected $name;

            /**
             * @Cache("NONSTRICT_READ_WRITE")
             * @ManyToOne(targetEntity="Country")
             * @JoinColumn(name="country_id", referencedColumnName="id")
             */
            protected $country;

            /**
             * @Cache("NONSTRICT_READ_WRITE")
             * @OneToMany(targetEntity="City", mappedBy="state")
             */
            protected $cities;

            // other properties and methods
        }

    .. code-block:: xml

        <?xml version="1.0" encoding="utf-8"?>
        <doctrine-mapping xmlns="http://doctrine-project.org/schemas/orm/doctrine-mapping" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://doctrine-project.org/schemas/orm/doctrine-mapping http://doctrine-project.org/schemas/orm/doctrine-mapping.xsd">
          <entity name="State">

            <cache usage="NONSTRICT_READ_WRITE" />

            <id name="id" type="integer" column="id">
              <generator strategy="IDENTITY"/>
            </id>

            <field name="name" type="string" column="name"/>
            
            <many-to-one field="country" target-entity="Country">
              <cache usage="NONSTRICT_READ_WRITE" />

              <join-columns>
                <join-column name="country_id" referenced-column-name="id"/>
              </join-columns>
            </many-to-one>

            <one-to-many field="cities" target-entity="City" mapped-by="state">
              <cache usage="NONSTRICT_READ_WRITE"/>
            </one-to-many>
          </entity>
        </doctrine-mapping>

    .. code-block:: yaml

        State:
          type: entity
          cache:
            usage : NONSTRICT_READ_WRITE
          id:
            id:
              type: integer
              id: true
              generator:
                strategy: IDENTITY
          fields:
            name:
              type: string

          manyToOne:
            state:
              targetEntity: Country
              joinColumns:
                country_id:
                  referencedColumnName: id
              cache:
                usage : NONSTRICT_READ_WRITE

          oneToMany:
            cities:
              targetEntity:City
              mappedBy: state
              cache:
                usage : NONSTRICT_READ_WRITE


> Note: for this to work, the target entity must also be marked as cacheable.

Cache usage
~~~~~~~~~~~

Basic entity cache

.. code-block:: php

    <?php
    $em->persist(new Country($name));
    $em->flush();                         // Hit database to insert the row and put into cache

    $em->clear();                         // Clear entity manager

    $country1  = $em->find('Country', 1); // Retrieve item from cache

    $country->setName("New Name");
    $em->persist($country);
    $em->flush();                         // Hit database to update the row and update cache

    $em->clear();                         // Clear entity manager

    $country2  = $em->find('Country', 1); // Retrieve item from cache
                                          // Notice that $country1 and $country2 are not the same instance.


Association cache

.. code-block:: php

    <?php
    // Hit database to insert the row and put into cache
    $em->persist(new State($name, $country));
    $em->flush();

    // Clear entity manager
    $em->clear();

    // Retrieve item from cache
    $state = $em->find('State', 1);

    // Hit database to update the row and update cache entry
    $state->setName("New Name");
    $em->persist($state);
    $em->flush();

    // Create a new collection item
    $city = new City($name, $state);
    $state->addCity($city);

    // Hit database to insert new collection item,
    // put entity and collection cache into cache.
    $em->persist($city);
    $em->persist($state);
    $em->flush();

    // Clear entity manager
    $em->clear();

    // Retrieve item from cache
    $state = $em->find('State', 1);

    // Retrieve association from cache
    $country = $state->getCountry();

    // Retrieve collection from cache
    $cities = $state->getCities();

    echo $country->getName();
    echo $state->getName();

    // Retrieve each collection item from cache
    foreach ($cities as $city) {
        echo $city->getName();
    }

.. note::

    Notice that all entities should be marked as cacheable.

Using the query cache
---------------------

The second level cache stores the entities, associations and collections.
The query cache stores the results of the query but as identifiers, entity values are actually stored in the 2nd level cache.

.. note::

    Query cache should always be used in conjunction with the second-level-cache for those entities which should be cached.

.. code-block:: php

    <?php
    /* @var $em \Doctrine\ORM\EntityManager */

    // Execute database query, store query cache and entity cache
    $result1 = $em->createQuery('SELECT c FROM Country c ORDER BY c.name')
        ->setCacheable(true)
        ->getResult();

    $em->clear()

    // Check if query result is valid and load entities from cache
    $result2 = $em->createQuery('SELECT c FROM Country c ORDER BY c.name')
        ->setCacheable(true)
        ->getResult();

Cache mode
~~~~~~~~~~

The Cache Mode controls how a particular query interacts with the second-level cache:

* ``Cache::MODE_GET`` - May read items from the cache, but will not add items.
* ``Cache::MODE_PUT`` - Will never read items from the cache, but will add items to the cache as it reads them from the database.
* ``Cache::MODE_NORMAL`` - May read items from the cache, and add items to the cache.
* ``Cache::MODE_REFRESH`` - The query will never read items from the cache, but will refresh items to the cache as it reads them from the database.

.. code-block:: php

    <?php
    /* @var $em \Doctrine\ORM\EntityManager */
    // Will refresh the query cache and all entities the cache as it reads from the database.
    $result1 = $em->createQuery('SELECT c FROM Country c ORDER BY c.name')
        ->setCacheMode(Cache::MODE_GET)
        ->setCacheable(true)
        ->getResult();

.. note::

    The the default query cache mode is ```Cache::MODE_NORMAL```

DELETE / UPDATE queries
~~~~~~~~~~~~~~~~~~~~~~~

DQL UPDATE / DELETE statements are ported directly into a database and bypass the second-level cache,
Entities that are already cached will NOT be invalidated.
However the cached data could be evicted using the cache API or an special query hint.


Execute the ``UPDATE`` and invalidate ``all cache entries`` using ``Query::HINT_CACHE_EVICT``

.. code-block:: php

    <?php
    // Execute and invalidate
    $this->_em->createQuery("UPDATE Entity\Country u SET u.name = 'unknown' WHERE u.id = 1")
        ->setHint(Query::HINT_CACHE_EVICT, true)
        ->execute();


Execute the ``UPDATE`` and invalidate ``all cache entries`` using the cache API

.. code-block:: php

    <?php
    // Execute
    $this->_em->createQuery("UPDATE Entity\Country u SET u.name = 'unknown' WHERE u.id = 1")
        ->execute();
    // Invoke Cache API
    $em->getCache()->evictEntityRegion('Entity\Country');


Execute the ``UPDATE`` and invalidate ``a specific cache entry`` using the cache API

.. code-block:: php

    <?php
    // Execute
    $this->_em->createQuery("UPDATE Entity\Country u SET u.name = 'unknown' WHERE u.id = 1")
        ->execute();
    // Invoke Cache API
    $em->getCache()->evictEntity('Entity\Country', 1);

Using the repository query cache
--------------------------------

As well as ``Query Cache`` all persister queries store only identifier values for an individual query.
All persister use a single timestamps cache region keeps track of the last update for each persister,
When a query is loaded from cache, the timestamp region is checked for the last update for that persister.
Using the last update timestamps as part of the query key invalidate the cache key when an update occurs.

.. code-block:: php

    <?php
    // load from database and store cache query key hashing the query + parameters + last timestamp cache region..
    $entities   = $em->getRepository('Entity\Country')->findAll();

    // load from query and entities from cache..
    $entities   = $em->getRepository('Entity\Country')->findAll();

    // update the timestamp cache region for Country
    $em->persist(new Country('zombieland'));
    $em->flush();
    $em->clear();

    // Reload from database.
    // At this point the query cache key if not logger valid, the select goes straight
    $entities   = $em->getRepository('Entity\Country')->findAll();

Cache API
---------

Caches are not aware of changes made by another application.
However, you can use the cache API to check / invalidate cache entries.

.. code-block:: php

    <?php
    /* @var $cache \Doctrine\ORM\Cache */
    $cache = $em->getCache();

    $cache->containsEntity('Entity\State', 1)      // Check if the cache exists
    $cache->evictEntity('Entity\State', 1);        // Remove an entity from cache
    $cache->evictEntityRegion('Entity\State');     // Remove all entities from cache

    $cache->containsCollection('Entity\State', 'cities', 1);   // Check if the cache exists
    $cache->evictCollection('Entity\State', 'cities', 1);      // Remove an entity collection from cache
    $cache->evictCollectionRegion('Entity\State', 'cities');   // Remove all collections from cache

Limitations
-----------

Composite primary key
~~~~~~~~~~~~~~~~~~~~~

Composite primary key are supported by second level cache,
however when one of the keys is an association the cached entity should always be retrieved using the association identifier.
For performance reasons the cache API does not extract from composite primary key.

.. code-block:: php

    <?php
    /**
     * @Entity
     */
    class Reference
    {
        /**
         * @Id
         * @ManyToOne(targetEntity="Article", inversedBy="references")
         * @JoinColumn(name="source_id", referencedColumnName="article_id")
         */
        private $source;

        /**
         * @Id
         * @ManyToOne(targetEntity="Article")
         * @JoinColumn(name="target_id", referencedColumnName="article_id")
         */
        private $target;
    }

    // Supported
    /* @var $article Article */
    $article = $em->find('Article', 1);

    // Supported
    /* @var $article Article */
    $article = $em->find('Article', $article);

    // Supported
    $id        = array('source' => 1, 'target' => 2);
    $reference = $em->find('Reference', $id);

    // NOT Supported
    $id        = array('source' => new Article(1), 'target' => new Article(2));
    $reference = $em->find('Reference', $id);

Distributed environments
~~~~~~~~~~~~~~~~~~~~~~~~

Some cache driver are not meant to be used in a distributed environment.
Load-balancer for distributing workloads across multiple computing resources
should be used in conjunction with distributed caching system such as memcached, redis, riak ...

Caches should be used with care when using a load-balancer if you don't share the cache.
While using APC or any file based cache update occurred in a specific machine would not reflect to the cache in other machines.


Paginator
~~~~~~~~~

Count queries generated by ``Doctrine\ORM\Tools\Pagination\Paginator`` are not cached by second-level cache.
Although entities and query result are cached count queries will hit the database every time.
