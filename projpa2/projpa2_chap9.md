# Criteria API

## Overview

### The Criteria API

Let’s begin with a simple example to demonstrate the syntax and usage of the Criteria API. The following JP QL query returns all the employees in the company with the name of “John Smith:”

	SELECT e
	FROM Employee e
	WHERE e.name = 'John Smith'

And here is the equivalent query constructed using the Criteria API:

	CriteriaBuilder cb = em.getCriteriaBuilder();
	CriteriaQuery<Employee> c = cb.createQuery(Employee.class);
	Root<Employee> emp = c.from(Employee.class);
	c.select(emp).where(cb.equal(emp.get("name"), "John Smith"));

**The JP QL keywords SELECT, FROM, WHERE and LIKE have matching methods in the form of select(), from(), where(), and like().**

### Parameterized Types

The API uses parameterized types extensively: almost every interface and method declaration uses Java generics in one form or another. Generics allow the compiler to detect many cases of incompatible type usage and, like the Java collection API, removes the need for casting in most cases.

This code is functionally identical to the original example, but just happens to be more prone to errors during development.

	CriteriaBuilder cb = em.getCriteriaBuilder();
	CriteriaQuery c = cb.createQuery(Employee.class);
	Root emp = c.from(Employee.class);
	c.select(emp).where(cb.equal(emp.get(“name”), "John Smith"));

### Dynamic Queries

To demonstrate a good potential use of the Criteria API, we will look at a common use case in many enterprise applications: crafting dynamic queries where the structure of the criteria is not known until runtime.

Consider a web application used to search for employee contact information. This is a common feature in many large organizations that allows users to search by name, department, phone number, or even location, either separately or using a combination of query terms.

Employee Search Using Dynamic JP QL Query :

	@Stateless
	public class SearchService {
		
		@PersistenceContext(unitName="EmployeeHR")
		EntityManager em;
		
		public List<Employee> findEmployees(String name, String deptName,
			String projectName, String city) {
		
			StringBuffer query = new StringBuffer();
			query.append("SELECT DISTINCT e ");
			query.append("FROM Employee e LEFT JOIN e.projects p ");
			query.append("WHERE ");
		
			List<String> criteria = new ArrayList<String>();
			
			if (name != null) { criteria.add("e.name = :name"); }
			if (deptName != null) { criteria.add("e.dept.name = :dept"); }
			if (projectName != null) { criteria.add("p.name = :project"); }
			if (city != null) { criteria.add("e.address.city = :city"); }
			if (criteria.size() == 0) {
				throw new RuntimeException("no criteria");
			}

			for (int i = 0; i < criteria.size(); i++) {
				if (i > 0) { query.append(" AND "); }
				query.append(criteria.get(i));
			}
		
			Query q = em.createQuery(query.toString());
			if (name != null) { q.setParameter("name", name); }
			if (deptName != null) { q.setParameter("dept", deptName); }
			if (projectName != null) { q.setParameter("project", projectName); }
			if (city != null) { q.setParameter("city", city); }

			return (List<Employee>)q.getResultList();
		}
	}

Employee Search Using Criteria API :

	@Stateless
	public class SearchService {
		
		@PersistenceContext(unitName="EmployeeHR")
		EntityManager em;
		
		public List<Employee> findEmployees(String name, String deptName, String projectName, String city) {
		
		CriteriaBuilder cb = em.getCriteriaBuilder();
		CriteriaQuery<Employee> c = cb.createQuery(Employee.class);
		Root<Employee> emp = c.from(Employee.class);
		c.select(emp);
		c.distinct(true);
		Join<Employee,Project> project = emp.join("projects", JoinType.LEFT);

		List<Predicate> criteria = new ArrayList<Predicate>();
		
		if (name != null) {
			ParameterExpression<String> p =	cb.parameter(String.class, "name");
			criteria.add(cb.equal(emp.get("name"), p));
		}

		if (deptName != null) {
			ParameterExpression<String> p = cb.parameter(String.class, "dept");
			criteria.add(cb.equal(emp.get("dept").get("name"), p));
		}

		if (projectName != null) {
			ParameterExpression<String> p = cb.parameter(String.class, "project");
			criteria.add(cb.equal(project.get("name"), p));
		}
		
		if (city != null) {
			ParameterExpression<String> p =	cb.parameter(String.class, "city");
			criteria.add(cb.equal(emp.get("address").get("city"), p));
		}

		if (criteria.size() == 0) {
			throw new RuntimeException("no criteria");
		} else if (criteria.size() == 1) {
			c.where(criteria.get(0));
		} else {
			c.where(cb.and(criteria.toArray(new Predicate[0])));
		}

		TypedQuery<Employee> q = em.createQuery(c);

		if (name != null) { q.setParameter("name", name); }
		if (deptName != null) { q.setParameter("dept", deptName); }
		if (project != null) { q.setParameter("project", projectName); }
		if (city != null) { q.setParameter("city", city); }

		return q.getResultList();
		}
	}

Unfortunately, the designers of the Collection.toArray() method decided that, in order
to avoid casting the return type, an array to be populated should also be passed in as an argument or an empty array in the case where we want the collection to create the array for us. The syntax in the example is shorthand for the following code:

	Predicate[] p = new Predicate[criteria.size()];
	p = criteria.toArray(p);
	c.where(cb.and(p));

The TypedQuery interface is used to obtain strongly typed query results. Query definitions
created with the Criteria API have their result type bound using Java generics and therefore always yield a TypedQuery object from the createQuery() method of the EntityManager interface.

## Building Criteria API Queries

### Creating a Query Definition

The CriteriaBuilder interface provides three methods for creating a new select query definition, depending on the desired result type of the query. The first and most common method is the createQuery(Class<T>) method, passing in the class corresponding to the result of the query. This is the approach we used in Listing 9-2. The second method is createQuery(), without any parameters, and corresponds to a query with a result type of Object. The third method, createTupleQuery(), is used for projection or report queries where the SELECT clause of the query contains more than one expression and you wish to work with the result in a more strongly typed manner. It is really just a
convenience method that is equivalent to invoking createQuery(Tuple.class). Note that Tuple is an interface that contains an assortment of objects or data and applies typing to the aggregate parts. It can be used whenever multiple items are returned and you want to combine them into a single typed object.

### Basic Structure

***Voir résumé maison***

### Criteria Objects and Mutability

The first issue we need to consider is one of mutability. The majority of objects created through the Criteria API are in fact immutable. There are no setter methods or mutating methods on these interfaces. Almost all of the objects created from the methods on the CriteriaBuilder interface fall into this category.

The use of immutable objects means that the arguments passed into the CriteriaBuilder methods are rich in detail. All relevant information must be passed in so that the object can be complete at the time of its creation. The advantage of this approach is that it facilitates chained invocations of methods. Because no mutating methods have to be invoked on the objects returned from the methods used to build expressions, control can immediately continue to the next component in the expression.

Only the CriteriaBuilder methods that create query definition objects produce truly mutable results.
The CriteriaQuery and Subquery objects are intended to be modified many times by invoking methods such as select(), from(), and where(). But even here care must be taken as invoking methods twice can have one of two different effects. In most cases, invoking a method twice replaces the contents within the object with the argument being supplied. For example, invoking select() twice with two different arguments results in only the argument from the second invocation actually remaining as part of the query definition.

In some cases, however, invoking a method twice is in fact addition. Invoking from() twice with different arguments results in multiple query roots being added to the query definition. While we refer to these cases in the sections where they are described, you should be familiar with the Javadoc comments on the Criteria API as they also call out this behavior.

The second issue is the presence of getter methods on Criteria API objects. These behave as expected, returning information about the component of the query that each object represents. But it is worth noting that such methods are primarily of interest to tool developers who wish to work with query definitions in a truly generic way. In the vast majority of cases, and those that we demonstrate in this chapter, you will not have to make use of the getter methods in the construction of your criteria queries.

### Query Roots and Path Expressions

#### Query Roots

The AbstractQuery interface (parent of CriteriaQuery) provides the from() method to define the abstract schema type that will form the basis for the query. This method accepts an entity type as a parameter and adds a new root to the query. A root in a criteria query corresponds to an identification variable in JP QL, which in turn corresponds to a range variable declaration or join expression. Earlier, we used the following code to obtain our query root:

	CriteriaQuery<Employee> c = cb.createQuery(Employee.class);
	Root<Employee> emp = c.from(Employee.class);

The from() method returns an instance of Root corresponding to the entity type. The Root interface is itself extended from the From interface, which exposes functionality for joins. The From interface extends Path, which further extends Expression and then Selection, allowing the root to be used in other parts of the query definition. The role of each of these interfaces will be described in later sections. Calls to the from() method are additive. Each call adds another root to the query, resulting in a Cartesian product when more than one root is defined if no further constraints are applied in the WHERE clause. The following example from Chapter 8 demonstrates multiple query
roots, replacing a conventional join with the more traditional SQL approach:

	SELECT DISTINCT d
	FROM Department d, Employee e
	WHERE d = e.department

To convert this query to the Criteria API we need to invoke from() twice, adding both the Department and Employee entities as query roots. The following example demonstrates this approach:

	CriteriaQuery<Department> c = cb.createQuery(Department.class);
	Root<Department> dept = c.from(Department.class);
	Root<Employee> emp = c.from(Employee.class);
	c.select(dept)
		.distinct(true)
		.where(cb.equal(dept, emp.get("department")));

#### Path Expressions

	emp.get("address").get("city")

=>

	CriteriaQuery<Employee> c = cb.createQuery(Employee.class);
	Root<Employee> emp = c.from(Employee.class);
	c.select(emp).where(cb.equal(emp.get("address").get("city"), "New York"));

### The SELECT Clause

#### Selecting Single Expressions

Thus far, we have been passing in a query root to the select() method, therefore indicating that we want the entity to be the result of the query. We could also supply a single-valued expression such as selecting an attribute from an entity or any compatible scalar expression. The following example demonstrates this approach by selecting the name attribute of the Employee entity:

	CriteriaQuery<String> c = cb.createQuery(String.class);
	Root<Employee> emp = c.from(Employee.class);
	c.select(emp.<String>get("name"));

This query will return all employee names, including any duplicates. Duplicate results from a query may be removed by invoking distinct(true) from the AbstractQuery interface. This is identical in behavior to the DISTINCT keyword in a JP QL query.

Also note the unusual syntax we used to declare that the “name” attribute was of type String. The type of theexpression provided to the select() method must be compatible with the result type used to create the CriteriaQuery object. For example, if the CriteriaQuery object was created by invoking createQuery(Project.class) on the CriteriaBuilder interface, then it will be an error to attempt to set an expression resolving to the Employee entity using the select() method. When a method call such as select() uses generic typing in order to enforce a compatibility constraint, the type may be prefixed to the method name in order to qualify it in cases where the type could not otherwise be automatically determined. We need to use that approach in this case because the select() method has
been declared as follows: 

	CriteriaQuery<T> select(Selection<? extends T> selection);

The argument to select() must be a type that is compatible with the result type of the query definition. The get() method returns a Path object, but that Path object is always of type Path<Object> because the compiler cannot infer the correct type based on the attribute name. To declare that the attribute is really of type String, we need to qualify the method invocation accordingly. This syntax has to be used whenever the Path is being passed as an argument for which
the parameter has been strongly typed, such as the argument to the select() method and certain CriteriaBuilder expression methods. We have not had to use them so far in our examples because we have been using them in methods like equal(), where the parameter was declared to be of type Expression<?>. Because the type is wildcarded, it is valid to pass in an argument of type Path<Object>. Later in the chapter, we will look at the strongly typed versions of the Criteria API methods that remove this requirement.

#### Selecting Multiple Expressions

When defining a SELECT clause that involves more than one expression, the Criteria API approach required depends on how the query definition was created. If the result type is Tuple, then a CompoundSelection<Tuple> object must be passed to select(). If the result type is a non-persistent class that will be created using a constructor expression, then the argument must be a CompoundSelection<[T]> object, where [T] is the class type of the non-persistent class. Finally, if the result type is an array of objects, then a CompoundSelection<Object[]> object must be provided.
These objects are created with the tuple(), construct() and array() methods of the CriteriaBuilder interface, respectively. The following example demonstrates how to provide multiple expressions to a Tuple query:

	CriteriaQuery<Tuple> c= cb.createTupleQuery();
	Root<Employee> emp = c.from(Employee.class);
	c.select(cb.tuple(emp.get("id"), emp.get("name")));

As a convenience, the multiselect() method of the CriteriaQuery interface may also be used to set the SELECT clause. The multiselect() method will create the appropriate argument type given the result type of the query. This can take three forms depending on how the query definition was created. 

The first form is for queries that have Object or Object[] as their result type. The list of expressions that make up each result are simply passed to the multiselect() method.
	
	CriteriaQuery<Object[]> c = cb.createQuery(Object[].class);
	Root<Employee> emp = c.from(Employee.class);
	c.multiselect(emp.get("id"), emp.get("name"));

The third and final form is for queries with constructor expressions that result in non-persistent types. The multiselect() method is again invoked with a list of expressions, but it uses the type of the query to figure out and automatically create the appropriate constructor expression, in this case a data transfer object of type EmployeeInfo.

	CriteriaQuery<EmployeeInfo> c = cb.createQuery(EmployeeInfo.class);
	Root<Employee> emp = c.from(Employee.class);
	c.multiselect(emp.get("id"), emp.get("name"));

This is equivalent to the following:

	CriteriaQuery<EmployeeInfo> c = cb.createQuery(EmployeeInfo.class);
	Root<Employee> emp = c.from(Employee.class);
	c.select(cb.construct(EmployeeInfo.class,
	emp.get("id"),
	emp.get("name")));

As convenient as the multiselect() method is for constructor expressions, there are still cases where you will need to use the construct() method from the CriteriaBuilder interface. For example, if the result type of the query is Object[] and it also includes a constructor expression for only part of the results, the following would be required:

	CriteriaQuery<Object[]> c = cb.createQuery(Object[].class);
	Root<Employee> emp = c.from(Employee.class);
	c.multiselect( emp.get("id"),cb.construct( EmployeeInfo.class, emp.get("id"),emp.get("name")) );

#### Using Aliases

	CriteriaQuery<Tuple> c= cb.createTupleQuery();
	Root<Employee> emp = c.from(Employee.class);
	c.multiselect(emp.get("id").alias("id"), emp.get("name").alias("fullName"));

The alias() method is an exception to the rule that only the query definition interfaces, CriteriaQuery and Subquery, contain mutating operations. Invoking alias() changes the original Selection object and returns it from the method invocation.

	TypedQuery<Tuple> q = em.createQuery(c);
	for (Tuple t : q.getResultList()) {
		String id = t.get("id", String.class);
		String fullName = t.get("fullName", String.class);
		// ...
	}

### The FROM Clause

#### Inner and Outer Joins

Join expressions are created using the join() method of the From interface, which is extended both by Root, which we covered earlier, and Join, which is the object type returned by creating join expressions. This means that any query root may join, and that joins may chain with one another. The join() method requires a path expression argument and optionally an argument to specify the type of join, JoinType.INNER or JoinType.LEFT, for inner and outer joins respectively.

The join will have two parameterized types: the type of the source and the type of the target. This maintains the type safety on both sides of the join, and makes it clear what types are being joined.

	Join<Employee,Project> project = emp.join("projects", JoinType.LEFT);

Had the JoinType.LEFT argument been omitted, the join type would have defaulted to be an inner join. Just as in JP QL, multiple joins may be associated with the same From instance. For example, to navigate across the directs relationship of Employee and then to both the Department and Project entities would require the following, which assumes inner joining:

	Join<Employee,Employee> directs = emp.join("directs");
	Join<Employee,Project> projects = directs.join("projects");
	Join<Employee,Department> dept = directs.join("dept");

Joins may also be cascaded in a single statement. The resulting join will be typed by the source and target of the last join in the statement:

	Join<Employee,Project> project = dept.join("employees").join("projects");

Joins across collection relationships that use Map are a special case. JP QL uses the KEY and VALUE keywords to extract the key or value of a Map element for use in other parts of the query. In the Criteria API, these operators are handled by the key() and value() methods of the MapJoin interface. Consider the following example assuming a Map join across the phones relationship of the Employee entity:

	SELECT e.name, KEY(p), VALUE(p) FROM Employee e JOIN e.phones p

To create this query using the Criteria API, we need to capture the result of the join as a MapJoin, in this case using the joinMap() method. The MapJoin object has three type parameters: the source type, key type, and value type. It can look a little more daunting, but makes it explicit what types are involved. 

	CriteriaQuery<Object> c = cb.createQuery();
	Root<Employee> emp = c.from(Employee.class);
	MapJoin<Employee,String,Phone> phone = emp.joinMap("phones");
	c.multiselect(emp.get("name"), phone.key(), phone.value());

We need to use the joinMap() method in this case because there is no way to overload the join() method to return a Join object or MapJoin object when all we are passing in is the name of the attribute. Collection, Set, and List relationships are likewise handled with the joinCollection(), joinSet(), and joinList() methods for those cases where a specific join interface must be used. The strongly typed version of the join() method, which we will demonstrate later, is able to handle all join types though the single join() call.

#### Fetch Joins

***A "fetch" join allows associations or collections of values to be initialized along with their parent objects using a single select. This is particularly useful in the case of a collection. It effectively overrides the outer join and lazy declarations of the mapping file for associations and collections.***

As with JP QL, the Criteria API supports the fetch join, a query construct that allows data to be prefetched into the persistence context as a side effect of a query that returns a different, but related, entity. The Criteria API builds fetch joins through the use of the fetch() method on the FetchParent interface. It is used instead of join() in cases where fetch semantics are required and accepts the same argument types. Consider the following example we used in the previous chapter to demonstrate fetch joins of single-valued relationships:

	SELECT e FROM Employee e JOIN FETCH e.address

To re-create this query with the Criteria API, we use the fetch() method.

	CriteriaQuery<Employee> c = cb.createQuery(Employee.class);
	Root<Employee> emp = c.from(Employee.class);
	emp.fetch("address");
	c.select(emp);

Note that when using the fetch() method the return type is Fetch, not Join. **Fetch objects are not paths and may not be extended or referenced anywhere else in the query. Collection-valued fetch joins are also supported and use similar syntax**. In the following example, we demonstrate how to fetch the Phone entities associated with each Employee, using an outer join to prevent Employee
entities from being skipped if they don’t have any associated Phone entities. We use the **distinct()** setting to remove any duplicates.

	CriteriaQuery<Employee> c = cb.createQuery(Employee.class);
	Root<Employee> emp = c.from(Employee.class);
	emp.fetch("phones", JoinType.LEFT);
	c.select(emp).distinct(true);

### The WHERE Clause

As you saw in Table 9-1 and in several examples, the WHERE clause of a query in the Criteria API is set through the where() method of the AbstractQuery interface. The where() method accepts either zero or more Predicate objects, or a single Expression<Boolean> argument. Each call to where() will render any previously set WHERE expressions to be discarded and replaced with the newly passed-in ones.

### Building Expressions

***Table 9-2. JP QL to CriteriaBuilder Predicate Mapping***

	JP QL Operator 	 |	CriteriaBuilder Method
	------------------------------------------
	AND 				and()
	OR 					or()
	NOT 				not()
	= 					equal()
	<> 					notEqual()
	>                   greaterThan(), gt()
	>=                  greaterThanOrEqualTo(), ge()
	<                   lessThan(), lt()
	<=                  lessThanOrEqualTo(), le()
	BETWEEN             between()
	IS NULL             isNull()
	IS NOT NULL         isNotNull()
	EXISTS              exists()
	NOT EXISTS          not(exists())
	IS EMPTY            isEmpty()
	IS NOT EMPTY 		isNotEmpty()
	MEMBER OF 			isMember()
	NOT MEMBER OF 		isNotMember()
	LIKE 				like()
	NOT LIKE 			notLike()
	IN 					in()
	NOT IN 				not(in())

***Table 9-3. JP QL to CriteriaBuilder Scalar Expression Mapping***

	JP QL Expression 	 |	CriteriaBuilder Method
	----------------------------------------------
	ALL 					all()
	ANY 					any()
	SOME 					some()
	-                       neg(), diff()
    +                       sum()
    *                       prod()
    /                       quot()
    COALESCE                coalesce()
    NULLIF                  nullif()
    CASE                    selectCase()

***Table 9-4. JP QL to CriteriaBuilder Function Mapping***

	JP QL Function 	 |	CriteriaBuilder Method
	------------------------------------------
	ABS 				abs()
	CONCAT 				concat()
	CURRENT_DATE 		currentDate()
	CURRENT_TIME 		currentTime()
	CURRENT_TIMESTAMP 	currentTimestamp()
	LENGTH 				length()
	LOCATE 				locate()
	LOWER 				lower()
	MOD 				mod()
	SIZE 				size()
	SQRT 				sqrt()
	SUBSTRING 			substring()
	UPPER 				upper()
	TRIM 				trim()

***Table 9-5. JP QL to CriteriaBuilder Aggregate Function Mapping***

	JP QL Aggregate Function 	 |	CriteriaBuilder Method
	------------------------------------------------------
	AVG 							avg()
	SUM 							sum(), sumAsLong(), sumAsDouble()
	MIN 							min(), least()
	MAX 							max(), greatest()
	COUNT 							count()
	COUNT DISTINCT 					countDistinct()

#### Predicates

****Listing 9-3. Predicate Construction Using Conjunction****

	Predicate criteria = cb.conjunction();
	if (name != null) {
		ParameterExpression<String> p = cb.parameter(String.class, "name");
		criteria = cb.and(criteria, cb.equal(emp.get("name"), p));
	}
	if (deptName != null) {
		ParameterExpression<String> p = cb.parameter(String.class, "dept");
		criteria = cb.and(criteria, cb.equal(emp.get("dept").get("name"), p));
	}
	if (projectName != null) {
		ParameterExpression<String> p = cb.parameter(String.class, "project");
		criteria = cb.and(criteria, cb.equal(project.get("name"), p));
	}
	if (city != null) {
		ParameterExpression<String> p = cb.parameter(String.class, "city");
		criteria = cb.and(criteria, cb.equal(emp.get("address").get("city"), p));
	}
	if (criteria.getExpressions().size() == 0) {
		throw new RuntimeException("no criteria");
	}

#### Literals

#### Parameters

****Listing 9-4. Creating Parameter Expressions****

	CriteriaQuery<Employee> c = cb.createQuery(Employee.class);
	Root<Employee> emp = c.from(Employee.class);
	c.select(emp);
	ParameterExpression<String> deptName = cb.parameter(String.class, "deptName");
	c.where(cb.equal(emp.get("dept").get("name"), deptName));

equivalent to :

	CriteriaQuery<Employee> c = cb.createQuery(Employee.class);
	Root<Employee> emp = c.from(Employee.class);
	c.select(emp).where(cb.equal(emp.get("dept").get("name"),
	cb.parameter(String.class, "deptName")));

#### Subqueries

****Listing 9-5. Modified Employee Search With Subqueries****

	@Stateless
	public class SearchService {

		@PersistenceContext(unitName="EmployeeHR")
		EntityManager em;

		public List<Employee> findEmployees(String name, String deptName, String projectName, String city) {

			CriteriaBuilder cb = em.getCriteriaBuilder();
			CriteriaQuery<Employee> c = cb.createQuery(Employee.class);
			Root<Employee> emp = c.from(Employee.class);
			c.select(emp);
			
			// ...

			if (projectName != null) {
				Subquery<Employee> sq = c.subquery(Employee.class);
				Root<Project> project = sq.from(Project.class);
				Join<Project,Employee> sqEmp = project.join("employees");

				sq.select(sqEmp).where(cb.equal(project.get("name"),
				cb.parameter(String.class, "project")));

				criteria.add(cb.in(emp).value(sq));
			}
		
			// ...

	}

The equivalent JP QL query with only Project criteria would be:

	SELECT e
	FROM Employee e
	WHERE e IN (SELECT emp FROM Project p JOIN p.employees emp WHERE p.name = :project)

For example, we could rewrite the previous example to use EXISTS instead of IN and shift the conditional expression into the WHERE clause of the subquery.

	if (projectName != null) {
		Subquery<Project> sq = c.subquery(Project.class);
		Root<Project> project = sq.from(Project.class);
		Join<Project,Employee> sqEmp = project.join("employees");

		sq.select(project).where(cb.equal(sqEmp, emp),
		cb.equal(project.get("name"),
		cb.parameter(String.class,"project")));

		criteria.add(cb.exists(sq));
	}

This time the query takes the following form in JP QL:

	SELECT e
	FROM Employee e
	WHERE EXISTS (SELECT p FROM Project p JOIN p.employees emp WHERE emp = e AND p.name = :name)

We can still take this example further and reduce the search space for the subquery by moving the reference to the Employee root to the FROM clause of the subquery and joining directly to the list of projects specific to that employee. In JP QL, we would write this as follows:

	SELECT e
	FROM Employee e
	WHERE EXISTS (SELECT p FROM e.projects p WHERE p.name = :name)

In order to re-create this query using the Criteria API, we are confronted with a dilemma. We need to base the query on the Root object from the parent query but the from() method only accepts a persistent class type. 

The solution is the correlate() method from the Subquery interface. It performs a similar function to the from() method of the AbstractQuery interface, but does so with Root and Join objects from the parent query. The following example demonstrates how to use correlate() in this case:

	if (projectName != null) {

		Subquery<Project> sq = c.subquery(Project.class);
		Root<Employee> sqEmp = sq.correlate(emp);
		Join<Employee,Project> project = sqEmp.join("projects");

		sq.select(project).where(cb.equal(project.get("name"),
		cb.parameter(String.class,"project")));
		criteria.add(cb.exists(sq));
	}

Before we leave subqueries in the Criteria API, there is one more corner case with correlated subqueries to explore: referencing a join expression from the parent query in the FROM clause of a subquery. Consider the following example that returns projects containing managers with direct reports earning an average salary higher than a user-defined threshold:

	SELECT p
	FROM Project p JOIN p.employees e
	WHERE TYPE(p) = DesignProject AND
	e.directs IS NOT EMPTY AND
	(SELECT AVG(d.salary)
	FROM e.directs d) >= :value

When creating the Criteria API query definition for this query, we must correlate the employees attribute of Project and then join it to the direct reports in order to calculate the average salary. This example also demonstrates the use of the type() method of the Path interface in order to do a polymorphic comparison of types:

	CriteriaQuery<Project> c = cb.createQuery(Project.class);
	Root<Project> project = c.from(Project.class);
	Join<Project,Employee> emp = project.join("employees");
	Subquery<Number> sq = c.subquery(Number.class);
	Join<Project,Employee> sqEmp = sq.correlate(emp);
	Join<Employee,Employee> directs = sqEmp.join("directs");

	c.select(project).where(
		cb.equal(project.type(), DesignProject.class),
		cb.isNotEmpty(emp.<Collection>get("directs")),
		cb.ge(sq.select(cb.avg(directs.get("salary"))),
		cb.parameter(Number.class, "value")));

#### In Expressions

	SELECT e
	FROM Employee e
	WHERE e.address.state IN ('NY', 'CA')

To convert this query to the Criteria API, we must invoke the value() method of the  CriteriaBuilder. In interface to set the state identifiers we are interested in querying, like so:

	CriteriaQuery<Employee> c = cb.createQuery(Employee.class);
	Root<Employee> emp = c.from(Employee.class);
	c.select(emp)
		.where(cb.in(emp.get("address").get("state")).value("NY").value("CA"));

OR

	CriteriaQuery<Employee> c = cb.createQuery(Employee.class);
	Root<Employee> emp = c.from(Employee.class);
	c.select(emp)
		.where(emp.get("address").get("state").in("NY","CA"));

More complicated example, query using an IN expression in which the department of an employee is
tested against a list generated from a subquery.: 

	SELECT e
	FROM Employee e
	WHERE e.department IN
		(SELECT DISTINCT d
		FROM Department d JOIN d.employees de JOIN de.project p
		WHERE p.name LIKE 'QA%')

convert in :

	CriteriaQuery<Employee> c = cb.createQuery(Employee.class);
	Root<Employee> emp = c.from(Employee.class);
	Subquery<Department> sq = c.subquery(Department.class);
	Root<Department> dept = sq.from(Department.class);
	Join<Employee,Project> project = dept.join("employees").join("projects");

	sq.select(dept.<Integer>get("id"))
	.distinct(true)
	.where(cb.like(project.<String>get("name"), "QA%"));
	c.select(emp)
	.where(cb.in(emp.get("dept").get("id")).value(sq));

#### Case Expressions

	SELECT p.name,
		CASE WHEN TYPE(p) = DesignProject THEN 'Development'
			 WHEN TYPE(p) = QualityProject THEN 'QA'
		     ELSE 'Non-Development'
		END
	FROM Project p
	WHERE p.employees IS NOT EMPTY

The selectCase() method of the CriteriaBuilder interface is used to create the CASE expression. For the general form it takes no arguments and returns a CriteriaBuilder.Case object that we may use to add the conditional expressions to the CASE statement. The following example demonstrates this approach:

	CriteriaQuery<Object[]> c = cb.createQuery(Object[].class);
	Root<Project> project = c.from(Project.class);
	
	c.multiselect(project.get("name"),
		cb.selectCase()
			.when(cb.equal(project.type(), DesignProject.class),
				"Development")
			.when(cb.equal(project.type(), QualityProject.class),
				"QA")
			.otherwise("Non-Development")
	).where(cb.isNotEmpty(project.<List<Employee>>get("employees")));

The last example we will cover in this section concerns the JP QL COALESCE expression.

	SELECT COALESCE(d.name, d.id)
	FROM Department d

Building a COALESCE expression with the Criteria API requires a helper interface like the other examples we have looked at in this section, but it is closer in form to the IN expression than the CASE expressions. Here we invoke the coalesce() method without arguments to get back a CriteriaBuilder.Coalesce object that we then use the value() method of to add values to the COALESCE expression. The following example demonstrates this approach:

	CriteriaQuery<Object> c = cb.createQuery();
	Root<Department> dept = c.from(Department.class);
	c.select(cb.coalesce()
	.value(dept.get("name"))
	.value(dept.get("id")));

Convenience versions of the coalesce() method also exist for the case where only two expressions are
being compared.

	CriteriaQuery<Object> c = cb.createQuery();
	Root<Department> dept = c.from(Department.class);
	c.select(cb.coalesce(dept.get("name"),dept.get("id")));

#### Function Expressions

Function expressions are created with the function() method of the CriteriaBuilder interface. It requires as arguments the database function name, the expected return type, and a variable list of arguments, if any, that should be passed to the function. The return type is an Expression, so it can be used in many other places within the query. The following example invokes a database function to capitalize the first letter of each word in a department name:

	CriteriaQuery<String> c = cb.createQuery(String.class);
	Root<Department> dept = c.from(Department.class);
	c.select(cb.function("initcap", String.class, dept.get("name")));

#### Downcasting

JPA 2.1 introduced support for type downcasting when querying over an entity inheritance hierarchy via the TREAT operation in JP QL. In the Criteria API, this operation is exposed via the treat() method of the CriteriaBuilder interface. It may be used when constructing joins to limit results to a specific subclass or as part of general criteria expressions in order to access state fields from specific subclasses.

The treat() method has been overloaded to return either Join or Path objects depending on the type of arguments. Recall the example in Chapter 8 that demonstrated using treat() to limit an Employee query to only those employees who are working on a QualityProject for which the quality rating is greater than five: 

	SELECT e FROM Employee e JOIN TREAT(e.projects AS QualityProject) qp
	WHERE qp.qualityRating > 5

The criteria equivalent of this query is as follows:

	CriteriaQuery<Employee> q = cb.createQuery(Employee.class);
	Root<Employee> emp = q.from(Employee.class);
	Join<Employee,QualityProject> project = 
		cb.treat(emp.join(emp.get("projects"),QualityProject.class);
	q.select(emp)
		.where(cb.gt(project.<Integer>get("qualityRating"), 5));

In this example, treat() accepts a Join object and returns a Join object and can be referenced in the WHERE clause because it is saved in a variable. When used directly in a criteria expression, such as in a WHERE clause, it accepts a Path or a Root object and returns a Path that corresponds to the subclass requested. For example, we could use the treat() method to access the quality rating or the design phase in a query across projects. 

	CriteriaQuery<Project> q = cb.createQuery(Project.class);
	Root<Project> project = q.from(Project.class);
	q.select(project)
		.where(
			cb.or(
				cb.gt(cb.treat(project, QualityProject.class). <Integer>get("qualityRating"), 5),
				cb.gt(cb.treat(project, DesignProject.class). <Integer>get("designPhase"), 3)
		)
	);

#### Outer Join Criteria

	SELECT e FROM Employee e JOIN e.projects p ON p.name = 'Zooby'

The on() method takes a single predicate object to represent the join condition. The criteria equivalent of the JP QL query above would be expressed as:

	CriteriaQuery<Employee> q = cb.createQuery(Employee.class);
	Root<Employee> emp = q.from(Employee.class);
	Join<Employee,Project> project = emp.join("projects", JoinType.LEFT)
		.on(cb.equal(project.get("name"), "Zooby"));
	q.select(emp);

### The ORDER BY Clause

	CriteriaQuery<Tuple> c = cb.createQuery(Tuple.class);
	Root<Employee> emp = c.from(Employee.class);
	Join<Employee,Department> dept = emp.join("dept");
	c.multiselect(dept.get("name"), emp.get("name"));
	c.orderBy( cb.desc(dept.get("name")), cb.asc(emp.get("name")) );

The equivalent JP QL for the query shown in the previous example is as follows:

	SELECT d.name, e.name
	FROM Employee e JOIN e.dept d
	ORDER BY d.name DESC, e.name

### The GROUP BY and HAVING Clauses

Consider the following example from the previous chapter:

	SELECT e, COUNT(p)
	FROM Employee e JOIN e.projects p
	GROUP BY e
	HAVING COUNT(p) >= 2

To re-create this example with the Criteria API, we will need to make use of both aggregate functions and the grouping methods. The following example demonstrates this conversion:

	CriteriaQuery<Object[]> c = cb.createQuery(Object[].class);
	Root<Employee> emp = c.from(Employee.class);
	Join<Employee,Project> project = emp.join("projects");
	c.multiselect(emp, cb.count(project))
		.groupBy(emp)
		.having(cb.ge(cb.count(project),2));

## Bulk Update and Delete

	UPDATE Employee e
	SET e.salary = e.salary + 5000
	WHERE EXISTS (SELECT p FROM e.projects p WHERE p.name = 'Release2')

Using the Criteria API, the same query would be constructed as follows:

	CriteriaUpdate<Employee> q = cb.createCriteriaUpdate(Employee.class);
	Root<Employee> e = q.from(Employee.class);
	Subquery<Project> sq = c.subquery(Project.class);
	Root<Employee> sqEmp = sq.correlate(e);
	Join<Employee,Project> project = sqEmp.join("projects");
	sq.select(project)
		.where(cb.equal(project.get("name"),"Release2"));
	q.set(emp.get("salary"), cb.sum(emp.get("salary"), 5000))
		.where(cb.exists(sq));

The set() method may be invoked multiple times for the same query. It also supports the same invocation chaining capability that is used by other query methods in the Criteria API. Bulk delete operations are very straightforward with the Criteria API. The following example converts the JP QL
DELETE query example from Chapter 8 to use the Criteria API:

	DELETE FROM Employee e
	WHERE e.department IS NULL

	CriteriaDelete<Employee> q = cb.createCriteriaDelete(Employee.class);
	Root<Employee> emp = c.from(Employee.class);
	q.where(cb.isNull(emp.get("dept"));

If executed, this example will remove all employees that are not assigned to any department.

## Strongly Typed Query Definitions

### The Metamodel API

For example, to obtain information about the Employee class we have been demonstrating in this chapter, we would use the entity() method.

	Metamodel mm = em.getMetamodel();
	EntityType<Employee> emp_ = mm.entity(Employee.class);

***Figure 9-2. Metamodel interfaces page 256***

To further expand on this example, consider a tool that inspects a persistent unit and prints out summary information to the console. To enumerate all of the attributes and their types, we could use the following code:

	public <T> void listAttributes(EntityType<T> type) {
		for (Attribute<? super T, ?> attr : type.getAttributes()) {
			System.out.println(attr.getName() + " " +
			attr.getJavaType().getName() + " " +
			attr.getPersistentAttributeType());
		}
	}	

For the Employee entity, this would result in the following:

	id int BASIC
	name java.lang.String BASIC
	salary float BASIC
	dept com.acme.Department MANY_TO_ONE
	address com.acme.Address MANY_TO_ONE
	directs com.acme.Employee ONE_TO_MANY
	phones com.acme.Phone ONE_TO_MANY
	projects com.acme.Project ONE_TO_MANY

### Strongly Typed API Overview

The string-based API within the Criteria API centers around constructing path expressions: join(), fetch(), and get() all accept string arguments. The strongly typed API within the Criteria API also supports path expressions by extending the same methods, but is also present in all aspects of the CriteriaBuilder interface, simplifying typing and enforcing type safety where possible.

The strongly typed API draws its type information from the metamodel API classes we introduced in the previous section. For example, the join() method is overloaded to accept arguments of type ***SingularAttribute, CollectionAttribute, SetAttribute, ListAttribute, and MapAttribute***. Each overloaded version uses the type information associated with the attribute interface to create the appropriate return type such as MapJoin for arguments of type MapAttribute.

To demonstrate this behavior, we will revisit an example from earlier in the chapter where we were forced to use joinMap() with the string-based API in order to access the MapJoin object. This time we will use the metamodel API to obtain entity and attribute type information and pass it to the Criteria API methods.

	CriteriaQuery<Object> c = cb.createQuery();
	Root<Employee> emp = c.from(Employee.class);
	EntityType<Employee> emp_ = emp.getModel();
	MapJoin<Employee,String,Phone> phone =
		emp.join(emp_.getMap("phones", String.class, Phone.class)
	);
	c.multiselect(
		emp.get(emp_.getSingularAttribute("name", String.class)), phone.key(), phone.value()
	);

There are several things to note about this example. First is the use of getModel(). This method exists on many of the Criteria API interfaces as a shortcut to the underlying metamodel object. We assign it to a variable, emp_, and add an underscore by convention to help denote it as a metamodel type. Second are the two calls to methods from the EntityType interface. The getMap() invocation returns the MapAttribute object for the phones attribute while the getSingularAttribute() invocation returns the SingularAttribute object for the name attribute. Again, we have to supply the type information for the attribute, partly to satisfy the generic type requirements of the method invocation but also as a type checking mechanism. Had any of the arguments been incorrect, an exception would have been thrown. Also note that the join() method no longer qualifies the collection type yet returns the correct MapJoin instance. The join() method is overloaded to behave correctly in the presence of the different collection attribute interfaces from the metamodel API.

The potential for error in using the metamodel objects is actually the heart of what makes it strongly typed. By enforcing that the type information is available, the Criteria API is able to ensure that not only are the appropriate objects being passed around as method arguments but also that compatible types are used in various expression building methods.

### The Canonical Metamodel

Our usage of the metamodel API so far has opened the doors to strong type checking but at the expense of readability and increased complexity. The metamodel APIs are not complex, but they are verbose. To simplify their usage, JPA also provides a canonical metamodel for a persistence unit.
The canonical metamodel consists of dedicated classes, typically generated, one per persistent class, that contain static declarations of the metamodel objects associated with that persistent class. This allows you to access the same information exposed through the metamodel API, but in a form that applies directly to your persistent classes. Listing 9-7 shows an example of a canonical metamodel class.

***Listing 9-7. The Canonical Metamodel Class for Employee***

	@StaticMetamodel(Employee.class)
		public class Employee_ {
		public static volatile SingularAttribute<Employee, Integer> id;
		public static volatile SingularAttribute<Employee, String> name;
		public static volatile SingularAttribute<Employee, String> salary;
		public static volatile SingularAttribute<Employee, Department> dept;
		public static volatile SingularAttribute<Employee, Address> address;
		public static volatile CollectionAttribute<Employee, Project> project;
		public static volatile MapAttribute<Employee, String, Phone> phones;
	}

Each canonical metamodel class contains a series of static fields, one for each attribute in the persistent class. Each field in the canonical metamodel class is of the metamodel type that corresponds to the type of the like-named field or property in the persistent class. If a persistent field or property in an entity is of a primitive type or a single-valued relationship, then the like-named field in the canonical metamodel class will be of type SingularAttribute. Similarly, if a persistent field or property is collection-valued, then the field in the canonical metamodel class will be of type ListAttribute, SetAttribute, MapAttribute, or CollectionAttribute, depending upon the type of collection. Additionally, each canonical metamodel class is annotated with
@StaticMetamodel, which identifies the persistent class it is modeling.

#### Using the Canonical Metamodel

We are now in the position of being able to leverage the metamodel without actually using the metamodel API. As an example, we can convert the example from the previous section to use the statically generated canonical classes.

	CriteriaQuery<Object> c = cb.createQuery();
	Root<Employee> emp = c.from(Employee.class);
	MapJoin<Employee,String,Phone> phone = emp.join(Employee_.phones);
	c.multiselect(emp.get(Employee_.name), phone.key(), phone.value());

This is a much more concise approach to using the metamodel objects than the interfaces we discussed earlier while offering the exact same benefits. Coupled with a development environment that has good code completion features, you may find it more convenient to develop using the canonical metamodel than with the string-based API.

We can convert a more complex example to illustrate using the canonical metamodel. Listing 9-8 converts the example in Listing 9-6, showing an IN expression that uses a subquery, from using the string-based attribute names to using the strongly typed approach.

***Listing 9-8. Strongly Typed Query***
	
	CriteriaQuery<Employee> c = cb.createQuery(Employee.class);
	Root<Employee> emp = c.from(Employee.class);
	Subquery<Department> sq = c.subquery(Department.class);
	Root<Department> dept = sq.from(Department.class);
	Join<Employee,Project> project =
		dept.join(Department_.employees).join(Employee_.projects);
	
	sq.select(dept.get(Department_.id))
		.distinct(true)
		.where(cb.like(project.get(Project_.name), "QA%"));
	c.select(emp)
		.where(cb.in(emp.get(Employee_.dept).get(Department_.id)).value(sq));

Note that there are two main differences between this example and Listing 9-6. First, the use of typed metamodel static fields to indicate entity attributes helps avert typing errors in attribute strings. Second, there is no longer the need to do inlined typing to convert the Path<Object>, returned from the get() method, to a narrower type.

The stronger typing solves that problem for us. All the examples in this chapter can be easily converted to use the canonical metamodel classes by simply changing the string-based attributes to the corresponding static fields of the generated metamodel classes. For example, emp.get("name") can be replaced with emp.get(Employee_.name), and so on.

#### Generating the Canonical Metamodel

The generation tools offered by providers may vary widely in function or in operation. Generation may involve reading the persistence.xml file, as well as accessing annotations on entities and XML mapping files to determine what the metamodel classes should look like. Since the specification does not require such tools to even exist, a provider may choose to not support it at all, expecting that if developers want to use the canonical metamodel classes they will handcode them. Most providers do offer some kind of generation tool, though; it’s just a matter of understanding how that vendor-specific tool works. It might run statically as a separate command line tool, or it might use the compiler hook offered in the JDK (starting in Java SE 6) to look at the entities and generate the
classes at compile-time. For example, to run the command line mode of the tool shipped with the EclipseLink Reference Implementation, you could set the javac -processor and -proc:only options. These two options indicate the EclipseLink code/annotation processor1 for the compiler to invoke, and instruct the compiler to call only the processor but not do any actual compilation.

	javac -processor org.eclipse.persistence.internal.jpa.modelgen.CanonicalModelProcessor
		-proc:only
		-classpath lib/*.jar;punit
		*.java

***The options are on separate lines to make them easier to see. It is assumed that the lib directory contains the necessary EclipseLink JAR and JPA interface JAR, and that the META-INF/persistence.xml is in the punit directory.***

Metamodel generation tools will also typically run in an IDE, and there will likely be IDE-specific configuration necessary to direct the incremental compiler to use the tool’s annotation processor. In some IDEs, there must be an additional code/annotation processor JAR to configure. The generated metamodel classes will need to go in a specific directory and be on the build classpath so the criteria queries that reference them can compile. Consult the IDE help files on how annotation processors or APT is supported, as well as the provider documentation on what to configure
in order to enable generation in a given IDE.

### Choosing the Right Type of Query

[/]


