# Query Language

Queries written using this language can be portably compiled to SQL on all major database servers.

## Introducing JP QL

### Terminology

Queries fall into one of four categories: select, aggregate, update, and delete. In query expressions, entities are referred to by name. If an entity has not been explicitly named (using the name attribute of the @Entity annotation, for example), the unqualified class name is used by default. This name is the abstract schema name of the entity in the context of a query.

### Example Data Model

![Example application domain model](projpa2_ch8_datamodelchapitre8.png)

### Example Application

## Select Queries

	SELECT <select_expression>	FROM <from_clause>	[WHERE <conditional_expression>]	[ORDER BY <order_by_clause>]

**The simplest form of a select query consists of two mandatory parts: the SELECT clause and the FROM clause.**

### SELECT Clause

#### Path Expressions

Path expressions are the building blocks of queries. They are used to navigate out from an entity, either across a relationship to another entity (or collection of entities) or to one of the persistent properties of an entity. 

#### Entities and Objects

	SELECT d FROM Department d

The keyword OBJECT can be used to indicate that the result type of the query is the entity bound to the identification variable. It has no impact on the query, but it can be used as a visual clue.	
	SELECT OBJECT(d) FROM Department d

The only problem with using OBJECT is that even though path expressions can resolve to an entity type, the syntax of the OBJECT keyword is limited to identification variables. The expression OBJECT(e.department) is illegal even though Department is an entity type. 

To remove the duplicates, the DISTINCT operator must be used:	SELECT DISTINCT e.department FROM Employee e

The following query is **illegal**:	SELECT d.employees FROM Department d

The path expression d.employees is a collection-valued path that produces a collection type. Restricting queries in this way prevents the provider from having to combine successive rows from the database into a single result object. It is possible to select embeddable objects navigated to in a path expression. The following query returns only the ContactInfo embeddable objects for all the employees: 

	SELECT e.contactInfo FROM Employee e

**The thing to remember about selecting embeddables is that the returned objects will not be managed. If you issue a query to return employees (select e FROM Employee e) and then from the results navigate to their ContactInfo embedded objects, you would be obtaining embeddables that were managed. Changes to any one of those objects would be saved when the transaction committed. Changing any of the ContactInfo object results returned from a query that selected the ContactInfo directly, however, would have no persistent effect.**

#### Combining Expressions

	SELECT e.name, e.salary FROM Employee eWhen this is executed, a collection of zero or more instances of arrays of type Object will be returned. Each array in this example has two elements, the first being a String containing the employee name and the second being a Double containing the employee salary. 

#### Constructor Expressions

A more powerful form of the SELECT clause involving multiple expressions is the constructor expression, which specifies that the results of the query are to be stored using a user-specified object type. Consider the following query:	SELECT NEW example.EmployeeDetails(e.name, e.salary, e.department.name) 	FROM Employee e
The result type of this query is the example.EmployeeDetails Java class

### FROM Clause

#### Identification Variables

Every query must have at least one identification variable defined in the FROM clause, and that variable must correspond to an entity type. Range variable declarations use the syntax **<entity_name> [AS] <identifier>**.

#### Joins

Joins occur whenever any of the following conditions are met in a select query : 1. Two or more range variable declarations are listed in the FROM clause and appear in the select clause.2. The JOIN operator is used to extend an identification variable using a path expression.3. A path expression anywhere in the query navigates across an association field, to the sameor a different entity.4. One or more WHERE conditions compare attributes of different identification variables.

An inner join between two entities returns the objects from both entity types that satisfy all the join conditions. Path navigation from one entity to another is a form of inner join. The outer join of two entities is the set of objects from both entity types that satisfy the join conditions plus the set of objects from one entity type (designated as the left entity) that have no matching join condition in the other.

##### Inner Joins

###### JOIN Operator and Collection Association Fields

The syntax of an inner join using the JOIN operator is **[INNER] JOIN <path_expression> [AS] <identifier>**. Consider the following query:	SELECT p FROM Employee e JOIN e.phones p

By joining the two entities together, this query returns all the Phone entity instances associated with employees in the company.

onsider the equivalent SQL form of the previous query written using the traditional join form:	
	SELECT p.id, p.phone_num, p.type, p.emp_id	FROM emp e, phone p	WHERE e.id = p.emp_id

For example, instead of returning Phone entity instances, phone numbers can be returned instead.	SELECT p.number FROM Employee e JOIN e.phones p

###### JOIN Operator and Single-Valued Association Fields

The JOIN operator works with both collection-valued association path expressions and single-valued association path expressions :
	
	SELECT d FROM Employee e JOIN e.department d

equivalent to :

	SELECT e.department FROM Employee e

Other one : 

	SELECT DISTINCT e.department	FROM Project p JOIN p.employees e	WHERE p.name = 'Release1' AND      e.address.state = 'CA'

equivalent to :

	SELECT DISTINCT d	FROM Project p JOIN p.employees e JOIN e.department d JOIN e.address a	WHERE p.name = 'Release1' AND		￼a.state = 'CA'

Therefore, the actual SQL for such a query uses five tables, not four.	SELECT DISTINCT d.id, d.name	FROM project p, emp_projects ep, emp e, dept d, address a	WHERE p.id = ep.project_id AND      ep.emp_id = e.id AND      e.dept_id = d.id AND      e.address_id = a.id AND      p.name = 'Release1' AND      a.state = 'CA'

###### Join Conditions in the WHERE Clause

The previous join example between the Employee and Department entities could also have been written like this :
	SELECT DISTINCT d	FROM Department d, Employee e	WHERE d = e.department

	SELECT d, m	FROM Department d, Employee m	WHERE d = m.department AND      m.directs IS NOT EMPTY

###### Multiple Joins

More than one join can be cascaded if necessary. For example, the following query returns the distinct set of projects belonging to **employees who belong to a department :**	SELECT DISTINCT p	FROM Department d JOIN d.employees e JOIN e.projects p

##### Map Joins

For example, consider the case where the phones relationship of the Employee entity is modeled as a map, where the key is the number type (work, cell, home, etc.) and the value is the phone number. The following query enumerates the phone numbers for all employees:

	SELECT e.name, p
	FROM Employee e JOIN e.phones p
This behavior can be highlighted explicitly through the use of the VALUE keyword. For example, the preceding query is **functionally** identical to the following:	SELECT e.name, VALUE(p)	FROM Employee e JOIN e.phones pTo access the key instead of the value for a given map item, we can use the KEY keyword to override the default behavior and return the key value for a given map item. The following example demonstrates adding the phone type to the previous query:	SELECT e.name, KEY(p), VALUE(p)	FROM Employee e JOIN e.phones p	WHERE KEY(p) IN ('Work', 'Cell')Finally, in the event that we want both the key and the value returned together in the form of ajava.util.Map.Entry object, we can specify the ENTRY keyword in the same fashion. Note that the ENTRY keyword can only be used in the SELECT clause. The KEY and VALUE keywords can also be used as part of conditional expressions in the WHERE and HAVING clauses of the query. Note that in each of the map join examples we joined an entity against##### Outer JoinsAn outer join between two entities produces a domain in which only one side of the relationship is required to be complete. In other words, the outer join of Employee to Department across the employee department relationship returns all employees and the department to which the employee has been assigned, but the department is returned only if it is available.An outer join is specified using the following syntax: **LEFT [OUTER] JOIN <path_expression> [AS]<identifier>.** The following query demonstrates an outer join between two entities:	SELECT e, d	FROM Employee e LEFT JOIN e.department d***If the employee has not been assigned to a department, the department object (the second element of the Object array) will be null.***In a typical provider SQL generation, you will see that the previous query would be equivalent to the following:	SELECT e.id, e.name, e.salary, e.manager_id, e.dept_id, e.address_id, d.id, d.name	FROM employee e LEFT OUTER JOIN department d	ON (d.id = e.department_id)For example, we can modify the previous JP QL query to have an additional ON condition tolimit the departments returned to only those that have a ‘QA’ prefix:	SELECT e, d	FROM Employee e LEFT JOIN e.department d	ON d.name LIKE 'QA%'This query still returns **all of the employees, but the results will not include any departments not matching the added ON condition.** The generated SQL would look like:	SELECT e.id, e.name, e.salary, e.department_id, e.manager_id, e.address_id,	d.id, d.name	FROM employee e left outer join department d	ON ((d.id = e.department_id) and (d.name like 'QA%'))It's very different to this one :	SELECT e, d FROM Employee e LEFT JOIN e.department d	WHERE d.name LIKE 'QA%' The WHERE clause results in inner join semantics between Employee and Department, so this query would only return the employees who were in a department with a ‘QA’ prefixed name.##### Fetch JoinsFetch joins are intended to help application designers optimize their database access and prepare query results for detachment. They allow queries to specify one or more relationships that should be navigated and prefetched by the query engine so that they are not lazy loaded later at runtime.For example, if we have an Employee entity with a lazy loading relationship to its address, the following query can be used to indicate that the relationship should be resolved eagerly during query execution:	SELECT e FROM Employee e JOIN FETCH e.addressThe result of executing the query is still a collection of Employee entity instances, except that the address relationship on each entity will not cause a secondary trip to the database when it is accessed.A consequence of implementing fetch joins in this way is that fetching a collection association results in duplicate results. For example, consider a department query where the employees relationship of the Department entity is eagerly fetched. The fetch join query, this time using an outer join to ensure that departments without employees are retrieved, would be written as follows:	SELECT d	FROM Department d LEFT JOIN FETCH d.employeesExpressed in JP QL, the provider interpretation would replace the fetch with an outer join across the employees relationship:	SELECT d, e	FROM Department d LEFT JOIN d.employees e### Where Clause#### Input ParametersPositional notation is defined by prefixing the variable number with a question mark. Consider the following query:	SELECT e	FROM Employee e	WHERE e.salary > ?1Using the Query interface, any double value, or value that is type-compatible with the salary attribute, can be bound into the first parameter in order to indicate the lower limit for employee salaries in this query. The same positional parameter can occur more than once in the query. The value bound into the parameter will be substituted for each of its occurrences.Named parameters are specified using a colon followed by an identifier :	SELECT e	FROM Employee e	WHERE e.salary > :sal#### Basic Expression Form1. Navigation operator (.)2. Unary +/–3. Multiplication (*) and division (/)4. Addition (+) and subtraction (–)5. Comparison operators =, >, >=, <, <=, <>, [NOT] BETWEEN, [NOT] LIKE, [NOT] IN, IS [NOT] NULL, IS [NOT] EMPTY, [NOT] MEMBER [OF]6. Logical operators (AND, OR, NOT)

#### BETWEEN Expressions

Any employee making $40,000 to $45,000 inclusively is included in the results :

	SELECT e
	FROM Employee e
	WHERE e.salary BETWEEN 40000 AND 45000

This is identical to the following query using basic comparison operators:

	SELECT e
	FROM Employee e
	WHERE e.salary >= 40000 AND e.salary <= 45000

**The BETWEEN operator can also be negated with the NOT operator.**

#### LIKE Expressions

The wildcard characters used by the pattern string are **the underscore (_) for single character
wildcards and the percent sign (%) for multicharacter wildcards.**

	SELECT d
	FROM Department d
	WHERE d.name LIKE '__Eng%'

We are using a prefix of two underscore characters to wildcard the first **two characters** of the string candidates, so example department names to match this query would be “CAEngOtt” or “USEngCal”, but not “CADocOtt”. Note **that pattern matches are case-sensitive**.

If the pattern string contains an underscore or percent sign that should be literally matched, the ESCAPE clause can be used to specify a character that, when prefixing a wildcard character, indicates that it should be treated literally:

	SELECT d
	FROM Department d
	WHERE d.name LIKE 'QA\_%' ESCAPE '\'

**Escaping the underscore makes it a mandatory part of the expression. For example, “QA_East” would match, but “QANorth” would not.**

#### Subqueries

Subqueries can be used in the WHERE and HAVING clauses of a query. A subquery is a complete select query inside a pair of parentheses that is embedded within a conditional expression. The results of executing the subquery (which will be either a scalar result or a collection of values) are then evaluated in the context of the conditional expression. Subqueries are a powerful technique for solving the most complex query scenarios.

Consider the following query:

	SELECT e
	FROM Employee e
	WHERE e.salary = (SELECT MAX(emp.salary)
						FROM Employee emp)

This query returns the employee with the highest salary from among all employees. 

A subquery consisting of an aggregate query (described later in this chapter) is used to return the maximum salary value, and then this result is used as the key to filter the employee list by salary. A subquery can be used in most conditional exp ressions and can appear on either the left or right side of an expression.

The scope of an identifier variable name begins in the query where it is defined and extends down into any subqueries. Identifiers in the main query can be referenced by a subquery, and identifiers introduced by a subquery can be referenced by any subquery that it creates. If a subquery declares an identifier variable of the same name, it overrides the parent declaration and prevents the subquery from referring to the parent variable.

	SELECT e
	FROM Employee e
	WHERE EXISTS (SELECT 1
					FROM Phone p
					WHERE p.employee = e AND p.type = 'Cell')

**This query returns all the employees who have a cell phone number.** This is also an example of a subquery that returns a collection of values. The **EXISTS** expression in this example returns true if any results are returned by the subquery. **Returning the literal 1 from the subquery is a standard practice with EXISTS expressions because the actual results selected by the subquery do not matter; only the number of results is relevant.**

The advantage in using the correlated subquery is that the main query remains unburdened by joins to other entities.

The FROM clause of a subquery can also create **new identification variables out of path expressions using an identification variable from the main query**. For example, the previous query could also have been written as follows:

	SELECT e
	FROM Employee e
	WHERE EXISTS (SELECT 1
					FROM e.phones p
					WHERE p.type = 'Cell')

In this version of the query, the subquery uses the collection association path phones from the Employee identification variable e in the subquery. This is then mapped to a local identification variable p that is used to filter the results by phone type. Each occurrence of p refers to a single phone associated with the employee.

To better illustrate how the translator handles this query, consider the equivalent query written in SQL:

	SELECT e.id, e.name, e.salary, e.manager_id, e.dept_id, e.address_id
	FROM emp e
	WHERE EXISTS (SELECT 1
					FROM phone p
					WHERE p.emp_id = e.id AND
					p.type = 'Cell')

The expression e.phones is converted to the table mapped by the Phone entity. The WHERE clause for the subquery then adds the necessary join condition to correlate the subquery to the primary query, in this case the expression p.emp_id = e.id. The join criteria applied to the PHONE table results in all the phones owned by the related employee.

#### IN Expressions

The IN expression can be used to check whether a single-valued path expression is a member of a collection. The collection can be defined inline as a set of literal values or can be derived from a subquery. The following query demonstrates the literal notation by selecting all the employees who live in New York or California :

	SELECT e
	FROM Employee e
	WHERE e.address.state IN ('NY', 'CA')

The subquery form of the expression is similar, replacing the literal list with a nested query. The following query returns employees who work in departments that are contributing to projects beginning with the prefix “QA”:

	SELECT e
	FROM Employee e
	WHERE e.department IN (SELECT DISTINCT d
							FROM Department d JOIN d.employees de JOIN de.projects p
							WHERE p.name LIKE 'QA%')

The IN expression can also be negated using the NOT operator. For example, the following query returns all the Phone entities representing phone numbers other than for the office or home:

	SELECT p
	FROM Phone p
	WHERE p.type NOT IN ('Office', 'Home')

##### Collection Expressions

The IS EMPTY operator is the logical equivalent of IS NULL, but for collections. Queries can use the **IS EMPTY** operator or its negated form **IS NOT EMPTY** to check whether a collection association path resolves to an empty collection or has at least one value. For example, the following query returns all employees who are managers by virtue of having at least one direct report:

	SELECT e
	FROM Employee e
	WHERE e.directs IS NOT EMPTY

Note that IS EMPTY expressions are translated to SQL as subquery expressions. The query translator can make use of an aggregate subquery or use the SQL EXISTS expression. Therefore, the following query is equivalent to the previous one:

	SELECT m
	FROM Employee m
	WHERE (SELECT COUNT(e)
			FROM Employee e
			WHERE e.manager = m) > 0

The **MEMBER OF** operator and its negated form **NOT MEMBER OF** are a shorthand way of checking whether an entity is a member of a collection association path. The following query returns all managers who are incorrectly entered as reporting to themselves:

	SELECT e
	FROM Employee e
	WHERE e MEMBER OF e.directs

A more typical use of the MEMBER OF operator is in conjunction with an input parameter. For example,
the following query selects all employees who are assigned to a specified project:
	
	SELECT e
	FROM Employee e
	WHERE :project MEMBER OF e.projects

Like the IS EMPTY expression, the MEMBER OF expression will be translated to SQL using either an EXISTS expression or the subquery form of the IN expression. The previous example is equivalent to the following query:

	SELECT e
	FROM Employee e
	WHERE :project IN (SELECT p
						FROM e.projects p)

#### EXISTS Expressions

The EXISTS condition returns true if a subquery returns any rows. Examples of EXISTS were demonstrated earlier in the introduction to subqueries. The EXISTS operator can also be negated with the NOT operator. The following query selects all employees who do not have a cell phone:

	SELECT e
	FROM Employee e
	WHERE NOT EXISTS (SELECT p
						FROM e.phones p
						WHERE p.type = 'Cell')

#### ANY, ALL, and SOME Expressions

This query returns the managers who are paid less than all the employees who work for them.

	SELECT e
	FROM Employee e
	WHERE e.directs IS NOT EMPTY AND
	e.salary < ALL (SELECT d.salary
				FROM e.directs d)

The subquery is evaluated, and then each value of the subquery is compared to the left-hand expression, in this case the manager salary. When the ALL operator is used, the comparison between the left side of the equation and all subquery results must be true for the overall condition to be true. The ANY operator behaves similarly, but the overall condition is true as long as at least one of the comparisons between the expression and the subquery result is true. For example, if ANY were specified instead of ALL in the previous example, the result of the query would be all the managers who were paid less than at least one of their employees. The SOME operator is an alias for the ANY operator.

There is symmetry between IN expressions and the ANY operator. Consider the following variation of the project department example used previously, modified only slightly to use ANY instead of IN:

	SELECT e
	FROM Employee e
	WHERE e.department = ANY (SELECT DISTINCT d
								FROM Department d JOIN d.employees de JOIN de.projects p
								WHERE p.name LIKE 'QA%')

### Inheritance and Polymorphism

#### Subclass Discrimination

In the example model, Project is a base class for QualityProject and DesignProject. If an identification variable is formed from the Project entity, the query results will include a mixture of Project, QualityProject, and DesignProject objects and the results can be cast to the subclasses by the caller as necessary. The following query retrieves all projects with at least one employee:

	SELECT p
	FROM Project p
	WHERE p.employees IS NOT EMPTY

The following example demonstrates using a type expression to return only design and quality projects:
	
	SELECT p
	FROM Project p
	WHERE TYPE(p) = DesignProject OR TYPE(p) = QualityProject

**Note that there are no quotes around the DesignProject and QualityProject identifiers. These are treated as entity names in JP QL, not as strings.** Despite this distinction, input parameters can be used in place of hard-coded names in query strings. Creating a parameterized query that returns instances of a given subclass type is straightforward, as illustrated by the following query:

	SELECT p
	FROM Project p
	WHERE TYPE(p) = :projectType

#### Downcasting

In most cases at least one of the subclasses contains some additional state, such as the qaRating attibute in the QualityProject. A subclass attribute could be accessed directly if the query ranged over only the subclass entities, but when the query ranges over a superclass, downcasting must be used. Downcasting is the technique of making an expression that refers to a superclass be applied to a specific subclass. It is achieved through the use of the TREAT operator. **TREAT can be used in the WHERE clause to filter the results based on subtype state of the instances.** The following query returns all of the design projects plus all of the quality projects that have a quality rating greater than 4:

	SELECT p
	FROM Project p
	WHERE TREAT(p AS QualityProject).qaRating > 4
	OR TYPE(p) = DesignProject

**The syntax of the expression begins with the TREAT keyword, followed by its parenthesized  argument. The argument is a path expression, followed by the AS keyword and then the entity name of the target subtype.**

### Scalar Expressions

A scalar expression is a **literal value, arithmetic sequence, function expression, type expression, or case expression that resolves to a single scalar value**. It can be used in the **SELECT** clause to format projected fields in report queries or as part of conditional expressions in the **WHERE or HAVING** clause of a query. **Subqueries that resolve to scalar values are also considered scalar expressions, but can be used only when composing criteria in the WHERE clause of a query. Subqueries can never be used in the SELECT clause.**

#### Literals

**There are a number of different literal types that can be used in JP QL, including strings,  numerics, booleans, enums, entity types, and temporal types.**

Throughout this chapter, we have shown many examples of **string, integer, and boolean literals**. **Single quotes are used to demarcate string literals and escaped within a string by prefixing the quote with another single quote. Exact and approximate numerics can be defined according to the conventions of the Java programming language or by using the standard SQL-92 syntax**. Boolean values are represented by the literals **TRUE and FALSE**.

Queries can reference **Java enum types by specifying the fully qualified name of the enum class**. The following example demonstrates using an enum in a conditional expression, using the PhoneType enum demonstrated in Listing 5-8 from Chapter 5: 

	SELECT e
	FROM Employee e JOIN e.phoneNumbers p
	WHERE KEY(p) = com.acme.PhoneType.Home

Temporal literals are specified using the JDBC escape syntax, which defines that curly braces enclose the literal. The first character in the sequence is either a “d” or a “t” to indicate that the literal is a date or time, respectively. If the literal represents a timestamp, “ts” is used instead. Following the type indicator is a space separator, and then the actual date, time, or timestamp information wrapped in single quotes. The general forms of the three temporal literal types, with accompanying examples are as follows:

	{d 'yyyy-mm-dd'} e.g. {d '2009-11-05'}
	{t 'hh-mm-ss'} e.g. {t '12-45-52'}
	{ts 'yyyy-mm-dd hh-mm-ss.f'} e.g. {ts '2009-11-05 12-45-52.325'}

#### Function Expressions

- **ABS(number) :** The ABS function returns the unsigned version of the number argument. The result type is the same as the argument type (integer, float, or double).

- **CONCAT(string1, string2) :** The CONCAT function returns a new string that is the concatenation of its arguments, string1 and string2.
    
- **CURRENT_DATE :** The CURRENT_DATE function returns the current date as defined by the
database server.

- **CURRENT_TIME :** The CURRENT_TIME function returns the current time as defined by the
database server.

- **CURRENT_TIMESTAMP :** The CURRENT_TIMESTAMP function returns the current timestamp as
defined by the database server.

- **INDEX(identification variable) :** The INDEX function returns the position of an entity within an ordered list.
 
- **LENGTH(string) :** The LENGTH function returns the number of characters in the string argument.

- **LOCATE(string1, string2 [, start]) :** The LOCATE function returns the position of string1 in string2, optionally starting at the position indicated by start. The result is zero if the string
cannot be found.

- **LOWER(string) :** The LOWER function returns the lowercase form of the string argument.
MOD(number1, number2) The MOD function returns the modulus of numeric arguments number1
and number2 as an integer.

- **SIZE(collection) :** The SIZE function returns the number of elements in the collection, or zero
if the collection is empty.

- **SQRT(number) :** The SQRT function returns the square root of the number argument as a double.
SUBSTRING(string, start, end) The SUBSTRING function returns a portion of the input string, starting
at the index indicated by start up to length characters. String indexes are
measured starting from one.

- **UPPER(string) :** The UPPER function returns the uppercase form of the string argument.

- **TRIM([[LEADING|TRAILING|BOTH] [char] FROM] string) :** The TRIM function removes leading and/or trailing characters from a string. If the optional LEADING, TRAILING, or BOTH keyword is not used, both leading and trailing characters

#### Native Database Functions

The following query invokes a database function named **“shouldGetBonus.”** The id of the employee’s department and the projects he works on are passed as parameters and the function return type is a boolean. The result creates a condition that makes the query return the set of all employees who get a bonus.

	SELECT DISTINCT e
	FROM Employee e JOIN e.projects p
	WHERE FUNCTION('shouldGetBonus', e.department.id, p.id)

#### CASE Expressions

	CASE {WHEN <cond_expr> THEN <scalar_expr>}+ ELSE <scalar_expr> END

The following example demonstrates the general case expression, enumerating the name and type of each project that has employees assigned to it: 

	SELECT p.name,
		CASE WHEN TYPE(p) = DesignProject THEN 'Development'
			 WHEN TYPE(p) = QualityProject THEN 'QA'
			 ELSE 'Non-Development'
		END
	FROM Project p
	WHERE p.employees IS NOT EMPTY

Case expressions are a powerful tool for transforming entity data in report queries.

The third form of the case expression is the coalesce expression. This form of the case expression accepts a sequence of one or more scalar expressions. It has the following form:

	COALESCE(<scalar_expr> {,<scalar_expr>}+)

The scalar expressions in the COALESCE expression are resolved in order. The first one to return a non-null value becomes the result of the expression. The following example demonstrates this usage, returning either the descriptive name of each department or the department identifier if no name has been defined:

	SELECT COALESCE(d.name, d.id)	
	FROM Department d

The fourth and final form of the case expression is somewhat unusual. It accepts two scalar expressions and resolves both of them. If the results of the two expressions are equal, the result of the expression is null. Otherwise it returns the result of the first scalar expression. This form of the case expression is identified by the NULLIF keyword.

	NULLIF(<scalar_expr1>, <scalar_expr2>)

One useful trick with NULLIF is to exclude results from an aggregate function. For example, the following query returns a count of all departments and a count of all departments not named ‘QA’:

	SELECT COUNT(*), COUNT(NULLIF(d.name, 'QA'))
	FROM Department d

If the department name is ‘QA’, NULLIF will return NULL, which will then be ignored by the COUNT function. Aggregate functions ignore NULL values, and are described later in the “Aggregate Queries” section.

### ORDER BY Clause

Queries can optionally be sorted using ORDER BY and one or more expressions consisting of identification variables, result variables, a path expression resolving to a single entity, or a path expression resolving to a persistent state field. The optional keywords ASC or DESC after the expression can be used to indicate ascending or descending sorts, respectively. The default sort order is ascending. The following example demonstrates sorting by a single field:

	SELECT e
	FROM Employee e
	ORDER BY e.name DESC

Multiple expressions can also be used to refine the sort order:

	SELECT e, d
	FROM Employee e JOIN e.department d
	ORDER BY d.name, e.name DESC

A result variable can be declared in the SELECT clause for the purpose of specifying an item to be ordered. A result variable is effectively an alias for its assigned selection item. It saves the ORDER BY clause from having to duplicate path expressions from the SELECT clause and permits referencing computed selection items and items that use aggregate functions. The following query defines two result variables in the SELECT clause and then uses them to order the results in the ORDER BY clause:

	SELECT e.name, e.salary * 0.05 AS bonus, d.name AS deptName	
	FROM Employee e JOIN e.department d
	ORDER BY deptName, bonus DESC

If the SELECT clause of the query uses state field path expressions, the ORDER BY clause is limited to the same path expressions used in the SELECT clause. For example, **the following query is not legal:**

	SELECT e.name
	FROM Employee e
	ORDER BY e.salary DESC

Because the result type of the query is the employee name, which is of type String, the remainder of the Employee state fields are no longer available for ordering.

## Aggregate Queries

An aggregate query groups results and applies aggregate functions to obtain summary information about query results. A query is considered an aggregate query if it uses an aggregate function or possesses **a GROUP BY clause and/or a HAVING clause**.

	SELECT <select_expression>
	FROM <from_clause>
	[WHERE <conditional_expression>]
	[GROUP BY <group_by_clause>]
	[HAVING <conditional_expression>]
	[ORDER BY <order_by_clause>]

The SELECT, FROM, and WHERE clauses behave much the same as previously described under select queries, with the exception of some restrictions on how the SELECT clause is formulated.

	SELECT AVG(e.salary)
	FROM Employee e

AVG is an aggregate function that takes a numeric state field path expression as an argument and calculates the average over the group. Because there was no GROUP BY clause specified, the group here is the entire set of employees. Now consider this variation, where the result has been grouped by the department name:

	SELECT d.name, AVG(e.salary)
	FROM Department d JOIN d.employees e
	GROUP BY d.name

This query returns the name of each department and the average salary of the employees in that department. The Department entity is joined to the Employee entity across the employees relationship and then formed into a group defined by the department name.

This can be extended further to filter the data so **that manager salaries are not included**:

	SELECT d.name, AVG(e.salary)
	FROM Department d JOIN d.employees e
	WHERE e.directs IS EMPTY
	GROUP BY d.name

Finally, we can extend this one last time to return only the departments where the average salary is greater than $50,000. 
	
	SELECT d.name, AVG(e.salary)
	FROM Department d JOIN d.employees e
	WHERE e.directs IS EMPTY
	GROUP BY d.name
	HAVING AVG(e.salary) > 50000

### Aggregate Functions

Five aggregate functions can be placed in the select clause of a query: **AVG, COUNT, MAX, MIN, and SUM.**

#### AVG

The AVG function takes a state field path expression as an argument and calculates the average value of that state field over the group. The state field type must be numeric, and the result is returned as a Double.

#### COUNT

The COUNT function takes either an identification variable or a path expression as its argument. This path expression can resolve to a state field or a single-valued association field. The result of the function is a Long value representing the number of values in the group. The argument to the COUNT function can optionally be preceded by the keyword DISTINCT, in which case duplicate values are eliminated before counting. The following query counts the number of phones associated with each employee as well as the number of distinct number types (cell, office, home, and so on):

	SELECT e, COUNT(p), COUNT(DISTINCT p.type)
	FROM Employee e JOIN e.phones p
	GROUP BY e

#### MAX

The MAX function takes a state field expression as an argument and returns the maximum value in the group for that state field.

#### MIN

The MIN function takes a state field expression as an argument and returns the minimum value in the group for that state field.

#### SUM

The SUM function takes a state field expression as an argument and calculates the sum of the values in that state field over the group. The state field type must be numeric, and the result type must correspond to the field type. **For example, if a Double field is summed, the result will be returned as a Double. If a Long field is summed, the response will be returned as a Long.**

### GROUP BY Clause

The GROUP BY clause defines the grouping expressions over which the results will be aggregated. A grouping expression must either be a single-valued path expression (state field, embeddable leading to a stat e field, or single-valued association field) or an identification variable. If an identification variable is used, the entity must not have any serialized state or large object fields. The following query counts the number of employees in each department :

	SELECT d.name, COUNT(e)
	FROM Department d JOIN d.employees e
	GROUP BY d.name

	SELECT d.name, COUNT(e), AVG(e.salary)
	FROM Department d JOIN d.employees e
	GROUP BY d.name

This variation of the query calculates the average salary of all employees in each department in addition to counting the number of employees in the department. Multiple grouping expressions can also be used to further break down the results:

	SELECT d.name, e.salary, COUNT(p)
	FROM Department d JOIN d.employees e JOIN e.projects p
	GROUP BY d.name, e.salary

In the absence of a GROUP BY clause, aggregate functions will be applied to the entire result set as a single group. For example, the following query returns the number of employees and their average salary across the entire company:

	SELECT COUNT(e), AVG(e.salary)
	FROM Employee e

### HAVING Clause

The HAVING clause defines a filter to be applied after the query results have been grouped. It is effectively a secondary WHERE clause, and its definition is the same: the keyword HAVING followed by a conditional expression. The key difference with the HAVING clause is that its conditional expressions are mostly limited to state fields or single-valued association fields included in the group.

Conditional expressions in the HAVING clause can also make use of aggregate functions over the elements used for grouping, or aggregate functions that appear in the SELECT clause. In many respects, the primary use of the HAVING clause is to restrict the results based on the aggregate result values. The following query uses this technique to retrieve all employees assigned to two or more projects:

	SELECT e, COUNT(p)
	FROM Employee e JOIN e.projects p
	GROUP BY e
	HAVING COUNT(p) >= 2

### Update Queries

	UPDATE <entity name> [[AS] <identification variable>]
	SET <update_statement> {, <update_statement>}*
	[WHERE <conditional_expression>]

The following simple example demonstrates the update query by giving employees who make $55,000 a year a raise to $60,000:

	UPDATE Employee e
	SET e.salary = 60000
	WHERE e.salary = 55000

The WHERE clause of an UPDATE statement functions the same as a SELECT statement and can use the
identification variable defined in the UPDATE clause in expressions. A slightly more complex but more realistic update query would be to award a $5,000 raise to employees who worked on a particular project:

	UPDATE Employee e
	SET e.salary = e.salary + 5000
	WHERE EXISTS (SELECT p
	FROM e.projects p
	WHERE p.name = 'Release2')

More than one property of the target entity can be modified with a single UPDATE statement. For example, the following query updates the phone exchange for employees in the city of Ottawa and changes the terminology of the phone type from “Office” to “Business”:

	UPDATE Phone p
	SET p.number = CONCAT('288', SUBSTRING(p.number, LOCATE(p.number, '-'), 4)), p.type = 'Business'
	WHERE p.employee.address.city = 'Ottawa' AND
		p.type = 'Office'

### Delete Queries

	DELETE FROM <entity name> [[AS] <identification variable>] [WHERE <condition>]

	DELETE FROM Employee e WHERE e.department IS NULL

The WHERE clause for a DELETE statement functions the same as it would for a SELECT statement. All conditional expressions are available to filter the set of entities to be removed. If the WHERE clause is not provided, all entities of the given type are removed. Delete queries are polymorphic. Any entity subclass instances that meet the criteria of the delete query will also be deleted. Delete queries do not honor cascade rules, however. No entities other than the type referenced in the query and its subclasses will be removed, even if the entity has relationships to other entities with cascade removes enabled.