# Chapter 4 : Object relational mapping

## Persistence Annotations

## Accessing Entity State

### Field Access

Annotating the fields of the entity will cause the provider to use field access to get and set the state of the entity.
Getter and Setter methods might or might not be present, but if they are present, they are ignored by the provider.
All fields must be declared as either *protected, package or private*. Public fields are disallowed because it would open up
the state fields to access by any unprotected class in the JVM => Bad practice + defeat the provider implementation.

@Entity
public class Employee{
	@Id private long id;
	private String name;
	private long salary;

	public long getId(){return id;}
	public void setId(long id){this.id=id;}

	public String getName(){return name;}
	public void setName(String name){this.name=name;}

	public long getSalary(){return salary;}
	public void setSalary(long salary){this.salary=salary;}
}

### Property Access

When property access mode is used, the same contract as for JavaBeans applies, and there must be getter and setter methods for the persistent properties.
The type of property is determined by the return type of the gette method and must be the same as the type of the single parameter passed into the setter method.
Both methods must be either *public or protected* visibility. The mapping annotations must be on the getter method.

@Entity
public class Employee{
	private long id;
	private String name;
	private long salary;

	@Id public long getId(){return id;}
	public void setId(long id){this.id=id;}

	public String getName(){return name;}
	public void setName(String name){this.name=name;}

	public long getSalary(){return wage;}
	public void setSalary(long salary){this.wage=salary;}
}

### Mixed Access

@Entity
@Access(AccessType.FIELD)
public class Employee{ ... }

@Transient private String phoneNum

@Access( AccessType.PROPERTY ) @Column(name="PHONE")
protected String getPhoneNumberForDb(){ ... }

#### Complete Employee entity class annotated to use property access for only one property
@Entity
@Access( AccessType.FIELD )
public class Employee{

	public static final String LOCAL_AREA_CODE = "613";

	@Id private long id;
	private String name;
	private long salary;
	@Transient private String phoneNum;

	public long getId(){return id;}
	public void setId(long id){this.id=id;}

	public String getName(){return name;}
	public void setName(String name){this.name=name;}

	public long getSalary(){return salary;}
	public void setSalary(long salary){this.salary=salary;}

	public String getPhoneNumber(){return phoneNum;}
	public void setPhoneNumber(String phoneNum){this.phoneNum=phoneNum;}

	@Access( AccessType.PROPERTY ) @Column(name="PHONE")
	public String getPhoneNumberForDb(){
		if( phoneNum.length()==10 ){
			return phoneNum;
		}else{
			return LOCAL_AREA_CODE + phoneNum;
		}
	}
	public void setPhoneNumberForDb(String num){
		if(num.startsWith(LOCAL_AREA_CODE) ){
			phoneNum = num.substring(3);
		}else{
			phoneNum = num;
		}
	}
}

### Mapping to a Table

@Entity
@Table(name="EMP")
public class Employee{ ... }

@Entity
@Table(name="EMP", schema="HR")
public class Employee{ ... }

@Entity
@Table(name="EMP", catalog="HR")
public class Employee{ ... }

### Mapping simple types

* Primitive Java types :  byte, boolean, char, short, int, float, long, double
* Wrapper Classes of primitives Java Types : Byte, Boolean, Character, Short, Integer, Float, Long, Double
* Byte and character array types : byte[], Byte[], char[], Character[]
* Large numeric types : java.math.BigInteger, java.math.BigDecimal
* Strings : java.lang.String
* Java Temporal Types : java.util.Date, java.util.Calendar
* JDBC Temporal Types : java.sql.Date, java.sql.Time, java.sql.Timestamp
* Enumerated types : Any system or user-defined enumerated type
* Serializable objects : Any system or user-defined serializable type

### Column Mappings

@Entity
public class Employee{
	@Id
	@Column(name="EMP_ID")
	private long id;
	private String name;
	@Column(name="SAL")
	private long salary;
	@Column(name="COMM")
	private String comments;
	// ...
}

===> 
 _____________
|  EMPLOYEE   |
 -------------
| PK | EMP_ID |
 -------------
|      NAME   |
|      SAL    |
|      COMM   |
 -------------

### Lazy Fetching

@Entity
public class Employee{
	//...
	@Column(name="COMM")
	@Basic(fetch=FetchType.LAZY)
	private String comments;
	// ...
}

### Large Objects

@Entity
public class Employee{
	//...
	@Column(name="PIC")
	@Basic(fetch=FetchType.LAZY)
	private byte[] picture;
	// ...
}

### Enumerated Types
public enum EmployeeType{
	FULL_TIME_EMPLOYEE,
	PART_TIME_EMPLOYEE,
	CONTRACT_EMPLOYEE
}

@Entity
public class Employee{
	//...
	private EmployeeTypetype; 
	// ...
}

// IF ENUM changes ==> pb, because FULL_TIME_EMPLOYEE=0, PART_TIME_EMPLOYEE = 1

@Entity
public class Employee{
	//...
	@Enumerated(EnumType.String)
	private EmployeeTypetype; 
	// ...
}

// not anymore pb apart if strings changes... the order is not important anymore

### Temporal Types

@Entity
public class Employee{
	//...
	@Temporal(TemporalType.DATE)
	private Calendar dob;

	@Temporal(TemporalType.DATE)
	@Column(name="S_DATE")
	private Date startDate;

	// ...
}

### Transient State

@Entity
public class Employee{
	@ID private long id;
	private String name;
	private long salary;
	transient private String translatedName;
	//...

	public String toString(){
		if(translatedName == null ){
			translatedName = ResourceBundle.getBundle("EmpResources").getString("Employee");
		}
		return translatedName + ": " + id + " " + name;
	}
}

## Mapping the primary key

### Overriding the PK

The @Column annotation can be used to override the column name that the id attribute is mapped to.
Primary keys are assumed to be insertable, but not nullable or updatable.
When overridin a PK column the nullable and updatable elements should not be overriden. Only in the very specific circumstance of mapping
the same column to multiple fields/relationships should the insertable element be set to false.

### PK types

* Primitive Java types :  byte, char, short, int, long
* Wrapper Classes of primitives Java Types : Byte, Character, Short, Integer, Long
* Strings : java.lang.String
* Large numeric types : java.math.BigInteger
* Java Temporal Types : java.util.Date, java.sql.Date

Ok but you have to avoid it : float, double, FLOAT, DOUBLE, BigDecimal

### Identifier Generation

@GeneratedValue

Strategies : AUTO, TABLE, SEQUENCE, IDENTITY (values of GenerationType enum class)

#### Automatic Id Generation

@Entity
public class Employee{
	@Id @GeneratedValue(Strategy=GenerationType.AUTO)
	private long id;
	// ...
}

DEV CONF

#### Id Generation Using a Table

One table = 2 columns = one for type / integral type that sotres the actual id sequence that is being generated, the value stored is the last identifier that was allocated in the sequence.
Each defined generator represents a row in th table.

@Entity
public class Employee{
	@Id @GeneratedValue(Strategy=GenerationType.TABLE)
	private long id;
	// ...
}

Provider choose his own table....

@Entity
public class Employee{
	@TableGenerator(name="Emp_Gen")
	@Id @GeneratedValue(Strategy=GenerationType.TABLE)
	private long id;
	// ...
}

@TableGenerator doesn't create tables...

@TableGenerator(name="Emp_Gen", table="ID_GEN", pkColumnName="GEN_NAME", valueColumnName="GEN_VAL")

@TableGenerator(name="Address_Gen", table="ID_GEN", pkColumnName="GEN_NAME", valueColumnName="GEN_VAL", pkColumnValue="Addr_Gen", initialValue=10000, allocationSize=100)
@Id @GeneratedValue(Strategy=GenerationType.TABLE)
private long id;

 ____________________
|      ID_GEN        |
 --------------------
|  EMP_ID | EMP_ID   |
 --------------------
| Emp_Gen  | 0       |
| Addr_Gen | 10000   |
 --------------------

==> SQL
CREATE TABLE id_gen (
	gen_name VARCHAR(80),
	gen_val INTEGER,
	CONSTRAINT pk_id_gen
	PRIMARY KEY (gen_name)
);
INSERT INTO id_gen (gen_name, gen_val) VALUES ('Emp_Gen', 0);
INSERT INTO id_gen (gen_name, gen_val) VALUES ('Addr_Gen', 10000);

#### Id Generation Using a Database Sequence

@Id @GeneratedValue(strategy=GenerationType.SEQUENCE)
private long id;

@SequenceGenerator(name="Emp_Gen", sequenceName="Emp_Seq")
@Id @GeneratedValue(generator="Emp_Gen")
private long getId;

CREATE SEQUENCE Emp_Seq
	MINVALUE 1
	START WITH 1
	INCREMENT BY 50

#### Id Generation Using Database Identity

@Id @GeneratedValue(strategy=GenerationType.IDENTITY)
private long id;
There is no generator annotation for IDENTITY because it must be defined as part of the database schema
definition for the primary key column of the entity. Because each entity primary key column defines its own identity
characteristic, IDENTITY generation cannot be shared across multiple entity types.

## Relationships

### Relationship Concepts

#### Roles

A relationship from an Employee to the Project that they work on would be bidirectional. The Employee should know its Project, and the Project should point to the Employee working on it. 
The arrows going in both directions indicate the bidirectionality of the relationship : 
* Employee and Project in a bidirectional relationship : Source/Target : Employee <-> Source/Target : Project
An Employee and its Address would likely be modeled as a unidirectional relationship because the Address is not expected to ever need to know its resident. If it did, of course, 
then it would need to become a bidirectional relationship. Figure 4-5 shows this relationship. Because the relationship is unidirectional, the arrow points from the Employee to the Address.
* Employee in a unidirectional relationship with Address : Source  : Employee -> Target : Address

### Cardinality

Employee *->1 Department
Employee *->* Department

* Each employee can work on a number of projects.
• Many employees can work on the same project.
• Each project can have a number of employees working on it.
• Many projects can have the same employee working on them.

### Ordinality

#### Mappings Overview

Each one of the mappings is named for the cardinality of the source and target roles. As shown in the previous sections, a bidirectional relationship can be viewed as a pair of two unidirectional mappings. Each of these mappings is really a unidirectional relationship mapping, and if we take the cardinalities of the source and target of the relationship and combine them together in that order, permuting them with the two possible values of “one” and “many”, we end up with the following names given to the mappings:
1. Many-to-one
2. One-to-one
3. One-to-many
4. Many-to-many

#### Single-Valued Associations

An association from an entity instance to another entity instance (where the cardinality of the target is “one”) is called a single-valued association. The many-to-one and one-to-one relationship mappings fall into this category because the source entity refers to at most one target entity. We will discuss these relationships and some of their variants first.

##### Many-to-One Mappings

 ______________________               ______________________  
|      Employee        |             |      Department      |
 ----------------------               ----------------------   
| id: Int              | ----------> | id: Int              |
| name : String        | *      0..1 | name : String        | 
| salary : long        |              ----------------------
 ----------------------

@Entity
public class Employee {
	// ...
	@ManyToOne
	private Department department;
	// ...
}

##### Using Join Columns

In the database, a relationship mapping means that one table has a reference to another table. The database term for a column that refers to a key (usually the primary key) in another table is a foreign key column. In JPA, they’re called join columns, and the @JoinColumn annotation is the primary annotation used to configure these types of columns.

 ______________________               ______________________  
|      EMPLOYEE        |             |      DEPARTMENT      |
 ----------------------               ----------------------   
| Pk  | ID             | ->o---o-|-> | Pk  | ID             |
 ----------------------  *      0..1  ----------------------
|     | NAME           |             |     | NAME           |
|     | SALARY         |              ----------------------
| FK1 | DEPT_ID        |              
 ----------------------

@Entity
public class Employee {
	@Id private long id;
	@ManyToOne
	@JoinColumn(name="DEPT_ID")
	private Department department;
	// ...
}

##### Bidirectional One-to-One Mappings

 ______________________               ______________________  
|      Employee        |             |      ParkingSpace    |
 ----------------------               ----------------------   
| id: Int              | ----------> | id: Int              |
| name : String        | 0..1   0..1 | lot : Int            | 
| salary : long        |             | location : String    | 
 ----------------------               ----------------------

  ______________________               ______________________  
|      EMPLOYEE        |             |      PARKING_SPACE   |
 ----------------------               ----------------------   
| Pk  | ID             | -|-o--o-|-> | Pk  | ID             |
 ----------------------  0..1   0..1  ----------------------
|     | NAME           |             |     | LOT            |
|     | SALARY         |             |     | LOCATION       | 
| FK1 | PSPACE_ID      |              ----------------------   
 ----------------------

@Entity
public class Employee {
	@Id private long id;
	private String name;
	@OneToOne
	@JoinColumn(name="PSPACE_ID")
	private ParkingSpace parkingSpace;
	// ...
}

#### Collection-Valued Associations

##### One-to-Many Mappings

 ______________________               ______________________  
|      Employee        |             |      Department      |
 ----------------------               ----------------------   
| id: Int              | ----------> | id: Int              |
| name : String        | *      0..1 | name: String         | 
| salary : long        |              ----------------------
 ----------------------               


@Entity
public class Department {
	@Id private long id;
	private String name;
	@OneToMany(mappedBy="department")
	private Collection<Employee> employees;
	// ...
}

@Entity
public class Department {
	@Id private long id;
	private String name;
	@OneToMany(targetEntity=Employee.class, mappedBy="department")
	private Collection employees;
	// ...
}

There are two important points to remember when defining bidirectional one-to-many (or many-to-one) relationships:
• The many-to-one side should be the owning side, so the join column should be defined on that side.
• The one-to-many mapping should be the inverse side, so the mappedBy element should be used.

##### Many-to-Many Mappings

 ______________________               ______________________  
|      Employee        |             |      Project         |
 ----------------------               ----------------------   
| id: Int              | ----------> | id: Int              |
| name : String        | *      *    | name: String         | 
| salary : long        |              ----------------------
 ----------------------      

@Entity
public class Employee {
	@Id private long id;
	private String name;
	@ManyToMany
	private Collection<Project> projects;
	// ...
}

@Entity
public class Project {
	@Id private long id;
	private String name;
	@ManyToMany(mappedBy="projects")
	private Collection<Employee> employees;
	// ...
}

The first is a mathematical inevitability: when a many-to-many relationship is bidirectional, both sides of the relationship are many-to-many mappings.

The second difference is that there are no join columns on either side of the relationship. You will see in the next section that the only way to implement a many-to-many relationship is with a separate join table. The consequence of not having any join columns in either of the entity tables is that there is no way to determine which side is the owner of the relationship. Because every bidirectional relationship as to have both an owning side and an inverse side.

##### Using Join Columns

Because the multiplicity of both sides of a many-to-many relationship is plural, neither of the two entity tables can store an unlimited set of foreign key values in a single entity row. We must use a third table to associate the two entity types. This association table is called a join table, and each many-to-many relationship must have one. They might be used for the other relationship types as well, but are not required and are therefore less common.

A join table consists simply of two foreign key or join columns to refer to each of the two entity types in the relationship. A collection of entities is then mapped as multiple rows in the table, each of which associates one entity with another. The set of rows that contain a given entity identifier in the source foreign key column represents the collection of entities related to that given entity.


 ______________________               ______________________               ______________________         
|      EMPLOYEE        |             |      DEPARTMENT      |             |      DEPARTMENT      |
 ----------------------               ----------------------               ----------------------      
| Pk  | ID             | -||------o< | Pk, FK1 | EMP_ID     | >o-----||-- | Pk  | ID             |
 ----------------------              | Pk, FK2 | PROJ_I     |              ----------------------
|     | NAME           |             |                      |             |     | NAME           |
|     | SALARY         |              ----------------------               ----------------------           
 ----------------------

@Entity
public class Employee {
	
	@Id private long id;
	private String name;
	
	@ManyToMany
	@JoinTable(name="EMP_PROJ",
		joinColumns=@JoinColumn(name="EMP_ID"),
		inverseJoinColumns=@JoinColumn(name="PROJ_ID"))
	private Collection<Project> projects;
	// ...
}

##### Unidirectional Collection Mappings

 ______________________               ______________________  
|      Employee        |             |      Phone           |
 ----------------------               ----------------------   
| id: Int              | ----------> | id: Int              |
| name : String        | 0..1      * | type: String         | 
| salary : long        |             | number: String       | 
 ----------------------                ---------------------

 ______________________               ______________________               ______________________         
|      EMPLOYEE        |             |      EMP_PHONE       |             |      DEPARTMENT      |
 ----------------------               ----------------------               ----------------------      
| Pk  | ID             | -||------o< | Pk, FK1 | PHONE_ID   | -|o----||-- | Pk  | ID             |
 ----------------------              | Pk, FK2 | EMP_ID     |              ----------------------
|     | NAME           |             |                      |             |     | TYPE           |
|     | SALARY         |              ----------------------              |     | NUM            | 
|     | DEPT_ID        |                                                   ----------------------
 ----------------------

@Entity
public class Employee {
	@Id private long id;
	private String name;
	
	@OneToMany
	@JoinTable(name="EMP_PHONE",
		joinColumns=@JoinColumn(name="EMP_ID"),
		inverseJoinColumns=@JoinColumn(name="PHONE_ID"))
	private Collection<Phone> phones;
	// ...
}