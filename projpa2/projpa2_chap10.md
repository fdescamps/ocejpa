# Advanced Object-Relational Mapping

## Table and Column Names

In previous sections, we have shown the names of tables and columns as uppercase identifiers. We did this, first, because it helps differentiate them from Java identifiers and, second, because the SQL standard defines that undelimited database identifiers do not respect case, and most tend to display them in uppercase.

Anywhere a table or column name is specified, or is defaulted, the identifier string is passed through to the JDBC driver exactly as it is specified, or defaulted. For example, when no table name is specified for the Employee entity, then the name of the table assumed and used by the provider will be Employee, which by SQL definition is no different from EMPLOYEE. The provider is neither required nor expected to do anything to try to adjust the identifiers before passing them to the database driver.

The following annotations should, therefore, be equivalent in that they refer to the same table in a SQL standard compliant database:

	@Table(name="employee")
	@Table(name="Employee")
	@Table(name="EMPLOYEE")

Some database names are intended to be case-specific and must be explicitly delimited. For example, a table might be called EmployeeSales, but without case distinction would become EMPLOYEESALES, clearly less readable and harder to ascertain its meaning. While it is by no means common, or good practice, a database in theory could have an EMPLOYEE table as well as an Employee table. These would need to be delimited in order to distinguish between the two. The method of delimiting is the use of a second set of double quotes, which must be escaped, around the identifier. The escaping mechanism is the backslash (the \ character), which would cause the following annotations to
refer to different tables:

	@Table(name="\"Employee\"")
	@Table(name="\"EMPLOYEE\"")

Notice that the outer set of double quotes is just the usual delimiter of strings in annotation elements, but the inner double quotes are preceded by the backslash to cause them to be escaped, indicating that they are part of the string, not string terminators.

When using an XML mapping file, the identifier is also delimited by including quotes in the identifier name. For example, the following two elements represent different columns:

	<column name="&quot;ID&quot;"/>
	<column name="&quot;Id&quot;"/>

By including the empty delimited-identifiers element in the XML mapping file, all identifiers in the
persistence unit will be treated as delimited, and quotes will be added to them when they are passed to the driver. The only catch is that there is no way to override this setting. Once the delimited-identifiers flag is turned on, all identifiers must be specified exactly as they exist in the database. Furthermore, if you decide to turn on the delimited-identifiers option, make sure you remove any escaped quotes in your identifier names or you will find that they will be included in the name. Using escaping in addition to the delimited identifiers option will take the escaped quotes and wrap them with further quotes, making the escaped ones become part of the identifier.

## Converting Entity State

### Creating a Converter

***Listing 10-1. AttributeConverter Interface***

	public interface AttributeConverter<X,Y> {
		public Y convertToDatabaseColumn (X attribute);
		public X convertToEntityAttribute (Y dbData);
	}

***Listing 10-2. Boolean-to-Integer Converter***

	@Converter
	public class BooleanToIntegerConverter implements AttributeConverter<Boolean,Integer> {
		
		public Integer convertToDatabaseColumn (Boolean attrib) {
			return (attrib ? 1 : 0);
		}
	
		public Boolean convertToEntityAttribute (Integer dbData) {
			return (dbData > 0)
		}

	}

### Declarative Attribute Conversion

#### Converting Embedded Attributes

If the attribute to be converted is part of an embeddable type and we are converting it from within a referencing entity, then we use the attributeName element to specify the attribute to convert. For example, if the bonded attribute from above was contained within a SecurityInfo embeddable object and the Employee entity had a securityInfo attribute, then we could convert it as in Listing 10-3:

***Listing 10-3. Converting an Embedded Attribute*** 
	
	@Entity
	public class Employee {
		// ...
		@Convert(converter=BooleanToIntegerConverter.class, attributeName="bonded")
		private SecurityInfo securityInfo;
		// ...
	}

	@Embeddable
	public class SecurityInfo {
		private Boolean bonded;
		// ...
	}

In the “Complex Embedded Objects” section later in this chapter we will describe more advanced usages of embeddables; in particular, the dot notation will be shown as a means to override nested embeddables. This same notation may also be used in the attributeName of the @Convert annotation to reference a nested embedded attribute.

#### Converting Collections

	@ElementCollection
	@Convert(converter=BooleanToIntegerConverter.class)
	private List<Boolean> securityClearances;

if we wanted to convert the employee last name to be stored as uppercase characters (assuming we have defined the corresponding converter class), we would annotate the attribute as shown in Listing 10-4:

***Listing 10-4. Converting an Embeddable Attribute Key in a Relationship Map***

	@Entity
	public class Department {
		
		// ...
		@ManyToMany
		@Convert(converter=UpperCaseConverter.class, attributeName="key.lastName")
		private Map<EmployeeName, Employee> employees;
		// ...

	}

#### Limitations

### Automatic Conversion

When we defined a converter to convert a boolean to an integer, we likely had in mind that it would be used in very specific places, on one or possibly a few attributes. You generally don’t want to convert every boolean attribute in your domain to an integer. However, if you frequently use a more semantically rich data type, such as the URL class, then you might want every attribute of that type to be converted. You can do this by setting the autoApply option on the @Converter annotation. In Listing 10-5, a URL converter is declared with the autoApply option enabled. This will cause every persistent attribute of type URL in the persistence unit to be converted to a string when the entity that contains it is written to the database.

***Listing 10-5. URL-to-String Converter***

	@Converter(autoApply=true)
	public class URLConverter implements AttributeConverter<URL,String> {

		public String convertToDatabaseColumn (URL attrib) {
			return attrib.toString();
		}
		public URL convertToEntityAttribute (String dbData) {
			return new URL(dbData);
		}

	}

We can override the conversion on a per-attribute basis. If, instead of the auto-applied converter, we want to use a different converter for a given attribute, then we can annotate the attribute with the @Convert annotation and specify the converter class we want to use. Alternatively, if we want to disable the conversion altogether and let the provider revert to serialization of the URL, then we can use the disableConversion attribute:

	@Convert(disableConversion=true)
	URL homePage;

### Converters and Queries

	SELECT e FROM Employee e WHERE e.bonded = true

This query will work fine if bonded is set to be converted from boolean to integer. The generated SQL will have converted both the bonded attribute and the literal “true” to the corresponding integer by invoking the convertToDatabaseColumn() method on it and the equals operator will work just as well on integers as it does on booleans. However, we may want to query for all of the employees who are not bonded:

	SELECT e FROM Employee e WHERE NOT e.bonded

If we try to execute this query the parser will have no problem with it, but when it comes time to execute it the resulting SQL will contain a NOT and the value of e.bonded will have been converted to be an integer. This will generally cause a database exception since the NOT operation cannot be applied to an integer.

## Complex Embedded Objects

### Advanced Embedded Mappings

As an example, let’s bring back our Employee and embedded Address objects from Chapter 4 and update
the model just a little bit. Insert a ContactInfo object, containing the address plus the phone information, into each employee. Instead of having an address attribute, our Employee entity would now have an attribute named contactInfo, of type ContactInfo, annotated with @Embedded. The model is shown in Figure 10-1.

The ContactInfo class contains an embedded object, as well as some relationships, and would be annotated as shown in Listing 10-6.

***Listing 10-6. Embeddable ContactInfo Class***

	@Embeddable @Access(AccessType.FIELD)
	public class ContactInfo {

		@Embedded
		private Address residence;

		@ManyToOne
		@JoinColumn(name="PRI_NUM")
		private Phone primaryPhone;

		@ManyToMany @MapKey(name="type")
		@JoinTable(name="EMP_PHONES")
		private Map<String, Phone> phones;
		// ...

	}

The Address class remains the same as in Listing 4-26, but we have added more depth to our contact
information. Within the ContactInfo embeddable, we have the address as a nested embedded object, but we also have an additional unidirectional relationship to the phone number serving as the primary contact number. A bidirectional many-to-many relationship to the employee’s phone numbers would have a default join table named EMPLOYEE_PHONE, and on the Phone side the relationship attribute would refer to a list of Employee instances, with the mappedBy element being the qualified name of the embedded relationship attribute. By qualified, we mean that it must first contain the attribute within Employee that contains the embeddable, as well as a dot separator character (.) and the relationship attribute within the embeddable. Listing 10-7 shows the Phone class and its mapping back to the Employee entity.

***Listing 10-7. Phone Class Referring To Embedded Attribute***

	@Entity
	public class Phone {
		
		@Id private String num;
		@ManyToMany(mappedBy="contactInfo.phones")
		private List<Employee> employees;
		
		private String type;
		// ...

	}

### Overriding Embedded Relationships

Suppose the many-to-many relationship in ContactInfo was unidirectional and Phone didn’t have a reference back to the Employee that embedded the ContactInfo. We might want to embed instances of ContactInfo within a Customer entity as well. The CUSTOMER table, however, might have a PRI_CONTACT foreign key column instead of PRI_NUM, and of course we would not be able to share the same join table for both Employee and Customer relationships to the Phone. The resulting Customer class is shown in Listing 10-8. 

***Listing 10-8. Customer Class Embedding ContactInfo***

	@Entity
	public class Customer {
	
		@Id int id;
		@Embedded
		@AttributeOverride(name="address.zip", column=@Column(name="ZIP"))
		@AssociationOverrides({
			@AssociationOverride(name="primaryPhone", joinColumns=@JoinColumn(name="EMERG_PHONE")),
			@AssociationOverride(name="phones", joinTable=@JoinTable(name="CUST_PHONE"))
		})
		private ContactInfo contactInfo;
		// ...
	}

We can override the zip attribute in the address that is embedded within contactInfo by using
@AttributeOverride and navigating to the attribute in the nested embedded Address object. Because we are overriding two associations we need to use the plural variant of @AssociationOverrides. Note that if there had not been a join table explicitly specified for the phones attribute, then the default join table name would have been different depending upon which entity was embedding the ContactInfo. Since the default name is composed partly of the name of the owning entity, the table joining the Employee entity to the Phone entity would have defaulted to EMPLOYEE_PHONE, whereas in Customer the join table would have defaulted to CUSTOMER_PHONE

## Compound Primary Keys

There are two options available for having compound primary keys in an entity, depending on how the entity class is structured. Both of them require the use of a separate class containing the primary key fields called a primary key class; the difference between the two options is determined by what the entity class contains. Primary key classes must include method definitions for equals() and hashCode() in order to be able to be stored and keyed on by the persistence provider, and their fields or properties must be in the set of valid identifier types listed in the previous chapter. They must also be public, implement Serializable, and have a no-arg constructor.

As an example of a compound primary key, we will look at the Employee entity again, only this time the employee number is specific to the country where he works. Two employees in different countries can have the same employee number, but only one can be used within any given country. Figure 10-2 shows the EMPLOYEE table structured with a compound primary key to capture this requirement. Given this table definition, we will now look at how to map the Employee entity using the two different styles of primary key class.

### Id Class

The first and most basic type of primary key class is an id class. Each field of the entity that makes up the primary key is marked with the @Id annotation. The primary key class is defined separately and associated with the entity by using the @IdClass annotation on the entity class definition. Listing 10-9 demonstrates an entity with a compound primary key that uses an id class. The accompanying id class is shown in Listing 10-10.

***Listing 10-9. Using an Id Class***

	@Entity
	@IdClass(EmployeeId.class)
	public class Employee {

		@Id private String country;

		@Id
		@Column(name="EMP_ID")
		private int id;

		private String name;

		private long salary;
		// ...

	}

The primary key class must contain fields or properties that match the primary key attributes in the entity in both name and type. Listing 10-10 shows the EmployeeId primary key class. It has two fields, one to represent the country and one to represent the employee number. We have also supplied equals() and hashCode() methods to allow the class to be used in sorting and hashing operations.

***Listing 10-10. The EmployeeId Id Class***

	public class EmployeeId implements Serializable {
		
		private String country;
		private int id;
		
		public EmployeeId() {}

		public EmployeeId(String country, int id) {
			this.country = country;
			this.id = id;
		}

		public String getCountry() { return country; }

		public int getId() { return id; }

		public boolean equals(Object o) {
			return ((o instanceof EmployeeId) &&
			country.equals(((EmployeeId)o).getCountry()) &&
			id == ((EmployeeId)o).getId());
		}

		public int hashCode() {
			return country.hashCode() + id;
		}

	}

Note that there are no setter methods on the EmployeeId class. Once it has been constructed using the primary key values, it can’t be changed. We do this to enforce the notion that a primary key value cannot be changed, even when it is made up of multiple fields. Because the @Id annotation was placed on the fields of the entity, the provider will also use field access when it needs to work with the primary key class.

The id class is useful as a structured object that encapsulates all of the primary key information. For example, when doing a query based upon the primary key, such as the find() method of the EntityManager interface, an instance of the id class can be used as an argument instead of some unstructured and unordered collection of primary key data. Listing 10-11 shows a code snippet that searches for an Employee with a given country name and employee number. A new instance of the EmployeeId class is constructed using the method arguments and then used as the argument to the find() method.

***Listing 10-11. Invoking a Primary Key Query on an Entity with an Id Class***
	
	EmployeeId id = new EmployeeId(country, id);
	Employee emp = em.find(Employee.class, id);

### Embedded Id Class

An entity that contains a single field of the same type as the primary key class is said to use an embedded id class. The embedded id class is just an embedded object that happens to be composed of the primary key components. We use an @EmbeddedId annotation to indicate that it is not just a regular embedded object but also a primary key class. When we use this approach, there are no @Id annotations on the class, nor is the @IdClass annotation used. You can think of @EmbeddedId as the logical equivalent to putting both @Id and @Embedded on the field.

Like other embedded objects, the embedded id class must be annotated with @Embeddable, but the access type might differ from that of the entity that uses it. Listing 10-12 shows the EmployeeId class again, this time as an embeddable primary key class. The getter methods, equals() and hashCode() implementations are the same as the previous version from Listing 10-10.

***Listing 10-12. Embeddable Primary Key Class***

	@Embeddable
	public class EmployeeId {

		private String country;

		@Column(name="EMP_ID")
		private int id;

		public EmployeeId() {}

		public EmployeeId(String country, int id) {
			this.country = country;
			this.id = id;
		}
		// ...

	}

Using the embedded primary key class is no different than using a regular embedded type, except that the annotation used on the attribute is @EmbeddedId instead of @Embedded. Listing 10-13 shows the Employee entity adjusted to use the embedded version of the EmployeeId class. Note that since the column mappings are present on the embedded type, we do not specify the mapping for EMP_ID as was done in the case of the id class. If the embedded primary key class is used by more than one entity, then the @AttributeOverride annotation can be used to customize mappings just as you would for a regular embedded type. To return the country and id attributes of the primary key from getter methods, we must delegate to the embedded id object to obtain the values.

***Listing 10-13. Using an Embedded Id Class***

	@Entity
	public class Employee {

		@EmbeddedId private EmployeeId id;

		private String name;
		private long salary;

		public Employee() {}

		public Employee(String country, int id) {
			this.id = new EmployeeId(country, id);
		}

		public String getCountry() { return id.getCountry(); }
		public int getId() { return id.getId(); }
		// ...

	}

***Listing 10-14. Referencing an Embedded Id Class in a Query***

	public Employee findEmployee(String country, int id) {
		return (Employee)
			em.createQuery("SELECT e " +
				"FROM Employee e " +
				"WHERE e.id.country = ?1 AND e.id.id = ?2")
			.setParameter(1, country)
			.setParameter(2, id)
			.getSingleResult();
	}

## Derived Identifiers

When an identifier in one entity includes a foreign key to another entity, it’s called a derived identifier. Because the entity containing the derived identifier depends upon another entity for its identity, the first is called the dependent entity. The entity that it depends upon is the target of a many-to-one or one-to-one relationship from the dependent entity, and is called the parent entity. Figure 10-3 shows an example of a data model for the two kinds of entities, with DEPARTMENT table representing the parent entity and PROJECT table representing the dependent entity. Note that in this example there is an additional name primary key column in PROJECT, meaning that the corresponding Project entity has an identifier attribute that is not part of its relationship to Department. The dependent object cannot exist without a primary key, and since that primary key consists of the foreign key to the parent entity it should be clear that a new dependent entity cannot be persisted without the relationship to the parent entity being established. It is undefined to modify the primary key of an existing entity, thus the one-to-one or many-to-one relationship that is part of a derived identifier is likewise immutable and must not be reassigned to a new entity once the dependent entity has been persisted or already exists.

Department ( Pk ID ; name ) and Project ( PK, FK1 DEPT_ID ; PK NAME ; START_DATE : END_DATE )

### Basic Rules for Derived Identifiers

Most of the rules for derived identifiers can be summarized in a few general statements, although applying the rules together might not be quite as easy. We will go through some of the cases later to explain them, and even show an exception case or two to keep it interesting, but to lay the groundwork for those use cases the rules can be laid out as follows:

* A dependent entity might have multiple parent entities (i.e., a derived identifier might include
multiple foreign keys).
* A dependent entity must have all its relationships to parent entities set before it can be
persisted.
* If an entity class has multiple id attributes, then not only must it use an id class, but there must
also be a corresponding attribute of the same name in the id class as each of the id attributes
in the entity.
* Id attributes in an entity might be of a simple type, or of an entity type that is the target of a
many-to-one or one-to-one relationship.
* If an id attribute in an entity is of a simple type, then the type of the matching attribute in the
id class must be of the same simple type.
* If an id attribute in an entity is a relationship, then the type of the matching attribute in the
id class is of the same type as the primary key type of the target entity in the relationship
(whether the primary key type is a simple type, an id class, or an embedded id class).
* If the derived identifier of a dependent entity is in the form of an embedded id class, then
each attribute of that id class that represents a relationship should be referred to by a @MapsId
annotation on the corresponding relationship attribute.

### Shared Primary Key

A simple, if somewhat less common case, is when the derived identifier is composed of a single attribute that is the relationship foreign key. As an example, suppose there was a bidirectional one-to-one relationship between Employee and EmployeeHistory entities. Because there is only ever one EmployeeHistory per Employee, we might decide to share the primary key. In Listing 10-15, if the EmployeeHistory is the dependent entity, then we indicate that the relationship foreign key is the identifier by annotating the relationship with @Id.

***Listing 10-15. Derived Identifier with Single Attribute***

	@Entity
	public class EmployeeHistory {

		// ...
		@Id
		@OneToOne
		@JoinColumn(name="EMP_ID")
		private Employee employee;

		// ...
	}

The primary key type of EmployeeHistory is going to be of the same type as Employee, so if Employee has a simple integer identifier then the identifier of EmployeeHistory is also going to be an integer. If Employee has a compound primary key, either with an id class or an embedded id class, then EmployeeHistory is going to share the same id class (and should also be annotated with the @IdClass annotation). The problem is that this trips over the id class rule that there should be a matching attribute in the entity for each attribute in its id class. This is the exception to the rule, because of the very fact that the id class is shared between both parent and dependent entities.

Occasionally, somebody might want the entity to contain a primary key attribute as well as the relationship attribute, with both attributes mapped to the same foreign key column in the table. Even though the primary key attribute is unnecessary in the entity, some people might want to define it separately for easier access. Despite the fact that the two attributes map to the same foreign key column (which is also the primary key column), the mapping does not have to be duplicated in both places. The @Id annotation is placed on the identifier attribute and @MapsId annotates the relationship attribute to indicate that it is mapping the id attribute as well. This is shown in Listing 10-16. Note that physical mapping annotations (e.g. @Column) should not be specified on the empId attribute since @MapsId is indicating that the relationship attribute is where the mapping occurs.

***Listing 10-16. Derived Identifier with Shared Mappings***

	@Entity
	public class EmployeeHistory {

		// ...

		@Id
		int empId;

		@MapsId
		@OneToOne
		@JoinColumn(name="EMP_ID")
		private Employee employee;

		// ...

	}

### Multiple Mapped Attributes

A more common case is probably the one in which the dependent entity has an identifier that includes not only a relationship, but also some state of its own. We will use the example shown in Figure 10-3, where a Project has a compound identifier composed of a name and a foreign key to the department that manages it. With the unique identifier being the combination of its name and department, no department would be permitted to create more than one project with the same name. However, two different departments may choose the same name for their own projects. Listing 10-17 illustrates the trivial mapping of the Project identifier using @Id on both the name and dept attributes.

***Listing 10-17. Project with Dependent Identifier***

	@Entity
	@IdClass(ProjectId.class)
	public class Project {

		@Id private String name;

		@Id
		@ManyToOne
		@JoinColumns({
		@JoinColumn(name="DEPT_NUM",
			referencedColumnName="NUM"),
		@JoinColumn(name="DEPT_CTY",
			referencedColumnName="COUNTRY")})
		private Department dept;

		// ...
	}

The compound identifier means that we must also specify the primary key class using the @IdClass annotation. Recall our rule that primary key classes must have a matching named attribute for each of the id attributes on the entity, and usually the attributes must also be of the same type. However, this rule only applies when the attributes are of simple types, not entity types. If @Id is annotating a relationship, then that relationship attribute is going to be of some target entity type, and the rule extension is that the primary key class attribute must be of the same type as
the primary key of the target entity. This means that the ProjectId class specified as the id class for Project in Listing 10-17 must have an attribute named name, of type String, and another named dept that will be the same type as the primary key of Department. If Department has a simple integer primary key, then the dept attribute in ProjectId will be of type int, but if Department has a compound primary key, with its own primary key class, say DeptId, then the dept attribute in ProjectId would be of type DeptId, as shown in Listing 10-18. 

***Listing 10-18. ProjectId and DeptId Id Classes***

	public class ProjectId implements Serializable {
	
		private String name;
		private DeptId dept;
		public ProjectId() {}
	
		public ProjectId(DeptId deptId, String name) {
			this.dept = deptId;
			this.name = name;
		}

		// ...
	}

	public class DeptId implements Serializable {
	
		private int number;
		private String country;

		public DeptId() {}
		public DeptId (int number, String country) {
			this.number = number;
			this.country = country;
		}

		// ...
	}

### Using EmbeddedId

It is also possible to have a derived identifier when one or the other (or both) of the entities uses @EmbeddedId. When the id class is embedded, the non-relationship identifier attributes are mapped within the embeddable id class, as usual, but the attributes in the embedded id class that correspond to relationships are mapped by the relationship attributes in the entity. Listing 10-19 shows how the derived identifier is mapped in the Project class when an embedded id class is used. We annotate the relationship attribute with @MapsId(“dept”), indicating that it is also specifying the mapping for the dept attribute of the embedded id class. The dept attribute of ProjectId is of the same primary key type as Department in Listing 10-20.

***Listing 10-19. Project and Embedded ProjectId Class***

	@Entity
	public class Project {

		@EmbeddedId private ProjectId id;

		@MapsId("dept")
		@ManyToOne
		@JoinColumns({
			@JoinColumn(name="DEPT_NUM", referencedColumnName="NUM"),
			@JoinColumn(name="DEPT_CTRY", referencedColumnName="CTRY")})
		private Department department;

		// ...
	}

	@Embeddable
	public class ProjectId implements Serializable {

		@Column(name="P_NAME")
		private String name;

		@Embedded
		private DeptId dept;

		// ...
	}

***Listing 10-20. Department and Embedded DeptId Class***

	@Entity
	public class Department {

		@EmbeddedId private DeptId id;

		@OneToMany(mappedBy="department")
		private List<Project> projects;

		// ...
	}
	
	@Embeddable
	public class DeptId implements Serializable {

		@Column(name="NUM")
		private int number;

		@Column(name="CTRY")
		private String country;

		// ...
	}

## Advanced Mapping Elements

Additional elements may be specified on the @Column and @JoinColumn annotations (and their @MapKeyColumn, @MapKeyJoinColumn, and @OrderColumn relatives).

### Read-Only Mappings

	@Entity
	public class Employee {

		@Id
		@Column(insertable=false)
		private int id;

		@Column(insertable=false, updatable=false)
		private String name;

		@Column(insertable=false, updatable=false)
		private long salary;

		@ManyToOne
		@JoinColumn(name="DEPT_ID", insertable=false, updatable=false)
		private Department department;

		// ...
	}

We don’t need to worry about the identifier mapping being modified, because it is illegal to modify identifiers. The other mappings, though, are marked as not being able to be inserted or updated, so we are assuming that there are already entities in the database to be read in and used. No new entities will be persisted, and existing entities will never be updated.

### Optionality

When the optional element is specified as false, it indicates to the provider that the field or property mapping may not be null. The API does not actually define what the behavior is in the case when the value is null, but the provider may choose to throw an exception or simply do something else. For basic mappings, it is only a hint and can be completely ignored. The optional element may also be used by the provider when doing schema generation, because, if optional is set to true, then the column in the database must also be nullable.

Because the API does not go into any detail about ordinality of the object model, there is a certain amount of non-portability associated with using it. An example of setting the manager to be a required attribute is shown in Listing 10-22. The default value for optional is true, making it necessary to be specified only if a false value is needed.

***Listing 10-22. Using Optional Mappings***

	@Entity
	public class Employee {

		// ...
		@ManyToOne(optional=false)
		@JoinColumn(name="DEPT_ID", insertable=false, updatable=false)
		private Department department;

		// ...
	}

## Advanced Relationships

### Using Join Tables

***Listing 10-23. Many-to-One Mapping Using a Join Table***

	@Entity
	public class Employee {

		@Id private int id;
		private String name;

		private long salary;

		@ManyToOne
		@JoinTable(name="EMP_DEPT")
		private Department department;

		// ...

	}

### Avoiding Join Tables

It is also possible to map a unidirectional mapping without using a join table. It requires the foreign key to be in the target table, or “many” side of the relationship, even though the target object does not have any reference to the “one” side. This is called a unidirectional one-to-many target foreign key mapping, because the foreign key is in the target table instead of a join table.
To use this mapping, we first indicate that the one-to-many relationship is unidirectional by not specifying any mappedBy element in the annotation. Then we specify a @JoinColumn annotation on the one-to-many attribute to indicate the foreign key column. The catch is that the join column that we are specifying applies to the table of the target object, not the source object in which the annotation appears.

The example in Listing 10-24 shows how simple it is to map a unidirectional one-to-many mapping using a target foreign key. The DEPT_ID column refers to the table mapped by Employee, and is a foreign key to the DEPARTMENT table, even though the Employee entity does not have any relationship attribute back to Department.

***Listing 10-24. Unidirectional One-to-Many Mapping Using a Target Foreign Key***

	@Entity
	public class Department {

		@Id private int id;
		@OneToMany
		@JoinColumn(name="DEPT_ID")
		private Collection<Employee> employees;

		// ...

	}

### Compound Join Columns

Now that we are getting into more complex scenarios, let’s add a more interesting relationship to the mix. Let’s say that employees have managers and that each manager has a number of employees that work for him. You may not find that very interesting until you realize that managers are themselves employees, so the join columns are actually self-referential, that is, referring to the same table they are stored in. Figure 10-4 shows the EMPLOYEE table with this relationship

***Figure 10-4. EMPLOYEE table with self-referencing compound foreign key***

****EMPLOYEE****

PK-COUNTRY

PK-EMP_ID

---NAME

---SALARY

FK1 MGR_COUNTRY

FK1 MGR_ID

****MGR = MANAGER****

Listing 10-25 shows a version of the Employee entity that has a manager relationship, which is many-to-one from each of the managed employees to the manager, and a one-to-many directs relationship from the manager to its managed employees.

***Listing 10-25. Self-referencing Compound Relationships***

	@Entity
	@IdClass(EmployeeId.class)
	public class Employee {

		@Id private String country;

		@Id
		@Column(name="EMP_ID")
		private int id;

		@ManyToOne
		@JoinColumns({
			@JoinColumn(name="MGR_COUNTRY", referencedColumnName="COUNTRY"),
			@JoinColumn(name="MGR_ID", referencedColumnName="EMP_ID")
		})
		private Employee manager;

		@OneToMany(mappedBy="manager")
		private Collection<Employee> directs;

		// ...
	}

Another example to consider is in the join table of a many-to-many relationship. We can revisit the Employee and Project relationship described in Chapter 4 to take into account our compound primary key in Employee. The new table structure for this relationship is shown in Figure 10-5.

EMPLOYEE( PK COUNTRY ; PK EMP_ID ; NAME ; SALARY )
EMP_PROJECT( PK, FK1 EMP_COUNTRY ; PK, FK1 EMP_ID ; PK_FK2 PROJECT_ID )
PROJECT( PK ID, NAME )

If we keep the Employee entity as the owner, where the join table is defined, then the mapping for this relationship will be as shown in Listing 10-26.

	@Entity
	@IdClass(EmployeeId.class)
	public class Employee {

		@Id private String country;

		@Id
		@Column(name="EMP_ID")
		private int id;

		@ManyToMany
		@JoinTable(
			name="EMP_PROJECT",
			joinColumns={
				@JoinColumn(name="EMP_COUNTRY", referencedColumnName="COUNTRY"),
				@JoinColumn(name="EMP_ID", referencedColumnName="EMP_ID")},
			inverseJoinColumns=@JoinColumn(name="PROJECT_ID"))
		private Collection<Project> projects;

		// ...
	}

### Orphan Removal

The orphanRemoval element provides a convenient way of modeling parent-child relationships, or more specifically privately owned relationships. We differentiate these two because privately owned is a particular variety of parentchild in which the child entity may only be a child of one parent entity, and may not ever belong to a different parent. Once it is removed from the parent, it is considered orphaned and is deleted by the provider.

Only relationships with single cardinality on the source side can enable orphan removal, which is why the orphanRemoval option is defined on the @OneToOne and @OneToMany relationship annotations, but on neither of the @ManyToOne or @ManyToMany annotations.

When specified, the orphanRemoval element causes the child entity to be removed when the relationship between the parent and the child is broken. This can be done either by setting to null the attribute that holds the related entity, or additionally in the one-to-many case by removing the child entity from the collection. The provider is then responsible, at flush or commit time (whichever comes first), for removing the orphaned child entity.


***Listing 10-27. Employee Class with Orphan Removal of Evaluation Entities***
	
	@Entity
	public class Employee {

		@Id private int id;

		@OneToMany(orphanRemoval=true)
		private List<Evaluation> evals;

		// ...
	}

Suppose an employee receives an unfair evaluation from a manager. The employee might go to the manager to correct the information and the evaluation might be modified, or the employee might have to appeal the evaluation, and if successful the evaluation might simply be removed from the employee record. This would cause it to be deleted from the database as well. If the employee decided to leave the company, then when the employee is removed from the system his evaluations will be automatically removed along with him.

If the collection in the relationship was a Map, keyed by a different entity type, then orphan removal would only apply to the entity values in the Map, not to the keys. This means that entity keys are never privately owned.

### Mapping Relationship State

There are times when a relationship actually has state associated with it. For example, let’s say that we want to maintain the date an employee was assigned to work on a project. Storing the state on the employee is possible but less helpful, since the date is really coupled to the employee’s relationship to a particular project (a single entry in the many-to-many association). Taking an employee off a project should really just cause the assignment date to go away, so storing it as part of the employee means that we have to ensure that the two are consistent with each other, which
can be bothersome. In UML, we would show this kind of relationship using an association class. 

***Figure 10-6 shows an example of this technique.***
***Figure 10-7. Join table with additional state***

When we get to the object model, however, it becomes much more problematic. The issue is that Java has no inherent support for relationship state. Relationships are just object references or pointers, hence no state can ever exist on them. State exists on objects only, and relationships are not first-class objects. The Java solution is to turn the relationship into an entity that contains the desired state and map the new entity to what was previously the join table. The new entity will have a many-to-one relationship to each of the existing entity types, and each of the entity types will have a one-to-many relationship back to the new entity representing the relationship. The primary key of the new entity will be the combination of the two relationships to the two entity  types. Listing 10-28 shows all of the participants in the Employee and Project relationship.

***Listing 10-28. Mapping Relationship State with an Intermediate Entity***
	
	@Entity
	public class Employee {
	
		@Id private int id;

		// ...

		@OneToMany(mappedBy="employee")
		private Collection<ProjectAssignment> assignments;

		// ...
	}


	@Entity
	public class Project {

		@Id private int id;

		// ...

		@OneToMany(mappedBy="project")
		private Collection<ProjectAssignment> assignments;

		// ...
	}


	@Entity
	@Table(name="EMP_PROJECT")
	@IdClass(ProjectAssignmentId.class)
	public class ProjectAssignment {

		@Id
		@ManyToOne
		@JoinColumn(name="EMP_ID")
		private Employee employee;

		@Id
		@ManyToOne
		@JoinColumn(name="PROJECT_ID")
		private Project project;

		@Temporal(TemporalType.DATE)
		@Column(name="START_DATE", updatable=false)
		private Date startDate;
		// ...

	}


	public class ProjectAssignmentId implements Serializable {

		private int employee;
		private int project;

		// ...
	}

## Multiple Tables

To account for this, entities may be mapped across multiple tables by making use of the @SecondaryTable annotation and its plural @SecondaryTables form. The default table or the table defined by the @Table annotation is called the primary table, and any additional ones are called secondary tables. We can then distribute the data in an entity across rows in both the primary table and the secondary tables simply by defining the secondary tables as annotations on the entity and then specifying when we map each field or property which table the column is in. We do this by specifying the name of the table in the table element in @Column or @JoinColumn. We did not need to use this element earlier, because the default value of table is the name of the primary table.

To demonstrate the use of a secondary table, consider the data model shown in Figure 10-8. There is a primary key relationship between the EMP and EMP_ADDRESS tables. The EMP table stores the primary employee information, while the address information has been moved to the EMP_ADDRESS table.

***Figure 10-8. EMP and EMP_ADDRESS tables***

	Table : EMP ( PK ID ; NAME ; SALARY )
	Table : EMP_ADDRESS ( PK, FK1 EMP_ID ; STREET ; CITY ; STATE ; ZIP_CODE )

***Listing 10-29. Mapping an Entity Across Two Tables***

	@Entity
	@Table(name="EMP")
	@SecondaryTable(name="EMP_ADDRESS",pkJoinColumns=@PrimaryKeyJoinColumn(name="EMP_ID"))
	public class Employee {

		@Id private int id;
		private String name;

		private long salary;

		@Column(table="EMP_ADDRESS")
		private String street;

		@Column(table="EMP_ADDRESS")
		private String city;

		@Column(table="EMP_ADDRESS")
		private String state;

		@Column(name="ZIP_CODE", table="EMP_ADDRESS")
		private String zip;

		// ...
	}

In Chapter 4, we learned how to use the schema or catalog elements in @Table to qualify the primary table to be in a particular database schema or catalog. This is also valid in the @SecondaryTable annotation.

Previously, when discussing embedded objects, we mapped the address fields of the Employee entity into an Address embedded type. With the address data in a secondary table, it is still possible to do this by specifying the mapped table name as part of the column information in the @AttributeOverride annotation. Listing 10-30 demonstrates this approach. Note that we have to enumerate all of the fields in the embedded type even though the column names may match the correct default values.

***Listing 10-30. Mapping an Embedded Type to a Secondary Table***

	@Entity
	@Table(name="EMP")
	@SecondaryTable(name="EMP_ADDRESS",
	pkJoinColumns=@PrimaryKeyJoinColumn(name="EMP_ID"))
	public class Employee {

		@Id private int id;
		private String name;

		private long salary;

		@Embedded
		@AttributeOverrides({
			@AttributeOverride(name="street", 
				column=@Column(table="EMP_ADDRESS")),
			@AttributeOverride(name="city", 
				column=@Column(table="EMP_ADDRESS")),
			@AttributeOverride(name="state", 
				column=@Column(table="EMP_ADDRESS")),
			@AttributeOverride(name="zip",
				column=@Column(name="ZIP_CODE", table="EMP_ADDRESS"))
		})
		private Address address;

		// ...
	}

Let’s consider a more complex example involving multiple tables and compound primary keys. Figure 10-9 shows the table structure we wish to map. In addition to the EMPLOYEE table, there are two secondary tables, ORG_STRUCTURE and EMP_LOB. The ORG_STRUCTURE table stores employee and manager reporting information. The EMP_LOB table stores large objects that are infrequently fetched during normal query options. Moving large objects to a secondary table is a common design technique in many database schemas.

***Figure 10-9. Secondary tables with compound primary key relationships***
	
	Table EMP_LOB : (PK, FK1 COUNTRY ; PK, FK1 : ID ; PHOTO ; COMMENTS )
	Table EMPLOYEE : (PK COUNTRY ; PK : EMP_ID ; NAME ; SALARY ) 
	Table ORG_STRUCTURE : (PK, FK1 COUNTRY ; PK, FK1 : EMP_ID ; MGR_COUNTRY ; MGR_ID )

***Listing 10-31. Mapping an Entity with Multiple Secondary Tables***

	@Entity
	@IdClass(EmployeeId.class)
	public class Employee {

		@Id private String country;

		@Id
		@Column(name="EMP_ID")
		private int id;

		private String name;

		private long salary;
		// ...

	}
	
	@Entity
	@IdClass(EmployeeId.class)
	@SecondaryTables({
		@SecondaryTable(name="ORG_STRUCTURE", pkJoinColumns={
			@PrimaryKeyJoinColumn(name="COUNTRY", referencedColumnName="COUNTRY"),
			@PrimaryKeyJoinColumn(name="EMP_ID", referencedColumnName="EMP_ID")}),
		@SecondaryTable(name="EMP_LOB", pkJoinColumns={
			@PrimaryKeyJoinColumn(name="COUNTRY", referencedColumnName="COUNTRY"),
			@PrimaryKeyJoinColumn(name="ID", referencedColumnName="EMP_ID")})
	})
	public class Employee {

		@Id private String country;

		@Id
		@Column(name="EMP_ID")
		private int id;

		@Basic(fetch=FetchType.LAZY)
		@Lob
		@Column(table="EMP_LOB")
		private byte[] photo;

		@Basic(fetch=FetchType.LAZY)
		@Lob
		@Column(table="EMP_LOB")
		private char[] comments;

		@ManyToOne
		@JoinColumns({
			@JoinColumn(name="MGR_COUNTRY", referencedColumnName="COUNTRY",
				table="ORG_STRUCTURE"),
			@JoinColumn(name="MGR_ID", referencedColumnName="EMP_ID",
				table="ORG_STRUCTURE")
		})
		private Employee manager;

		// ...
	}

## Inheritance

### Class Hierarchies

#### Mapped Superclasses

****Annotations such as @Table are not permitted on mapped superclasses because the state defined in them applies only to its entity subclasses.****

***Listing 10-32. Entities Inheriting from a Mapped Superclass page 294*** 
	
	@Entity
	public class Employee {

		@Id private int id;
		private String name;

		@Temporal(TemporalType.DATE)
		@Column(name="S_DATE")
		private Date startDate;

		// ...
	}

	@Entity
	public class ContractEmployee extends Employee {

		@Column(name="D_RATE")
		private int dailyRate;

		private int term;

		// ...
	}

	@MappedSuperclass
	public abstract class CompanyEmployee extends Employee {

		private int vacation;

		// ...
	}

	@Entity
	public class FullTimeEmployee extends CompanyEmployee {

		private long salary;
		private long pension;

		// ...
	}

	@Entity
	public class PartTimeEmployee extends CompanyEmployee {

		@Column(name="H_RATE")
		private float hourlyRate;

		// ...
	}

#### Transient Classes in the Hierarchy

Classes in an entity hierarchy, that are not entities or mapped superclasses, are called transient classes. Entities may extend transient classes either directly or indirectly through a mapped superclass. When an entity inherits from a transient class, the state defined in the transient class is still inherited in the entity, but it is not persistent.

***Listing 10-33. Entity Inheriting from a Transient Superclass***
	
	public abstract class CachedEntity {
	
		private long createTime;
		public CachedEntity() { createTime = System.currentTimeMillis(); }
		public long getCacheAge() { return System.currentTimeMillis() - createTime; }
	}
	
	@Entity
	public class Employee extends CachedEntity {
	
		public Employee() { super(); }
	
		// ...
	}

#### Abstract and Concrete Classes

...

### Inheritance Models

#### Single-Table Strategy

##### Discriminator Column

You may have noticed an extra column named EMP_TYPE in Figure 10-11 that was not mapped to any field in any of the classes in Figure 10-10. This field has a special purpose and is required when using a single table to model inheritance. It is called a discriminator column and is mapped using the @DiscriminatorColumn annotation in conjunction with the @Inheritance annotation we have already learned about. The name element of this annotation specifies the name of the column that should be used as the discriminator column, and if not specified will be defaulted to a column named “DTYPE”.

A discriminatorType element dictates the type of the discriminator column. Some applications prefer to use strings to discriminate between the entity types, while others like using integer values to indicate the class. The type of the discriminator column may be one of three predefined discriminator column types: INTEGER, STRING, or CHAR. If the discriminatorType element is not specified, then the default type of STRING will be assumed.

##### Discriminator Value

Every row in the table will have a value in the discriminator column called a discriminator value, or a class indicator, to indicate the type of entity that is stored in that row. Every concrete entity in the inheritance hierarchy, therefore, needs a discriminator value specific to that entity type so that the provider can process or assign the correct entity type when it loads and stores the row. The way this is done is to use a @DiscriminatorValue annotation on each concrete entity class. The string value in the annotation specifies the discriminator value that instances of the class will get assigned when they are inserted into the database. This will allow the provider to recognize instances of the class when it issues queries. This value should be of the same type as was specified or defaulted as the discriminatorType element in the @DiscriminatorColumn annotation.

If no @DiscriminatorValue annotation is specified, then the provider will use a provider-specific way of obtaining the value. If the discriminatorType is STRING, then the provider will just use the entity name as the class indicator string. If the discriminatorType is INTEGER, then we would either have to specify the discriminator values for every entity class or none of them. If we were to specify some but not others, then we could not guarantee that a provider-generated value would not overlap with one that we specified. Listing 10-34 shows how our Employee hierarchy is mapped to a single-table strategy.

***Listing 10-34. Entity Hierarchy Mapped Using Single-Table Strategy***

	@Entity
	@Table(name="EMP")
	@Inheritance
	@DiscriminatorColumn(name="EMP_TYPE")
	public abstract class Employee { ... }
	
	@Entity
	public class ContractEmployee extends Employee { ... }
	@MappedSuperclass
	public abstract class CompanyEmployee extends Employee { ... }
	
	@Entity
	@DiscriminatorValue("FTEmp")
	public class FullTimeEmployee extends CompanyEmployee { ... }
	
	@Entity(name="PTEmp")
	public class PartTimeEmployee extends CompanyEmployee { ... }

The Employee class is the root class, so it establishes the inheritance strategy and discriminator column. We have assumed the default strategy of SINGLE_TABLE and discriminator type of STRING.
Neither the Employee nor the CompanyEmployee classes have discriminator values, because discriminator values should not be specified for abstract entity classes, mapped superclasses, transient classes, or any abstract classes for that matter. Only concrete entity classes use discriminator values since they are the only ones that actually get stored and retrieved from the database. 

The ContractEmployee entity does not use a @DiscriminatorValue annotation, because the default string “ContractEmployee”, which is the default entity name that is given to the class, is just what we want. The FullTimeEmployee class explicitly lists its discriminator value to be “FTEmp”, so that is what is stored in each row for instances of FullTimeEmployee. Meanwhile, the PartTimeEmployee class will get “PTEmp” as its discriminator value because it set its entity name to be “PTEmp”, and the entity name gets used as the discriminator value when none is specified. In Figure 10-12, we can see a sample of some of the data that we might find given the earlier model and settings. We can see from the EMP_TYPE discriminator column that there are three different types of concrete entities. We also see null values in the columns that do not apply to an entity instance.

#### Joined Strategy

***Listing 10-35. Entity Hierarchy Mapped Using the Joined Strategy***

	@Entity
	@Table(name="EMP")
	@Inheritance(strategy=InheritanceType.JOINED)
	@DiscriminatorColumn(name="EMP_TYPE", discriminatorType=DiscriminatorType.INTEGER)
	public abstract class Employee { ... }

	@Entity
	@Table(name="CONTRACT_EMP")

	@DiscriminatorValue("1")
	public class ContractEmployee extends Employee { ... }

	@MappedSuperclass
	public abstract class CompanyEmployee extends Employee { ... }

	@Entity
	@Table(name="FT_EMP")
	@DiscriminatorValue("2")
	public class FullTimeEmployee extends CompanyEmployee { ... }

	@Entity
	@Table(name="PT_EMP")
	@DiscriminatorValue("3")
	public class PartTimeEmployee extends CompanyEmployee { ... }

#### Table-per-Concrete-Class Strategy

***Listing 10-36. Entity Hierarchy Mapped Using Table-per-Concrete-Class Strategy***
	
	@Entity
	@Inheritance(strategy=InheritanceType.TABLE_PER_CLASS)
	public abstract class Employee {

		@Id private int id;
		private String name;
	
		@Temporal(TemporalType.DATE)
		@Column(name="S_DATE")
		private Date startDate;

		// ...
	}

	@Entity
	@Table(name="CONTRACT_EMP")
	@AttributeOverrides({
		@AttributeOverride(name="name", column=@Column(name="FULLNAME")),
		@AttributeOverride(name="startDate", column=@Column(name="SDATE"))
	})
	public class ContractEmployee extends Employee {

		@Column(name="D_RATE")
		private int dailyRate;

		private int term;

		// ...
	}

	@MappedSuperclass
	public abstract class CompanyEmployee extends Employee {

		private int vacation;

		@ManyToOne
		private Employee manager;

		// ...
	}

	@Entity @Table(name="FT_EMP")
	public class FullTimeEmployee extends CompanyEmployee {

		private long salary;
		@Column(name="PENSION")

		private long pensionContribution;

		// ...
	}

	@Entity
	@Table(name="PT_EMP")
	@AssociationOverride(name="manager", joinColumns=@JoinColumn(name="MGR"))
	public class PartTimeEmployee extends CompanyEmployee {

		@Column(name="H_RATE")
		private float hourlyRate;

		// ...
	}

### Mixed Inheritance

***Listing 10-37. Entity Hierarchy Mapped Using Mixed Strategies***

	@Entity
	@Table(name="EMP")
	@Inheritance(strategy=InheritanceType.JOINED)
	@DiscriminatorColumn(name="EMP_TYPE")
	public abstract class Employee {

		@Id private int id;

		private String name;

		@Temporal(TemporalType.DATE)
		@Column(name="S_DATE")
		private Date startDate;

		// ...
	}

	@Entity
	@Table(name="CONTRACT_EMP")
	public class ContractEmployee extends Employee {

		@Column(name="D_RATE") private int dailyRate;
		private int term;

		// ...
	}

	@Entity
	@Table(name="COMPANY_EMP")
	@Inheritance(strategy=InheritanceType.SINGLE_TABLE)
	public abstract class CompanyEmployee extends Employee {

		private int vacation;

		// ...
	}

	@Entity
	public class FullTimeEmployee extends CompanyEmployee {

		private long salary;

		@Column(name="PENSION")
		private long pensionContribution;

		// ...
	}

	@Entity
	public class PartTimeEmployee extends CompanyEmployee {

		@Column(name="H_RATE")
		private float hourlyRate;

		// ...
	}

