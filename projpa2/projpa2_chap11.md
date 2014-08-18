# Advanced Queries

## SQL Queries

### Native Queries vs. JDBC

The main reason to consider using SQL queries in JPA, instead of just issuing JDBC queries, is when the result of the query will be converted back into entities. As an example, let’s consider a typical use case for SQL in an application that uses JPA. Given the employee id for a manager, the application needs to determine all the employees who report to that manager either directly or indirectly. For example, if the query were for a senior manager, the results would include all the managers who report to that senior manager as well as the employees who report to those managers. This type of query cannot be implemented by using JP QL, but a database such as Oracle natively supports hierarchical queries for just this purpose. Listing 11-1 demonstrates the typical sequence of JDBC calls to execute this query and transform the results into entities for use by the application.

***Listing 11-1. Querying Entities Using SQL and JDBC***

	@Stateless
	public class OrgStructureBean implements OrgStructure {

		private static final String ORG_QUERY =
			"SELECT emp_id, name, salary " +
			"FROM emp " +
			"START WITH manager_id = ? " +
			"CONNECT BY PRIOR emp_id = manager_id";

		@Resource
		DataSource hrDs;

		public List findEmployeesReportingTo(int managerId) {

			Connection conn = null;
			PreparedStatement sth = null;

			try {

				conn = hrDs.getConnection();
				sth = conn.prepareStatement(ORG_QUERY);
				sth.setLong(1, managerId);
				ResultSet rs = sth.executeQuery();
				ArrayList<Employee> result = new ArrayList<Employee>();
				while (rs.next()) {
					Employee emp = new Employee();
					emp.setId(rs.getInt(1));
					emp.setName(rs.getString(2));
					emp.setSalary(rs.getLong(3));
					result.add(emp);
				}
				return result;

			} catch (SQLException e) {
				throw new EJBException(e);
			}
		}

	}

Now consider the alternative syntax supported by JPA, as shown in Listing 11-2. By simply indicating that the result of the query is the Employee entity, the query engine uses the object-relational mapping of the entity to figure out which result columns map to the entity properties and builds the result set accordingly.

***Listing 11-2. Querying Entities Using SQL and the Query Interface***

	@Stateless
	public class OrgStructureBean implements OrgStructure {

	private static final String ORG_QUERY =
		"SELECT emp_id, name, salary, manager_id, dept_id, address_id " +
		"FROM emp " +
		"START WITH manager_id = ? " +
		"CONNECT BY PRIOR emp_id = manager_id";

	@PersistenceContext(unitName="EmployeeService")
	EntityManager em;

	public List findEmployeesReportingTo(int managerId) {
		return em.createNativeQuery(ORG_QUERY, Employee.class)
			.setParameter(1, managerId)
			.getResultList();
		}
	}

Not only is the code much easier to read but it also makes use of the same Query interface that can be used for JP QL queries. This helps to keep application code consistent because it needs to concern itself only with the EntityManager and Query interfaces.

An unfortunate result of adding the TypedQuery interface in JPA 2.0 is that the createNativeQuery() method was already defined in JPA 1.0 to accept a SQL string and a result class and return an untyped Query interface. Now there is no backward compatible way to return a TypedQuery instead of a Query. The regrettable consequence is that when the createNativeQuery() method is called with a result class argument one might mistakenly think it will produce a TypedQuery, like createQuery() and createNamedQuery() do when a result class is passed in.

### Defining and Executing SQL Queries

If the column aliases of the query do not match up exactly with the object-relational mappings for the entity, or if the results contain both entity and non-entity results, SQL result set mapping metadata is required. SQL result set mappings are defined as persistence unit metadata and are referenced by name. When the createNativeQuery() method is invoked with a SQL query string and a result set mapping name, the query engine uses this mapping to build the result set. SQL result set mappings are discussed in the next section.

Named SQL queries are defined using the @NamedNativeQuery annotation. This annotation may be placed on any entity and defines the name of the query as well as the query text. Like JP QL named queries, the name of the query must be unique within the persistence unit. If the result type is an entity, the resultClass element may be used to indicate the entity class. If the result requires a SQL mapping, the resultSetMapping element may be used to specify the mapping name. Listing 11-3 shows how the hierarchical query demonstrated earlier would be defined as a named query.

***Listing 11-3. Using an Annotation to Define a Named Native Query***

	@NamedNativeQuery(
		name="orgStructureReportingTo",
		query=	"SELECT emp_id, name, salary, manager_id, dept_id, address_id " +
				"FROM emp " +
				"START WITH manager_id = ? " +
				"CONNECT BY PRIOR emp_id = manager_id",
		resultClass=Employee.class
	)

One advantage of using named SQL queries is that the application can use the createNamedQuery() method on the EntityManager interface to create and execute the query. The fact that the named query was defined using SQL instead of JP QL is not important to the caller. A further benefit is that createNamedQuery() can return a TypedQuery whereas the createNativeQuery() method returns an untyped Query. 

Listing 11-4 demonstrates the reporting structure bean again, this time using a named query. The other advantage of using named queries instead of dynamic queries is that they can be overridden using XML mapping files. A query originally specified in JP QL can be overridden with a SQL version, and vice versa. This technique is described in Chapter 13.

***Listing 11-4. Executing a Named SQL Query***
	
	@Stateless
	public class OrgStructureBean implements OrgStructure {

		@PersistenceContext(unitName="EmployeeService")
		EntityManager em;

		public List<Employee> findEmployeesReportingTo(int managerId) {
			return em.createNamedQuery("orgStructureReportingTo",
				Employee.class)
				.setParameter(1, managerId)
				.getResultList();
		}

	}

***Listing 11-5. Using SQL INSERT and DELETE Statements***

	@Stateless
	@TransactionAttribute(TransactionAttributeType.REQUIRES_NEW)
	public class LoggerBean implements Logger {

		private static final String INSERT_SQL =
			"INSERT INTO message_log (id, message, log_dttm) " +
			" VALUES(id_seq.nextval, ?, SYSDATE)";

		private static final String DELETE_SQL =
			"DELETE FROM message_log";

		@PersistenceContext(unitName="Logger")
		EntityManager em;

		public void logMessage(String message) {
			em.createNativeQuery(INSERT_SQL)
				.setParameter(1, message)
				.executeUpdate();
		}

		public void clearMessageLog() {
			em.createNativeQuery(DELETE_SQL)
				.executeUpdate();
		}
	}

****Executing SQL statements that make changes to data in tables mapped by entities is generally discouraged. Doing so may cause cached entities to be inconsistent with the database because the provider cannot track changes made to entity state that has been modified by data-manipulation statements.****

### SQL Result Set Mapping

It is not always the case that the names match up, nor is it always the case that only a single entity type is returned. JPA provides SQL result set mappings to handle these scenarios. A SQL result set mapping is defined using the @SqlResultSetMapping annotation. It may be placed on an entity class and consists of a name (unique within the persistence unit) and one or more entity and column mappings. The entity result class argument on the createNativeQuery() method is really a shortcut to specifying a simple SQL result set mapping. The following mapping is equivalent to specifying Employee.class in a call to createNativeQuery():

	@SqlResultSetMapping(
		name="EmployeeResult",
		entities=@EntityResult(entityClass=Employee.class)
	)

#### Mapping Foreign Keys

	SELECT emp_id, name, salary, manager_id, dept_id, address_id
	FROM emp START WITH manager_id IS NULL
	CONNECT BY PRIOR emp_id = manager_id

The MANAGER_ID, DEPT_ID, and ADDRESS_ID columns all map to the join columns of associations on the Employee entity. An Employee instance returned from this query can use the methods getManager(), getDepartment(), and getAddress(), and the results will be as expected. The persistence provider will retrieve the associated entity based on the foreign key value read in from the query. There is no way to populate collection associations from a SQL query. Entity instances constructed from this example are effectively the same as they would have been had they been returned from a JP QL query.

#### Multiple Result Mappings

A query may return more than one entity at a time. This is most often useful if there is a one-to-one relationship between two entities; otherwise, the query will result in duplicate entity instances. Consider the following query:

	SELECT emp_id, name, salary, manager_id, dept_id, address_id,
	id, street, city, state, zip
	FROM emp, address
	WHERE address_id = id

The SQL result set mapping to return both the Employee and Address entities out of this query is defined in Listing 11-6. Each entity is listed in an @EntityResult annotation, an array of which is assigned to the entities element. The order in which the entities are listed is not important. The query engine uses the column names of the query to match against entity mapping data, not column position.

***Listing 11-6. Mapping a SQL Query that Returns Two Entity Types***

	@SqlResultSetMapping(
		name="EmployeeWithAddress",
		entities={	@EntityResult(entityClass=Employee.class),
					@EntityResult(entityClass=Address.class)}
	)

#### Mapping Column Aliases

If the column aliases in the SQL statement do not directly match up with the names specified in the column mappings for the entity, field result mappings are required for the query engine to make the correct association. Suppose, for example, that both the EMP and ADDRESS tables listed in the previous example used the column ID for their primary key. The query would have to be altered to alias the ID columns so that they are unique: 

	SELECT emp.id AS emp_id, name, salary, manager_id, dept_id, address_id,
	address.id, street, city, state, zip
	FROM emp, address
	WHERE address_id = address.id

The @FieldResult annotation is used to map column aliases to the entity attributes in situations where the name in the query is not the same as the one used in the column mapping. Listing 11-7 shows the mapping required to convert the EMP_ID alias to the id attribute of the entity. More than one @FieldResult may be specified, but only the mappings that are different need to be specified. This can be a partial list of entity attributes.

***Listing 11-7. Mapping a SQL Query with Unknown Column Aliases***

	@SqlResultSetMapping(
		name="EmployeeWithAddress",
		entities={	
			@EntityResult( 	entityClass=Employee.class, 
							fields=@FieldResult(
								name="id",
								column="EMP_ID")),
			@EntityResult(entityClass=Address.class)
		}
	)

#### Mapping Scalar Result Columns

SQL queries are not limited to returning only entity results, although it is expected that this will be the primary use case. Consider the following query:

	SELECT e.name AS emp_name, m.name AS manager_name
	FROM emp e, emp m
	WHERE e.manager_id = m.emp_id (+)
	START WITH e.manager_id IS NULL
	CONNECT BY PRIOR e.emp_id = e.manager_id

Non-entity result types, called scalar result types, are mapped using the @ColumnResult annotation. One or more column mappings may be assigned to the columns attribute of the mapping annotation. The only attribute available for a column mapping is the column name. Listing 11-8 shows the SQL mapping for the employee and manager hierarchical query.

***Listing 11-8. Scalar Column Mappings***
	
	@SqlResultSetMapping(
		name="EmployeeAndManager",
		columns={	
			@ColumnResult(name="EMP_NAME"),
			@ColumnResult(name="MANAGER_NAME")
		}
	)

Let’s look at a more complex example in which this would be the case. A report for an application needs to see information about each department, showing the manager, the number of employees, and the average salary. The following JP QL query produces the correct report:

	SELECT d, m, COUNT(e), AVG(e.salary)
	FROM Department d 	LEFT JOIN d.employees e
						LEFT JOIN d.employees m
	WHERE m IS NULL OR m IN (	SELECT de.manager
								FROM Employee de
								WHERE de.department = d)
	GROUP BY d, m

This query is particularly challenging because there is no direct relationship from Department to the Employee who is the manager of the department. Therefore, the employees relationship must be joined twice: once for the employees assigned to the department and once for the employee in that group who is also the manager. This is possible because the subquery reduces the second join of the employees relationship to a single result. We also need to accommodate the fact that there might not be any employees currently assigned to the department, and further that a department might not have a manager assigned. This means that each of the joins must be an outer join and that we further have to use an OR condition to allow for the missing manager in the WHERE clause. 

Once in production, it is determined that the SQL query generated by the provider is not performing well, so the DBA proposes an alternate query that takes advantage of the inline views possible with the Oracle database. The query to accomplish this result is shown in Listing 11-9.

***Listing 11-9. Department Summary Query***
	
	SELECT d.id, d.name AS dept_name,
		e.emp_id, e.name, e.salary, e.manager_id, e.dept_id,
		e.address_id,
		s.tot_emp, s.avg_sal
	FROM dept d,
		(	SELECT *
			FROM emp e
			WHERE EXISTS(
				SELECT 1 FROM emp WHERE manager_id = e.emp_id)) e,
		( 	SELECT d.id, COUNT(*) AS tot_emp, AVG(e.salary) AS avg_sal
			FROM dept d, emp e
			WHERE d.id = e.dept_id (+)
			GROUP BY d.id) s
	WHERE d.id = e.dept_id (+) AND
	d.id = s.id

Fortunately, mapping this query is a lot easier than reading it. The query results consist of a Department entity, an Employee entity, and two scalar results, the number of the employees and the average salary. Listing 11-10 shows the mapping for this query.

***Listing 11-10. Mapping for the Department Query***

	@SqlResultSetMapping(
		name="DepartmentSummary",
		entities={
			@EntityResult(	entityClass=Department.class,
							fields=@FieldResult(name="name", column="DEPT_NAME")),
			@EntityResult(entityClass=Employee.class)
		},
		columns={	@ColumnResult(name="TOT_EMP"),
					@ColumnResult(name="AVG_SAL")
		}
	)

#### Mapping Compound Keys

***Listing 11-11. SQL Query Returning Employee and Manager***
	
	SELECT e.country, e.emp_id, e.name, e.salary,
		e.manager_country, e.manager_id, m.country AS mgr_country,
		m.emp_id AS mgr_id, m.name AS mgr_name, m.salary AS mgr_salary,
		m.manager_country AS mgr_mgr_country, m.manager_id AS mgr_mgr_id
	FROM emp e, emp m
	WHERE e.manager_country = m.country AND
		e.manager_id = m.emp_id

The result set mapping for this query depends on the type of primary key class used by the target entity. Listing 11-12 shows the mapping in the case where an id class has been used. For the primary key, each attribute is listed as a separate field result. For the foreign key, each primary key attribute of the target entity (the Employee entity again in this example) is suffixed to the name of the relationship attribute.

***Listing 11-12. Mapping for Employee Query Using id Class***

	@SqlResultSetMapping(
		name="EmployeeAndManager",
		entities={
			@EntityResult(entityClass=Employee.class),
			@EntityResult(
				entityClass=Employee.class,
				fields={
					@FieldResult(name="country", column="MGR_COUNTRY"),
					@FieldResult(name="id", column="MGR_ID"),
					@FieldResult(name="name", column="MGR_NAME"),
					@FieldResult(name="salary", column="MGR_SALARY"),
					@FieldResult(	name="manager.country",
									column="MGR_MGR_COUNTRY"),
					@FieldResult(name="manager.id", column="MGR_MGR_ID")
				}
			)
		}
	)

If Employee uses an embedded id class instead of an id class, the notation is slightly different. We have to include the primary key attribute name as well as the individual attributes within the embedded type. Listing 11-13 shows the result set mapping using this notation.

***Listing 11-13. Mapping for Employee Query Using Embedded id Class***

	@SqlResultSetMapping(
		name="EmployeeAndManager",
		entities={
			@EntityResult(entityClass=Employee.class),
			@EntityResult(
				entityClass=Employee.class,
				fields={
					@FieldResult(name="id.country", column="MGR_COUNTRY"),
					@FieldResult(name="id.id", column="MGR_ID"),
					@FieldResult(name="name", column="MGR_NAME"),
					@FieldResult(name="salary", column="MGR_SALARY"),
					@FieldResult(	name="manager.id.country",
									column="MGR_MGR_COUNTRY"),
					@FieldResult(name="manager.id.id", column="MGR_MGR_ID")
				}
			)
		}
	)


#### Mapping Inheritance

Assume that the Employee entity had been mapped to the table shown in Figure 10-11 from Chapter 10.
To understand aliasing a discriminator column, consider the following query that returns data from another EMPLOYEE_STAGE table structured to use single-table inheritance:

	SELECT id, name, start_date, daily_rate, term, vacation,
	hourly_rate, salary, pension, type
	FROM employee_stage

To convert the data returned from this query to Employee entities, the following result set mapping would be used:

	@SqlResultSetMapping(
		name="EmployeeStageMapping",
		entities=
			@EntityResult(
				entityClass=Employee.class,
				discriminatorColumn="TYPE",
				fields={
					@FieldResult(name="startDate", column="START_DATE"),
					@FieldResult(name="dailyRate", column="DAILY_RATE"),
					@FieldResult(name="hourlyRate", column="HOURLY_RATE")
				}
			)
		)

#### Mapping to Non-Entity Types

Consider the constructor expression example that we introduced in Chapter 8 to create instances of the EmployeeDetails data type by selecting specific fields from the Employee entity:

	SELECT NEW example.EmployeeDetails(e.name, e.salary, e.department.name)
	FROM Employee e

In order to replace this query with its native equivalent and achieve the same result, we must define both the replacement native query and an SqlResultSetMapping that defines how the query results map to the user-specified type. Let’s begin by defining the SQL native query to replace the JP QL.

	SELECT e.name, e.salary, d.name AS deptName
	FROM emp e, dept d
	WHERE e.dept_id = d.id

The mapping for this query is more verbose than the JP QL equivalent. Unlike JP QL where the columns are implicitly mapped to the constructor based on their position, native queries must fully define the set of data that will be mapped to the constructor in the mapping annotation. The following example demonstrates the mapping for this query:

	@SqlResultSetMapping(
		name="EmployeeDetailMapping",
		classes={
			@ConstructorResult(
				targetClass=example.EmployeeDetails.class,
				columns={
					@ColumnResult(name="name"),
					@ColumnResult(name="salary", type=Long.class),
					@ColumnResult(name="deptName")
	})})

### Parameter Binding

/

### Stored Procedures

One of the important new features added to JPA 2.1 is the ability to map and invoke stored procedures from a database. JPA 2.1 introduces first class support for stored procedure queries on the EntityManager and defines a new query type, StoredProcedureQuery, which extends Query and better handles the range of options open to developers who leverage stored procedures in their applications.

#### Defining and Executing Stored Procedure Queries

JPA stored procedure definitions support the main parameter types defined for JDBC stored procedures: IN, OUT, INOUT, and REF_CURSOR. As their name suggests, IN and OUT parameter types pass data to the stored procedure or return it to the caller, respectively. INOUT parameter types combine IN and OUT behavior into a single type that can both accept and return a value to the caller. REF_CURSOR parameter types are used to return result sets to the caller. Each of these types has a corresponding enum value defined on the ParameterMode type. 

Stored procedures are assumed to return all values through parameters except in the case where the database supports returning a result set (or multiple result sets) from a stored procedure. Databases that support this method of returning result sets usually do so as an alternative to the use of REF_CURSOR parameter types. Unfortunately, stored procedure behavior is very vendor specific, and this is an example where writing to a particular feature of a database is unavoidable in application code.

##### Scalar Parameter Types

To begin, let’s consider a simple stored procedure named “hello” that accepts a single string argument named “name” and returns a friendly greeting to the caller via the same parameter. The following example demonstrates how to define and execute such a stored procedure:

	StoredProcedureQuery q = em.createStoredProcedureQuery("hello"); 
	q.registerStoredProcedureParameter("name", String.class, ParameterMode.INOUT);
	q.execute();
	String value = q.getOutputParameterValue("name");

##### Result Set Parameter Types

Now let’s move to a more sophisticated example in which we have a stored procedure named “fetch_emp” that retrieves employee data and returns it to the caller via a REF_CURSOR parameter. In this case, we will use the overloaded form of createStoredProcedureQuery() to indicate that the result set consists of Employee entities.

The following example demonstrates this approach:

	StoredProcedureQuery q = em.createStoredProcedureQuery("fetch_emp");
	q.registerStoredProcedureParameter("empList", void.class, ParameterMode.REF_CURSOR);
	if (q.execute()) {
		List<Employee> emp = (List<Employee>)q.getOutputParameterValue("empList");
		// ...
	}

The first thing to note about this example is that we are not specifying a parameter class type for the empList parameter. Whenever REF_CURSOR is specified as the parameter type, the class type argument to registerStoredProcedureParameter() is ignored. The second thing to note is that we are checking the return value of execute() to see any result sets were returned during the execution of the query. If no result sets are returned (or the query only returns scalar values), execute() will return false. In the event that no records were returned and we did not check the result of execute(), then the parameter value would be null.

Previously it was noted that some databases allow result sets to be returned from stored procedures without having to specify a REF_CURSOR parameter. In this case, we can simplify the example by using the getResultList() method of the query interface to directly access the results:

	StoredProcedureQuery q = em.createStoredProcedureQuery("fetch_emp");
	List<Employee> emp = (List<Employee>)q.getResultList();
	// ...

As a shortcut for executing queries, both getResultList() and getSingleResult() implicitly invoke execute(). In the case where the stored procedure returns more than one result set, each call to getResultList() will return the next result set in the sequence.

#### Stored Procedure Mapping

Stored procedure queries may be declared using the @NamedStoredProcedureQuery annotation and then
referred to by name using the createNamedStoredProcedureQuery() method on EntityManager. This simplifies the code required to invoke the query and allows for more complex mappings than are possible using the StoredProcedureQuery interface. Like other JPA query types, names for stored procedure queries must be unique within the scope of a persistence unit.

We’ll begin by declaring the simple “hello” example using annotations and then progressively move to more complex examples. The “hello” stored procedure would be mapped as follows:

	@NamedStoredProcedureQuery(
		name="hello",
		procedureName="hello",
		parameters={
			@StoredProcedureParameter(	name="name", type=String.class,
										mode=ParameterMode.INOUT)
	})

Here we can see that the native procedure name must be specified in addition to the name of the stored procedure query. It is not defaulted as it is when using the createStoredProcedureQuery() method. As with the programmatic example, all parameters must be specified using the same arguments as would be used with registerStoredProcedureParameter().

The “fetch_emp” query is mapped similarly, but in this example we must also specify a mapping for the result set type:

	@NamedStoredProcedureQuery(
		name="fetch_emp",
		procedureName="fetch_emp",
		parameters={
			@StoredProcedureParameter(
				name="empList", type=void.class,
				mode=ParameterMode.REF_CURSOR)
		},
		resultClasses=Employee.class)

The list of classes provided to the resultClasses field is a list of entity types. In the version of this example where REF_CURSOR is not used and the results are returned directly from the stored procedure, the empList parameter would simply be omitted. The resultClasses field would be sufficient. It is important to note that the entities referenced in resultClasses must match the order in which result set parameters are declared for the stored procedure. For example, if there are two REF_CURSOR parameters, “empList” and “deptList”, then the resultClasses field must contain Employee and Department in that order.

For cases where the result sets returned by the query do not natively map to entity types, SQL result set mappings may also be included in the NamedStoredProcedureQuery annotation using the resultSetMappings field. For example, a stored procedure that performed the employee and manager query defined in Listing 11-11 (and mapped in Listing 11-12) would be mapped as follows:

	@NameStoredProcedureQuery(
		name="queryEmployeeAndManager",
		procedureName="fetch_emp_and_mgr",
		resultSetMappings = "EmployeeAndManager"
	)

## Entity Graphs

Unlike what its name implies, an entity graph is not really a graph of entities, but rather a template for specifying entity and embeddable attributes. It serves as a pattern that can be passed into a find method or query to specify which entity and embeddable attributes should be fetched in the query result. In more concrete terms, entity graphs are used to override at runtime the fetch settings of attribute mappings. For example, if an attribute is mapped to be eagerly fetched (set to FetchType.EAGER) it can set to be lazily fetched for a single execution of a query. This feature is
similar to what has been referred to by some as the ability to set a fetch plan, load plan, or fetch group.

The structure of an entity graph is fairly simple in that there are really only three types of objects that are contained within it.

* ***Entity graph nodes:*** There is exactly one entity graph node for every entity graph. It is the root of the entity graph and represents the root entity type. It contains all of the attribute nodes for
that type, plus all of the subgraph nodes for types associated with the root entity type.1
* ***Attribute nodes:*** Each attribute node represents an attribute in an entity or embeddable type. If it is for a basic attribute, then there will be no subgraph associated with it, but if it is for a relationship attribute, embeddable attribute, or element collection of embeddables, then it
may refer to a named subgraph.
* ***Subgraph nodes:*** A subgraph node is the equivalent of an entity graph node in that it contains attribute nodes and subgraph nodes, but represents a graph of attributes for an entity or
embeddable type that is not the root type.

### Entity Graph Annotations

#### Basic Attribute Graphs

Let’s use the domain model from Figure 8-1 as a starting point. In that model there is an Address entity with some basic attributes. We could create a simple named entity graph by annotating the Address entity, as shown in Listing 11-14. 

***Listing 11-14. Named Entity Graph with Basic Attributes***

	@Entity
	@NamedEntityGraph(
	attributeNodes={
		@NamedAttributeNode("street"),
		@NamedAttributeNode("city"),
		@NamedAttributeNode("state"),
		@NamedAttributeNode("zip")}
	)

	public class Address {

		@Id private long id;
		private String street;
		private String city;
		private String state;
		private String zip;
		// ...
	}

The entity graph is assigned the default name of “Address” and all of the attributes are explicitly included in the attributeNodes element, meaning that they should be fetched. There is actually a shortcut for this case by means of a special includeAllAttributes element in the @NamedEntityGraph annotation:

	@Entity
	@NamedEntityGraph(includeAllAttributes=true)
	public class Address { ... }

Since the Address entity contains only eager attributes anyway (all basic attributes default to being eager), we can make it even shorter:

	@Entity
	@NamedEntityGraph
	public class Address { ... }

Annotating the class without listing any attributes is a shorthand for defining a named entity graph that is composed of the default fetch graph for that entity. Putting the annotation on the class causes the named entity graph to be created and referenceable by name in a query.

#### Using Subgraphs

The Address entity was a rather simple entity, though, and didn’t even have any relationships. The Employee entity is more complicated and requires that we add in some subgraphs, as shown in Listing 11-15.

***Listing 11-15. Named Entity Graph with Subgraphs***
	
	@Entity
	@NamedEntityGraph(name="Employee.graph1",
		attributeNodes={
			@NamedAttributeNode("name"),
			@NamedAttributeNode("salary"),
			@NamedAttributeNode(value="address"),
			@NamedAttributeNode(value="phones", subgraph="phone"),
			@NamedAttributeNode(value="department" subgraph="dept")},
		subgraphs={
			@NamedSubgraph(name="phone",
				attributeNodes={
					@NamedAttributeNode("number"),
					@NamedAttributeNode("type")
				}
			),
			@NamedSubgraph(name="dept",
				attributeNodes={
					@NamedAttributeNode("name")
				}
			)
		}
	)
	public class Employee {

		@Id
		private int id;

		private String name;

		private long salary;

		@Temporal(TemporalType.DATE)
		private Date startDate;

		@OneToOne
		private Address address;

		@OneToMany(mappedBy="employee")
		private Collection<Phone> phones = new ArrayList<Phone>();

		@ManyToOne
		private Department department;

		@ManyToOne
		private Employee manager;

		@OneToMany(mappedBy="manager")
		private Collection<Employee> directs = new ArrayList<Employee>();

		@ManyToMany(mappedBy="employees")
		private Collection<Project> projects = new ArrayList<Project>();

		// ...
	}

#### Entity Graphs with Inheritance

***Listing 11-17. Named Entity Graph with Inheritance***

	@Entity
	@NamedEntityGraph(
		name="Employee.graph3",
		attributeNodes={
			@NamedAttributeNode("name"),
			@NamedAttributeNode(value="projects", subgraph="project")
		},
		subgraphs={
			@NamedSubgraph(
				name="project", type=Project.class,
				attributeNodes={
					@NamedAttributeNode("name")
				}
			),
			@NamedSubgraph(
				name="project", type=QualityProject.class,
				attributeNodes={
					@NamedAttributeNode("qaRating")
					}
				)
			}
		)		
	public class Employee { ... }

The attribute fetch state for the projects attribute is defined in a subgraph called “project”, but as you can see, there are two subgraphs named “project”. Each of the possible classes/subclasses that could be in that relationship has a subgraph named “project” and includes its defined state that should be fetched, plus a type element to identify which subclass it is. However, since DesignProject does not introduce any new state, we don’t need to include a subgraph named “project” for that class.

The inheritance case that this doesn’t cover is when the root entity class is itself a superclass. For this special case there is a subclassSubgraphs element to list the root entity subclasses. For example, if there was a ContractEmployee subclass of Employee that had an additional hourlyRate attribute that we wanted fetched, then we could use the entity graph shown in Listing 11-18.

***Listing 11-18. Named Entity Graph with Root Inheritance***

	@Entity
	@NamedEntityGraph(
		name="Employee.graph4",
		attributeNodes={
			@NamedAttributeNode("name"),
			@NamedAttributeNode("address"),
			@NamedAttributeNode(value="department", subgraph="dept")
		},
		subgraphs={
		@NamedSubgraph(name="dept",
			attributeNodes={
				@NamedAttributeNode("name")
			}
		)
	},
	subclassSubgraphs={
		@NamedSubgraph(
			name="notUsed", type=ContractEmployee.class,
			attributeNodes={
				@NamedAttributeNode("hourlyRate")
			}
		)
	})
	public class Employee { ... }

#### Map Key Subgraphs

To illustrate having a Map with an embeddable key subgraph, we will use an EmployeeName embeddable class similar to Listing 5-13, and modify our Department entity slightly to be similar to Listing 5-14. We list the type definitions in Listing 11-19 and add a named entity graph.

***Listing 11-19. Named Entity Graph with Map Key Subgraph***

	@Embeddable
	public class EmployeeName {
		private String firstName;
		private String lastName;
		// ...
	}

	@Entity
	@NamedEntityGraph(
		name="Department.graph1",
		attributeNodes={
			@NamedAttributeNode("name"),
			@NamedAttributeNode(
				value="employees",
				subgraph="emp",
				keySubgraph="empName")
		},
		subgraphs={
			@NamedSubgraph(
				name="emp",
				attributeNodes={
					@NamedAttributeNode(
						value="name",
						subgraph="empName"),
					@NamedAttributeNode("salary")
				}
			),
			@NamedSubgraph(
				name="empName",
				attributeNodes={
					@NamedAttributeNode("firstName"),
					@NamedAttributeNode("lastName")
				}
			)
		}
	)

	public class Department {
		
		@Id private int id;
		private String name;
		
		@OneToMany(mappedBy="department")
		@MapKey(name="name")
		private Map<EmployeeName, Employee> employees;
		// ...
	}

#### Entity Graph API

The way to get started building a new entity graph is to use the createEntityGraph() factory method on EntityManager. It takes the root entity class as a parameter and returns a new EntityGraph instance typed to the entity class:

	EntityGraph<Address> graph = em.createEntityGraph(Address.class);

The next step is to add attribute nodes to the entity graph. The adding methods are designed to do most of the work of creating the node structures for you. We can use the variable-arg addAttributeNodes() method to add the attributes that will not have subgraphs associated with them:

	graph.addAttributeNodes("street","city", "state", "zip");

This will create an AttributeNode object for each of the named attribute parameters and add it to the entity graph. There is unfortunately no method equivalent to the includeAllAttributes element in the @NamedEntityGraph annotation.

There are also strongly typed method equivalents to the ones that take string-based attribute names. The typed versions use the metamodel, so you would need to ensure that the metamodel has been generated for your domain model (see Chapter 9). A sample invocation of the strongly typed addAttributeNodes() method would be: 

	graph.addAttributeNodes(Address_.street, Address_.city, Address_.state, Address_.zip);

***Listing 11-20. Dynamic Entity Graph with Subgraphs***
	
	EntityGraph<Employee> graph = em.createEntityGraph(Employee.class);
	graph.addAttributeNodes("name","salary", "address");
	Subgraph<Phone> phone = graph.addSubgraph("phones");
	phone.addAttributeNodes("number", "type");
	Subgraph<Department> dept = graph.addSubgraph("department");
	dept.addAttributeNodes("name");

The example in Listing 11-16 illustrated having an entity graph containing a second definition of the root entity class, as well as having one subgraph refer to another. Listing 11-21 shows that the API actually suffers in this case because it doesn’t allow sharing of a subgraph within the same entity graph. Since there is no API to pass in an existing Subgraph instance, we need to construct two identical named employee subgraphs.

***Listing 11-21. Dynamic Entity Graph with Multiple Type Definitions***

EntityGraph<Employee> graph = em.createEntityGraph(Employee.class);
graph.addAttributeNodes("name","salary", "address");
Subgraph<Phone> phone = graph.addSubgraph("phones");
phone.addAttributeNodes("number", "type");
Subgraph<Employee> namedEmp = phone.addSubgraph("employee");
namedEmp.addAttributeNodes("name");
Subgraph<Department> dept = graph.addSubgraph("department");
dept.addAttributeNodes("name");
Subgraph<Employee> mgrNamedEmp = graph.addSubgraph("manager");
mgrNamedEmp.addAttributeNodes("name");
The inheritance example in Listing 11-17 can be translated into an API-based version. When a related class is
actually a class hierarchy, then each call to addSubgraph() can take the class as a parameter to distinguish between
the different subclasses, as shown in Listing 11-22.

***Listing 11-22. Dynamic Entity Graph with Inheritance***

	EntityGraph<Employee> graph = em.createEntityGraph(Employee.class);
	graph.addAttributeNodes("name","salary", "address");
	Subgraph<Project> project = graph.addSubgraph("projects", Project.class);
	project.addAttributeNodes("name");
	Subgraph<QualityProject> qaProject = graph.addSubgraph("projects", QualityProject.class);
	qaProject.addAttributeNodes("qaRating");

When inheritance exists at the root entity class, the addSubclassSubgraph() method should be used. The class is the only parameter that is required. The API version of the annotation in Listing 11-18 is shown in Listing 11-23.

***Listing 11-23. Dynamic Entity Graph with Root Inheritance***

	EntityGraph<Employee> graph = em.createEntityGraph(Employee.class);
	graph.addAttributeNodes("name","address");
	graph.addSubgraph("department").addAttributeNodes("name");
	graph.addSubclassSubgraph(ContractEmployee.class).addAttributeNodes("hourlyRate");

Note that in Listing 11-23 we make use of the fact that no further subgraphs are being added to the subgraphs connected to the entity graph node, so neither of the created subgraphs are saved in stack variables. Rather, the addAttributeNodes() method is invoked directly on each of the Subgraph results of the addSubgraph() and addSubclassSubgraph() methods. Our final example to convert is the Map example from Listing 11-19. The API equivalent is shown in Listing 11-24. The root entity is the Department entity and the Map is keyed by the EmployeeName embeddable.

***Listing 11-24. Dynamic Entity Graph with Map Key Subgraph***

	EntityGraph<Department> graph = em.createEntityGraph(Department.class);
	graph.addAttributeNodes("name");
	graph.addSubgraph("employees").addAttributeNodes("salary");
	graph.addKeySubgraph("employees").addAttributeNodes("firstName", "lastName");

In the above example, the addKeySubgraph() method was invoked on the root entity graph node but the same method also exists on Subgraph, so a key subgraph can be added at any level where a Map occurs.

### Managing Entity Graphs

#### Accessing Named Entity Graphs

	EntityGraph<?> empGraph2 = em.getEntityGraph("Employee.graph2");

If there are many entity graphs defined on a single class and we have a reason to sequence through them, we can do so using the class-based accessor method. The following code looks at the attribute names of the root entity classes for each of the Employee entity graphs:

	List<EntityGraph<? super Employee>> egList =
		em.getEntityGraphs(Employee.class);
	for (EntityGraph<? super Employee> graph : egList) {
		System.out.println("EntityGraph: " + graph.getName());
		List<AttributeNode<?>> attribs = graph.getAttributeNodes();
		for (AttributeNode<?> attr : attribs) {
			System.out.println(" Attribute: " + attr.getAttributeName());
		}
	}

#### Adding Named Entity Graphs

	em.getEntityManagerFactory().addNamedEntityGraph("Employee.graphX", graph);

#### Creating New Entity Graphs From Named Ones

***Listing 11-25. Creating an Entity Graph from an Existing Graph***
	
	EntityGraph<?> graph = em.createEntityGraph("Employee.graph2");
	graph.addAttributeNodes("projects");
	em.getEntityManagerFactory().addNamedEntityGraph("Employee.newGraph", graph);

### Using Entity Graphs

#### Fetch Graphs

Let’s look at an example of using a fetch graph. In Listing 11-16, we defined an entity graph for the Employee entity. Since it was defined as a named entity graph, we can access it using the getEntityGraph() method and use it as the value for our javax.persistence.fetchgraph property.

	Map<String,Object> props = new HashMap<String,Object>();
	props.put("javax.persistence.fetchgraph",
	em.getEntityGraph("Employee.graph2"));
	Employee emp = em.find(Employee.class, empId, props);

We can just as easily pass in the dynamic version that we created in Listing 11-21. If the dynamic graph was being created in the “graph” variable, then we could simply pass it in to the find() method, or as a query hint:

	EntityGraph<Employee> graph = em.createEntityGraph(Employee.class);
	// ... (compose graph as in Listing 11-21)
	TypedQuery<Employee> query = em.createQuery(
		"SELECT e FROM Employee e WHERE e.salary > 50000", Employee.class);
	query.setHint("javax.persistence.fetchgraph",graph);
	List<Employee> results = query.getResultList();

#### Load Graphs

***Listing 11-26. Triggering a Lazy Relationship***
	
	@Stateless
	public class EmployeeService {

		@PersistenceContext(unitName="EmployeeService")
		private EntityManager em;
		public List findAll() {
		EntityGraph<Employee> graph = em.createEntityGraph(Employee.class);
		graph.addAttributeNodes("department");
		TypedQuery<Employee> query = em.createQuery(
			"SELECT e FROM Employee e", Employee.class);
		query.setHint("javax.persistence.loadgraph", graph);
		return query.getResultList();
		}
		// ...
	}

#### Best Practices for Fetch and Load Graphs

/