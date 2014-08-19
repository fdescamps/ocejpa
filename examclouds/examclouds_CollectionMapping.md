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