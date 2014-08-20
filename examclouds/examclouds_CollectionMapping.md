http://examclouds.com/

# Collection Mapping

1. ***What can be stored as mapping collections in JPA 2.0?***
Collections of entities, embeddables, basic types.

2. ***Difference between relationships and element collections in JPA 2.0.*** Element collections are collections of embeddables and basic types.

3. ***Which annotation is used to define ElementCollection in JPA 2.0?*** @ElementCollection

4. ***What should be used to store element collection in JPA 2.0?*** @CollectionTable

5. ***Default values for table and columns of element collection in JPA 2.0.*** The table name will default to the name of the referencing entity, appended with an underscore and the name of the entity attribute that contains the element collection. The join column default is similarly the name of the referencing entity, appended with an underscore and the name of the primary key column of the entity table.

6. ***Where data are stored for this code?***

		@Entity 
		public class Employee { 

			@Id 
			private int id; 

			private String name; 

			private long salary; 

			// ... 

			@ElementCollection(targetClass=VacationEntry.class) 
			private Collection vacationBookings; 

			@ElementCollection 
			private Set<String> nickNames; 

			// ... 
		} 

		@Embeddable 
		public class VacationEntry { 

			@Temporal(TemporalType.DATE) 
			private Calendar startDate; 

			@Column(name="DAYS") 
			private int daysTaken; 

			// ... 
		}

		// Answer : 

		We map the fields or properties of the embeddable type to the columns in the collection table instead of to the primary table of the entity, with the usual column name defaulting rules applying. 

		When the element collection contains basic types, the values are also stored in a column in the collection table, with the default column name being the name of the entity attribute. Applying this rule, we would see that the nicknames will be stored in the NICKNAMES column. After all the defaults are applied, the mapped tables would look like those :

		Employee ( pk id ; name ; salary ) 
		EMPLOYEE_VACATIONBOOKING ( pk,fk1 EMPLOYEE_ID ; pk startDate ; pk days ) 
		EMPLOYEE_NICKNAMES ( pk,fk1 EMPLOYEE_ID ; pk nickNames)

7. ***Override attributes, table name, join column of a of VacationEntry***

		@Entity 
		public class Employee { 

			@Id 
			private int id; 
	
			private String name; 
			private long salary; 

			// ... 

			@ElementCollection(targetClass=VacationEntry.class) 
			private Collection vacationBookings; 

			@ElementCollection 
			private Set<String> nickNames; 

			// ... 
		} 

		@Embeddable 
		public class VacationEntry { 
		
			@Temporal(TemporalType.DATE) 
			private Calendar startDate; 

			@Column(name="DAYS") 
			private int daysTaken; 

			// ... 
		}

		// Answer :

		@Entity 
		public class Employee { 

			@Id 
			private int id; 

			private String name; 

			private long salary; 

			// ? 

			@ElementCollection(targetClass=VacationEntry.class) 
			@CollectionTable( 
				name="VACATION", 
				joinColumns=@JoinColumn(name="EMP_ID")) 
			@AttributeOverride(
				name="daysTaken",
				column=@Column(name="DAYS_ABS")) 
			private Collection vacationBookings; 

			@ElementCollection 
			@Column(name="NICKNAME") 
			private Set<String> nickNames; 

			// ... 
		} 


		In order to override the name of the column in which the nicknames are stored, we can use the @Column annotation, remembering again that the name specifies a column in the collection table, not the entity table. 

		Employee ( pk id ; name ; salary )
		Vacation ( pk,fk1 EMP_ID ; pk startDate ; pk DAYS_ABS ) 
		EMPLOYEE_NICKNAMES ( pk,fk1  EMPLOYEE_ID ; Pk nickNames )

8. ***Ways to determine the order of the List in JPA 2.0.*** The first is to map it so that it is ordered according to state that exists in each entity or element in the List. This is the easiest method and is less intrusive on the data model. The second involves maintaining the order of the List in an additional database column. It is more Java-friendly in that it supports the traditional ordering semantics of a Java List, but can be far less performant.

9. ***How to indicate the attribute to order by in JPA 2.0?*** We indicate the attribute to order by in the @OrderBy annotation. The value of the annotation is a string that contains one or more comma-separated fields or properties of the object being ordered. Each of the attributes can be optionally followed by an ASC or DESC keyword to define whether the attribute should be ordered in ascending or descending order. If the direction is not specified, the property will be ordered in ascending order. 

		@Entity 
	    public class Course {
	       ...
	       @ManyToMany
	       @OrderBy("lastname ASC")
	       public List<Student> getStudents() {...};
	       ...
	    }
	


	    @Entity 
	    public class Student {
	       ...
	       @ManyToMany(mappedBy="students")
	       @OrderBy // ordering by primary key is assumed
	       public List<Course> getCourses() {...};
	       ...
	    }



	 	@Entity 
	    public class Person {
	         ...
	       @ElementCollection
	       @OrderBy("zipcode.zip, zipcode.plusFour")
	       public Set<Address> getResidences() {...};
	       ...
	    }
	 
	    @Embeddable 
	    public class Address {
	       protected String street;
	       protected String city;
	       protected String state;
	       @Embedded protected Zipcode zipcode;
	    }
	 
	    @Embeddable 
	    public class Zipcode {
	       protected String zip;
	       protected String plusFour;
	    }

10. ***If the List is a relationship and references entities, how the List will be ordered by specifying @OrderBy with no fields or properties?*** If the List is a relationship and references entities, specifying @OrderBy with no fields or properties, or not specifying it at all, will cause the List to be ordered by the primary keys of the entities in the List.

11. ***If the List is an element collection of basic types, how the List will be ordered by specifying @OrderBy with no fields or properties?*** In the case of an element collection of basic types the List will be ordered by the values of the elements.

12. ***If the List is an Element collection of embeddable types, how the List will be ordered by specifying @OrderBy with no fields or properties?*** Element collections of embeddable types will result in the List being defaulted to be in some undefined order, typically in the order returned by the database in the absence of any ORDER BY clause.

13. ***Order it by name*** 
 
		@Entity 
		public class Department { 
			
			// ... 

 			@OrderBy("name ASC")
			@OneToMany(mappedBy="department") 
			private List<Employee> employees; 

			// ... 
		}

14. ***Order by name, if name had been embedded in an embedded Employee field called info that was of embeddable type EmployeeInfo.***

		@OrderBy("info.name ASC")
		@OneToMany(mappedBy="department") 
		private List<Employee> employees;

15. ***If Employee has a status, order by status and then by name:*** 

		@OrderBy("status DESC,name  ASC")
		@OneToMany(mappedBy="department") 
		private List<Employee> employees;

16. ***The prerequisite for using an attribute in an @OrderBy annotation in JPA 2.0.*** The prerequisite for using an attribute in an @OrderBy annotation is that the attribute type should be comparable, meaning that it supports comparison operators.

17. ***If the order of list isn't specified by entity's attribute, how it can be declared?*** A designated database column should be created to store it. This column is called the order column.

18. ***Which annotation has stronger persistent ordering @OrderBy or @OrderColumn?*** @OrderColumn

19. ***Does @OrderColumn annotation preclude the use of @OrderBy and vice versa in JPA 2.0?*** Yes

20. ***Write example with @OrderColumn and write tables.***

		@Entity 
		public class PrintQueue { 

			@Id
		 	private String name; 

			// ... 

			@OneToMany(mappedBy="queue")
			@OrderColumn(name="PRINT_ORDER") 
			private List<PrintJob> jobs; 

			// ... 
		} 

		PRINTQUEUE ( pk NAME )
		PRINTJOB ( pk id ; Fk1 QUEUE ; PRINT_ORDER )

21. ***What is the usual practice of mapping the physical columns in JPA 2.0?*** The practice of mapping the physical columns on the owning side because that is the side that owns the table in which they apply.

22. ***On which side @OrderColumn is used(owning or not) in JPA 2.0?*** The order column can be an exception to this rule when the relationship is bidirectional because the column is always defined beside the List that it is ordering, even though it is in the table mapped by the owning many-to-one entity side.

23. ***Default value of @OrderColumn in JPA 2.0.***  The name just defaults to the name of the entity attribute, appended by the '_ORDER' string.

24. ***Where order column is usually stored?*** The table that the order column is stored in depends on the mapping that @OrderColumn is being applied to. It is usually in the table that stores the entity or element being stored.

25. ***Where order column is stored for one to many relationship?*** The entity being stored is PrintJob, and the order column would be stored in the PRINTJOB table.

26. ***Where order column is stored for element collection?*** In the collection table.

27. ***Where order column is stored for many to many relationship?*** The order column is in the join table.

28. ***Basic rules to follow for being keys in JPA 2.0.***

	1. They must be comparable and respond appropriately to the hashCode() method, and equals() method when necessary. 
	2. They should also be unique, at least within the domain of a particular collection instance, so that values are not lost or overwritten in memory. 
	3. Keys should not be changed, or more specifically, the parts of the key object that are used in the hashCode() and equals() methods must not be changed while the object is acting as a key in a Map.

29. ***When keys are basic or embeddable types, where they are stored?*** When keys are basic or embeddable types, they are stored directly in the table being referred to. Depending upon the type of mapping, it can be either the target entity table, join table, or collection table.

30. ***When keys are entities, where they are stored?*** When keys are entities, only the foreign key is stored in the table because entities are stored in their own table, and their identity in the database must be preserved.

31. ***What determines which kind of mapping must be used in JPA 2.0?*** It is always the type of the value object in the Map that determines what kind of mapping must be used.

32. ***If the values of collection are entities what kind of mapping must be used in JPA 2.0?*** If the values are entities, the Map must be mapped as a one-to-many or many-to-many relationship. 

33. ***If the values of the Map are either embeddable or basic types, what kind of mapping must be used?*** If the values of the Map are either embeddable or basic types, the Map is mapped as an element collection.

34. ***How to override the name of the column that stores the values in the Collection?*** @Column.

35. ***Which annotation should be used to indicate the column in the collection table that stores the basic key?*** @MapKeyColumn.

36. ***Default name for key in JPA 2.0.*** When the annotation is not specified, the key is stored in a column named after the mapped collection attribute, appended with the '_KEY' suffix.

37. ***Add annotations for private Map <String, String> phoneNumbers. Specify table name, key column, value column.*** 

		@ElementCollection 
		@CollectionTable(name="EMP_PHONE") 
		@MapKeyColumn(name="PHONE_TYPE") 
		@Column(name="PHONE_NUM") 
		private Map<String, String> phoneNumbers;

38. ***Write tables for :***
	
		@Entity 
		public class Employee { 
		
			@Id 
			private int id; 

			private String name; 

			private long salary; 

			@ElementCollection 
			@CollectionTable(name="EMP_PHONE") 
			@MapKeyColumn(name="PHONE_TYPE") 
			@Column(name="PHONE_NUM") 
			private Map<String, String> phoneNumbers; 

			// ... 
		}



		Employee ( pk : id ; name ; salary) 
		EMP_PHONE ( Pk,fk1 : EMPLOYEE_ID ; pk : PHONE_TYPE ; PHONE_NUM)

39. ***Annotation for enumerated value of collection in JPA 2.0.*** @Enumerated

40. ***Annotation for enumerated key of collection in JPA 2.0.*** @MapKeyEnumerated(EnumType.STRING)

41. ***How to specify the temporal type in JPA 2.0, when the key is of type java.util.Date?*** @MapKeyTemporal

42. ***Map it, specifying table name, key column, value column, change enum type private Map<PhoneType, String> phoneNumbers;***
		
		@ElementCollection 
		@CollectionTable(name="EMP_PHONE") 
		@MapKeyEnumerated(EnumType.STRING) 
		@MapKeyColumn(name="PHONE_TYPE") 
		@Column(name="PHONE_NUM") 
		private Map<PhoneType, String> phoneNumbers;

43. ***Write the table for :*** 

		@Entity 
		public class Department { 
	
			@Id 
			private int id; 
	
			private String name; 
	
			private long salary; 
	
			@OneToMany(mappedBy="department") 
			@MapKeyColumn(name="CUB_ID") 
			private Map<String, Employee> employeesByCubicle; 
	
			// ... 
		}
	
	
		
		Employee( pk id ; name ; salary ; DEPARTEMENT ; CUB_ID )

44. ***Rewrite 43 for many to many case. Specify name of table and columns.***

		@Entity 
		public class Department { 

			@Id 
			private int id; 

			private String name; 

			private long salary;  

			@ManyToMany 
			@JoinTable(
				name="DEPT_EMP", 
				joinColumns=@JoinColumn(name="DEPT_ID"), 
				inverseJoinColumns=@JoinColumn(name="EMP_ID")) 
			@MapKeyColumn(name="CUB_ID") 
			private Map<String, Employee> employeesByCubicle; 
			// ... 
		}
	
45. ***If we did not override the key column with @MapKeyColumn, what is then the default value in JPA 2.0?*** If we did not override the key column with @MapKeyColumn, it would have been defaulted as the name of the collection attribute suffixed by '_KEY'. This would have produced a dreadful-looking EMPLOYEESBYCUBICLE_KEY.

46. ***Write tables for 44.***

		Employee( pk id ; name ; salary )
	    DEPT_EMP( pk1 fk1 : DEPT_ID ; pk2 fk2 : EMP_ID ; CUB_ID )
		DEPARTMENT( pk id ; name )

47. ***Write tables for 44 with 45.****

		Employee( pk id ; name ; salary )
	    DEPT_EMP( pk1 fk1 : DEPT_ID ; pk2 fk2 : EMP_ID ; EMPLOYEESBYCUBICLE_KEY )
		DEPARTMENT( pk id ; name )

48. ***Can map be used on both sides of many-to-many mappings?*** You can use a Map on only one side of a many-to-many relationship; it makes no difference which side.

49. ***How to key by entity attribute?*** @MapKey

50. ***Add mapping assuming that the key is id of employee*** 

		@Entity 
		public class Department { 

			// ... 
			private Map<Integer, Employee> employees; 

			// ... 
		}



		@Entity 
		public class Department { 

			// ... 
	
			@OneToMany(mappedBy="department") 
			@MapKey(name="id")
			private Map<Integer, Employee> employees; 

			// ... 
		}