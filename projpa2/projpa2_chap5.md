# Chapter 5 : Collection Mapping

This chapter goes into more detail about how we can map more sophisticated collection types, such as persistently ordered Lists, and Maps with keys and values that are of various object types.

## Relationships and Element Collections

When we speak of mapping collections, there are actually three kinds of objects that we can store in mapped collections. We can map collections of entities, embeddables, or basic types, and each one requires a certain level of understanding to be correctly mapped and efficiently used.

Relationships define associations between independent entities, whereas element collections contain objects that are dependent upon the referencing entity, and can be retrieved only through the entity that contains them.

A practical difference between relationships and element collections is the annotation that is used to denote them. A relationship minimally requires the relationship annotation, either @OneToMany or @ManyToMany, whereas an element collection is indicated by the @ElementCollection annotation. Assuming the VacationEntry embeddable class in Listing 5-1, Listing 5-2 shows an example of an element collection of embeddables in the vacationBookings attribute, as well as an element collection of basic types (String) in the nickNames attribute.

@Embeddable
public class VacationEntry {
	@Temporal(TemporalType.DATE)
	private Calendar startDate;
	@Column(name="DAYS")
	private int daysTaken;
	// ...
}

@Entity
public class Employee {
	@Id private int id;
	private String name;
	private long salary;
	// ...
	@ElementCollection(targetClass=VacationEntry.class)
	private Collection vacationBookings;
	@ElementCollection
	private Set<String> nickNames;
	// ...
}

You can see from Listing 5-2 that, like the relationship annotations, the @ElementCollection annotation includes a targetClass element that is used to specify the class if the Collection does not define the type of element contained in it. It also includes a fetch element to indicate whether the collection should be lazily loaded. A more interesting aspect of the mappings in Listing 5-2 is the absence of any additional metadata. Recall that the elements that are being stored in the collections are not entities, so they do not have any mapped table. Embeddables are supposed to be stored in the same table as the entity that refers to them, but if there is a collection of embeddables, how would it be possible to store a multiplicity of like-mapped objects in a single row? Similarly for basic types, we could not map each nickname String to a column in the EMPLOYEE table and expect to store multiple strings in a single row. For this reason, element collections require a separate table called a collection table. Every collection table must have a join column that refers to the containing entity table. Additional columns in the collection table are used to map the attributes of the embeddable element, or the basic element state if the element is of a basic type.

We can specify a collection table using a @CollectionTable annotation, which allows us to designate the name of the table, as well as the join column. Default values will apply if the annotation or specific elements within that annotation are not specified. The table name will default to the name of the referencing entity, appended with an underscore and the name of the entity attribute that contains the element collection. The join column default is similarly the name of the referencing entity, appended with an underscore and the name of the primary key column of the entity table. Because no collection tables were specified in either of the element collections in the vacationBookings and nickNames attributes of the Employee entity defined in Listing 5-2, they are defaulted to use collection tables named EMPLOYEE_ VACATIONBOOKINGS and EMPLOYEE_NICKNAMES, respectively. The join column in each of the collection tables will be EMPLOYEE_ID, which is just the name of the entity combined with the mapped Employee primary key column.

We map the fields or properties of the embeddable type to the columns in the collection table instead of to the primary table of the entity, with the usual column name defaulting rules applying. When the element collection contains basic types, the values are also stored in a column in the collection table, with the default column name being the name of the entity attribute. Applying this rule, the nicknames would be stored in the NICKNAMES column. After all the defaults are applied, the mapped tables would look like those in Figure 5-1.

 ______________________               ___________________________                    
|      EMPLOYEE        |             | EMPLOYEE_VACATIONBOOKINGS |           
 ----------------------               ---------------------------                  
| Pk  | ID             | -||------o< | Pk, FK1 | EMPLOYEE_ID     |
 ----------------------              | Pk      | STARTDATE       |            
|     | NAME           |             | Pk      | DAYS            |            
|     | SALARY         |              ---------------------------                                              
|     |                |              ___________________________
|     |                |             | EMPLOYEE_NICKNAMES        |   
|     |                |              ---------------------------
|     |                | -||------o< | Pk, FK1 | EMPLOYEE_ID     |
 ----------------------              | Pk      | NICKNAMES       |
                                      ---------------------------  
Figure 5-1. EMPLOYEE entity table and mapped collection tables

When we first discussed embeddables, you saw how the attributes were mapped within the embeddable object but could be overridden when embedded inside other entities or embeddables. We used the @AttributeOverride annotation to override the column names. The same annotation can also be used to override the embedded attributes in the elements of an element collection. In Listing 5-3, the daysTaken attribute is being remapped, using @AttributeOverride, from being stored in the DAYS column to being stored in the DAYS_ABS column. One important difference between using @AttributeOverride on simple embedded mappings and using it to override the columns of embeddables in an element collection is that in the latter case the column specified by @AttributeOverride actually applies to the collection table, not to the entity table.

@Entity
public class Employee {
	
	@Id private int id;
	private String name;
	private long salary;
	// ...
	
	@ElementCollection(targetClass=VacationEntry.class)
	@CollectionTable(
		name="VACATION",
		joinColumns=@JoinColumn(name="EMP_ID"))
	@AttributeOverride(name="daysTaken",
		column=@Column(name="DAYS_ABS"))
	private Collection vacationBookings;
	
	@ElementCollection
	@Column(name="NICKNAME")
	private Set<String> nickNames;
	// ...

}

 ______________________               ___________________________                    
|      EMPLOYEE        |             | EMPLOYEE_VACATIONBOOKINGS |           
 ----------------------               ---------------------------                  
| Pk  | ID             | -||------o< | Pk, FK1 | EMP_ID          |
 ----------------------              | Pk      | STARTDATE       |            
|     | NAME           |             | Pk      | DAYS_ABS        |            
|     | SALARY         |              ---------------------------                                              
|     |                |              __________________________
|     |                |             | EMPLOYEE_NICKNAMES       |   
|     |                |              --------------------------
|     |                | -||------o< | Pk, FK1 | EMPLOYEE_ID    |
 ----------------------              | Pk      | NICKNAME       |
                                      --------------------------  
Figure 5-2. EMPLOYEE entity table and mapped collection tables with overrides

## Using Different Collection Types

### Sets or Collections

The most common collection type used in associations is the standard Collection superinterface. This is used when it doesn’t matter which implementation is underneath and when the common Collection methods are all that is required to access the entities stored in it.
A Set will prevent duplicate elements from being inserted and might be a simpler and more concise collection model, while a vanilla Collection interface is the most generic. Neither of these interfaces requires additional annotations beyond the original mapping annotation to further specify them. They are used the same way as if they held non-persistent objects. An example of using a Set interface for an element collection and a Collection for another is in Listing 5-3.

### Lists

Another common collection type is the List. A List is typically used when the entities or elements are to be retrieved in some user-defined order. Because the notion of row order in the database is not commonly defined, the task of determining the ordering must lie with the application. 
There are two ways to determine the order of the List. The first is to map it so that it is ordered according to state that exists in each entity or element in the List. This is the easiest method and is less intrusive on the data model.
The second involves maintaining the order of the List in an additional database column. It is more Java-friendly in that it supports the traditional ordering semantics of a Java List, but can be far less performant, as you will see in the following sections.

#### Ordering By Entity or Element Attribute

 ______________________               ___________________________                    
|      PRINTQUEUE      |             |          PRINTJOB         |           
 ----------------------               ---------------------------                  
| Pk  | NAME           | -|o------o< | Pk      | ID              |
 ----------------------              | FK1     | QUEUE           |            
|     |                |             |         | PRINT_ORDER     |            
|     |                |              ---------------------------                                              
 ----------------------            
                                     

#### Persistently Ordered Lists

@Entity
public class PrintQueue {
	
	@Id private String name;
	// ...

	@OneToMany(mappedBy="queue")
	@OrderColumn(name="PRINT_ORDER")
	private List<PrintJob> jobs;
	// ...

}

### Maps

The Map is a very common collection that is used in virtually every application and offers the ability to associate a key object with an arbitrary value object. The various underlying implementations are expected to use fast hashing techniques to optimize direct access to the keys.
There is a great deal of flexibility with Map types in JPA, given that the keys and values can be any combination of entities, basic types, and embeddables. Permuting the three types in the two key and value positions renders nine distinct Map types. We will give detailed explanations of the most common combinations in the following sections.

#### Keys and Values

Although basic types, embeddable types, or entity types can be Map keys, remember that if they are playing the role of key, they must follow the basic rules for keys. They must be comparable and respond appropriately to the hashCode() method, and equals() method when necessary2. They should also be unique, at least within the domain of a particular collection instance, so that values are not lost or overwritten in memory. Keys should not be changed, or more specifically, the parts of the key object tha t are used in the hashCode() and equals() methods must not be changed while the object is acting as a key in a Map.

#### Keying By Basic Type

@Entity
public class Employee {
	@Id private int id;
	private String name;
	private long salary;
	
	@ElementCollection
	@CollectionTable(name="EMP_PHONE")
	@MapKeyColumn(name="PHONE_TYPE")
	@Column(name="PHONE_NUM")
	private Map<String, String> phoneNumbers;

	// ...
}

The only new annotation is @MapKeyColumn, which is used to indicate the column in the collection table that stores the basic key. When the annotation is not specified, the key is stored in a column named after the mapped collection attribute, appended with the “_KEY” suffix. In

 ______________________               ___________________________                    
|      EMPLOYEE        |             |         EMP_PHONE         |           
 ----------------------               ---------------------------                  
| Pk  | ID             | -||------o< | Pk, FK1 | EMPLOYEE_ID     |
 ----------------------              | Pk      | PHONE_TYPE      |            
|     | NAME           |             |         | PHONE_NUM       |            
|     | SALARY         |              ---------------------------                                              
 ----------------------              
Figure 5-4. EMPLOYEE entity table and EMP_PHONE collection table


We should really improve our model, though, because using a String key to store something that is constrained to be one of only three values (“Home”, “Mobile”, or “Work”), is not great style. An appropriate improvement would be to use an enumerated type instead of String. We can define our enumerated type as follows: public enum PhoneType { Home, Mobile, Work }

Listing 5-8. Element Collection of Strings with Enumerated Type Keys
@Entity
public class Employee {
	@Id private int id;
	private String name;
	private long salary;
	
	@ElementCollection
	@CollectionTable(name="EMP_PHONE")
	@MapKeyEnumerated(EnumType.STRING)
	@MapKeyColumn(name="PHONE_TYPE")
	@Column(name="PHONE_NUM")
	private Map<PhoneType, String> phoneNumbers;
	
	// ...
}

Listing 5-9. One-to-Many Relationship Using a Map with String Key
@Entity
public class Department {
	@Id private int id;
	
	@OneToMany(mappedBy="department")
	@MapKeyColumn(name="CUB_ID")
	private Map<String, Employee> employeesByCubicle;
	// ...
}

What if an employee could split his time between multiple departments? We would have to change our model to a many-to-many relationship and use a join table. The @MapKeyColumn will be stored in the join table that references the two entities. The relationship is mapped in Listing 5-10. 

Listing 5-10. Many-to-Many Relationship Using a Map with String Keys

@Entity
public class Department {
	@Id private int id;
	private String name;
	
	@ManyToMany
	@JoinTable(name="DEPT_EMP",
		joinColumns=@JoinColumn(name="DEPT_ID"),
		inverseJoinColumns=@JoinColumn(name="EMP_ID"))
	@MapKeyColumn(name="CUB_ID")
	private Map<String, Employee> employeesByCubicle;
	// ...
}

 ______________________               ___________________________               ___________________________                    
|      EMPLOYEE        |             |         DEPT_EMP          |             |         DEPARTMENT        |            
 ----------------------               ---------------------------               ---------------------------                  
| Pk  | ID             | -||------o< | Pk, FK1 | EMP_ID          | >o------||- | Pk  | ID                  |
 ----------------------              | Pk, FK2 | DEPT_ID         |             |     | NAME                |            
|     | NAME           |             |         | CUB_ID          |              ---------------------------            
|     | SALARY         |              ---------------------------                                              
 ----------------------    

 Figure 5-5. EMPLOYEE and DEPARTMENT entity tables and DEPT_EMP join table

#### Keying by Entity Attribute

If each department keeps track of the employees in it, as in our previous example in Listing 5-4, we could use a Map and key on the Employee id for quick Employee lookup. The updated Department mapping is shown in Listing 5-11.

Listing 5-11. One-to-Many Relationship Keyed by Entity Attribute

@Entity
public class Department {
	// ...
	@OneToMany(mappedBy="department")
	@MapKey(name="id")
	private Map<Integer, Employee> employees;
	// ...
}

One of the reasons why the identifier attribute is used for the key is because it fits the key criteria nicely. It responds to the necessary comparison methods, hashCode() and equals(), and it is guaranteed to be unique.

#### Keying by Embeddable Type

##### Sharing Embeddable Key Mappings with Values

Listing 5-12. Employee Entity
@Entity
public class Employee {
	@Id private int id;
	@Column(name="F_NAME")
	private String firstName;
	
	@Column(name="L_NAME")
	private String lastName;
	private long salary;
	// ...
}

Listing 5-13. EmployeeName Embeddable with Read-Only Mappings
	
@Embeddable
public class EmployeeName {
	
	@Column(name="F_NAME", insertable=false, updatable=false)
	private String first_Name;
	
	@Column(name="L_NAME", insertable=false, updatable=false)
	private String last_Name;
	// ...
}

Listing 5-14. One-to-Many Relationship Keyed by Embeddable
@Entity
public class Department {
	// ...
	@OneToMany(mappedBy="department")
	private Map<EmployeeName, Employee> employees;
	// ...
}

#### Overriding Embeddable Attributes

Listing 5-15. Employee Entity with Embedded Attribute
@Entity
public class Employee {
	@Id private int id;

	@Embedded
	private EmployeeName name;
	private long salary;
	// ...
}

Listing 5-16. EmployeeName Embeddable
@Embeddable
public class EmployeeName {
	@Column(name="F_NAME")
	private String first_Name;
	@Column(name="L_NAME")
	private String last_Name;
	// ...
}

Listing 5-17. Many-to-Many Map Keyed by Embeddable Type with Overriding
@Entity
public class Department {
	@Id private int id;
	
	@ManyToMany
	@JoinTable(name="DEPT_EMP",
		joinColumns=@JoinColumn(name="DEPT_ID"),
		inverseJoinColumns=@JoinColumn(name="EMP_ID"))
	@AttributeOverrides({
	@AttributeOverride(
		name="first_Name",
		column=@Column(name="EMP_FNAME")),
	@AttributeOverride(
		name="last_Name",
		column=@Column(name="EMP_LNAME"))
	})
	private Map<EmployeeName, Employee> employees;

	// ...
}

 ______________________               ___________________________               ___________________________                    
|      EMPLOYEE        |             |         DEPT_EMP          |             |         DEPARTMENT        |            
 ----------------------               ---------------------------               ---------------------------                  
| Pk  | ID             | -||------o< | Pk, FK1 | EMP_ID          | >o------||- | Pk  | ID                  |
 ----------------------              | Pk, FK2 | DEPT_ID         |             |     | NAME                |            
|     | F_NAME         |             |         | EMP_FNAME       |              ---------------------------            
|     | L_NAME         |             |         | EMP_FLAME       |  
|     | SALARY         |              ---------------------------                                              
 ----------------------    
Figure 5-6. DEPARTMENT and EMPLOYEE entity tables and DEPT_EMP join table

@ElementCollection
@AttributeOverrides({
	@AttributeOverride(name="key.first_Name",
	column=@Column(name="EMP_FNAME")),
	@AttributeOverride(name="key.last_Name",
	column=@Column(name="EMP_LNAME"))
})
private Map<EmployeeName, EmployeeInfo> empInfos;

#### Keying by Entity

Listing 5-18. Element Collection Map Keyed by EntityType
@Entity
public class Department {
	
	@Id private int id;
	private String name;
	// ...
	
	@ElementCollection
	@CollectionTable(name="EMP_SENIORITY")
	@MapKeyJoinColumn(name="EMP_ID")
	@Column(name="SENIORITY")
	private Map<Employee, Integer> seniorities;
	// ...
}

 ______________________               ___________________________               ___________________________                    
|      EMPLOYEE        |             |         DEPT_EMP          |             |         DEPARTMENT        |            
 ----------------------               ---------------------------               ---------------------------                  
| Pk  | ID             | -||------o< | Pk, FK1 | DEPARTMENT_ID   | >o------||- | Pk  | ID                  |
 ----------------------              | Pk, FK2 | EMP_ID          |             |     | NAME                |            
|     | NAME           |             |         | SENIORITY       |             |     | SALARY              |         
|     |                |             |         |                 |              ---------------------------     
|     |                |              ---------------------------                                              
 ----------------------  
 Figure 5-7. DEPARTMENT and EMPLOYEE entity tables with EMP_SENIORITY collection table

#### Untyped Maps

If we did not want to (or were not able to) use the typed parameter version of Map<KeyType, ValueType>, we would define it using the non-parameterized style of Map shown in Listing 5-19.

Listing 5-19. One-to-Many Relationship Using a Non-parameterized Map 
@Entity
public class Department {
	// ...
	@OneToMany(targetEntity=Employee.class, mappedBy="department")
	@MapKey
	private Map employees;
	// ...
}

The targetEntity element only ever indicates the type of the Map value. Of course, if the Map holds an element collection and not a relationship, the targetClass element of @ElementCollection is used to indicate the value type of the Map.

Listing 5-20. Untyped Element Collection of Strings with String Keys
@Entity
public class Employee {
	@Id private int id;
	private String name;
	private long salary;
	
	@ElementCollection(targetClass=String.class)
	@CollectionTable(name="EMP_PHONE")
	@MapKeyColumn(name="PHONE_TYPE")
	@MapKeyClass(String.class)
	@Column(name="PHONE_NUM")
	private Map phoneNumbers;
	// ...
}

#### Rules for Maps

Learning about the various Map variants can get kind of confusing given that you can choose any one of three different kinds of key types and three different kinds of value types. Below are some of the basic rules of using a Map.
	• Use the @MapKeyClass and targetEntity/targetClass elements of the relationship and element collection mappings to specify the classes when an untyped Map is used.
	• Use @MapKey with one-to-many or many-to-many relationship Map that is keyed on an attribute of the target entity.
	• Use @MapKeyJoinColumn to override the join column of the entity key.
	• Use @Column to override the column storing the values of an element collection of basic types.
	• Use @MapKeyColumn to override the column storing the keys when keyed by a basic type.
	• Use @MapKeyTemporal and @MapKeyEnumerated if you need to further qualify a basic key that is a temporal or enumerated type.
	• Use @AttributeOverride with a “key.” or “value.” prefix to override the column of an embeddable attribute type that is a Map key or a value, respectively.

PAGE 112 super tableau

#### Duplicates

#### Null Values

#### Best Practices

When using a List, do not assume that it is ordered automatically if you have not actually specified any ordering. The List order might be affected by the database results, which are only partially deterministic with respect to how they are ordered. There are no guarantees that such an ordering will be the same across multiple executions.
• It will generally be possible to order the objects by one of their own attributes. Using the @OrderBy annotation will always be the best approach when compared to a persistent List that must maintain the order of the items within it by updating a specific order column. Use the order column only when it is impossible to do otherwise.
• Map types are very helpful, but they can be relatively complicated to properly configure. Once you reach that stage, however, the modeling capabilities that they offer and the loose association support that can be leveraged makes them ideal candidates for various kinds of relationships and element collections.
• As with the List, the preferred and most efficient use of a Map is to use an attribute of the target object as a key, making a Map of entities keyed by a basic attribute type the most common and useful. It will often solve most of the problems you encounter. A Map of basic keys and values can be a useful configuration for associating one basic object with another.
• Avoid using embedded objects in a Map, particularly as keys, because their identity is typically not defined. Embeddables in general should be treated with care and used only when absolutely necessary.
• Support for duplicate or null values in collections is not guaranteed, and is not recommended even when possible. They will cause certain types of operations on the collection type to be slower and more database-intensive, sometimes amounting to a combination of record deletion and insertion instead of simple updates.