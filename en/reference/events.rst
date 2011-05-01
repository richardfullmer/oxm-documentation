Events
======

Doctrine 2 features a lightweight event system that is part of the
Common package.

The Event System
----------------

The event system is controlled by the ``EventManager``. It is the
central point of Doctrine's event listener system. Listeners are
registered on the manager and events are dispatched through the
manager.

.. code-block:: php

    <?php
    $evm = new EventManager();

Now we can add some event listeners to the ``$evm``. Let's create a
``EventTest`` class to play around with.

.. code-block:: php

    <?php
    class EventTest
    {
        const preFoo = 'preFoo';
        const postFoo = 'postFoo';
    
        private $_evm;
    
        public $preFooInvoked = false;
        public $postFooInvoked = false;
    
        public function __construct($evm)
        {
            $evm->addEventListener(array(self::preFoo, self::postFoo), $this);
        }
    
        public function preFoo(EventArgs $e)
        {
            $this->preFooInvoked = true;
        }
    
        public function postFoo(EventArgs $e)
        {
            $this->postFooInvoked = true;
        }
    }
    
    // Create a new instance
    $test = new EventTest($evm);

Events can be dispatched by using the ``dispatchEvent()`` method.

.. code-block:: php

    <?php
    $evm->dispatchEvent(EventTest::preFoo);
    $evm->dispatchEvent(EventTest::postFoo);

You can easily remove a listener with the ``removeEventListener()``
method.

.. code-block:: php

    <?php
    $evm->removeEventListener(array(self::preFoo, self::postFoo), $this);

The Doctrine 2 event system also has a simple concept of event
subscribers. We can define a simple ``TestEventSubscriber`` class
which implements the ``\Doctrine\Common\EventSubscriber`` interface
and implements a ``getSubscribedEvents()`` method which returns an
array of events it should be subscribed to.

.. code-block:: php

    <?php
    class TestEventSubscriber implements \Doctrine\Common\EventSubscriber
    {
        public $preFooInvoked = false;
    
        public function preFoo()
        {
            $this->preFooInvoked = true;
        }
    
        public function getSubscribedEvents()
        {
            return array(TestEvent::preFoo);
        }
    }
    
    $eventSubscriber = new TestEventSubscriber();
    $evm->addEventSubscriber($eventSubscriber);

Now when you dispatch an event any event subscribers will be
notified for that event.

.. code-block:: php

    <?php
    $evm->dispatchEvent(TestEvent::preFoo);

Now you can test the ``$eventSubscriber`` instance to see if the
``preFoo()`` method was invoked.

.. code-block:: php

    <?php
    if ($eventSubscriber->preFooInvoked) {
        echo 'pre foo invoked!';
    }

Naming convention
~~~~~~~~~~~~~~~~~

Events being used with the Doctrine 2 EventManager are best named
with camelcase and the value of the corresponding constant should
be the name of the constant itself, even with spelling. This has
several reasons:


-  It is easy to read.
-  Simplicity.
-  Each method within an EventSubscriber is named after the
   corresponding constant. If constant name and constant value differ,
   you MUST use the new value and thus, your code might be subject to
   codechanges when the value changes. This contradicts the intention
   of a constant.

An example for a correct notation can be found in the example
``EventTest`` above.

Lifecycle Events
----------------

The EntityManager and UnitOfWork trigger a bunch of events during
the life-time of their registered entities.


-  preRemove - The preRemove event occurs for a given entity before
   the respective EntityManager remove operation for that entity is
   executed.  It is not called for a DQL DELETE statement.
-  postRemove - The postRemove event occurs for an entity after the
   entity has been deleted. It will be invoked after the database
   delete operations. It is not called for a DQL DELETE statement.
-  prePersist - The prePersist event occurs for a given entity
   before the respective EntityManager persist operation for that
   entity is executed.
-  postPersist - The postPersist event occurs for an entity after
   the entity has been made persistent. It will be invoked after the
   database insert operations. Generated primary key values are
   available in the postPersist event.
-  preUpdate - The preUpdate event occurs before the database
   update operations to entity data. It is not called for a DQL UPDATE statement.
-  postUpdate - The postUpdate event occurs after the database
   update operations to entity data. It is not called for a DQL UPDATE statement.
-  postLoad - The postLoad event occurs for an entity after the
   entity has been loaded into the current EntityManager from the
   database or after the refresh operation has been applied to it.
-  loadClassMetadata - The loadClassMetadata event occurs after the
   mapping metadata for a class has been loaded from a mapping source
   (annotations/xml/yaml).
-  onFlush - The onFlush event occurs after the change-sets of all
   managed entities are computed. This event is not a lifecycle
   callback.

.. warning::

    Note that the postLoad event occurs for an entity
    before any associations have been initialized. Therefore it is not
    safe to access associations in a postLoad callback or event
    handler.


You can access the Event constants from the ``Events`` class in the
ORM package.

.. code-block:: php

    <?php
    use Doctrine\ORM\Events;
    echo Events::preUpdate;

These can be hooked into by two different types of event
listeners:


-  Lifecycle Callbacks are methods on the entity classes that are
   called when the event is triggered. They receive absolutely no
   arguments and are specifically designed to allow changes inside the
   entity classes state.
-  Lifecycle Event Listeners are classes with specific callback
   methods that receives some kind of ``EventArgs`` instance which
   give access to the entity, EntityManager or other relevant data.

.. note::

    All Lifecycle events that happen during the ``flush()`` of
    an EntityManager have very specific constraints on the allowed
    operations that can be executed. Please read the
    :ref:`reference-events-implementing-listeners` section very carefully
    to understand which operations are allowed in which lifecycle event.


Lifecycle Callbacks
-------------------

A lifecycle event is a regular event with the additional feature of
providing a mechanism to register direct callbacks inside the
corresponding entity classes that are executed when the lifecycle
event occurs.

.. code-block:: php

    <?php
    
    /** @Entity @HasLifecycleCallbacks */
    class User
    {
        // ...
    
        /**
         * @Column(type="string", length=255)
         */
        public $value;
    
        /** @Column(name="created_at", type="string", length=255) */
        private $createdAt;
    
        /** @PrePersist */
        public function doStuffOnPrePersist()
        {
            $this->createdAt = date('Y-m-d H:m:s');
        }
    
        /** @PrePersist */
        public function doOtherStuffOnPrePersist()
        {
            $this->value = 'changed from prePersist callback!';
        }
    
        /** @PostPersist */
        public function doStuffOnPostPersist()
        {
            $this->value = 'changed from postPersist callback!';
        }
    
        /** @PostLoad */
        public function doStuffOnPostLoad()
        {
            $this->value = 'changed from postLoad callback!';
        }
    
        /** @PreUpdate */
        public function doStuffOnPreUpdate()
        {
            $this->value = 'changed from preUpdate callback!';
        }
    }

Note that when using annotations you have to apply the
@HasLifecycleCallbacks marker annotation on the entity class.

If you want to register lifecycle callbacks from YAML or XML you
can do it with the following.

.. code-block:: yaml

    User:
      type: entity
      fields:
    # ...
        name:
          type: string(50)
      lifecycleCallbacks:
        prePersist: [ doStuffOnPrePersist, doOtherStuffOnPrePersistToo ]
        postPersist: [ doStuffOnPostPersist ] 

XML would look something like this:

.. code-block:: xml

    <?xml version="1.0" encoding="UTF-8"?>
    
    <doctrine-mapping xmlns="http://doctrine-project.org/schemas/orm/doctrine-mapping"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://doctrine-project.org/schemas/orm/doctrine-mapping
                              /Users/robo/dev/php/Doctrine/doctrine-mapping.xsd">
    
        <entity name="User">
    
            <lifecycle-callbacks>
                <lifecycle-callback type="prePersist" method="doStuffOnPrePersist"/>
                <lifecycle-callback type="postPersist" method="doStuffOnPostPersist"/>
            </lifecycle-callbacks>
    
        </entity>
    
    </doctrine-mapping>

You just need to make sure a public ``doStuffOnPrePersist()`` and
``doStuffOnPostPersist()`` method is defined on your ``User``
model.

.. code-block:: php

    <?php
    // ...
    
    class User
    {
        // ...
    
        public function doStuffOnPrePersist()
        {
            // ...
        }
    
        public function doStuffOnPostPersist()
        {
            // ...
        }
    }

The ``key`` of the lifecycleCallbacks is the name of the method and
the value is the event type. The allowed event types are the ones
listed in the previous Lifecycle Events section.

Listening to Lifecycle Events
-----------------------------

Lifecycle event listeners are much more powerful than the simple
lifecycle callbacks that are defined on the entity classes. They
allow to implement re-usable behaviors between different entity
classes, yet require much more detailed knowledge about the inner
workings of the EntityManager and UnitOfWork. Please read the
*Implementing Event Listeners* section carefully if you are trying
to write your own listener.

To register an event listener you have to hook it into the
EventManager that is passed to the EntityManager factory:

.. code-block:: php

    <?php
    $eventManager = new EventManager();
    $eventManager->addEventListener(array(Events::preUpdate), MyEventListener());
    $eventManager->addEventSubscriber(new MyEventSubscriber());
    
    $entityManager = EntityManager::create($dbOpts, $config, $eventManager);

You can also retrieve the event manager instance after the
EntityManager was created:

.. code-block:: php

    <?php
    $entityManager->getEventManager()->addEventListener(array(Events::preUpdate), MyEventListener());
    $entityManager->getEventManager()->addEventSubscriber(new MyEventSubscriber());

.. _reference-events-implementing-listeners:

Implementing Event Listeners
----------------------------

This section explains what is and what is not allowed during
specific lifecycle events of the UnitOfWork. Although you get
passed the EntityManager in all of these events, you have to follow
this restrictions very carefully since operations in the wrong
event may produce lots of different errors, such as inconsistent
data and lost updates/persists/removes.

For the described events that are also lifecycle callback events
the restrictions apply as well, with the additional restriction
that you do not have access to the EntityManager or UnitOfWork APIs
inside these events.

prePersist
~~~~~~~~~~

There are two ways for the ``prePersist`` event to be triggered.
One is obviously when you call ``EntityManager#persist()``. The
event is also called for all cascaded associations.

There is another way for ``prePersist`` to be called, inside the
``flush()`` method when changes to associations are computed and
this association is marked as cascade persist. Any new entity found
during this operation is also persisted and ``prePersist`` called
on it. This is called "persistence by reachability".

In both cases you get passed a ``LifecycleEventArgs`` instance
which has access to the entity and the entity manager.

The following restrictions apply to ``prePersist``:


-  If you are using a PrePersist Identity Generator such as
   sequences the ID value will *NOT* be available within any
   PrePersist events.
-  Doctrine will not recognize changes made to relations in a pre
   persist event called by "reachability" through a cascade persist
   unless you use the internal ``UnitOfWork`` API. We do not recommend
   such operations in the persistence by reachability context, so do
   this at your own risk and possibly supported by unit-tests.

preRemove
~~~~~~~~~

The ``preRemove`` event is called on every entity when its passed
to the ``EntityManager#remove()`` method. It is cascaded for all
associations that are marked as cascade delete.

There are no restrictions to what methods can be called inside the
``preRemove`` event, except when the remove method itself was
called during a flush operation.

onFlush
~~~~~~~

OnFlush is a very powerful event. It is called inside
``EntityManager#flush()`` after the changes to all the managed
entities and their associations have been computed. This means, the
``onFlush`` event has access to the sets of:


-  Entities scheduled for insert
-  Entities scheduled for update
-  Entities scheduled for removal
-  Collections scheduled for update
-  Collections scheduled for removal

To make use of the onFlush event you have to be familiar with the
internal UnitOfWork API, which grants you access to the previously
mentioned sets. See this example:

.. code-block:: php

    <?php
    class FlushExampleListener
    {
        public function onFlush(OnFlushEventArgs $eventArgs)
        {
            $em = $eventArgs->getEntityManager();
            $uow = $em->getUnitOfWork();
    
            foreach ($uow->getScheduledEntityInsertions() AS $entity) {
    
            }
    
            foreach ($uow->getScheduledEntityUpdates() AS $entity) {
    
            }
    
            foreach ($uow->getScheduledEntityDeletions() AS $entity) {
    
            }
    
            foreach ($uow->getScheduledCollectionDeletions() AS $col) {
    
            }
    
            foreach ($uow->getScheduledCollectionUpdates() AS $col) {
    
            }
        }
    }

The following restrictions apply to the onFlush event:


-  Calling ``EntityManager#persist()`` does not suffice to trigger
   a persist on an entity. You have to execute an additional call to
   ``$unitOfWork->computeChangeSet($classMetadata, $entity)``.
-  Changing primitive fields or associations requires you to
   explicitly trigger a re-computation of the changeset of the
   affected entity. This can be done by either calling
   ``$unitOfWork->computeChangeSet($classMetadata, $entity)`` or
   ``$unitOfWork->recomputeSingleEntityChangeSet($classMetadata, $entity)``.
   The second method has lower overhead, but only re-computes
   primitive fields, never associations.

preUpdate
~~~~~~~~~

PreUpdate is the most restrictive to use event, since it is called
right before an update statement is called for an entity inside the
``EntityManager#flush()`` method.

Changes to associations of the updated entity are never allowed in
this event, since Doctrine cannot guarantee to correctly handle
referential integrity at this point of the flush operation. This
event has a powerful feature however, it is executed with a
``PreUpdateEventArgs`` instance, which contains a reference to the
computed change-set of this entity.

This means you have access to all the fields that have changed for
this entity with their old and new value. The following methods are
available on the ``PreUpdateEventArgs``:


-  ``getEntity()`` to get access to the actual entity.
-  ``getEntityChangeSet()`` to get a copy of the changeset array.
   Changes to this returned array do not affect updating.
-  ``hasChangedField($fieldName)`` to check if the given field name
   of the current entity changed.
-  ``getOldValue($fieldName)`` and ``getNewValue($fieldName)`` to
   access the values of a field.
-  ``setNewValue($fieldName, $value)`` to change the value of a
   field to be updated.

A simple example for this event looks like:

.. code-block:: php

    <?php
    class NeverAliceOnlyBobListener
    {
        public function preUpdate(PreUpdateEventArgs $eventArgs)
        {
            if ($eventArgs->getEntity() instanceof User) {
                if ($eventArgs->hasChangedField('name') && $eventArgs->getNewValue('name') == 'Alice') {
                    $eventArgs->setNewValue('name', 'Bob');
                }
            }
        }
    }

You could also use this listener to implement validation of all the
fields that have changed. This is more efficient than using a
lifecycle callback when there are expensive validations to call:

.. code-block:: php

    <?php
    class ValidCreditCardListener
    {
        public function preUpdate(PreUpdateEventArgs $eventArgs)
        {
            if ($eventArgs->getEntity() instanceof Account) {
                if ($eventArgs->hasChangedField('creditCard')) {
                    $this->validateCreditCard($eventArgs->getNewValue('creditCard'));
                }
            }
        }
    
        private function validateCreditCard($no)
        {
            // throw an exception to interrupt flush event. Transaction will be rolled back.
        }
    }

Restrictions for this event:


-  Changes to associations of the passed entities are not
   recognized by the flush operation anymore.
-  Changes to fields of the passed entities are not recognized by
   the flush operation anymore, use the computed change-set passed to
   the event to modify primitive field values.
-  Any calls to ``EntityManager#persist()`` or
   ``EntityManager#remove()``, even in combination with the UnitOfWork
   API are strongly discouraged and don't work as expected outside the
   flush operation.

postUpdate, postRemove, postPersist
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The three post events are called inside ``EntityManager#flush()``.
Changes in here are not relevant to the persistence in the
database, but you can use this events to

postLoad
~~~~~~~~

This event is called after an entity is constructed by the
EntityManager.

Load ClassMetadata Event
------------------------

When the mapping information for an entity is read, it is populated
in to a ``ClassMetadataInfo`` instance. You can hook in to this
process and manipulate the instance.

.. code-block:: php

    <?php
    $test = new EventTest();
    $metadataFactory = $em->getMetadataFactory();
    $evm = $em->getEventManager();
    $evm->addEventListener(Events::loadClassMetadata, $test);
    
    class EventTest
    {
        public function loadClassMetadata(\Doctrine\ORM\Event\LoadClassMetadataEventArgs $eventArgs)
        {
            $classMetadata = $eventArgs->getClassMetadata();
            $fieldMapping = array(
                'fieldName' => 'about',
                'type' => 'string',
                'length' => 255
            );
            $classMetadata->mapField($fieldMapping);
        }
    }


