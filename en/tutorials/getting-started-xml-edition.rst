Getting Started
===============

Doctrine 2 is a project that aims to handle the persistence of the
domain model in a non-interfering way. The Data Mapper pattern is
at the heart of this project, aiming for a complete separation of
the domain/business logic from the persistence in a relational
database management system. The benefit of Doctrine for the
programmer is the ability to focus solely on the business logic and
worry about persistence only as a secondary task. This doesn't mean
persistence is not important to Doctrine 2, however it is our
belief that there are considerable benefits for object-oriented
programming if persistence and entities are kept perfectly
separated.

What are Entities?
------------------

Entities are lightweight PHP Objects that don't need to extend any
abstract base class or interface. An entity class must not be final
or contain final methods. Additionally it must not implement
**clone** nor **wakeup** or :doc:`do so safely <../cookbook/implementing-wakeup-or-clone>`.

An entity contains persistable properties. A persistable property
is an instance variable of the entity that is saved into and retrieved from the database
by Doctrine's data mapping capabilities.

An Example Model: Bug Tracker
-----------------------------

For this Getting Started Guide for Doctrine we will implement the
Bug Tracker domain model from the
`Zend\_Db\_Table <http://framework.zend.com/manual/en/zend.db.table.html>`_
documentation. Reading their documentation we can extract the
requirements to be:


-  A Bugs has a description, creation date, status, reporter and
   engineer
-  A bug can occur on different products (platforms)
-  Products have a name.
-  Bug Reporter and Engineers are both Users of the System.
-  A user can create new bugs.
-  The assigned engineer can close a bug.
-  A user can see all his reported or assigned bugs.
-  Bugs can be paginated through a list-view.

.. warning::

    This tutorial is incrementally building up your Doctrine 2
    knowledge and even lets you make some mistakes, to show some common
    pitfalls in mapping Entities to a database. Don't blindly
    copy-paste the examples here, it is not production ready without
    the additional comments and knowledge this tutorial teaches.


A first prototype
-----------------

A first simplified design for this domain model might look like the
following set of classes:

.. code-block:: php

    <?php
    class Bug
    {
        public $id;
        public $description;
        public $created;
        public $status;
        public $products = array();
        public $reporter;
        public $engineer;
    }
    class Product
    {
        public $id;
        public $name;
    }
    class User
    {
        public $id;
        public $name;
        public $reportedBugs = array();
        public $assignedBugs = array();
    }

.. warning::

    This is only a prototype, please don't use public properties with
    Doctrine 2 at all, the "Queries for Application Use-Cases" section
    shows you why. In combination with proxies public properties can
    make up for pretty nasty bugs.


Because we will focus on the mapping aspect, no effort is being
made to encapsulate the business logic in this example. All
persistable properties are public in visibility. We will soon see
that this is not the best solution in combination with Doctrine 2,
one restriction that actually forces you to encapsulate your
properties. For persistence Doctrine 2 actually uses Reflection to
access the values in all your entities properties.

Many of the fields are single scalar values, for example the 3 ID
fields of the entities, their names, description, status and change
dates. Doctrine 2 can easily handle these single values as can any
other ORM. From a point of our domain model they are ready to be
used right now and we will see at a later stage how they are mapped
to the database.

There are also several references between objects in this domain
model, whose semantics are discussed case by case at this point to
explain how Doctrine handles them. In general each OneToOne or
ManyToOne Relation in the Database is replaced by an instance of
the related object in the domain model. Each OneToMany or
ManyToMany Relation is replaced by a collection of instances in the
domain model.

If you think this through carefully you realize Doctrine 2 will
load up the complete database in memory if you access one object.
However by default Doctrine generates Lazy Load proxies of entities
or collections of all the relations that haven't been explicitly
retrieved from the database yet.

To be able to use lazyload with collections, simple PHP arrays have
to be replaced by a generic collection interface Doctrine which
tries to act as array as much as possible using ArrayAccess,
IteratorAggregate and Countable interfaces. The class is the most
simple implementation of this interface.

Now that we know this, we have to clear up our domain model to cope
with the assumptions about related collections:

.. code-block:: php

    <?php
    use Doctrine\Common\Collections\ArrayCollection;
    
    class Bug
    {
        public $products = null;
    
        public function __construct()
        {
            $this->products = new ArrayCollection();
        }
    }
    
    class User
    {
        public $reportedBugs = null;
        public $assignedBugs = null;
    
        public function __construct()
        {
            $this->reportedBugs = new ArrayCollection();
            $this->assignedBugs = new ArrayCollection();
        }
    }

Whenever an entity is recreated from the database, an Collection
implementation of the type Doctrine is injected into your entity
instead of an array. Compared to the ArrayCollection this
implementation helps the Doctrine ORM understand the changes that
have happened to the collection which are noteworthy for
persistence.

.. warning::

    Lazy load proxies always contain an instance of
    Doctrine's EntityManager and all its dependencies. Therefore a
    var\_dump() will possibly dump a very large recursive structure
    which is impossible to render and read. You have to use
    ``Doctrine\Common\Util\Debug::dump()`` to restrict the dumping to a
    human readable level. Additionally you should be aware that dumping
    the EntityManager to a Browser may take several minutes, and the
    Debug::dump() method just ignores any occurrences of it in Proxy
    instances.


Because we only work with collections for the references we must be
careful to implement a bidirectional reference in the domain model.
The concept of owning or inverse side of a relation is central to
this notion and should always be kept in mind. The following
assumptions are made about relations and have to be followed to be
able to work with Doctrine 2. These assumptions are not unique to
Doctrine 2 but are best practices in handling database relations
and Object-Relational Mapping.


-  Changes to Collections are saved or updated, when the entity on
   the *owning* side of the collection is saved or updated.
-  Saving an Entity at the inverse side of a relation never
   triggers a persist operation to changes to the collection.
-  In a one-to-one relation the entity holding the foreign key of
   the related entity on its own database table is *always* the owning
   side of the relation.
-  In a many-to-many relation, both sides can be the owning side of
   the relation. However in a bi-directional many-to-many relation
   only one is allowed to be.
-  In a many-to-one relation the Many-side is the owning side by
   default, because it holds the foreign key.
-  The OneToMany side of a relation is inverse by default, since
   the foreign key is saved on the Many side. A OneToMany relation can
   only be the owning side, if its implemented using a ManyToMany
   relation with join table and restricting the one side to allow only
   UNIQUE values per database constraint.

.. note::

    Consistency of bi-directional references on the inverse side of a
    relation have to be managed in userland application code. Doctrine
    cannot magically update your collections to be consistent.


In the case of Users and Bugs we have references back and forth to
the assigned and reported bugs from a user, making this relation
bi-directional. We have to change the code to ensure consistency of
the bi-directional reference:

.. code-block:: php

    <?php
    class Bug
    {
        private $engineer;
        private $reporter;
    
        public function setEngineer($engineer)
        {
            $engineer->assignedToBug($this);
            $this->engineer = $engineer;
        }
    
        public function setReporter($reporter)
        {
            $reporter->addReportedBug($this);
            $this->reporter = $reporter;
        }
    
        public function getEngineer()
        {
            return $this->engineer;
        }
    
        public function getReporter()
        {
            return $this->reporter;
        }
    }
    class User
    {
        private $reportedBugs = null;
        private $assignedBugs = null;

        public function addReportedBug($bug)
        {
            $this->reportedBugs[] = $bug;
        }
    
        public function assignedToBug($bug)
        {
            $this->assignedBugs[] = $bug;
        }
    }

I chose to name the inverse methods in past-tense, which should
indicate that the actual assigning has already taken place and the
methods are only used for ensuring consistency of the references.
You can see from ``User::addReportedBug()`` and
``User::assignedToBug()`` that using this method in userland alone
would not add the Bug to the collection of the owning side in
``Bug::$reporter`` or ``Bug::$engineer``. Using these methods and
calling Doctrine for persistence would not update the collections
representation in the database.

Only using ``Bug::setEngineer()`` or ``Bug::setReporter()``
correctly saves the relation information. We also set both
collection instance variables to protected, however with PHP 5.3's
new features Doctrine is still able to use Reflection to set and
get values from protected and private properties.

The ``Bug::$reporter`` and ``Bug::$engineer`` properties are
Many-To-One relations, which point to a User. In a normalized
relational model the foreign key is saved on the Bug's table, hence
in our object-relation model the Bug is at the owning side of the
relation. You should always make sure that the use-cases of your
domain model should drive which side is an inverse or owning one in
your Doctrine mapping. In our example, whenever a new bug is saved
or an engineer is assigned to the bug, we don't want to update the
User to persist the reference, but the Bug. This is the case with
the Bug being at the owning side of the relation.

Bugs reference Products by a uni-directional ManyToMany relation in
the database that points from from Bugs to Products.

.. code-block:: php

    <?php
    class Bug
    {
        private $products = null;
    
        public function assignToProduct($product)
        {
            $this->products[] = $product;
        }
    
        public function getProducts()
        {
            return $this->products;
        }
    }

We are now finished with the domain model given the requirements.
From the simple model with public properties only we had to do
quite some work to get to a model where we encapsulated the
references between the objects to make sure we don't break its
consistent state when using Doctrine.

However up to now the assumptions Doctrine imposed on our business
objects have not restricting us much in our domain modelling
capabilities. Actually we would have encapsulated access to all the
properties anyways by using object-oriented best-practices.

Metadata Mappings for our Entities
----------------------------------

Up to now we have only implemented our Entities as Data-Structures
without actually telling Doctrine how to persist them in the
database. If perfect in-memory databases would exist, we could now
finish the application using these entities by implementing code to
fulfil all the requirements. However the world isn't perfect and we
have to persist our entities in some storage to make sure we don't
loose their state. Doctrine currently serves Relational Database
Management Systems. In the future we are thinking to support NoSQL
vendors like CouchDb or MongoDb, however this is still far in the
future.

The next step for persistence with Doctrine is to describe the
structure of our domain model entities to Doctrine using a metadata
language. The metadata language describes how entities, their
properties and references should be persisted and what constraints
should be applied to them.

Metadata for entities are loaded using a
``Doctrine\ORM\Mapping\Driver\Driver`` implementation and Doctrine
2 already comes with XML, YAML and Annotations Drivers. This
Getting Started Guide will show the mappings for all Mapping Drivers.
References in the text will be made to the XML mapping.

Since we haven't namespaced our three entities, we have to
implement three mapping files called Bug.dcm.xml, Product.dcm.xml
and User.dcm.xml (or .yml) and put them into a distinct folder for mapping
configurations. For the annotations driver we need to use
doc-block comments on the entity classes themselves.

The first discussed definition will be for the Product, since it is
the most simple one:

.. configuration-block::

    .. code-block:: php

        <?php
        /**
         * @Entity @Table(name="products")
         */
        class Product
        {
            /** @Id @Column(type="integer") @GeneratedValue */
            public $id;
            /** @Column(type="string") */
            public $name;
        }

    .. code-block:: xml

        <doctrine-mapping xmlns="http://doctrine-project.org/schemas/orm/doctrine-mapping"
              xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
              xsi:schemaLocation="http://doctrine-project.org/schemas/orm/doctrine-mapping
                            http://doctrine-project.org/schemas/orm/doctrine-mapping.xsd">

              <entity name="Product" table="products">
                  <id name="id" type="integer" column="product_id">
                      <generator strategy="AUTO" />
                  </id>

                  <field name="name" column="product_name" type="string" />
              </entity>
        </doctrine-mapping>

    .. code-block:: yaml

        Product:
          type: entity
          table: products
          id:
            id:
              type: integer
              generator:
                strategy: AUTO
          fields:
            name:
              type: string

The top-level ``entity`` definition tag specifies information about
the class and table-name. The primitive type ``Product::$name`` is
defined as ``field`` attributes. The Id property is defined with
the ``id`` tag. The id has a ``generator`` tag nested inside which
defines that the primary key generation mechanism automatically
uses the database platforms native id generation strategy, for
example AUTO INCREMENT in the case of MySql or Sequences in the
case of PostgreSql and Oracle.

We then go on specifying the definition of a Bug:

.. configuration-block::
    .. code-block:: php

        <?php
        /**
         * @Entity @Table(name="bugs")
         */
        class Bug
        {
            /**
             * @Id @Column(type="integer") @GeneratedValue
             */
            public $id;
            /**
             * @Column(type="string")
             */
            public $description;
            /**
             * @Column(type="datetime")
             */
            public $created;
            /**
             * @Column(type="string")
             */
            public $status;

            /**
             * @ManyToOne(targetEntity="User", inversedBy="assignedBugs")
             */
            private $engineer;

            /**
             * @ManyToOne(targetEntity="User", inversedBy="reportedBugs")
             */
            private $reporter;

            /**
             * @ManyToMany(targetEntity="Product")
             */
            private $products;
        }

    .. code-block:: xml

        <doctrine-mapping xmlns="http://doctrine-project.org/schemas/orm/doctrine-mapping"
              xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
              xsi:schemaLocation="http://doctrine-project.org/schemas/orm/doctrine-mapping
                            http://doctrine-project.org/schemas/orm/doctrine-mapping.xsd">

            <entity name="Bug" table="bugs">
                <id name="id" type="integer">
                    <generator strategy="AUTO" />
                </id>

                <field name="description" type="text" />
                <field name="created" type="datetime" />
                <field name="status" type="string" />

                <many-to-one target-entity="User" field="reporter" inversed-by="reportedBugs" />
                <many-to-one target-entity="User" field="engineer" inversed-by="assignedBugs" />

                <many-to-many target-entity="Product" field="products" />
            </entity>
        </doctrine-mapping>

    .. code-block:: yaml

        Bug:
          type: entity
          table: bugs
          id:
            id:
              type: integer
              generator:
                strategy: AUTO
          fields:
            description:
              type: text
            created:
              type: datetime
            status:
              type: string
          manyToOne:
            reporter:
              targetEntity: User
              inversedBy: reportedBugs
            engineer:
              targetEntity: User
              inversedBy: assignedBugs
          manyToMany:
            products:
              targetEntity: Product
              

Here again we have the entity, id and primitive type definitions.
The column names are used from the Zend\_Db\_Table examples and
have different names than the properties on the Bug class.
Additionally for the "created" field it is specified that it is of
the Type "DATETIME", which translates the YYYY-mm-dd HH:mm:ss
Database format into a PHP DateTime instance and back.

After the field definitions the two qualified references to the
user entity are defined. They are created by the ``many-to-one``
tag. The class name of the related entity has to be specified with
the ``target-entity`` attribute, which is enough information for
the database mapper to access the foreign-table. Since
``reporter`` and ``engineer`` are on the owning side of a
bi-directional relation we also have to specify the ``inversed-by``
attribute. They have to point to the field names on the inverse
side of the relationship. We will see in the next example that the ``inversed-by``
attribute has a counterpart ``mapped-by`` which makes that
the inverse side.

The last missing property is the ``Bug::$products`` collection. It
holds all products where the specific bug is occurring in. Again
you have to define the ``target-entity`` and ``field`` attributes
on the ``many-to-many`` tag. Furthermore you have to specify the
details of the many-to-many join-table and its foreign key columns.
The definition is rather complex, however relying on the XML
auto-completion I got it working easily, although I forget the
schema details all the time.

The last missing definition is that of the User entity:

.. configuration-block::

    .. code-block:: php

        <?php
        /**
         * @Entity @Table(name="users")
         */
        class User
        {
            /**
             * @Id @GeneratedValue @Column(type="integer")
             * @var string
             */
            public $id;

            /**
             * @Column(type="string")
             * @var string
             */
            public $name;

            /**
             * @OneToMany(targetEntity="Bug", mappedBy="reporter")
             * @var Bug[]
             */
            protected $reportedBugs = null;

            /**
             * @OneToMany(targetEntity="Bug", mappedBy="engineer")
             * @var Bug[]
             */
            protected $assignedBugs = null;

    .. code-block:: xml

        <doctrine-mapping xmlns="http://doctrine-project.org/schemas/orm/doctrine-mapping"
              xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
              xsi:schemaLocation="http://doctrine-project.org/schemas/orm/doctrine-mapping
                            http://doctrine-project.org/schemas/orm/doctrine-mapping.xsd">

             <entity name="User" name="users">
                 <id name="id" type="integer">
                     <generator strategy="AUTO" />
                 </id>

                 <field name="name" type="string" />

                 <one-to-many target-entity="Bug" field="reportedBugs" mapped-by="reporter" />
                 <one-to-many target-entity="Bug" field="assignedBugs" mapped-by="engineer" />
             </entity>
        </doctrine-mapping>

    .. code-block:: yaml

        User:
          type: entity
          table: users
          id:
            id:
              type: integer
              generator:
                strategy: AUTO
          fields:
            name:
              type: string
          oneToMany:
            reportedBugs:
              targetEntity: Bug
              mappedBy: reporter
            assignedBugs:
              targetEntity: Bug
              mappedBy: engineer

Here are some new things to mention about the ``one-to-many`` tags.
Remember that we discussed about the inverse and owning side. Now
both reportedBugs and assignedBugs are inverse relations, which
means the join details have already been defined on the owning
side. Therefore we only have to specify the property on the Bug
class that holds the owning sides.

This example has a fair overview of the most basic features of the
metadata definition language.

Obtaining the EntityManager
---------------------------

Doctrine's public interface is the EntityManager, it provides the
access point to the complete lifecycle management of your entities
and transforms entities from and back to persistence. You have to
configure and create it to use your entities with Doctrine 2. I
will show the configuration steps and then discuss them step by
step:

.. code-block:: php

    <?php
    // Setup Autoloader (1)
    // See :doc:`Configuration <../reference/configuration>` for up to date autoloading details.
    
    $config = new Doctrine\ORM\Configuration(); // (2)
    
    // Proxy Configuration (3)
    $config->setProxyDir(__DIR__.'/lib/MyProject/Proxies');
    $config->setProxyNamespace('MyProject\Proxies');
    $config->setAutoGenerateProxyClasses((APPLICATION_ENV == "development"));
    
    // Mapping Configuration (4)
    $driverImpl = new Doctrine\ORM\Mapping\Driver\XmlDriver(__DIR__."/config/mappings/xml");
    //$driverImpl = new Doctrine\ORM\Mapping\Driver\XmlDriver(__DIR__."/config/mappings/yml");
    //$driverImpl = $config->newDefaultAnnotationDriver(__DIR__."/entities");
    $config->setMetadataDriverImpl($driverImpl);
    
    // Caching Configuration (5)
    if (APPLICATION_ENV == "development") {
        $cache = new \Doctrine\Common\Cache\ArrayCache();
    } else {
        $cache = new \Doctrine\Common\Cache\ApcCache();
    }
    $config->setMetadataCacheImpl($cache);
    $config->setQueryCacheImpl($cache);
    
    // database configuration parameters (6)
    $conn = array(
        'driver' => 'pdo_sqlite',
        'path' => __DIR__ . '/db.sqlite',
    );
    
    // obtaining the entity manager (7)
    $evm = new Doctrine\Common\EventManager()
    $entityManager = \Doctrine\ORM\EntityManager::create($conn, $config, $evm);

The first block sets up the autoloading capabilities of Doctrine. I
am registering the Doctrine namespace to the given path. To add
your own namespace you can instantiate another ``ClassLoader`` with
different namespace and path arguments. There is no requirement to
use the Doctrine ``ClassLoader`` for your autoloading needs, you
can use whatever suits you best.

The second block contains of the instantiation of the ORM
Configuration object. Besides the configuration shown in the next
blocks there are several others with are all explained in the
:doc:`Configuration section of the manual <../reference/configuration>`.

The Proxy Configuration is a required block for your application,
you have to specify where Doctrine writes the PHP code for Proxy
Generation. Proxies are children of your entities generated by
Doctrine to allow for type-safe lazy loading. We will see in a
later chapter how exactly this works. Besides the path to the
proxies we also specify which namespace they will reside under as
well as a flag ``autoGenerateProxyClasses`` indicating that proxies
should be re-generated on each request, which is recommended for
development. In production this should be prevented at all costs,
the proxy class generation can be quite costly.

The fourth block contains the mapping driver details. We will use
XML Mapping in this example, so we configure the ``XmlDriver``
instance with a path to mappings configuration folder where we put
the Bug.dcm.xml, Product.dcm.xml and User.dcm.xml.

In the 5th block the caching configuration is set. In production we
use caching only on a per request-basis using the ArrayCache. In
production it is literally required to use Apc, Memcache or XCache
to get the full speed out of Doctrine. Internally Doctrine uses
caching heavily for the Metadata and DQL Query Language so make
sure you use a caching mechanism.

The 6th block shows the configuration options required to connect
to a database, in my case a file-based sqlite database. All the
configuration options for all the shipped drivers are given in the
`DBAL Configuration section of the manual <http://www.doctrine-project.org/documentation/manual/2_0/en/dbal>`_.

The last block shows how the ``EntityManager`` is obtained from a
factory method, Here we also pass in an ``EventManager`` instance
which is optional. However using the EventManager you can hook in
to the lifecycle of entities, which is a common use-case, so you
know how to configure it already.

Generating the Database Schema
------------------------------

Now that we have defined the Metadata Mappings and bootstrapped the
EntityManager we want to generate the relational database schema
from it. Doctrine has a Command-Line-Interface that allows you to
access the SchemaTool, a component that generates the required
tables to work with the metadata.

For the command-line tool to work a cli-config.php file has to be
present in the project root directory, where you will execute the
doctrine command. Its a fairly simple file:

.. code-block:: php

    <?php
    $helperSet = new \Symfony\Components\Console\Helper\HelperSet(array(
        'em' => new \Doctrine\ORM\Tools\Console\Helper\EntityManagerHelper($entityManager)
    ));
    $cli->setHelperSet($helperSet);

You can then change into your project directory and call the
Doctrine command-line tool:

::

    doctrine@my-desktop> cd myproject/
    doctrine@my-desktop> doctrine orm:schema-tool:create

.. note::

    The ``doctrine`` command will only be present if you installed
    Doctrine from PEAR. Otherwise you will have to dig into the
    ``bin/doctrine.php`` code of your Doctrine 2 directory to setup
    your doctrine command-line client.

    See the
    :doc:`Tools section of the manual <../reference/tools>`
    on how to setup the Doctrine console correctly.


During the development you probably need to re-create the database
several times when changing the Entity metadata. You can then
either re-create the database:

::

    doctrine@my-desktop> doctrine orm:schema-tool:drop --force
    doctrine@my-desktop> doctrine orm:schema-tool:create

Or use the update functionality:

::

    doctrine@my-desktop> doctrine orm:schema-tool:update --force

The updating of databases uses a Diff Algorithm for a given
Database Schema, a cornerstone of the ``Doctrine\DBAL`` package,
which can even be used without the Doctrine ORM package. However
its not available in SQLite since it does not support ALTER TABLE.

Writing Entities into the Database
----------------------------------

Having created the schema we can now start and save entities in the
database. For starters we need a create user use-case:

.. code-block:: php

    <?php
    $newUsername = "beberlei";
    
    $user = new User();
    $user->name = $newUsername;
    
    $entityManager->persist($user);
    $entityManager->flush();

Products can also be created:

.. code-block:: php

    <?php
    $newProductName = "My Product";
    
    $product = new Product();
    $product->name = $newProductName;
    
    $entityManager->persist($product);
    $entityManager->flush();

So what is happening in those two snippets? In both examples the
class creation is pretty standard, the interesting bits are the
communication with the ``EntityManager``. To notify the
EntityManager that a new entity should be inserted into the
database you have to call ``persist()``. However the EntityManager
does not act on this, its merely notified. You have to explicitly
call ``flush()`` to have the EntityManager write those two entities
to the database.

You might wonder why does this distinction between persist
notification and flush exist? Doctrine 2 uses the UnitOfWork
pattern to aggregate all writes (INSERT, UDPATE, DELETE) into one
single fast transaction, which is executed when flush is called.
Using this approach the write-performance is significantly faster
than in a scenario where updates are done for each entity in
isolation. In more complex scenarios than the previous two, you are
free to request updates on many different entities and all flush
them at once.

Doctrine's UnitOfWork detects entities that have changed after
retrieval from the database automatically when the flush operation
is called, so that you only have to keep track of those entities
that are new or to be removed and pass them to
``EntityManager#persist()`` and ``EntityManager#remove()``
respectively. This comparison to find dirty entities that need
updating is using a very efficient algorithm that has almost no
additional memory overhead and can even save you computing power by
only updating those database columns that really changed.

We are now getting to the "Create a New Bug" requirement and the
code for this scenario may look like this:

.. code-block:: php

    <?php
    $reporter = $entityManager->find("User", $theReporterId);
    $engineer = $entityManager->find("User", $theDefaultEngineerId);
    
    $bug = new Bug();
    $bug->description = "Something does not work!";
    $bug->created = new DateTime("now");
    $bug->status = "NEW";
    
    foreach ($productIds AS $productId) {
        $product = $entityManager->find("Product", $productId);
        $bug->assignToProduct($product);
    }
    
    $bug->setReporter($reporter);
    $bug->setEngineer($engineer);
    
    $entityManager->persist($bug);
    $entityManager->flush();
    
    echo "Your new Bug Id: ".$bug->id."\n";

This is the first contact with the read API of the EntityManager,
showing that a call to ``EntityManager#find($name, $id)`` returns a
single instance of an entity queried by primary key. Besides this
we see the persist + flush pattern again to save the Bug into the
database.

See how simple relating Bug, Reporter, Engineer and Products is
done by using the discussed methods in the "A first prototype"
section. The UnitOfWork will detect this relations when flush is
called and relate them in the database appropriately.

Queries for Application Use-Cases
---------------------------------

List of Bugs
~~~~~~~~~~~~

Using the previous examples we can fill up the database quite a
bit, however we now need to discuss how to query the underlying
mapper for the required view representations. When opening the
application, bugs can be paginated through a list-view, which is
the first read-only use-case:

.. code-block:: php

    <?php
    $dql = "SELECT b, e, r FROM Bug b JOIN b.engineer e JOIN b.reporter r ORDER BY b.created DESC";
    
    $query = $entityManager->createQuery($dql);
    $query->setMaxResults(30);
    $bugs = $query->getResult();
    
    foreach($bugs AS $bug) {
        echo $bug->description." - ".$bug->created->format('d.m.Y')."\n";
        echo "    Reported by: ".$bug->getReporter()->name."\n";
        echo "    Assigned to: ".$bug->getEngineer()->name."\n";
        foreach($bug->getProducts() AS $product) {
            echo "    Platform: ".$product->name."\n";
        }
        echo "\n";
    }

The DQL Query in this example fetches the 30 most recent bugs with
their respective engineer and reporter in one single SQL statement.
The console output of this script is then:

::

    Something does not work! - 02.04.2010
        Reported by: beberlei
        Assigned to: beberlei
        Platform: My Product

.. note::

    **Dql is not Sql**

    You may wonder why we start writing SQL at the beginning of this
    use-case. Don't we use an ORM to get rid of all the endless
    hand-writing of SQL? Doctrine introduces DQL which is best
    described as **object-query-language** and is a dialect of
    `OQL <http://en.wikipedia.org/wiki/Object_Query_Language>`_ and
    similar to `HQL <http://www.hibernate.org>`_ or
    `JPQL <http://en.wikipedia.org/wiki/Java_Persistence_Query_Language>`_.
    It does not know the concept of columns and tables, but only those
    of Entity-Class and property. Using the Metadata we defined before
    it allows for very short distinctive and powerful queries.

    An important reason why DQL is favourable to the Query API of most
    ORMs is its similarity to SQL. The DQL language allows query
    constructs that most ORMs don't, GROUP BY even with HAVING,
    Sub-selects, Fetch-Joins of nested classes, mixed results with
    entities and scalar data such as COUNT() results and much more.
    Using DQL you should seldom come to the point where you want to
    throw your ORM into the dumpster, because it doesn't support some
    the more powerful SQL concepts.

    Besides handwriting DQL you can however also use the
    ``QueryBuilder`` retrieved by calling
    ``$entityManager->createQueryBuilder()`` which is a Query Object
    around the DQL language.

    As a last resort you can however also use Native SQL and a
    description of the result set to retrieve entities from the
    database. DQL boils down to a Native SQL statement and a
    ``ResultSetMapping`` instance itself. Using Native SQL you could
    even use stored procedures for data retrieval, or make use of
    advanced non-portable database queries like PostgreSql's recursive
    queries.


Array Hydration of the Bug List
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

In the previous use-case we retrieved the result as their
respective object instances. We are not limited to retrieving
objects only from Doctrine however. For a simple list view like the
previous one we only need read access to our entities and can
switch the hydration from objects to simple PHP arrays instead.
This can obviously yield considerable performance benefits for
read-only requests.

Implementing the same list view as before using array hydration we
can rewrite our code:

.. code-block:: php

    <?php
    $dql = "SELECT b, e, r, p FROM Bug b JOIN b.engineer e ".
           "JOIN b.reporter r JOIN b.products p ORDER BY b.created DESC";
    $query = $entityManager->createQuery($dql);
    $bugs = $query->getArrayResult();
    
    foreach ($bugs AS $bug) {
        echo $bug['description'] . " - " . $bug['created']->format('d.m.Y')."\n";
        echo "    Reported by: ".$bug['reporter']['name']."\n";
        echo "    Assigned to: ".$bug['engineer']['name']."\n";
        foreach($bug['products'] AS $product) {
            echo "    Platform: ".$product['name']."\n";
        }
        echo "\n";
    }

There is one significant difference in the DQL query however, we
have to add an additional fetch-join for the products connected to
a bug. The resulting SQL query for this single select statement is
pretty large, however still more efficient to retrieve compared to
hydrating objects.

Find by Primary Key
~~~~~~~~~~~~~~~~~~~

The next Use-Case is displaying a Bug by primary key. This could be
done using DQL as in the previous example with a where clause,
however there is a convenience method on the Entity Manager that
handles loading by primary key, which we have already seen in the
write scenarios:

.. code-block:: php

    <?php
    $bug = $entityManager->find("Bug", (int)$theBugId);

However we will soon see another problem with our entities using
this approach. Try displaying the engineer's name:

.. code-block:: php

    <?php
    echo "Bug: ".$bug->description."\n";
    echo "Engineer: ".$bug->getEngineer()->name."\n";

It will be null! What is happening? It worked in the previous
example, so it can't be a problem with the persistence code of
Doctrine. What is it then? You walked in the public property trap.

Since we only retrieved the bug by primary key both the engineer
and reporter are not immediately loaded from the database but are
replaced by LazyLoading proxies. Sample code of this proxy
generated code can be found in the specified Proxy Directory, it
looks like:

.. code-block:: php

    <?php
    namespace MyProject\Proxies;
    
    /**
     * THIS CLASS WAS GENERATED BY THE DOCTRINE ORM. DO NOT EDIT THIS FILE.
     */
    class UserProxy extends \User implements \Doctrine\ORM\Proxy\Proxy
    {
        // .. lazy load code here
    
        public function addReportedBug($bug)
        {
            $this->_load();
            return parent::addReportedBug($bug);
        }
    
        public function assignedToBug($bug)
        {
            $this->_load();
            return parent::assignedToBug($bug);
        }
    }

See how upon each method call the proxy is lazily loaded from the
database? Using public properties however we never call a method
and Doctrine has no way to hook into the PHP Engine to detect a
direct access to a public property and trigger the lazy load. We
need to rewrite our entities, make all the properties private or
protected and add getters and setters to get a working example:

.. code-block:: php

    <?php
    echo "Bug: ".$bug->getDescription()."\n";
    echo "Engineer: ".$bug->getEngineer()->getName()."\n";
    
    /**
    Bug: Something does not work!
    Engineer: beberlei
    */

Being required to use private or protected properties Doctrine 2
actually enforces you to encapsulate your objects according to
object-oriented best-practices.

Dashboard of the User
---------------------

For the next use-case we want to retrieve the dashboard view, a
list of all open bugs the user reported or was assigned to. This
will be achieved using DQL again, this time with some WHERE clauses
and usage of bound parameters:

.. code-block:: php

    <?php
    $dql = "SELECT b, e, r FROM Bug b JOIN b.engineer e JOIN b.reporter r ".
           "WHERE b.status = 'OPEN' AND e.id = ?1 OR r.id = ?1 ORDER BY b.created DESC";
    
    $myBugs = $entityManager->createQuery($dql)
                            ->setParameter(1, $theUserId)
                            ->setMaxResults(15)
                            ->getResult();
    
    foreach ($myBugs AS $bug) {
        echo $bug->getDescription()."\n";
    }

That is it for the read-scenarios of this example, we will continue
with the last missing bit, engineers being able to close a bug.

Number of Bugs
--------------

Until now we only retrieved entities or their array representation.
Doctrine also supports the retrieval of non-entities through DQL.
These values are called "scalar result values" and may even be
aggregate values using COUNT, SUM, MIN, MAX or AVG functions.

We will need this knowledge to retrieve the number of open bugs
grouped by product:

.. code-block:: php

    <?php
    $dql = "SELECT p.id, p.name, count(b.id) AS openBugs FROM Bug b ".
           "JOIN b.products p WHERE b.status = 'OPEN' GROUP BY p.id";
    $productBugs = $entityManager->createQuery($dql)->getScalarResult();
    
    foreach($productBugs as $productBug) {
        echo $productBug['name']." has " . $productBug['openBugs'] . " open bugs!\n";
    }

Updating Entities
-----------------

There is a single use-case missing from the requirements, Engineers
should be able to close a bug. This looks like:

.. code-block:: php

    <?php
    $bug = $entityManager->find("Bug", (int)$theBugId);
    $bug->close();
    
    $entityManager->flush();

When retrieving the Bug from the database it is inserted into the
IdentityMap inside the UnitOfWork of Doctrine. This means your Bug
with exactly this id can only exist once during the whole request
no matter how often you call ``EntityManager#find()``. It even
detects entities that are hydrated using DQL and are already
present in the Identity Map.

When flush is called the EntityManager loops over all the entities
in the identity map and performs a comparison between the values
originally retrieved from the database and those values the entity
currently has. If at least one of these properties is different the
entity is scheduled for an UPDATE against the database. Only the
changed columns are updated, which offers a pretty good performance
improvement compared to updating all the properties.

Entity Repositories
-------------------

For now we have not discussed how to separate the Doctrine query logic from your model.
In Doctrine 1 there was the concept of ``Doctrine_Table`` instances for this
seperation. The similar concept in Doctrine2 is called Entity Repositories, integrating
the `repository pattern <http://martinfowler.com/eaaCatalog/repository.html>`_ at the heart of Doctrine.

Every Entity uses a default repository by default and offers a bunch of convenience
methods that you can use to query for instances of that Entity. Take for example
our Product entity. If we wanted to Query by name, we can use:

.. code-block:: php

    <?php
    $product = $entityManager->getRepository('Product')
                             ->findOneBy(array('name' => $productName));

The method ``findOneBy()`` takes an array of fields or association keys and the values to match against.

If you want to find all entities matching a condition you can use ``findBy()``, for
example querying for all closed bugs:

.. code-block:: php

    <?php
    $bugs = $entityManager->getRepository('Bug')
                          ->findBy(array('status' => 'CLOSED'));

    foreach ($bugs AS $bug) {
        // do stuff
    }

Compared to DQL these query methods are falling short of functionality very fast.
Doctrine offers you a convenient way to extend the functionalities of the default ``EntityRepository``
and put all the specialized DQL query logic on it. For this you have to create a subclass
of ``Doctrine\ORM\EntityRepository``, in our case a ``BugRepository`` and group all
the previoiusly discussed query functionality in it:

.. code-block:: php

    <?php

    use Doctrine\ORM\EntityRepository;

    class BugRepository extends EntityRepository
    {
        public function getRecentBugs($number = 30)
        {
            $dql = "SELECT b, e, r FROM Bug b JOIN b.engineer e JOIN b.reporter r ORDER BY b.created DESC";

            $query = $this->getEntityManager()->createQuery($dql);
            $query->setMaxResults($number);
            return $query->getResult();
        }

        public function getRecentBugsArray($number = 30)
        {
            $dql = "SELECT b, e, r, p FROM Bug b JOIN b.engineer e ".
                   "JOIN b.reporter r JOIN b.products p ORDER BY b.created DESC";
            $query = $this->getEntityManager()->createQuery($dql);
            $query->setMaxResults($number);
            return $query->getArrayResult();
        }

        public function getUsersBugs($userId, $number = 15)
        {
            $dql = "SELECT b, e, r FROM Bug b JOIN b.engineer e JOIN b.reporter r ".
                   "WHERE b.status = 'OPEN' AND e.id = ?1 OR r.id = ?1 ORDER BY b.created DESC";

            return $this->getEntityManager()->createQuery($dql)
                                 ->setParameter(1, $userId)
                                 ->setMaxResults($number)
                                 ->getResult();
        }

        public function getOpenBugsByProduct()
        {
            $dql = "SELECT p.id, p.name, count(b.id) AS openBugs FROM Bug b ".
                   "JOIN b.products p WHERE b.status = 'OPEN' GROUP BY p.id";
            return $this->getEntityManager()->createQuery($dql)->getScalarResult();
        }
    }

To be able to use this query logic through ``$this->getEntityManager()->getRepository('Bug')``
we have to adjust the metadata slightly.

.. configuration-block::

    .. code-block:: php

        <?php
        /**
         * @Entity(repositoryClass="BugRepository")
         * @Table(name="bugs")
         */
        class Bug
        {
            //...
        }

    .. code-block:: xml

        <doctrine-mapping xmlns="http://doctrine-project.org/schemas/orm/doctrine-mapping"
              xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
              xsi:schemaLocation="http://doctrine-project.org/schemas/orm/doctrine-mapping
                            http://doctrine-project.org/schemas/orm/doctrine-mapping.xsd">

              <entity name="Bug" table="bugs" entity-repository="BugRepository">

              </entity>
        </doctrine-mapping>

    .. code-block:: yaml

        Product:
          type: entity
          repositoryClass: BugRepository

Now we can remove our query logic in all the places and instead use them through the EntityRepository.
As an example here is the code of the first use case "List of Bugs":

.. code-block:: php

    <?php
    $bugs = $entityManager->getRepository('Bug')->getRecentBugs();

    foreach($bugs AS $bug) {
        echo $bug->description." - ".$bug->created->format('d.m.Y')."\n";
        echo "    Reported by: ".$bug->getReporter()->name."\n";
        echo "    Assigned to: ".$bug->getEngineer()->name."\n";
        foreach($bug->getProducts() AS $product) {
            echo "    Platform: ".$product->name."\n";
        }
        echo "\n";
    }

Using EntityRepositories you can avoid coupling your model with specific query logic.
You can also re-use query logic easily throughout your application.

Conclusion
----------

This tutorial is over here, I hope you had fun. Additional content
will be added to this tutorial incrementally, topics will include:

-   More on Association Mappings
-   Lifecycle Events triggered in the UnitOfWork
-   Ordering of Collections

Additional details on all the topics discussed here can be found in
the respective manual chapters.


