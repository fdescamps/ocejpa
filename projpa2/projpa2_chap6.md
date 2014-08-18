# Chapter 6 : Entity Manager

## Persistence Contexts

* A persistence unit is a named configuration of entity classes.
* A persistence context is a managed set of entity instances. 
* Every persistence context is associated with a persistence unit, restricting the classes of the managed instances to the set defined by the persistence unit.
Saying that an entity instance is managed means that it is contained within a persistence context and it can be acted upon by an entity manager. It is for this reason that we say that an entity manager manages a persistence context.

## Entity Managers

### Container-Managed Entity Managers

In the Java EE environment, the most common way to acquire an entity manager is by using the @PersistenceContext annotation to inject one. An entity manager obtained in this way is called container-managed because the container manages the lifecycle of the entity manager, typically by proxying the one that it gets from the persistence provider. The application does not have to create it or close it. This is the style of entity manager we demonstrated in Chapter 3.

Container-managed entity managers come in two varieties. The style of a container-managed entity manager determines how it works with persistence contexts.
The first and most common style is called transaction-scoped.
This means that the persistence contexts managed by the entity manager are scoped by the active JTA transaction, ending when the transaction is complete. The second style is called extended. Extended entity managers work with a single persistence context that is tied to the lifecycle of a stateful session bean and are scoped to the life of that stateful session bean, potentially spanning multiple transactions.

#### Transaction-Scoped

All the entity manager examples that we have shown so far for the Java EE environment have been transaction-scoped entity
managers. A transaction-scoped entity manager is returned whenever the reference created by the @PersistenceContext annotation is resolved. As we mentioned in Chapter 3, a transaction-scoped entity manager is stateless, meaning that it can be safely stored on any Java EE component. Because the container manages it for us, it is also basically maintenance-free.

Listing 6-1. The ProjectService Session Bean
@Stateless
public class ProjectService {
	
	@PersistenceContext(unitName="EmployeeService")
	EntityManager em;
	
	public void assignEmployeeToProject(int empId, int projectId) {
		Project project = em.find(Project.class, projectId);
		Employee employee = em.find(Employee.class, empId);
		project.getEmployees().add(employee);
		employee.getProjects().add(project);
	}

	// ...
}

All container-managed entity managers depend on JTA
transactions because they can use the transaction as a way to track persistence contexts. Every time an operation is invoked on the entity manager, the container proxy for that entity manager checks to see whether a persistence context is associated with the container JTA transaction. If it finds one, the entity manager will use this persistence context. If it doesn’t find one, it creates a new persistence context and associates it with the transaction. When the transaction ends, the persistence context goes away.

Let’s walk through an example. Consider the assignEmployeeToProject() method from Listing 6-1. The first thing the method does is search for the Employee and Project instances using the find() operation. When the first find() method is invoked, the container checks for a transaction. By default, the container will ensure that a transaction is active whenever a session bean method starts, so the entity manager in this example will find one ready. It then checks for a persistence context. This is the first time any entity manager call has occurred, so there isn’t a persistence context yet. The entity manager creates a new one and uses it to find the project.

When the entity manager is used to search for the employee, it checks the transaction again and this time finds the one it created when searching for the project. It then reuses this persistence context to search for the employee. At this point, employee and project are both managed entity instances. The employee is then added to the project, updating both the employee and project entities. When the method call ends, the transaction is committed. Because the employee and project instances were managed, the persistence context can detect any state changes in them, and it updates the database during the commit. When the transaction is over, the persistence context goes away. This process is repeated every time one or more entity manager operations are invoked within a transaction.

#### Extended

order to describe the extended entity manager, we must first talk a little about stateful session beans. As you learned in Chapter 3, stateful session beans are designed to hold conversational state. Once acquired by a client, the same bean instance is used for the life of the conversation until the client invokes one of the methods marked @Remove on the bean. While the conversation is active, the business methods of the client can store and access information using the fields of the bean.

Our goal is to create a business object for a Department entity that provides business operations relating to that entity.

The business method init() is called by the client to initialize the department id. We then store this department id on the bean instance, and the addEmployee() method uses it to find the department and make the necessary changes. From the perspective of the client, they only have to set the department id once, and then subsequent operations always refer to the same department.

Listing 6-2. First Attempt at Department Manager Bean
@Stateful
public class DepartmentManager {
	
	@PersistenceContext(unitName="EmployeeService")
	EntityManager em;
	
	int deptId;
	public void init(int deptId) {
		this.deptId = deptId;
	}

	public void setName(String name) {
		Department dept = em.find(Department.class, deptId);
		dept.setName(name);
	}

	public void addEmployee(int empId) {
		Department dept = em.find(Department.class, deptId);
		Employee emp = em.find(Employee.class, empId);
		dept.getEmployees().add(emp);
		emp.setDepartment(dept);
	}

	// ...
	@Remove
	public void finished() {}
}

The first thing that should stand out when looking at this bean is that it seems unnecessary to have to search for the department every time. After all, we have the department id, so why not just store the Department entity instance as well? Listing 6-3 revises our first attempt by searching for the department once during the init() method and then reusing the entity instance for each business method.

Listing 6-3. Second Attempt at Department Manager Bean
@Stateful
public class DepartmentManager {
	
	@PersistenceContext(unitName="EmployeeService")
	EntityManager em;
	Department dept;
	
	public void init(int deptId) {
		dept = em.find(Department.class, deptId);
	}

	public void setName(String name) {
		dept.setName(name);
	}

	public void addEmployee(int empId) {
		Employee emp = em.find(Employee.class, empId);
		dept.getEmployees().add(emp);
		emp.setDepartment(dept);
	}

	// ...
	@Remove
	public void finished() {
	}
}

This version looks better suited to the capabilities of a stateful session bean. It is certainly more natural to reuse
the Department entity instance instead of searching for it each time. But there is a problem. The entity manager in
Listing 6-3 is transaction-scoped. Assuming there is no active transaction from the client, every method on the bean
will start and commit a new transaction because the default transaction attribute for each method is REQUIRED.
Because there is a new transaction for each method, the entity manager will use a different persistence context
each time.
Even though the Department instance still exists, the persistence context that used to manage it went away when
the transaction associated with the init() call ended. We refer to the Department entity in this case as being detached
from a persistence context. The instance is still around and can be used, but any changes to its state will be ignored.
For example, invoking setName() will change the name in the entity instance, but the changes will never be reflected
in the database.


Listing 6-4. Using an Extended Entity Manager
@Stateful
public class DepartmentManager {

	@PersistenceContext(unitName="EmployeeService",
		type=PersistenceContextType.EXTENDED)
	EntityManager em;
	Department dept;
	
	public void init(int deptId) {
		dept = em.find(Department.class, deptId);
	}
	
	public void setName(String name) {
		dept.setName(name);
	}

	public void addEmployee(int empId) {
		Employee emp = em.find(Employee.class, empId);
		dept.getEmployees().add(emp);
		emp.setDepartment(dept);
	}

	// ...
	@Remove
	public void finished() {
	}
}

has a special type attribute that can be set to either TRANSACTION or EXTENDED. These constants are defined by the PersistenceContextType enumerated type. TRANSACTION is the default and corresponds to the transaction-scoped entity managers we have been using up to now. EXTENDED means that an extended entity manager should be used. With this change made, the department manager bean now works as expected. Extended entity managers create a persistence context when a stateful session bean instance is created that lasts until the bean is removed. Unlike the persistence context of a transaction-scoped entity manager, which begins when the transaction begins and lasts until the end of a transaction, the persistence context of an extended entity manager will last for the entire length of the conversation. Because the Department entity is still managed by the same persistence context, whenever it is used in a transaction any changes will be automatically written to the database.

### Application-Managed Entity Managers

The entity manager in that example, and any entity manager that is created from the createEntityManager() call of an EntityManagerFactory instance, is what we call an application-managed entity manager. This name comes from the fact that the application, rather than the container, manages the lifecycle of the entity manager. Note that all open entity managers, whether container-managed or application-managed, are associated with an EntityManagerFactory instance. The factory used to create the entity manager can be accessed from the getEntityManagerFactory() call on the EntityManager interface.

Creating an application-managed entity manager is simple enough. All you need is an EntityManagerFactory to create the instance. What separates Java SE and Java EE for application-managed entity managers is not how you create the entity manager but how you get the factory. Listing 6-5 demonstrates use of the Persistence class to bootstrap an EntityManagerFactory instance that is then used to create an entity manager.

Listing 6-5. Application-Managed Entity Managers in Java SE
public class EmployeeClient {

	public static void main(String[] args) {
		
		EntityManagerFactory emf = Persistence.createEntityManagerFactory("EmployeeService");
		EntityManager em = emf.createEntityManager();
		List<Employee> emps = em.createQuery("SELECT e FROM Employee e").getResultList();

		for (Employee e : emps) {
			System.out.println(e.getId() + ", " + e.getName());
		}

		em.close();
		emf.close();
	}
}

The Persistence class offers two variations of the same createEntityManager() method that can be used to create an EntityManagerFactory instance for a given persistence unit name. The first, specifying only the persistence unit name, returns the factory created with the default properties defined in the persistence.xml file. The second form of the method call allows a map of properties to be passed in, adding to, or overriding the properties specified in persistence.xml. This form is useful when required JDBC properties might not be known until the application is started, perhaps with information provided as command-line parameters. The set of active properties for an entity manager can be determined via the getProperties() method on the EntityManager interface. We will discuss persistence unit properties in Chapter 14.


The best way to create an application-managed entity manager in Java EE is to use the @PersistenceUnit annotation to declare a reference to the EntityManagerFactory for a persistence unit. Once acquired, the factory can be used to create an entity manager, which can be used just as it would in Java SE. Listing 6-6 demonstrates injection of an EntityManagerFactory into a servlet and its use to create a short-lived entity manager in order to verify a user id. 

Listing 6-6. Application-Managed Entity Managers in Java EE
public class LoginServlet extends HttpServlet {
	
	@PersistenceUnit(unitName="EmployeeService")
	EntityManagerFactory emf;
	
	protected void doPost(HttpServletRequest request,HttpServletResponse response) {
		String userId = request.getParameter("user");
		// check valid user
		EntityManager em = emf.createEntityManager();
		try {
			User user = em.find(User.class, userId);
			if (user == null) {
				// return error page
				// ...
			}
		} finally {
			em.close();	
		}
		// ...
	}
}

One thing common to both of these examples is that the entity manager is explicitly closed with the close() call when it is no longer needed. This is one of the lifecycle requirements of an entity manager that must be performed manually in the case of application-managed entity managers; it is normally taken care of automatically by container-managed entity managers. Likewise, the EntityManagerFactory instance must also be closed, but only in the Java SE application. In Java EE, the container closes the factory automatically, so no extra steps are required.

If resource-local transactions are required for an operation, an application-managed entity manager is the only type of entity manager that can be configured with that transaction type within the server

## Transaction Management

There are two transaction management types supported by JPA. The first is resource-local transactions, which are the native transactions of the JDBC drivers that are referenced by a persistence unit. The second transaction management type is JTA transactions, which are the transactions of the Java EE server, supporting multiple participating resources, transaction lifecycle management, and distributed XA transactions. Container-managed entity managers always use JTA transactions, while application-managed entity managers can use either type. Because JTA is typically not available in Java SE applications, the provider needs to support only resource-local transactions in that environment. The default and preferred transaction type for Java EE applications is JTA.

### JTA Transaction Management

Transaction synchronization is the process by which a persistence context is registered with a transaction so that the persistence context can be notified when a transaction commits. The provider uses this notification to ensure that a given persistence context is correctly flushed to the database. 

Transaction association is the act of binding a persistence context to a transaction. You can also think of this as the active persistence context within the scope of that transaction. 

Transaction propagation is the process of sharing a persistence context between multiple container-managed entity managers in a single transaction.

There can be only one persistence context associated with and propagated across a JTA transaction. All container-managed entity managers in the same transaction must share the same propagated persistence context.

#### Transaction-Scoped Persistence Contexts

It is created by the container during a transaction and will be closed when the transaction completes. Transaction-scoped entity managers are responsible for creating transaction-scoped persistence contexts automatically when needed. We say only when needed because transaction-scoped persistence context creation is lazy. An entity manager will create a persistence context only when a method is invoked on the entity manager and when there is no persistence context available.

If one exists, the entity manager uses this persistence context to carry out the operation. If one does not exist, the entity manager requests a new persistence context from the persistence provider and then marks this new persistence context as the propagated persistence context for the transaction before carrying out the method call.

To demonstrate propagation of a transaction-scoped persistence context, we introduce an audit service bean that stores information about a successfully completed transaction. Listing 6-7 shows the complete bean implementation. The logTransaction() method ensures that an employee id is valid by attempting to find the employee using the entity manager.

Listing 6-7. AuditService Session Bean
@Stateless
public class AuditService {
	
	@PersistenceContext(unitName="EmployeeService")
	EntityManager em;
	
	public void logTransaction(int empId, String action) {
		// verify employee number is valid
		if (em.find(Employee.class, empId) == null) {
			throw new IllegalArgumentException("Unknown employee id");
		}
		LogRecord lr = new LogRecord(empId, action);
		em.persist(lr);
	}
}

Now consider the fragment from the EmployeeService session bean example shown in Listing 6-8. After an employee is created, the logTransaction() method of the AuditService session bean is invoked to record the “created employee” event.

Listing 6-8. Logging EmployeeService Transactions
@Stateless
public class EmployeeService {
	
	@PersistenceContext(unitName="EmployeeService")
	EntityManager em;
	
	@EJB AuditService audit;
	public void createEmployee(Employee emp) {
		em.persist(emp);
		audit.logTransaction(emp.getId(), "created employee");
	}
	// ...
}

Even though the newly created Employee is not yet in the database, the audit bean can find the entity and verify that it exists. This works because the two beans are actually sharing the same persistence context. The transaction attribute of the createEmployee() method is REQUIRED by default because no attribute has been explicitly set. The container will guarantee that a transaction is started before the method is invoked. When persist() is called on the entity manager, the container checks to see whether a persistence context is already associated with the transaction. Let’s assume in this case that this was the first entity manager operation in the transaction, so the container creates a new persistence context and marks it as the propagated one.

When the logTransaction() method starts, it issues a find() call on the entity manager from the AuditService. We are guaranteed to be in a transaction because the transaction attribute is also REQUIRED, and the containermanaged transaction from createEmployee() has been extended to this method by the container. When the find() method is invoked, the container again checks for an active persistence context. It finds the one created in the createEmployee() method and uses that persistence context to search for the entity. Because the newly created Employee instance is managed by this persistence context, it is returned successfully.

Now consider the case where logTransaction() has been declared with the REQUIRES_NEW transaction attribute instead of the default REQUIRED. Before the logTransaction() method call starts, the container will suspend the transaction inherited from createEmployee() and start a new transaction. When the find() method is invoked on the entity manager, it will check the current transaction for an active persistence context only to determine that one does not exist. A new persistence context will be created starting with the find() call, and this persistence context will be the active persistence context for the remainder of the logTransaction() call. Because the transaction started in createEmployee() has not yet committed, the newly created Employee instance is not in the database and therefore is not visible to this new persistence context. The find() method will return null, and the logTransaction() method will throw an exception as a result.

#### Extended Persistence Contexts

The lifecycle of an extended persistence context is tied to the stateful session bean to which it is bound. The stateful session bean is associated with a single extended persistence context that is created when the bean instance is created and closed when the bean instance is removed. This has implications for both the association and propagation characteristics of the extended persistence context.

Transaction association for extended persistence contexts is eager. In the case of container-managed transactions, as soon as a method call starts on the bean, the container automatically associates the persistence context with the transaction. Likewise in the case of bean-managed transactions, as soon as UserTransaction.begin() is invoked within a bean method, the container intercepts the call and performs the same association.

Because a transaction-scoped entity manager will use an existing persistence context associated with the transaction before it will create a new persistence context, it is possible to share an extended persistence context with other transaction-scoped entity managers. As long as the extended persistence context is propagated before any transaction-scoped entity managers are accessed, the same extended persistence context will be shared by all components.

Similar to the auditing EmployeeService bean demonstrated in Listing 6-8, consider the same change made to a DepartmentManager stateful session bean to audit when an employee is added to a department. Listing 6-9 shows
this example.

Listing 6-9. Logging Department Changes
@Stateful
public class DepartmentManager {
	
	@PersistenceContext(unitName="EmployeeService",
		type=PersistenceContextType.EXTENDED)
	EntityManager em;

	Department dept;
	
	@EJB AuditService audit;
	public void init(int deptId) {
		dept = em.find(Department.class, deptId);
	}
	
	public void addEmployee(int empId) {
		Employee emp = em.find(Employee.class, empId);
		dept.getEmployees().add(emp);
		emp.setDepartment(dept);
		audit.logTransaction(emp.getId(), "added to department " + dept.getName());
	}
	// ...
}

The addEmployee() method has a default transaction attribute of REQUIRED. Because the container eagerly associates extended persistence contexts, the extended persistence context stored on the session bean will be immediately associated with the transaction when the method call starts. This will cause the relationship between the managed Department and Employee entities to be persisted to the database when the transaction commits. It also means that the extended persistence context will now be shared by other transaction-scoped persistence contexts used in methods called from addEmployee().

The logTransaction() method in this example will inherit the transaction context from addEmployee() because its transaction attribute is the default REQUIRED, and a transaction is active during the call to addEmployee(). When the find() method is invoked, the transaction-scoped entity manager checks for an active persistence context and will find the extended persistence context from the DepartmentManager. It will then use this persistence context to execute the operation. All the managed entities from the extended persistence context become visible to the transaction-scoped entity manager.

##### Persistence Context Collision

We said earlier that only one persistence context could be propagated with a JTA transaction. We also said that the extended persistence context would always try to make itself the active persistence context. This can lead to situations in which the two persistence contexts collide with each other. Consider, for example, that a stateless session bean with a transaction-scoped entity manager creates a new persistence context and then invokes a method on a stateful session bean with an extended persistence context. During the eager association of the extended persistence context, the container will check to see whether there is already an active persistence context. If there is, it must be the same as the extended persistence context that it is trying to associate, or an exception will be thrown. In this example, the stateful session bean will find the transaction-scoped persistence context created by the stateless session bean, and the call into the stateful session bean method will fail. There can be only one active persistence context for a transaction. Figure 6-1 illustrates this case.

While extended persistence context propagation is useful if a stateful session bean with an extended persistence context is the first EJB to be invoked in a call chain, it limits the situations in which other components can call into the stateful session bean if they are also using entity managers. This might or might not be common depending on your application architecture, but it is something to keep in mind when planning dependencies between components. One way to work around this problem is to change the default transaction attribute for the stateful session bean that uses the extended persistence context. If the default transaction attribute is REQUIRES_NEW, any active transaction will be suspended before the stateful session bean method starts, allowing it to associate its extended persistence context with the new transaction. This is a good strategy if the stateful session bean calls in to other stateless session beans and needs to propagate the persistence context. Note that excessive use of the REQUIRES_NEW transaction attribute can lead to application performance problems because many more transactions than normal will be created, and active transactions will be suspended and resumed.

If the stateful session bean is largely self-contained; that is, it does not call other session beans and does not need its persistence context propagated, a default transaction attribute type of NOT_SUPPORTED can be worth considering. In this case, any active transaction will be suspended before the stateful session bean method starts, but no new transaction will be started. If there are some methods that need to write data to the database, those methods can be overridden to use the REQUIRES_NEW transaction attribute. 

Listing 6-10 repeats the DepartmentManager bean, this time with some additional getter methods and customized transaction attributes. We have set the default transaction attribute to REQUIRES_NEW to force a new transaction by default when a business method is invoked. For the getName() method, we don’t need a new transaction because no changes are being made, so it has been set to NOT_SUPPORTED. This will suspend the current transaction, but won’t result in a new transaction being created. With these changes, the DepartmentManager bean can be accessed in any situation, even if there is already an active persistence context.

Listing 6-10. Customizing Transaction Attributes to Avoid Collision

@Stateful
@TransactionAttribute(TransactionAttributeType.REQUIRES_NEW)
public class DepartmentManager {

	@PersistenceContext(unitName="EmployeeService",
		type=PersistenceContextType.EXTENDED)
	EntityManager em;
	
	Department dept;
	
	@EJB AuditService audit;
	public void init(int deptId) {
		dept = em.find(Department.class, deptId);
	}
	
	@TransactionAttribute(TransactionAttributeType.NOT_SUPPORTED)
	public String getName() { return dept.getName(); }
	public void setName(String name) { dept.setName(name); }
	
	public void addEmployee(int empId) {
		Employee emp = em.find(empId, Employee.class);
		dept.getEmployees().add(emp);
		emp.setDepartment(dept);
		audit.logTransaction(emp.getId(), "added to department " + dept.getName());
	}
	// ...
}

##### Persistence Context Inheritance 

In our example, clients worked with a DepartmentManager session bean instead of the actual Department entity instance. Because a department has a manager, it makes sense to extend this façade to the Employee entity as well. Listing 6-11 shows changes to the DepartmentManager bean so that it returns an EmployeeManager bean from the getManager() method in order to represent the manager of the department. The employeeManager bean is injected and then initialized during the invocation of the init() method. 

Listing 6-11. Creating and Returning a Stateful Session Bean
@Stateful
public class DepartmentManager {
	@PersistenceContext(unitName="EmployeeService",
		type=PersistenceContextType.EXTENDED)
	EntityManager em;
	
	Department dept;

	@EJB EmployeeManager manager;
	public void init(int deptId) {
		dept = em.find(Department.class, deptId);
		manager.init();
	}

	public EmployeeManager getManager() {
		return manager;
	}
	// ...
}

Should the init() method succeed or fail? So far based on what we have described, it looks like it should fail. When init() is invoked on the DepartmentManager bean, its extended persistence context will be propagated with the transaction. In the subsequent call to init() on the EmployeeManager bean, it will attempt to associate its own extended persistence context with the transaction, causing a collision between the two.

Perhaps surprisingly, this example actually works. When a stateful session bean with an extended persistence context creates another stateful session bean that also uses an extended persistence context, the child will inherit the parent’s persistence context. The EmployeeManager bean inherits the persistence context from the DepartmentManager bean when it is injected into the DepartmentManager instance. The two beans can now be used together within the same transaction.

#### Application-Managed Persistence Contexts

Like container-managed persistence contexts, application-managed persistence contexts can be synchronized with JTA transactions. Synchronizing the persistence context with the transaction means that a flush will occur if the transaction commits, but the persistence context will not be considered associated by any container-managed entity managers. There is no limit to the number of application-managed persistence contexts that can be synchronized with a transaction, but only one container-managed persistence context will ever be associated. This is one of the most important differences between application-managed and container-managed entity managers. An application-managed entity manager participates in a JTA transaction in one of two ways. If the persistence context is created inside the transaction, the persistence provider will automatically synchronize the persistence context with the transaction. If the persistence context was created earlier (outside of a transaction or in a transaction
that has since ended), the persistence context can be manually synchronized with the transaction by calling joinTransaction()on the EntityManager interface. Once synchronized, the persistence context will automatically be flushed when the transaction commits.

Listing 6-12 shows a variation of the DepartmentManager from Listing 6-11 that uses an application-managedentity manager instead of an extended entity manager.

Listing 6-12. Using Application-Managed Entity Managers with JTA
@Stateful
public class DepartmentManager {
	
	@PersistenceUnit(unitName="EmployeeService")
	EntityManagerFactory emf;
	
	EntityManager em;
	
	Department dept;
	
	public void init(int deptId) {
		em = emf.createEntityManager();
		dept = em.find(Department.class, deptId);
	}
	
	public String getName() {
		return dept.getName();
	}
	
	public void addEmployee(int empId) {
		em.joinTransaction();
		Employee emp = em.find(Employee.class, empId);
		dept.getEmployees().add(emp);
		emp.setDepartment(dept);
	}
	
	// ...
	@Remove
	public void finished() {
		em.close();
	}
}

Instead of injecting an entity manager, we are injecting an entity manager factory. Prior to searching for the entity, we manually create a new application-managed entity manager using the factory. Because  the container does not manage its lifecycle, we have to close it later when the bean is removed during the call to finished(). Like the container-managed extended persistence context, the Department entity remains managed after the call to init(). When addEmployee() is called, there is the extra step of calling joinTransaction() to notify the persistence context that it should synchronize itself with the current JTA transaction. Without this call, the changes to the department would not be flushed to the database when the transaction commits.

Because application-managed entity managers do not propagate, the only way to share managed entities with other components is to share the EntityManager instance. This can be achieved by passing the entity manager around as an argument to local methods or by storing the entity manager in a common place such as an HTTP session or singleton bean. Listing 6-13 demonstrates a servlet creating an application-managed entity manager and using it to instantiate the EmployeeService class we defined in Chapter 2. In these cases, care must be taken to ensure that access to the entity manager is done in a thread-safe manner. While EntityManagerFactory instances are thread-safe, EntityManager instances are not. Also, application code must not call joinTransaction() on the same entity manager in multiple concurrent transactions.

Listing 6-13. Sharing an Application-Managed Entity Manager
public class EmployeeServlet extends HttpServlet {
	@PersistenceUnit(unitName="EmployeeService")
	EntityManagerFactory emf;
	
	@Resource UserTransaction tx;
	protected void doPost(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
		
		// ...
		int id = Integer.parseInt(request.getParameter("id"));
		String name = request.getParameter("name");
		long salary = Long.parseLong(request.getParameter("salary"));
		tx.begin();
		
		EntityManager em = emf.createEntityManager();
		try {
			EmployeeService service = new EmployeeService(em);
			service.createEmployee(id, name, salary);
		} finally {
			em.close();
		}
		tx.commit();
		// ...

	}
}
Listing 6-13 demonstrates an additional characteristic of the application-managed entity manager in the presence of transactions. If the persistence context becomes synchronized with a transaction, changes will still be written to the database when the transaction commits, even if the entity manager is closed. This allows entity managers to be closed at the point where they are created, removing the need to worry about closing them after the transaction ends. Note that closing an application-managed entity manager still prevents any further use of the entity manager. It is only the persistence context that continues until the transaction has completed. 
There is a danger in mixing multiple persistence contexts in the same JTA transaction. This occurs when multiple application-managed persistence contexts become synchronized with the transaction or when application-managed persistence contexts become mixed with container-managed persistence contexts. When the transaction commits, each persistence context will receive notification from the transaction manager that changes should be written to the database. This will cause each persistence context to be flushed.
What happens if an entity with the same primary key is used in more than one persistence context? Which version of the entity gets stored? The unfortunate answer is that there is no way to know for sure. The container does not guarantee any ordering when notifying persistence contexts of transaction completion. As a result, it is critical for data integrity that entities never be used by more than one persistence context in the same transaction. When designing your application, we recommend picking a single persistence context strategy (container-managed or application-managed) and sticking to that strategy consistently.

#### Unsynchronized Persistence Contexts

### Resource-Local Transactions

### Transaction Rollback and Entity State

## Choosing an Entity Manager

## Entity Manager Operations

### Persisting an Entity

### Finding an Entity

### Removing an Entity

### Cascading Operations

#### Cascade Persist

#### Cascade Remove

### Clearing the Persistence Context

### Synchronization with the database 

### Detachment and Merging

#### Detachment

#### Merging detached entities

#### Working with Detached Entities

##### Planning for Detachment

###### Triggering Lazy Loading

###### Configuring Eager Loading

##### Avoiding Detachment

###### Transaction view

###### Entity Manager per Request

##### Merge Strategies

###### Session Façade

###### Edit Session

###### Conversation