# Chapitre 7 : Using Queries

JPA supports two methods for expressing queries to retrieve entities and other persistent data from the database: query languages and the Criteria API. 

The primary query language is **Java Persistence Query Language (JP QL)**, a database-independent query language that operates on the logical entity model as opposed to the physical data model. Queries may also be expressed in SQL to take advantage of the underlying database. 

We will explore using SQL queries with **JPA** in Chapter 11. The Criteria API provides an alternative method for constructing queries based on Java objects instead of query strings. Chapter 9 covers the Criteria API in detail.

## Java Persistence Query Language

JP QL significantly extends EJB QL. The following list describes some of the features available above and beyond EJB QL:* Single and multiple value result types* Aggregate functions, with sorting and grouping clauses
* A more natural join syntax, including support for both inner and outer joins* Conditional expressions involving subqueries* Update and delete queries for bulk data changes* Result projection into non-persistent classes

### Getting Started

	SELECT e FROM Employee e

The key difference between SQL and JP QL for this query is that instead of selecting from a table, an entity from the application domain model has been specified instead. This indicates that the result type of the query is the Employee entity, so executing this statement will result in a list of zero or more Employee instances.

Starting with an alias, we can navigate across entity relationships using the dot (**.**) operator. For example, if we want just the names of the employees, the following query will suffice:

	SELECT e.name FROM Employee e

We can also select an entity we didn’t even list in the FROM clause. 	SELECT e.department FROM Employee eAn employee has a many-to-one relationship with her department named department, so the result type of the query is the Department entity.

### Filtering Results

Just like SQL, JP QL supports the WHERE clause to set conditions on the data being returned. **The majority of operators commonly available in SQL are available in JP QL, including basic comparison operators; IN, LIKE, and BETWEEN expressions; numerous function expressions (such as SUBSTRING and LENGTH); and subqueries. The key difference for JP QL is that entity expressions and not column references are used.**

	SELECT e FROM Employee e WHERE e.department.name = 'NA42' AND e.address.state IN ('NY','CA')

### Projecting Results

Depending on how an entity is mapped to the database, this can be an expensive operation if much of the entity data is discarded. It would be useful to return only a subset of the properties from an entity.

	SELECT e.name, e.salary FROM Employee e Joins Between Entities

### Joins Between Entities

The result type of a select query cannot be a collection; it must be a single valued object such as an entity instance or persistent field type. **Expressions such as e.phones are illegal in the SELECT clause because they would result in Collection instances (each occurrence of e.phones is a collection, not an instance)**. Therefore, just as with SQL and tables, if we want to navigate along a collection association and return elements of that collection, we must join the two entities together. ***In the following code, a join between Employee and Phone entities is implicitly applied in order to retrieve all the cell phone numbers for a specific department:***

	SELECT p.number FROM Employee e, Phone p WHERE e = p.employee AND e.department.name = 'NA42' AND p.type = 'Cell'

The previous query can be rewritten to use the JOIN operator. Just as before, the alias p is of type Phone, only this time it refers to each of the phones in the e.phones collection:
	SELECT p.number FROM Employee e JOIN e.phones p WHERE  e.department.name = 'NA42' AND p.type = 'Cell'

### Aggregate Queries

The syntax for aggregate queries in JP QL is very similar to that of SQL. There are five supported aggregate functions **(AVG, COUNT, MIN, MAX, and SUM)**, and results may be grouped in the **GROUP BY** clause and filtered using the **HAVING** clause. Once again, the difference is the use of entity expressions when specifying the data to be aggregated. An aggregate JP QL query can use many of the aggregate functions within the same query:
	
	SELECT d, COUNT(e), MAX(e.salary), AVG(e.salary) FROM Department d JOIN d.employees e GROUP BY d HAVING COUNT(e) >= 5

### Query Parameters

	SELECT e FROM Employee e WHERE e.department = ?1 AND e.salary > ?2

	SELECT e FROM Employee e WHERE e.department = :dept AND e.salary > :base

## Defining Queries

### Dynamic Query Definition

	public class QueryService {	    @PersistenceContext(unitName="DynamicQueries")	    EntityManager em;	    public long queryEmpSalary(String deptName, String empName) {	        String query = "SELECT e.salary " +	                       "FROM Employee e " +	                       "WHERE e.department.name = '" + deptName +	                       "' AND " +	                       "      e.name = '" + empName + "'";	        return em.createQuery(query, Long.class).getSingleResult();	    }	}

Better with using paramets with a Dynamic Query

	public class QueryService {	    private static final String QUERY =	        "SELECT e.salary " +	        "FROM Employee e " +	        "WHERE e.department.name = :deptName AND " +	        "      e.name = :empName ";	    @PersistenceContext(unitName="QueriesUnit")	    EntityManager em;	    public long queryEmpSalary(String deptName, String empName) {	        return em.createQuery(QUERY, Long.class)	                 .setParameter("deptName", deptName)	                 .setParameter("empName", empName)	                 .getSingleResult();	} }

### Named Query Definition

Named queries are a powerful tool for organizing query definitions and improving application performance. A named query is defined using the @NamedQuery annotation, which may be placed on the class definition for any entity. 

	@NamedQuery(name="findSalaryForNameAndDepartment",            query="SELECT e.salary " +                  "FROM Employee e " +                  "WHERE e.department.name = :deptName AND " +                  "      e.name = :empName")


The garbage normally associated with repeated string concatenation will not apply here because the annotation will be processed only once at startup time and will be executed at runtime in query form.

**The name of the query is scoped to the entire persistence unit and must be unique within that scope.**

For example, the "findAll" query for the Employee entity would be named "Employee.findAll". It is undefined what should happen if two queries in the same persistence unit have the same name, but it is likely that either deployment of the application will fail or one will overwrite the other, leading to unpredictable results at runtime.

If more than one named query is to be defined on a class, they must be placed inside of a @NamedQueries annotation, which accepts an array of one or more @NamedQuery annotations. 
 	@NamedQueries({	    @NamedQuery(name="Employee.findAll",	                query="SELECT e FROM Employee e"),	    @NamedQuery(name="Employee.findByPrimaryKey",	                query="SELECT e FROM Employee e WHERE e.id = :id"),	    @NamedQuery(name="Employee.findByName",	                query="SELECT e FROM Employee e WHERE e.name = :name")	})


Because the query string is defined in the annotation, it cannot be altered by the application at runtime. This contributes to the performance of the application and helps to prevent the kind of security issues discussed in the previous section. Due to the static nature of the query string, any additional criteria required for the query must be specified using query parameters. 

	public class EmployeeService {	    @PersistenceContext(unitName="EmployeeService")	    EntityManager em;	    public Employee findEmployeeByName(String name) {	         return em.createNamedQuery("Employee.findByName",	}	// ... }

### Dynamic Named Queries

A dynamic query can be turned into a named query by using the EntityManagerFactory addNamedQuery() method.

	public class QueryService {
		    private static final String QUERY =	        "SELECT e.salary " +	        "FROM Employee e " +	        "WHERE e.department.name = :deptName AND " +	        "      e.name = :empName ";
		    @PersistenceContext(unitName="QueriesUnit")	    EntityManager em;
		    @PersistenceUnit(unitName="QueriesUnit")	    EntityManagerFactory emf;
		    @PostConstruct	    public void init() {	        TypedQuery<Long> q = em.createQuery(QUERY, Long.class);	        emf.addNamedQuery("findSalaryForNameAndDepartment", q);	    }
	
		public long queryEmpSalary(String deptName, String empName) {	    	return em.createNamedQuery("findSalaryForNameAndDepartment", Long.class)
				.setParameter("deptName", deptName)				.setParameter("empName", empName)				.getSingleResult();		}	// ... }

## Parameter Types

		@NamedQuery(name="findEmployeesAboveSal",            query="SELECT e " +                  "FROM Employee e " +                  "WHERE e.department = :dept AND " +                  "      e.salary > :sal")
This query highlights one of the nice features of JP QL in that entity types may be used as parameters. When the query is translated to SQL, the necessary primary key columns will be inserted into the conditional expression and paired with the primary key values from the parameter. It is not necessary to know how the primary key is mapped in order to write the query. Binding the parameters for this query is a simple case of passing in the required Department entity instance as well as a long representing the minimum salary value for the query. Listing 7-7 demonstrates how to bind the entity and primitive parameters required by this query.

public List<Employee> findEmployeesAboveSal(Department dept, long minSal) {    return em.createNamedQuery("findEmployeesAboveSal", Employee.class)                       .setParameter("dept", dept)                       .setParameter("sal", minSal)                       .getResultList();}

Binding Date Parameters : 
	public List<Employee> findEmployeesHiredDuringPeriod(Date start, Date end) {		return em.createQuery("SELECT e " +                   "FROM Employee e " +                   "WHERE e.startDate BETWEEN ?1 AND ?2",                   Employee.class)				.setParameter(1, start, TemporalType.DATE)				.setParameter(2, end, TemporalType.DATE)				.getResultList();


One thing to keep in mind with query parameters is that the same parameter can be used multiple times in the query string yet only needs to be bound once using the setParameter() method. For example, consider the following named query definition, where the "dept" parameter is used twice in the WHERE clause:

	@NamedQuery(name="findHighestPaidByDepartment",            query="SELECT e " +                  "FROM Employee e " +                  "WHERE e.department = :dept AND " +
					 "	e.salary= (SELECT MAX(e.salary) " +
					 "					FROM Employee e " +
					 "					WHERE e.department = :dept)")


To execute this query, the "dept" parameter needs to be set only once with setParameter(), as in the followingexample:
	public Employee findHighestPaidByDepartment(Department dept) {	    return em.createNamedQuery("findHighestPaidByDepartment", Employee.class)			.setParameter("dept", dept)			.getSingleResult();
	}

## Executing Queries

For queries that return values, the developer may choose to call either getSingleResult() if the query is expected to return a single result or getResultList() if more than one result may be returned.

The executeUpdate() method is used to invoke bulk update and delete queries.

The simplest form of query execution is via the getResultList() method. It returns a collection containingthe query results. If the query did not return any data, the collection is empty. The return type is specified as a List instead of a Collection in order to support queries that specify a sort order. If the query uses the ORDER BY clause to specify a sort order, the results will be put into the result list in the same order.

It demonstrates how a query might be used to generate a menu for a command-line application that displays the name of each employee working on a project as well as the name of the department that the employee is assigned to. The results are sorted by the name of the employee. Queries are unordered by default.

	public void displayProjectEmployees(String projectName) {	    List<Employee> result = em.createQuery(	                          "SELECT e " +	                          "FROM Project p JOIN p.employees e "+	                          "WHERE p.name = ?1 " +	                          "ORDER BY e.name",	                          Employee.class)	                    .setParameter(1, projectName)	                    .getResultList();
		int count = 0;	   for (Employee e : result) {	   		System.out.println(++count + ": " + e.getName() + ", " + e.getDepartment().getName());
		} 
	}

**Whereas getResultList() returns an empty collection when no results are available, getSingleResult() throws a NoResultException exception.**

**If multiple results are available after executing the query instead of the single expected result, getSingleResult() will throw a NonUniqueResultException exception.**

Query and TypedQuery objects may be reused as often as needed so long as the same persistence context that was used to create the query is still active. For transaction-scoped entity managers, this limits the lifetime of the Query or TypedQuery object to the life of the transaction. Other entity manager types may reuse them until the entity manager is closed or removed.
Listing 7-10 demonstrates caching a TypedQuery object instance on the bean class of a stateful session bean that uses an extended persistence context. Whenever the bean needs to find the list of employees who are currently not assigned to any project, it reuses the same unassignedQuery object that was initialized during PostConstruct.

	@Stateful	public class ProjectManager {
		    @PersistenceContext(unitName="EmployeeService",	                        type=PersistenceContextType.EXTENDED)	    EntityManager em;
		    TypedQuery<Employee> unassignedQuery;
		    @PostConstruct	    public void init() {	        unassignedQuery =	            em.createQuery("SELECT e " +
				                 "FROM Employee e " +									"WHERE e.projects IS EMPTY",									Employee.class);		}
	
		public List<Employee> findEmployeesWithoutProjects() {	        return unassignedQuery.getResultList();		}	// ... }

### Working with Query Results

There is a wide variety of results possible, depending on the makeup of the query. The following are just some of the types that may result from JP QL queries:* Basic types, such as String, the primitive types, and JDBC types* Entity types* An array of Object* User-defined types created from a constructor expression

**For developers used to JDBC, the most important thing to remember when using the Query and TypedQuery interfaces is that the results are not encapsulated in a JDBC ResultSet. The collection or single result corresponds directly to the result type of the query.**

#### Untyped Results

#### Optimizing Read-Only Queries

When the query results will not be modified, queries using transaction-scoped entity managers outside of a transaction can be more efficient than queries executed within a transaction when the result type is an entity. When query results are prepared within a transaction, the persistence provider has to take steps to convert the results
into managed entities. This usually entails taking a snapshot of the data for each entity in order to have a baseline to compare against when the transaction is committed. If the managed entities are never modified, the effort of converting the results into managed entities is wasted.
Outside of a transaction, in some circumstances the persistence provider may be able to optimize the case where the results will be detached immediately. Therefore it can avoid the overhead of creating the managed versions. Note that this technique does not work on application-managed or extended entity managers because their persistence context outlives the transaction. Any query result from this type of persistence context may be modified for later synchronization to the database even if there is no transaction.
When encapsulating query operations behind a bean with container-managed transactions, the easiest way to execute nontransactional queries is to use the NOT_SUPPORTED transaction attribute for the session bean method. This will cause any active transaction to be suspended, forcing the query results to be detached and enabling this optimization. Listing 7-11 shows an example of this technique using a stateless session bean.


	@Stateless	public class QueryService {	    @PersistenceContext(unitName="EmployeeService")	    EntityManager em;	    @TransactionAttribute(TransactionAttributeType.NOT_SUPPORTED)	    public List<Department> findAllDepartmentsDetached() {	        return em.createQuery("SELECT d FROM Department d",	                              Department.class)		}	// ... }

#### Special Result Types

Whenever a query involves **more than one expression** in the SELECT clause, **the result of the query will be a List of Object arrays**. Common examples include projection of entity fields and aggregate queries where grouping expressions or multiple functions are used. 

**Each element of the List is cast to an array of Object that is then used to extract the employee and department name information. We use an untyped query because the result has multiple elements in it.**

	public void displayProjectEmployees(String projectName) {
		List result = em.createQuery(
			"SELECT e.name, e.department.name " +
			"FROM Project p JOIN p.employees e " +
			"WHERE p.name = ?1 " +
			"ORDER BY e.name")
		.setParameter(1, projectName)
		.getResultList();

		int count = 0;
		for (Iterator i = result.iterator(); i.hasNext();) {
			Object[] values = (Object[]) i.next();
			System.out.println(++count + ": " +
			values[0] + ", " + values[1]);
		}
	}


Constructor expressions provide developers with a way to map an array of Object result types to custom objects. A constructor expression is defined in JP QL using the NEW operator in the SELECT clause. The argument to the NEW operator is the fully qualified name of the class that will be instantiated to hold the results for each row of data returned. The only requirement for this class is that it has a constructor with arguments matching the exact type and order that will be specified in the query. 

	package example;
	public class EmpMenu {
		private String employeeName;
		private String departmentName;
		public EmpMenu(String employeeName, String departmentName) {
			this.employeeName = employeeName;
			this.departmentName = departmentName;
		}
		public String getEmployeeName() { return employeeName; }
		public String getDepartmentName() { return departmentName; }
	}

	// ...

	public void displayProjectEmployees(String projectName) {
		
		List<EmpMenu> result =
			em.createQuery("SELECT NEW example.EmpMenu(" +
				"e.name, e.department.name) " +
				"FROM Project p JOIN p.employees e " +
				"WHERE p.name = ?1 " +
				"ORDER BY e.name",
			EmpMenu.class)
			.setParameter(1, projectName)
			.getResultList();

		int count = 0;
		for (EmpMenu menu : result) {
			System.out.println(++count + ": " +
			menu.getEmployeeName() + ", " +
			menu.getDepartmentName());
		}
	}

	// ...


### Query Paging

The Query and TypedQuery interfaces provide support for pagination via the setFirstResult() and
setMaxResults() methods. A persistence provider may choose to implement support for this feature in a number of different ways because not all databases benefit from the same approach. It’s a good
idea to become familiar with the way your vendor approaches pagination and what level of support exists in the target database platform for your application.

	@Stateful
	public class ResultPager {

		@PersistenceContext(unitName="QueryPaging")
		private EntityManager em;
	
		private String reportQueryName;
		private long currentPage;
		private long maxResults;
		private long pageSize;
	
		public long getPageSize() {
			return pageSize;
		}
	
		public long getMaxPages() {
			return maxResults / pageSize;
		}

		public void init(long pageSize, String countQueryName, String reportQueryName) {
			this.pageSize = pageSize;
			this.reportQueryName = reportQueryName;
			maxResults = em.createNamedQuery(countQueryName, Long.class).getSingleResult();
			currentPage = 0;
		}

		public List getCurrentResults() {
			return em.createNamedQuery(reportQueryName)
				.setFirstResult(currentPage * pageSize)
				.setMaxResults(pageSize)
				.getResultList();
		}
	
		public void next() {
			currentPage++;
		}

		public void previous() {
			currentPage--;
			if (currentPage < 0) {
				currentPage = 0;
			}
		}

		public long getCurrentPage() {
			return currentPage;
		}

		public void setCurrentPage(long currentPage) {
			this.currentPage = currentPage;
		}

		@Remove
		public void finished() {}
	}

### Queries and Uncommitted Changes Executing queries

To put this issue in context, consider a message board application, which has modeled conversation topics as **Conversation entities**. Each Conversation entity refers to one or more messages represented by a Message entity. Periodically, conversations are archived when the last message added to the conversation is more than 30 days old. This is accomplished by changing the status of the Conversation entity from ACTIVE to INACTIVE. The two queries to obtain the list of active conversations and the last message date for a given conversation are shown in Listing 7-16.

	@NamedQueries({
		@NamedQuery(name="findActiveConversations",
			query="SELECT c " +
					"FROM Conversation c " +
					"WHERE c.status = 'ACTIVE'"),
		@NamedQuery(name="findLastMessageDate",
			query="SELECT MAX(m.postingDate) " +
					"FROM Conversation c JOIN c.messages m " +
					"WHERE c = :conversation")
	})

The "findActiveConversations" query collects all the active conversations, while the "findLastMessageDate" returns the last date that a message was added to a Conversation entity. 

As the code iterates over the Conversation entities, it invokes the "findLastMessageDate" query for each one. As these two queries are related, it is reasonable for a persistence provider to assume that the results of the "findLastMessageDate" query will depend on the changes being made to the Conversation entities. **If the provider ensures the integrity of the "findLastMessageDate" query by flushing the persistence context, this could become a very expensive operation if hundreds of active conversations are being checked.**

	public void archiveConversations(Date minAge) {
		List<Conversation> active =
			em.createNamedQuery("findActiveConversations",Conversation.class)
				.getResultList();
		
		TypedQuery<Date> maxAge =
			em.createNamedQuery("findLastMessageDate", Date.class);
		
		for (Conversation c : active) {
			maxAge.setParameter("conversation", c);
			Date lastMessageDate = maxAge.getSingleResult();
			if (lastMessageDate.before(minAge)) {
				c.setStatus("INACTIVE");
			}
		}
	}

To offer more control over the integrity requirements of queries, the EntityManager and Query interfaces support a **setFlushMode()** method to set the flush mode, an indicator to the provider how it should handle pending changes and queries. There are two possible flush mode settings, **AUTO** and **COMMI**T, which are defined by the **FlushModeType** enumerated type. **The default setting is AUTO**, which means that** the provider should ensure that pending transactional changes are included in query results.** If a query might overlap with changed data in the persistence
context, this setting will ensure that the results are correct. The current flush mode setting may be retrieved via the **getFlushMode()** method.

The **COMMIT** flush mode tells the provider that queries don’t overlap with changed data in the persistence context, so it does not need to do anything in order to get correct results. Depending on how the provider implements its query integrity support, this might mean that it does not have to flush the persistence context before executing a query because you have indicated that there is no changed data in memory that would affect the results of the database query.

### Query Timeouts

Generally speaking, when a query executes it will block until the database query returns. In addition to the obvious concern about runaway queries and application responsiveness, it may also be a problem if the query is participating in a transaction and a timeout has been set on the JTA transaction or on the database. The timeout on the transaction or database may cause the query to abort early, but it will also cause the transaction to roll back, preventing any further work in the same transaction. 

If an application needs to set a limit on query response time without using a transaction or causing a transaction rollback, the javax.persistence.query.timeout property may be set on the query or as part of the persistence unit. This property defines the number of milliseconds that the query should be allowed to run before it is aborted.

This example uses the query hint mechanism, which we will discuss in more detail later in the “Query Hints” section. Setting properties on the persistence unit is covered in Chapter 14.

	public Date getLastUserActivity() {
		
		TypedQuery<Date> lastActive =
			em.createNamedQuery("findLastUserActivity", Date.class);
		
		lastActive.setHint("javax.persistence.query.timeout", 5000);
		
		try {
			return lastActive.getSingleResult();
		} catch (QueryTimeoutException e) {
			return null;
		}
	}

Unfortunately, setting a query timeout is not portable behavior. It may not be supported by all database platforms nor is it a requirement to be supported by all persistence providers. Therefore, applications that want to enable query timeouts must be prepared for three scenarios. The first is that the property is silently ignored and has no effect. The second is that the property is enabled and any select, update, or delete operation that runs longer than the specified timeout value is aborted, and a QueryTimeoutException is thrown. This exception may be handled and will not cause
any active transaction to be marked for rollback.

## Bulk Update and Delete

### Using Bulk Update and Delete

Bulk update of entities is accomplished with the UPDATE statement. This statement operates on a single entity type and sets one or more single-valued properties of the entity (either a state field or a single-valued association) subject to the conditions in the WHERE clause. It can also be used on embedded state in one or more embeddable objects referenced by the entity. In terms of syntax, it is nearly identical to the SQL version with the exception of using entity expressions instead of tables and columns. Listing 7-19 demonstrates using a bulk UPDATE statement. Note that the use of the REQUIRES_NEW transaction attribute type is significant and will be discussed following the examples.

	@Stateless
	public class EmployeeService {
		
		@PersistenceContext(unitName="BulkQueries")
		EntityManager em;
		
		@TransactionAttribute(TransactionAttributeType.REQUIRES_NEW)
		public void assignManager(Department dept, Employee manager) {
			em.createQuery("UPDATE Employee e " +
				"SET e.manager = ?1 " +
				"WHERE e.department = ?2")
				.setParameter(1, manager)
				.setParameter(2, dept)
				.executeUpdate();
		}
	}

Bulk removal of entities is accomplished with the DELETE statement. Again, the syntax is the same as the SQL version, except that the target in the FROM clause is an entity instead of a table, and the WHERE clause is composed of entity expressions instead of column expressions. 

	@Stateless
	public class ProjectService {
		
		@PersistenceContext(unitName="BulkQueries")
		EntityManager em;
		
		@TransactionAttribute(TransactionAttributeType.REQUIRES_NEW)
		public void removeEmptyProjects() {
			em.createQuery("DELETE FROM Project p " +
				"WHERE p.employees IS EMPTY")
				.executeUpdate();
		}
	}

**The first issue to consider when using these statements is that the persistence context is not updated to reflect the results of the operation. Bulk operations are issued as SQL against the database, bypassing the in-memory structures of the persistence context.** The developer can rely only on entities retrieved after the bulk operation completes.

When using transaction-scoped persistence contexts the bulk operation should either execute in a transaction all by itself or be the first operation in the transaction. Running the bulk operation in its own transaction is the preferred approach because it minimizes the chance of accidentally fetching data before the bulk change occurs.

Executing the bulk operation and then working with entities after it completes is also safe because then any find() operation or query will go to the database to get current results. The examples in Listing 7-19 and Listing 7-20 used the REQUIRES_NEW transaction attribute to ensure that the bulk operations occurred within their own transactions.

A typical strategy for persistence providers dealing with bulk operations is to invalidate any in-memory cache of data related to the target entity. This forces data to be fetched from the database the next time it is required. How much cached data gets invalidated depends on the sophistication of the persistence provider. If the provider can detect that the update impacts only a small range of entities, those specific entities may be invalidated, leaving other cached data in place. Such optimizations are limited, however, and if the provider cannot be sure of the scope of the change, the entire cache must be invalidated. This can have an impact on the performance of the application if bulk changes are a frequent occurrence.

### Bulk Delete and Relationships

	DELETE FROM Department d WHERE d.name IN ('CA13', 'CA19', 'NY30')

=> a PersistenceException exception is thrown, reporting that a foreign key integrity
constraint has been violated.

	UPDATE Employee e SET e.department = null WHERE e.department.name IN ('CA13', 'CA19', 'NY30')

and then
	
	DELETE FROM Department d WHERE d.name IN ('CA13', 'CA19', 'NY30')

## Query Hints

	public Employee findEmployeeNoCache(int empId) {
		TypedQuery<Employee> q = em.createQuery(
			"SELECT e FROM Employee e WHERE e.id = :empId", Employee.class);
		
		// force read from database
		q.setHint("eclipselink.cache-usage", "DoNotCheckCache");
		q.setParameter("empId", empId);
		
		try {
			return q.getSingleResult();
		} catch (NoResultException e) {
			return null;
		}
	}

If this query were to be executed frequently, a named query would be more efficient. The following named query definition incorporates the cache hint used earlier:
	
	@NamedQuery(name="findEmployeeNoCache",
				query="SELECT e FROM Employee e WHERE e.id = :empId",
				hints={@QueryHint(name="eclipselink.cache-usage",
				value="DoNotCheckCache")})

## Query Best Practices

### Named Queries

First and foremost, we recommend named queries whenever possible. Persistence providers will often take steps to precompile JP QL named queries to SQL as part of the deployment or initialization phase of an application. This avoids the overhead of continuously parsing JP QL and generating SQL. Even with a cache for converted queries, dynamic query definition will always be less efficient than using named queries.

As we discussed in the “Dynamic Query Definition” section, query parameters also help to avoid security issues caused by concatenating values into query strings.

### Report Queries

If you are executing queries that return entities for reporting purposes and have no intention of modifying the results, consider executing queries using a transaction-scoped entity manager but outside of a transaction. The persistence provider may be able to detect the lack of a transaction and optimize the results for detachment, often by skipping some of the steps required to create an interim managed version of the entity results.

### Vendor Hints

It is likely that vendors will entice you with a variety of hints to enable different performance optimizations for queries. Hints are often highly dependent on the target platform and may well have
to be changed over time as different aspects of the application impact the overall balance of performance. Keep hints decoupled from your code if at all possible.

### Stateless Beans

* Clients can execute queries by invoking an a • ppropriately named business method instead of
relying on a cryptic query name or multiple copies of the same query string.
* Bean methods can optimize their transaction usage depending on whether or not the results
need to be managed or detached.
* Using a transaction-scoped persistence context ensures that large numbers of entity instances
don’t remain managed long after they are needed.

### Bulk Update and Delete

If bulk update and delete operations must be used, ensure that they are executed only in an isolated transaction where no other changes are being made. There are many ways in which these queries can negatively impact an active persistence context. Interweaving these queries with other non-bulk operations requires careful management by the application.

Entity versioning and locking requires special consideration when bulk update operations are used. Bulk delete operations can have wide ranging ramifications depending on how well the persistence provider can react and adjust entity caching in response. Therefore, we view bulk update and delete operations as being highly specialized, to be used with care.

### Provider Differences

Become familiar with the approach used by your provider and whether or not it is configurable.

