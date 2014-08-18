1. Overview of the Java Persistence API

    * Describe the basics of Object Relational Mapping (ORM)
    * Define the key concepts of the Java Persistence API (entity, entity manager, and persistence unit)

2. Introducing the Auction Application

    * Describe the auction application
    * Define the domain objects of the auction application
    * Describe the implementation model for the auction system

3. Java Persistence API Entities

    * Describe the difference between objects and entities
    * Describe the difference between persistent fields and properties
    * Identify and use common Java Persistence API annotations, such as @Entity, @Id, @Table, and @Column

4. Understanding the Entity Manager

    * Describe the relationship between an entity manager, a persistence context, and a persistence unit
    * Describe the difference between a container-managed entity manager and an application-managed entity manager
    * Describe the entity life cycle

5. Modeling Entity Relationships

    * Examine association relationships in the data and object models
    * Use relationship properties to define associations
    * Implement one-to-one unidirectional associations
    * Implement one-to-one bidirectional associations
    * Implement many-to-one/one-to-many bidirectional associations
    * Implement many-to-many bidirectional associations
    * Implement many-to-many unidirectional associations
    * Examine fetch and cascade mode settings

6. Entity Inheritance and Object-Relational Mapping

    * Examine entity inheritance
    * Examining object/relational inheritance hierarchy mapping strategies
    * Inherit from an entity class
    * Inherit using a mapped superclass
    * Inherit from a non-entity class
    * Examine inheritance mapping strategies
    * Use an embeddable class

7. Persisting Enums and Collections

    * Persist entities that contain enums with @Enumerated
    * Persist entities that contain lists with @ElementCollection
    * Persist entities that contain maps with @ElementCollection

8. Introduction to Querying

    * Find an Entity by its primary key
    * Understand basic Java Persistence API query language queries
    * Understand native SQL queries
    * Understand basic Criteria API queries

9. Using the Java Persistence API Query Language

    * Examine the Java Persistence API query language
    * Create and use the SELECT statement
    * Create and use the UPDATE statement
    * Create and use the DELETE statement

10. Using the Java Persistence API Criteria API

    * Contrast queries that use the Criteria API with queries that use the Java Persistence query language
    * Describe the metamodel object approach to querying
    * Create Criteria API queries

11. Using the Java Persistence API in a Container

    * Use the Java Persistence API from a servlet
    * Use the Java Persistence API from a stateless session bean

12. Implementing Transactions and Locking

    * Describe the transaction demarcation management
    * Implement container-managed transactions (CMT)
    * Interact programmatically with an ongoing CMT transaction
    * Implement bean-managed transactions (BMT)
    * Apply transactions to the Java Persistence API

13. Advanced Java Persistence API Concepts

    * Specify composite primary keys
    * Override mappings with the @AttributeOverride and @AssociationOverride annotations
    * Understand entity listeners and callback methods

