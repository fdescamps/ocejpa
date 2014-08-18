http://examclouds.com/

# Object Relational Mapping

1. ***Allowed field access in JPA 2.0.*** Protected, package or private.

2. ***Where mapping annotation for property access in entity should be located?*** On Getter method.

3. ***How the type of property is determined in entity?*** The type of property is determined by the return type of the getter method and must be the same as the type of the single parameter passed into the setter method.

4. ***Access for get and set method for property access in entity.*** Get and set methods must have either public or protected visibility.

5. ***How to change access mode on the subclass of entity in JPA 2.0?*** @Access annotation with a specified access mode on the subclass entity will cause the default access type to be overridden for that entity subclass.

6. ***How to make persistent field or property to be accessed differently from the default access mode for that entity?***
 
      1. The first thing that must be done is to explicitly mark the default access mode for the class by annotating it with the @Access annotation and indicating the access type. Unless this is done, it will be undefined if both fields and properties are annotated. We would tag our Employee entity as having FIELD access: 
      		
			@Entity @Access(AccessType.FIELD) 
			class Employee { ... }
 
      2. The next step is to annotate the additional field or property with the @Access annotation, but this time specifying the opposite access type from what was specified at the class level. It might seem a little redundant, for example, to specify the access type of AccessType.PROPERTY on a persistent property because it is obvious by looking at it that it is a property, but doing so indicates that what you are doing is not an oversight, but a conscious exception to the default case. 
      			
			@Access(AccessType.PROPERTY) 
			@Column(name="PHONE") 
			protected String getPhoneNumberForDb() { ... } 

      3. The final thing to remember is that the corresponding field or property to the one being made persistent must be marked as transient so that the default accessing rules do not cause the same state to be persisted twice. For example, because we are adding a persistent property to an entity for which the default access type is through fields, the field in which the persistent property state is being stored in the entity must be annotated with @Transient:
       
			@Transient private String phoneNum;

7. ***How to change the default name of the table?*** 

			@Entity @Table(name="EMP") 
			public class Employee { ... }

8. ***The list of persistable types in JPA 2.0.***

       1. Primitive Java types: byte, int, short, long, boolean, char, float, double 
       2. Wrapper classes of primitive Java types: Byte, Integer, Short, Long, Boolean,Character, Float, Double 
       3. Byte and character array types: byte[], Byte[], char[], Character[] 
       4. Large numeric types: java.math.BigInteger, java.math.BigDecimal 
       5. Strings: java.lang.String 
       6. Java temporal types: java.util.Date, java.util.Calendar 
       7. JDBC temporal types: java.sql.Date, java.sql.Time, java.sql.Timestamp 
       8. Enumerated types: Any system or user-defined enumerated type 
       9. Serializable objects: Any system or user-defined serializable type

9. ***How explicitly mark on a field or property in entity to make it persistent?*** An optional @Basic annotation can be placed on a field or property to explicitly mark it as being persistent. This annotation is mostly for documentation purposes and is not required for the field or property to be persistent.

10. ***How to make column to be lazy loaded in entity?*** 
 
		@Basic(fetch=FetchType.LAZY) 
		@Column(name="COMM") 
		private String comments;

11. ***How to signal to the provider that it should use the LOB methods when passing and retrieving this data to and from the JDBC driver?*** The @Lob annotation acts as the marker annotation to fulfill this purpose.

12. ***Types for CLOBs in JPA 2.0.*** char[], Character[], and String.

13. ***Types for BLOBs in JPA 2.0.*** byte[], Byte[], and Serializable types.

14. ***How declare enumerated type in entity?*** Nothing special.

15. ***Default for enumerate types in JPA 2.0.*** Ordinal.

16. ***Values of EnumType.*** ORDINAL and STRING

17. ***How to store strings for the enumerated values?*** 
		
		@Enumerated(EnumType.STRING) 
		private EmployeeType type;

18. ***The list of supported temporal types in JPA 2.0.*** 

     1. java.sql.Date, 
     2. java.sql.Time, 
     3. java.sql.Timestamp, 
     4. java.util.Date, 
     5. java.util.Calendar.

19. ***How to use java.util.Date and java.util.Calendar in entity?*** This is done by annotating them with the @Temporal annotation and specifying the JDBC type as a value of the TemporalType enumerated type: 
	
		@Temporal(TemporalType.DATE) 
		private Calendar dob;

20. ***Write three enumerated values date annotation.*** There are three enumerated values of DATE, TIME, and TIMESTAMP to represent each of the java.sql types.

21. ***The difference between transient and @Transient.*** Attributes that are part of a persistent entity but not intended to be persistent can either be modified with the transient modifier in Java or be annotated with the @Transient annotation. If either is specified, the provider runtime will not apply its default mapping rules to the attribute on which it was specified. In cases where the non-persistent value should be retained across serialization, the annotation should be used instead of the modifier.

22. ***What is right for Primary keys in JPA 2.0: insertable, nullable or updatable?*** Primary keys are assumed to be insertable, but not nullable or updatable.

23. ***Types for id mappings in JPA 2.0.***

     1. Primitive Java types: byte, int, short, long, char; 
     2. Wrapper classes of primitive Java types: Byte, Integer, Short, Long, Character; 
     3. String: java.lang.String; 
     4. Large numeric type: java.math.BigInteger; 
     5. Temporal types: java.util.Date, java.sql.Date; 
     6. Floating point types such as float and double are permitted, as well as the Float and Double wrapper classes and java.math.BigDecimal, but they are discouraged because of the nature of rounding error and the untrustworthiness of the equals() operator when applied to them. Using floating types for primary keys is a risky endeavor and definitely not recommended.

24. ***Annotation for id generation in JPA 2.0.*** @GeneratedValue annotation.

25. ***Id generation strategies in JPA 2.0.*** AUTO, TABLE, SEQUENCE, or IDENTITY enumerated values of the GenerationType enumerated type.

26. ***How to write generation strategy?*** 
	
		@Id 
		@GeneratedValue(strategy=GenerationType.AUTO) 
		private int id;

27. ***Describe Table generation.*** An id generation table should have two columns. The first column is a string type used to identify the particular generator sequence. It is the primary key for all the generators in the table. The second column is an integer type that stores the actual id sequence that is being generated. The value stored in this column is the last identifier that was allocated in the sequence. Each defined generator represents a row in the table.

28. ***Describe the case when generation table strategy is indicated but no generator has been specified.*** Because the generation strategy is indicated but no generator has been specified, the provider will assume a table of its own choosing. If schema generation is used, it will be created; if not, the default table assumed by the provider must be known and must exist in the database.

29. ***How to specify the table that is to be used for id storage?*** This is done by defining a table generator that, contrary to what its name implies, does not actually generate tables. Rather, it is an identifier generator that uses a table to store them. We can define one by using a @TableGenerator annotation and then refer to it by name in the @GeneratedValue annotation: 

		@TableGenerator(name="Emp_Gen") 
		@Id 
        @GeneratedValue(generator="Emp_Gen") 
		private int id;

30. ***Where @TableGenerator can be defined in JPA 2.0?*** It can be defined on any attribute or class. Regardless of where it is defined, it will be available to the entire persistence unit. A good practice would be to define it locally on the id attribute if only one class is using it but to define it in XML.

31. ***How to specify @TableGenerator details in JPA 2.0?*** 
	
		@TableGenerator(
			name="Address_Gen", 
			table="ID_GEN", 
			pkColumnName="GEN_NAME", 
			valueColumnName="GEN_VAL", 
			pkColumnValue="Addr_Gen", 
			initialValue=10000, 
			allocationSize=100) 
		@Id 
		@GeneratedValue(generator="Address_Gen") 
        private int id;

32. ***Define Sequence generator.*** 
	
		@SequenceGenerator(name="Emp_Gen", sequenceName="Emp_Seq") 
		@Id 
		@GeneratedValue(generator="Emp_Gen") 
		private int getId;

33. ***Define identity generator.***
	
		@Id 
		@GeneratedValue(strategy=GenerationType.IDENTITY) 
		private int id;

34. ***Relationship concepts in JPA 2.0.*** 
    
   1. Directionality: bidirectional, unidirectional; 
   2. Cardinality:one, many; 
   3. Ordinality

35. ***Single-valued association.*** The many-to-one and one-to-one relationship.

36. ***Foreign key column in JPA.*** @JoinColumn(name='DEPT_ID')

37. ***Which mappings are always on the owning side of a relationship in JPA 2.0?*** Many-to-one mappings are always on the owning side of a relationship.

38. ***Default column name for many to one mapping in JPA 2.0.*** The name that is used as the default is formed from a combination of both the source and target entities. It is the name of the relationship attribute in the source entity, which is department in our example, plus an underscore character (\_), plus the name of the primary key column of the target entity. So if the Department entity were mapped to a table that had a primary key column named ID, the join column in the EMPLOYEE table would be assumed to be named DEPARTMENT\_ID.

39. ***Default column name for one to one mapping in JPA 2.0.*** The same as for many-to-one.

40. ***The value of mappedBy in JPA 2.0.*** The value of mappedBy is the name of the attribute in the owning entity that points back to the inverse entity.

		@Entity
		public class Department {
		
		    @Id @GeneratedValue(strategy=GenerationType.IDENTITY)
			private int id;
		
		    @OneToMany(mappedBy="department")
		    private Collection<Professor> employees;

			//...
		}

		@Entity
		public class Professor {

		    @Id @GeneratedValue(strategy=GenerationType.IDENTITY)
		    private int id;
		    
		    @ManyToOne
		    private Department department;

			//...
		}

41. ***Collection-Valued Associations in JPA 2.0.*** One-to-Many, many-to-Many.

42. ***Write example with using a Collection that is not type-parameterized.*** 

		@OneToMany(targetEntity=Employee.class, mappedBy="department") 
		private Collection employees;

43. ***Write Employee entity for for a many-to-many relationship*** 

		Employee( pk : id ; name ; salary)
		EMP_PROJ( Pk, fk1 :Emp_id ; Pk,fk2 : Proj_id)
	    Project(  pk : id ; name )
	
		@Entity
		public class Employee {
		
		    @Id @GeneratedValue(strategy=GenerationType.IDENTITY)
			private int id;

			private String name;
			private Double salary;
		
		    @ManyToMany
			@JoinTable(
				name="EMP_PROJ", 
				joinColumns=@JoinColumn(name="EMP_ID"), 		
				inverseJoinColumns=@JoinColumn(name="PROJ_ID")) 
		    private Collection<Project> projects;

			//...
		}

		@Entity
		public class Project {
		    @Id @GeneratedValue(strategy=GenerationType.IDENTITY)
		    private int id;
		    private String name;
		
		    @ManyToMany(mappedBy="projects")
		    private Collection<Employee> employees;
			//...
		}

44. ***Default values for many to many in JPA 2.0.*** When no @JoinTable annotation is present on the owning side, then a default join table named Owner_Inverse is assumed, where Owner is the name of the owning entity, and Inverse is the name of the inverse or non-owning entity. The default name of the join column that points to the owning entity is the name of the attribute on the inverse entity that points to the owning entity, appended by an underscore and the name of the primary key column of the owning entity table. So in our example, the Employee is the owning entity, and the Project has an employees attribute that contains the collection of Employee instances. The Employee entity maps to the EMPLOYEE table and has a primary key column of ID, so the defaulted name of the join column to the owning entity would be ****EMPLOYEES\_ID****. The inverse join column would be likewise defaulted to be ****PROJECTS\_ID****.

45. ***Write code and tables for unidirectional one-to-many mapping (Employee,Phone). Use joinTable.***

		Employee (pk : id ; name ; salary) 
		EMP_PHONE ( Pk, fk1 : Phone_id ; Pk,fk2 : Emp_id) 
		Phone ( pk : id ; type ; num)

		@Entity
		public class Employee {
		
		    @Id @GeneratedValue(strategy=GenerationType.IDENTITY)
			private int id;

			private String name;
			private Double salary;
		
		    @ManyToOne
			@JoinTable(
				name="EMP_PHONE", 
				joinColumns=@JoinColumn(name="EMP_ID"), 		
				inverseJoinColumns=@JoinColumn(name="PHONE_ID")) 
		    private Collection<Phone> phones;

			//...
		}

		@Entity
		public class Phone {
		
		    @Id @GeneratedValue(strategy=GenerationType.IDENTITY)
			private int id;

			private PhoneType type;
			private Integer number;
		
		    @ManyToOne(mappedBy="phones")
		    private Collection<Employee> employee;

			//...
		}

46. ***Default values for unidirectional one-to-many mapping in JPA 2.0.*** The name of the join table would default to ****EMPLOYEE\_PHONE**** and would have a join column named ****EMPLOYEE\_ID **** after the name of the Employee entity and its primary key column. The inverse join column would be named ****PHONES\_ID****, which is the concatenation of the phones attribute in the Employee entity and the ID primary key column of the PHONE table.

		EMPLOYEE (pk : ID ; name ; salary) 
		EMPLOYEE_PHONE ( Pk, fk1 : EMPLOYEE\_ID ; Pk,fk2 : PHONES\_ID) 
		PHONE ( pk : ID ; type ; num)

47. ***Default fetch mode for single-valued relationship in JPA 2.0.*** Eager.

48. ***Default fetch mode for multi-valued relationship in JPA 2.0.*** Lazy.

49. ***In bidirectional relationship cases, the fetch mode might be lazy on one side but eager on the other. Is it true?*** Yes.

50. ***Make this one to one lazy loaded: private ParkingSpace parkingSpace;***

		@OneToOne(fetch=FetchType.LAZY) 
		private ParkingSpace parkingSpace;

51. ***What is embedded object in JPA 2.0?*** An embedded object is one that is dependent on an entity for its identity.

52. ***How embedded object is stored in the db?*** In the database, however, the state of the embedded object is stored with the rest of the entity state in the database row, with no distinction between the state in the Java entity and that in its embedded object.

53. ***Can embedded object be shared among several entities in JPA 2.0?*** Even though embedded types can be shared or reused, the instances cannot. An embedded object instance belongs to the entity that references it; and no other entity instance, of that entity type or any other, can reference the same embedded instance.

54. ***How embedded object is persisted?*** Once a class has been designated as embeddable, then its fields and properties will be persistable as part of an entity.

55. ***Can access type be defined for embedded object?*** We might also want to define the access type of the embeddable object so it is accessed the same way regardless of which entity it is embedded in

56. ***How to define Access type for embedded object in JPA 2.0?*** 

		@Embeddable 
		@Access(AccessType.FIELD) 
		public class Address { 
			private String street; 
			private String city; 
			private String state; 
			
			@Column(name="ZIP_CODE") 
			private String zip; 

			// ... 
		}

57. ***Is it correct assuming that address is embeddable?***
 
		@Entity 
		public class Employee { 
			
			@Id private int id; 
			private String name; 
			private long salary; 
		
			private Address address; 

			// ... 
		}

		Yes.

58. ***How to override attributes in embedded class in JPA 2.0?***

		@Entity 
		public class Employee { 
	
			@Id 
			private int id; 
	
			private String name; 
			private long salary;
	 
			@Embedded
			@AttributeOverrides({ 
				@AttributeOverride(
					name="state", column=@Column(name="PROVINCE")), 
				@AttributeOverride(
					name="zip", column=@Column(name="POSTAL_CODE")) }) 
			private Address address; 
	
			// ... 
		 
		}

		@Entity 
		public class Company { 

			@Id 
			private String name; 

			@Embedded 
			private Address address; 

			// ... 
		}


59. ***Can methods or persistent instance variables of the entity class be final in JPA 2.0?***
No.

60. ***Can entity class be final in JPA 2.0?***
No.

61. ***Where primary key should be defined in JPA 2.0?*** The primary key must be defined on the entity class that is the root of the entity hierarchy or on a mapped superclass that is a (direct or indirect) superclass of all entity classes in the entity hierarchy.

62. ***How many times primary key can be defined in JPA 2.0?*** The primary key must be defined exactly once in an entity hierarchy.
	
63. ***Describe SEQUENCE generation.***Indicates that the persistence provider must assign primary keys for the entity using a database sequence.

Databases like Oracle DB is known for having custom defined sequence generator, which generates a running number that could be used in any query not restricting having just a row ID for primary key field. In certain aspect, it has more flexibility and it provides more control for applications. Taking an example of a table creation script in Oracle DB shown below

		CREATE TABLE APP_USERS
		(
		    APP_USERS_PK NUMBER(10) NOT NULL,
		    USERNAME VARCHAR2(255) NOT NULL,
		    PASSWORD VARCHAR2(255) NOT NULL
		);
		 
		ALTER TABLE APP_USERS ADD CONSTRAINT APP_USERS_C1 PRIMARY KEY(APP_USERS_PK);
		ALTER TABLE APP_USERS ADD CONSTRAINT APP_USERS_C2 UNIQUE(USERNAME);
		 
		CREATE SEQUENCE APP_USERS_SEQ START WITH 1 INCREMENT BY 1 NOCACHE NOCYCLE;

Looking at the SQL script, the APP\_USERS_\SEQ is the sequence generator to generate numerical value and it will be used for the primary key field APP\_USERS\_PK. In order to call the APP\_USERS\_SEQ to generate the sequential number for the primary key field, you should use have the strategy attribute in the @GeneratedValue annotation define as GenerationType.SEQUENCE and do mark the @Id field with @SequenceGenerator annotation shown below as well:

		@Entity
		@Table( name = "APP_USERS", catalog = "", schema = "SampleDatabaseSchema" )
		public class AppUsersEntity implements Serializable
		{
		    @Id
		    @SequenceGenerator( 
				name = "appUsersSeq", 
				sequenceName = "APP_USERS_SEQ", 
				allocationSize = 1, 
				initialValue = 1 )
		    @GeneratedValue( strategy = GenerationType.SEQUENCE, generator = "appUsersSeq" )
		    @Column( name = "APP_USERS_PK" )
		    private Long appUsersPk;
		 
		    //The rest of the codes...
		}

Looking in the @SequenceGenerator at the above sample code, the “name” attribute is a given name by you, it can be anything as long as it is unique through out the whole application. The “sequenceName” attribute should be filled with the name of the sequence generator when you execute the “CREATE SEQUENCE APP_USERS_SEQ …” DB statement, in this case, it is “APP_USERS_SEQ“. Then at the @GeneratedValue annotation, have the strategy attribute as GenerationType.SEQUENCE and the generator attribute value as the name of the @SequenceGenerator (NOT THE SEQUENCE GENERATOR IN Oracle DB), which is “appUsersSeq“.
	
24. ***Describe IDENTITY generation.*** Indicates that the persistence provider must assign primary keys for the entity using a database identity column. 

		CREATE TABLE APP_USERS
		(
		    APP_USERS_PK BIGINT NOT NULL AUTO_INCREMENT,
		    USERNAME VARCHAR(255) NOT NULL,
		    PASSWORD VARCHAR(255) NOT NULL,
		    PRIMARY KEY(APP_USERS_PK),
		    UNIQUE(USERNAME)
		);

We have a table which stores User login info with the APP\_USERS\_PK field as the primary key field and it is marked as AUTO_INCREMENT, which tells the database to insert a sequentially generated number when a record is inserted.

To map this in the Entity class, just define the strategy attribute in the @GeneratedValue annotation as GenerationType.IDENTITY as shown in the below

		@Entity
		@Table( name = "APP_USERS", catalog = "SampleDBName", schema = "" )
		public class AppUsersEntity implements Serializable
		{
		    @Id
		    @GeneratedValue( strategy = GenerationType.IDENTITY )
		    @Column( name = "APP_USERS_PK" )
		    private Long appUsersPk;
		 
		    //The rest of the codes...
		}

This will enable the Entity to leverage on the AUTO\_INCREMENT feature in automatically generating a sequential number as primary key when inserted into the database.


63. ***Describe AUTO generation.*** Specifying a strategy of AUTO allows persistence provider to select the strategy to use. Typically, the persistence provider picks TABLE as the strategy, since it is the most portable strategy. However, when AUTO is specified, schema generation must be used at least once in order for the default table to be created in the database.

The following example demonstrates the use of the AUTO strategy:

		@Entity
		public class Inventory implements Serializable {
		 
		        @Id
		        @GeneratedValue(strategy=GenerationType.AUTO)
		        private long id;


65. ***Describe TABLE generation.***

If you want to build an Enterprise Web Application that is portable and highly adaptable and deployable to various databases, the best way is to have a separate table which stores the sequence name with column for running numbers and incrementally update it whenever the number is used through the application. This approach is free from depending on the databases’ identity or sequence generator facilities, which gives you much freedom especially if you are writing apps that could be adaptive to a myriad of databases. I’ll just use a simple database table creation script for MySQL to illustrate this: 

		CREATE TABLE APP_USERS
		(
		    APP_USERS_PK BIGINT NOT NULL,
		    USERNAME VARCHAR(255) NOT NULL,
		    PASSWORD VARCHAR(255) NOT NULL,
		    PRIMARY KEY(APP_USERS_PK),
		    UNIQUE(USERNAME)
		);
		 
		CREATE TABLE APP_SEQ_STORE
		(
			APP_SEQ_NAME VARCHAR(255) NOT NULL,
			APP_SEQ_VALUE BIGINT NOT NULL,
			PRIMARY KEY(APP_SEQ_NAME)
		);
		 
		INSERT INTO APP_SEQ_STORE VALUES ('APP_USERS.APP_USERS.PK', 0);

Over here, we are consistently using the same table structure as the first example, with an additional table APP\_SEQ\_STORE to store the sequential numbers for the primary key. The APP\_SEQ\_STORE is more like a name-value pair table and we’ll have “APP\_USERS.APP\_USERS.PK” as the name which refers to the sequential number for this single entity.

** DO TAKE SPECIAL NOTE that I have initialized the APP_SEQ_STORE with an insert statement with the initial APP\_SEQ\_VALUE as 0. The initialization is required when running on GenerationType.TABLE mode!

As for the Java Entity codes, here it is:

	@Entity
	@Table( name = "APP_USERS", catalog = "SampleDBName", schema = "" )
	public class AppUsersEntity implements Serializable
	{
	    @Id
	    @Column( name = "APP_USERS_PK" )
	    @TableGenerator( 
			name = "appSeqStore", 
			table = "APP_SEQ_STORE", 
			pkColumnName = "APP_SEQ_NAME", 
			pkColumnValue = "APP_USERS.APP_USERS_PK", 
			valueColumnName = "APP_SEQ_VALUE", 
			initialValue = 1, 
			allocationSize = 1 )
	    @GeneratedValue( strategy = GenerationType.TABLE, generator = "appSeqStore" )
	    private Long appUsersPk;
	 
	    //The rest of the codes...
	}

Firstly, we need to have the @TableGenerator to be properly defined. Just give a unique name for the @TableGenerator through name attribute. Let’s name it as “appSeqStore” for now. The table attribute refers to the table which stores all sequence name and sequential number, which is “APP\_SEQ\_STORE“. The pkColumnName attribute should have the value referring to the table column of the sequence name, which is “APP\_SEQ\_NAME“. As for pkColumnValue, it is the unique sequence name that you’ve assigned during the insert statement earlier, which is “APP\_USERS.APP\_USERS\_PK” and lastly, the valueColumnName is the table column which holds the running number, which is “APP\_SEQ\_VALUE“. Do define the initialValue and allocationSize according to your application needs.

After that, just declare the @GeneratedValue annotation with strategy attribute as “GenerationType.TABLE” and the generator attribute value as what you’ve named the @TableGenerator, which is “appSeqStore“.

There you go, a working table-stored sequence generator for the primary key fields, without having to write a load-some of persisting codes for the APP_SEQ_STORE table.