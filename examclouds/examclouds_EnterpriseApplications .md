http://examclouds.com/

# Enterprise Applications 

1. ***Types of session beans :*** There are three types of session bean: stateless, stateful, and singleton.

2. ***Interaction with stateless session bean :*** Interaction with a stateless session bean begins at the start of a business method call and ends when the method call completes. There is no state that carries over from one business operation to the other.

3. ***Interaction with stateful session bean :*** An interaction with stateful session beans becomes more of a conversation that begins from the moment the client acquires a reference to the session bean and ends when the client explicitly releases it back to the server. Business operations on a stateful session bean can maintain state on the bean instance across calls.

4. ***When singleton bean was introduced?*** Introduced in EJB 3.1.

5. ***Feature of singleton session bean in terms of comparing it with other session beans, concurrency :*** Singleton session beans can be considered a hybrid of stateless and stateful session beans. All clients share the same singleton bean instance, so it becomes possible to share state across method invocations, but singleton session beans lack the conversational contract and mobility of stateful session beans. State on a singleton session bean also raises issues of concurrency that need to be taken into consideration when deciding whether or not to use this style of session bean.

6. ***How clients interact with beans?*** Clients never interact directly with a session bean instance. The client references and invokes an implementation of the business interface provided by the server. This implementation class acts as a proxy to the underlying bean implementation. 

7. ***What are the benefits of decoupling the client from the bean?*** It allows the server to intercept method calls in order to provide the services required by the bean, such as transaction management. It also allows the server to optimize and reuse instances of the session bean class as necessary.

8. ***How many interfaces can stateless and stateful session bean implement?*** Zero or more business interfaces that define what methods a client can invoke on the bean.

9. ***What if interface isn't specified?*** When no interface is defined then the set of public methods on the bean implementation class forms a logical client interface.

10. ***In which packages @Stateless annotation is defined?*** The @Stateless annotation are defined in either the javax.ejb or javax.annotation package.

11. ***Types of interfaces.*** Local and remote

12. ***Feature of local business interface.*** Implementation is accessible only to clients within the same application server.

13. ***How local interface is defined?*** With @Local annotation or without annotations.

14. ***What is the no-interface view?*** The no-interface view was introduced in EJB 3.1 to make it simpler to define a local session bean and for clients to access local session beans. To define bean with a no-interface view, the bean developer creates only the implementation class without implementing any business interface. The logical interface of the session bean consists of its public methods.

15. ***The requirements about the stateless session bean class definition.*** There are only a couple of caveats about the stateless session bean class definition. The first is that it needs a no-arg constructor, but the compiler normally generates this automatically when no other constructors are supplied. The second is that static fields should not be used, primarily because of bean redeployment issues.

16. ***Why stateless session beans should not use state?*** Many EJB containers create a pool of stateless session bean instances and then select an arbitrary instance to service each client request. Therefore, there is no guarantee that the same state will be used between calls, and hence it cannot be relied on. Any state placed on the bean class should be restricted to factory classes that are inherently stateless, such as DataSource.

17. ***What does manage the lifecycle of a stateless session bean?***  The server.


18. ***Which concurrency types can use Singleton session beans?*** Singleton session beans can use container-managed or bean-managed concurrency.

19. ***The default concurrency for Singleton session beans.*** The default is container-managed.

20. ***Why the constructor for the bean class is not very useful?*** The bean might have to acquire a resource such as a JDBC data source before business methods can be used. However, in order for the bean to acquire a resource, the server must first have completed initializing its services for the bean. This limits the usefulness of the constructor for the class because the bean won't have access to any resources until server initialization has completed.

21. ***Who decides when session bean is created and when deleted?*** The server decides when to create and remove bean instances. The application has no control over when or even how many instances of a particular stateless session bean are created or how long they will stay around.

22. ***When the services for the session bean are created?*** The server has to initialize services for the bean after it is constructed, but before the business logic of the bean is invoked.

23. ***How to allow both the server and the bean to achieve their initialization requirements?*** To allow both the server and the bean to achieve their initialization requirements, EJBs support lifecycle callback methods that are invoked by the server at various points in the bean's lifecycle.

24. ***Lifecycle callbacks for stateless session beans.*** For stateless session beans, there are two lifecycle callbacks: PostConstruct and PreDestroy.

25. ***When does server invoke the PostConstruct callback?*** The server will invoke the PostConstruct callback as soon as it has completed initializing all the container services for the bean. In effect, this replaces the constructor as the location for initialization logic because it is only here that container services are guaranteed to be available.

26. ***When does server invoke the PreDestroy callback?*** The server invokes the PreDestroy callback immediately before the server releases the bean instance to be garbage collected. Any resources acquired during PostConstruct that require explicit shutdown should be released during PreDestroy.

27. ***How many PostConstruct and PreDestroy callbacks can have stateless session bean?*** At most one.

28. ***What means local business interface?*** Local in this case means that a dependency on the session bean can be declared only by Java EE components that are running together in the same application server instance. It is not possible to use a session bean with a local interface from a remote client, for example.

29. ***What is remote business interface? How to make interface to be remote(2 options)?*** To accommodate remote clients, session beans can mark their business interface with the @Remote annotation to declare that it should be useable remotely. Marking an interface as being remote is equivalent to having it extend the java.rmi.Remote interface. The reference to the bean that gets acquired by a client is no longer a local reference on the server but a Remote Method Invocation (RMI) stub that will invoke operations on the session bean from across the network. No special support is required on the bean class to use remote interfaces.

30. ***Making an interface remote has consequences both in terms of performance and how arguments to business methods are handled. Describe this.*** Remote business interfaces can be used locally within a running server, but doing so might still result in network overhead if the method call is routed through the RMI layer. Arguments to methods on remote interfaces are also passed by value instead of passed by reference. This means that the argument is serialized even when the client is local to the session bean.

31. ***Why local interfaces for local clients are generally a better approach?*** Local interfaces preserve the semantics of regular Java method calls and avoid the costs associated with networking and RMI.

32. ***To which lock corresponds container-managed concurrency?*** It corresponds to a write lock on all business methods. All business method invocations are serialized so that only one client can access the bean at any given time.

33. ***Why the JNDI API look up is a somewhat cumbersome method of finding a resource?*** It is a somewhat cumbersome method of finding a resource because of the exception-handling requirements of JNDI.

34. ***Alternative syntax using the lookup() method on the EJB.*** EJBs also support an alternative syntax using the lookup() method of the EJBContext interface.

35. ***Differences of stateful session bean from stateless.*** 
    * The stateful bean class has state fields that are modified by the business methods of the bean. This is allowed because the client that uses the bean effectively has access to a private instance of the session bean on which to make changes. 
    * There are methods marked with the @Remove annotation. These are the methods that the client will use to end the conversation with the bean. After one of these methods has been called, the server will destroy the bean instance.

36. ***How many methods marked with the @Remove annotation can have stateful session bean?*** Every stateful session bean must define at least one method marked with the @Remove annotation, even if the method doesn't do anything other than serve as an end to the conversation.

37. ***Callback methods of stateful session bean.*** @PostConstruct, @PrePassivate, @PostActivate, @PreDestroy

38. ***What is a passivation process?*** Passivation is the process by which the server serializes the bean instance so that it can either be stored offline to free up resources or replicated to another server in a cluster.

39. ***What is an activation process?*** Activation is the process of deserializing a passivated session bean instance and making it active in the server once again.

40. ***When PrePassivate callback is invoked and what for?*** Before a bean is passivated, the server will invoke the PrePassivate callback. The bean uses this callback to prepare the bean for serialization, usually by closing any live connections to other server resources.

41. ***When PostActivate callback is invoked and what for?*** After a bean has been activated, the server will invoke the PostActivate callback. With the serialized instance restored, the bean must then reacquire any connections to other resources that the business methods of the bean might be depending on. Singleton session beans share the same lifecycle callbacks as a stateless session bean and server-managed resources such as persistence contexts behave the same as if they were part of a stateless session bean.

42. ***A key difference of singleton from other session bean in terms of creation and destroying.*** Unlike other session beans, the singleton can be created eagerly during application initialization and exist until the application shuts down. Once created, it will continue to exist until the container removes it.

43. ***If an exception occurs in singleton bean, will it continue to exists?*** Once created, it will continue to exist until the container removes it,regardless of any exceptions that occur during business method execution. This is a key difference from other session bean types because the bean instance will never be re-created in the event of a system exception.

44. ***If an exception occurs in stateless or stateful bean, will it continue to exists?*** Session bean types will never be re-created in the event of a system exception.

45. ***What is usually stored in singleton?*** The long life and shared instance of the singleton session bean make it the ideal place to store common application state, whether read-only or read-write. To safeguard access to this state, the singleton session bean provides a number of concurrency options depending on the needs of the application developer. Methods can be completely unsynchronized for performance, or automatically locked and managed by the container.   

46. ***Which interfaces uses singleton?*** Singleton session beans can include a local business interface or use a no-interface view.

47. ***Can container create singletons lazily if they do not specify eager?*** The container can create singletons that do not specify eager initialization lazily, but this is vendor-specific and cannot be assumed.

48. ***How specify that singleton session bean depends on another singleton session bean?*** When multiple singleton session beans depend on one another, the container needs to be informed of the order in which they should be instantiated. This is accomplished via the @DependsOn annotation on the bean class, which lists the names of other singleton session beans that must be created first.

49. ***The lifecycle callbacks for singleton session beans.*** The lifecycle callbacks for singleton session beans are the same as for stateless session beans: PostConstruct and PreDestroy.

50. ***The difference in PreDestroy for stateless session beans and singletons.*** The key difference here with respect to stateless session beans is that PreDestroy is invoked only when the application shuts down as a whole. It will therefore be called only once, whereas the lifecycle callbacks of stateless session beans are called frequently as bean instances are created and destroyed.

51. ***When @Lock(LockType.READ) is used?*** Not all business methods change the state of the bean. Those that do not can be safely run in a concurrent fashion without affecting the overall integrity of the bean. If there is no danger of corrupting the bean state through concurrent access, the @Lock(LockType.READ) annotation can be used to declare that such access is safe and places a read lock on the method.

52. ***The default business method behavior in terms of lock.*** Write locks.

53. ***Where lock annotation can be used?*** On class and method.

54. ***The way a lock works.*** Although locking semantics are declared in terms of business methods, conceptually the bean can be thought of as having a single lock on the instance. Business methods will acquire either read or write access on this lock depending on their @Lock declaration or default value. In terms of semantics, multiple readers are allowed to proceed concurrently, but as soon as a write lock is acquired, all other clients block until the write operation completes.

55. ***Example of declaring a lock.*** @Lock(LockType.READ)

56. ***How the singleton session bean can use bean-managed concurrency?*** For those who wish to have fine-grained control over concurrency, the singleton session bean can be configured to use bean-managed concurrency via the @ConcurrencyManagement(ConcurrencyManagementType.BEAN) annotation on the bean class. This effectively disables the container-managed concurrency and relies on the developer to use the appropriate Java concurrency primitives to ensure data safety. 

57. ***Does the @Lock annotation have effect for bean-managed concurrency?*** The @Lock annotation has no effect for bean-managed concurrency.

58. ***There are a number of cases in which bean-managed concurrency might be preferable to container-managed concurrency. Describe them.*** If the singleton session bean has no state, or if state operations are restricted to a small subset of methods, bean-managed concurrency will yield better performance. When container-managed concurrency is enabled, all business methods involve a lock of some kind, whether state operations are involved or not. Multiple sets of mutually exclusive state on the bean are also a candidate for bean-managed concurrency. With container-managed concurrency, only one write lock can be held at any time across all business methods. But if there are sets of state that are mutually exclusive, it might be safe to execute concurrent writes across different sets. Bean-managed concurrency with developer maintained locks will again yield better performance. Alternatively, refactoring the bean into multiple singleton session beans each focused on a single type of state will also improve the performance of container-managed concurrency.

59. ***What is the message-driven bean (MDB)?*** The message-driven bean (MDB) is the EJB component for asynchronous messaging. 

60. ***How clients requests to the MDB?*** Clients issue requests to the MDB using a messaging system such as Java Message Service (JMS). These requests are queued and eventually delivered to the MDB by the server. The server invokes the business interface of the MDB whenever it receives a message sent from a client. 

61. ***How the component contract of an MDB is defined?*** Although the component contract of a session bean is defined by its business interface, the component contract of an MDB is defined by the structure of the messages it is designed to receive.

62. ***Which interface MDB implements?*** In the case of message-driven beans, the bean class implements an interface specific to the messaging system the MDB is based on. The most common case is JMS, but other messaging systems are possible with the Java Connector Architecture (JCA). For JMS message-driven beans, the business interface is javax.jms.MessageListener, which defines a single method: onMessage().

63. ***How to mark the class as an MDB?*** The @MessageDriven annotation marks the class as an MDB.

64. ***How the activation configuration properties defined?*** The activation configuration properties is defined using the @ActivationConfigProperty annotations.

65. ***What activation configuration properties defined for?*** To tell the server the type of messaging system and any configuration details required by that system.

66. ***The example of MDB.*** 
67. 
		@MessageDriven( 
			activationConfig = { 
				@ActivationConfigProperty(
					propertyName="destinationType", 
					propertyValue="javax.jms.Queue"), 
				@ActivationConfigProperty(
					propertyName="messageSelector", 
					propertyValue="RECIPIENT=`ReportProcessor`") 
			}
		) 
		public class ReportProcessorBean implements javax.jms.MessageListener {
			public void onMessage(javax.jms.Message message) { // ... } 
		}

67. ***When the MDB from the previous example will be invoked?*** This MDB will be invoked only if the JMS message has a property named RECIPIENT in which the value is ReportProcessor.

68. ***What happens when MDB receives a message?*** Whenever the server receives a message, it invokes the onMessage() method with the message as the argument. Because there is no synchronous connection with a client, the onMessage() method does not return anything. However, the MDB can use session beans, data sources, or even other JMS resources to process and carry out an action based on the message.

69. ***What for Java EE components support the notion of references to resources that are defined in metadata for the component?*** The business logic of a Java EE component is not always self-contained. More often than not, the implementation depends on other resources hosted by the application server. This might include server resources such as a JDBC data source or JMS message queue, or application-defined resources such as a session bean or entity manager for a specific persistence unit. To manage these dependencies, Java EE components support the notion of references to resources that are defined in metadata for the component.

70. ***What is a reference?*** A reference is a named link to a resource.

71. ***How the reference can be resolved?*** Reference can be resolved dynamically at runtime from within application code or resolved automatically by the container when the component instance is created.

72. ***Which parts a reference consists of?*** A reference consists of two parts: a name and a target.

73. ***What for reference name is used?*** The name is used by application code to resolve the reference dynamically.

74. ***What for reference target is used?*** The server uses target information to find the resource the application is looking for.

75. ***What determines the type of resource to be located?*** The type of resource to be located determines the type of information required to match the target. Each resource reference requires a different set of information specific to the resource type it refers to.

76. ***Which annotations are used to declare reference?*** A reference is declared using one of the resource reference annotations: @Resource, @EJB, @PersistenceContext, or @PersistenceUnit

77. ***Where reference annotations can be placed?*** These annotations can be placed on a class, field or setter method. 

78. ***What the choice of location for reference determines?*** The choice of location determines the default name of the reference, and whether or not the server resolves the reference automatically.

79. ***What is a dependency lookup?*** Strategy for resolving dependencies in application code. This is the traditional form of dependency management in Java EE, in which the application code is responsible for using the Java Naming and Directory Interface (JNDI) to look up a named reference.

80. ***Which attribute all the resource annotations support?*** All the resource annotations support an attribute called name that defines the name of the reference.

81. ***When the name attribute for the resource annotation is mandatory?*** When the resource annotation is placed on the class definition, this attribute is mandatory.

82. ***When the server will generate a default name for reference?*** If the resource annotation is placed on a field or a setter method, the server will generate a default name.

83. ***When using dependency lookup, where annotations are typically placed?*** When using dependency lookup, annotations are typically placed at the class level, and the name is explicitly specified.

84. ***The role of the name in the reference?*** The role of the name is to provide a way for the client to resolve the reference dynamically.

85. ***What is environment naming context?*** Every Java EE application server supports JNDI, and each component has its own locally scoped JNDI naming context called the environment naming context. The name of the reference is bound into the environment naming context, and when it is looked up using the JNDI API, the server resolves the reference and returns the target of the reference.

86. ***Declare a dependency to the bean on a session bean on the class level.*** 

	    @Stateless
		@EJB(
			name="audit", 
			beanInterface=AuditService.class) 
		public class DeptServiceBean implements DeptService {...}

87. ***What for beanInterface element of the @EJB annotation is used?*** The beanInterface element of the @EJB annotation references the business interface of the session bean that the client is interested in. 

88. ***How to retrieve objects from a JNDI context?*** The lookup() method of the Context interface is the primary way to retrieve objects from a JNDI context.

89. ***What for the prefix 'java:comp/env/' is added?*** The prefix 'java:comp/env/' that was added to the reference name indicates to the server that the environment naming context should be searched to find the reference.

90. ***What happens if the name is incorrectly specified for lookup?*** If the name is incorrectly specified, a NamingException exception will be thrown when the lookup fails.

91. ***Which components support using the JNDI API to look up resource references from the environment naming context?*** Using the JNDI API to look up resource references from the environment naming context is supported by all Java EE components.

92. ***Subinterfaces of EJBContext.*** SessionContext and MessageDrivenContext.

93. ***Feature of EJBContext*** The EJBContext interface is available to any EJB and provides the bean with access to runtime services such as the timer service.

94. ***The advantages of EJBContext lookup() method over the JNDI API.*** The EJBContext lookup() method has two advantages over the JNDI API. The first is that the argument to the method is the name exactly as it was specified in the resource reference. The second is that only runtime exceptions are thrown from the lookup() method so the checked exception handling of the JNDI API can be avoided.

95. ***What happens when a resource annotation is placed on a field or setter method?*** When a resource annotation is placed on a field or setter method, two things occur. First, a resource reference is declared just as if it had been placed on the bean class, and the name for that resource will be bound into the environment naming context when the component is created. Second, the server does the lookup automatically on your behalf and sets the result into the instantiated class.

96. ***What is dependency injection?*** The process of automatically looking up a resource and setting it into the class is called dependency injection because the server is said to inject the resolved dependency into the class.

97. ***Forms of dependency injection.*** Field injection, setter injection.

98. ***What is field injection?*** Injecting a dependency into a field means that after the server looks up the dependency in the environment naming context, it assigns the result directly into the annotated field of the class.

99. ***Example of field injection.***

		@Stateless 
		public class DeptServiceBean implements DeptService { 
			@EJB AuditService audit; 
			// ... 
		}

100. ***When a resource annotation is placed on a field or setter method, how the reference name is generated?*** The generated name is the fully qualified class name, followed by a forward slash and then the name of the field or property. 

101. ***What is the reference name if the AuditService bean is located in the persistence.session package?*** This means that if the AuditService bean is located in the persistence.session package, the injected EJB referenced would be accessible in the environment naming context under the name 'persistence.session.AuditService/audit'.

		@Stateless 
		public class DeptServiceBean implements DeptService { 
			@EJB AuditService audit; 
			// ... 
		}

102. ***Can we override default name for field injection? How?*** Specifying the name element for the resource annotation will override this default value.

103. ***What happens when the server resolves the reference while setter injection?*** When the server resolves the reference, it will invoke the annotated setter method with the result of the lookup.

104. ***Example of using Setter Injection.***

		@Stateless 
		public class DeptServiceBean implements DeptService { 

			private AuditService audit; 
			
			@EJB 
			public void setAuditService( AuditService audit ) { 
				this.audit = audit; 
			} 
	
			// ... 
		}

105. ***The benefit of setter injection.*** This style of injection allows for private fields.

106. ***What happens if the unitName element is omitted in @PersistenceContext?*** If the unitName element is omitted, it is vendor-specific how the unit name for the persistence context is determined. Some vendors can provide a default value if there is only one persistence unit for an application, whereas others might require that the unit name be specified in a vendor-specific configuration file.

107. ***Is this code legal:*** After the warnings about using a state field in a stateless session bean, you might be wondering how this code is legal. After all, entity managers must maintain their own state to be able to manage a specific persistence context. The good news is that the specification was designed with Java EE integration in mind. The value injected into the bean is a container-managed proxy that acquires and releases persistence contexts on behalf of the application code. This is a powerful feature of the Java Persistence API in Java EE.
		
		@Stateless 
		public class EmployeeServiceBean implements EmployeeService { 
		
			@PersistenceContext(unitName="EmployeeService") 
			EntityManager em; 
			// ...

108. ***How the EntityManagerFactory for a persistence unit can be referenced?*** The EntityManagerFactory for a persistence unit can be referenced using the @PersistenceUnit annotation.

109. ***What happens if the persistent unit name is not specified in the annotation?*** If the persistent unit name is not specified in the annotation, it is vendor-specific how the name is determined.
 
110. ***Do Message-driven beans have client interface?*** Message-driven beans have no client interface, so they cannot be accessed directly.

111. ***Can Message-driven beans be injected?*** They cannot be injected.

112. ***What happens when a component needs to access an EJB?*** When a component needs to access an EJB, it declares a reference to that bean with the @EJB annotation. The server will search through all deployed session beans to find the one that implements the requested business interface.

113. ***What if two session beans implement the same business interface?*** In the rare case that two session beans implement the same business interface or if the client needs to access a session bean located in a different EJB jar, then the ****beanName**** element can also be specified to identify the session bean by its name.

114. ***What is the name of a session bean?*** The name of a session bean defaults to the unqualified class name of the bean class or it can be set explicitly by using the name element of the @Stateless and @Stateful annotations.

115. ***How to specify the beanName element on the injected bean?***

		@Stateless 
		public class DeptServiceBean implements DeptService { 
			@EJB(beanName="AuditServiceBean")
			AuditService audit; 
			// ... 
		}

116. ***When the @Resource annotation is used?*** It is used to define references to resource factories, message destinations, data sources, and other server resources.

117. ***The only additional element of @Resource annotation?*** The only additional element is resourceType which allows you to specify the type of resource if the server can't figure it out automatically. For example, if the field you are injecting into is of type Object, then there is no way for the server to know that you wanted a data source instead. The resourceType element can be set to javax.sql.DataSource to make the need explicit.

118. ***What is a transaction?*** A transaction is an abstraction that is used to group together a series of operations. Once grouped together, the set of operations is treated as a single unit, and all of the operations must succeed or none of them can succeed.

119. ***Basic transaction properties?*** Atomicity, Consistency, Isolation, Durability.

120. ***Describe transaction's Atomicity property.*** Either all the operations in a transaction are successful or none of them is. The success of every individual operation is tied to the success of the entire group.

121. ***Describe transaction's Consistency property.*** The resulting state at the end of the transaction adheres to a set of rules that define acceptability of the data. The data in the entire system is legal or valid with respect to the rest of the data in the system. The consistency property ensures that any transaction the database performs will take it from one consistent state to another.

122. ***Describe transaction's Isolation property.*** Changes made within a transaction are visible only to the transaction that is making the changes. Once a transaction commits the changes, they are atomically visible to other transactions

123. ***Describe transaction's Durability property*** The changes made within a transaction endure beyond the completion of the transaction.

124. ***The name of a transaction that meets all requirements.*** A transaction that meets all these requirements is said to be an ACID transaction.

125. ***Are all transactions are ACID transactions?*** No.

126. ***Transaction levels within the enterprise application server?*** Resource-local transaction, container transaction.

127. ***What is resource-local transaction*** The lowest and most basic transaction is at the level of the resource, which in our discussion is assumed to be a relational database fronted by a DataSource interface. This is called a resource-local transaction and is equivalent to a database transaction. These types of transactions are manipulated by interacting directly with the JDBC DataSource that is obtained from the application server. Resource-local transactions are used much more infrequently than container transactions.

128. ***What is container transaction? Which resources it can enlist?*** The broader container transaction uses the Java Transaction API (JTA) that is available in every compliant Java EE application server. This is the typical transaction that is used for enterprise applications and can involve or enlist a number of resources including data sources as well as other types of transactional resources. Resources defined using Java Connector Architecture (JCA) components can also be enlisted in the container transaction.

129. ***What for containers typically add their own layer on top of the JDBC DataSource?*** Containers typically add their own layer on top of the JDBC DataSource to perform functions such as connection management and pooling that make more efficient use of the resources and provide a seamless integration with the transaction-management system. This is also necessary because it is the responsibility of the container to perform the commit or rollback operation on the data source when the container transaction completes.

130. ***Which transaction container transactions use?*** Container transactions use JTA.

131. ***How transactions can be completed?*** Transactions can be completed in one of two ways. They can be committed, causing all of the changes to be persisted to the data store, or rolled back, indicating that the changes should be discarded.

132. ***What is transaction demarcation?*** The act of causing a transaction to either begin or complete is termed transaction demarcation.

133. ***How resource-local transactions are demarcated?*** Resource-local transactions are always demarcated explicitly by the application.

134. ***How container transactions are demarcated?*** Container transactions can either be demarcated automatically by the container or by using a JTA interface that supports application-controlled demarcation.

135. ***What is container-managed transaction?*** When the container takes over the responsibility of transaction demarcation, we call it container-managed transaction management.

136. ***What is bean-managed transaction?*** When the application is responsible for demarcation, we call it bean-managed transaction management.

137. ***Which transactions EJBs can use?*** EJBs can use either container-managed transactions or bean-managed transactions.

138. ***Which transactions servlets can use?*** Servlets are limited to the somewhat poorly named bean-managed transaction. 

139. ***The default transaction management style for an EJB component?*** The default transaction management style for an EJB component is container-managed.

140. ***How to configure an EJB to have its transactions demarcated one way or the other?*** To configure an EJB to have its transactions demarcated one way or the other, the @TransactionManagement annotation should be specified on the session or message-driven bean class. The TransactionManagementType enumerated type defines BEAN for bean-managed transactions and CONTAINER for container-managed transactions.

141. ***Example of changing the Transaction Management Type of a Bean.*** 
		
		@Stateless 
		@TransactionManagement(TransactionManagementType.BEAN) 
		public class ProjectServiceBean implements ProjectService { 
			// methods in this class manually control transaction demarcation
		}

142. ***Transaction attributes choices.*** MANDATORY, REQUIRED, REQUIRES_NEW, SUPPORTS, NOT_SUPPORTED, NEVER.

143. ***Describe MANDATORY transaction attributes. When it is usually used?*** MANDATORY: If this attribute is specified for a method, a transaction is expected to have already been started and be active when the method is called. If no transaction is active, an exception is thrown. This attribute is seldom used, but can be a development tool to catch transaction demarcation errors when it is expected that a transaction should already have been started.

144. ***Describe REQUIRED transaction attribute.*** REQUIRED: This attribute is the most common case in which a method is expected to be in a transaction. The container provides a guarantee that a transaction is active for the method. If one is already active, it is used; if one does not exist, a new transaction is created for the method execution.

145. ***Describe REQUIRES_NEW transaction attribute.*** REQUIRES_NEW: This attribute is used when the method always needs to be in its own transaction; that is, the method should be committed or rolled back independently of methods further up the call stack. It should be used with caution because it can lead to excessive transaction overhead.

146. ***Describe SUPPORTS transaction attribute.*** SUPPORTS: Methods marked with supports are not dependent on a transaction, but will tolerate running inside one if it exists. This is an indicator that no transactional resources are accessed in the method.

147. ***Describe NOT_SUPPORTED transaction attribute.*** NOT_SUPPORTED: A method marked to not support transactions will cause the container to suspend the current transaction if one is active when the method is called. It implies that the method does not perform transactional operations, but might fail in other ways that could undesirably affect the outcome of a transaction. This is not a commonly used attribute.

148. ***Describe NEVER transaction attribute.*** NEVER: A method marked to never support transactions will cause the container to throw an exception if a transaction is active when the method is called. This attribute is very seldom used, but can be a development tool to catch transaction demarcation errors when it is expected that transactions should already have been completed.

149. ***How to specify transaction attribute?*** The transaction attribute for a method can be indicated by annotating a session or messagedriven bean class, or one of its methods that is part of the business interface, with the @TransactionAttribute annotation.

150. ***The default transaction attribute value?*** REQUIRED.

151. ***Example of specifying transaction attribute.***
 
		@TransactionAttribute(TransactionAttributeType.SUPPORTS) 
		public void addItem(String item, Integer quantity) { 
			verifyItem(item, quantity); 
			// ... 
		}

152. ***How to cause a container-managed transaction to roll back?***Any bean wanting to cause a container-managed transaction to roll back can do so by invoking the setRollbackOnly() method on the EJBContext object. Although this will not cause the immediate rollback of the transaction, it is an indication to the container that the transaction should be rolled back when the time comes.

153. ***When transaction should be completed for Beans that use BMT?*** Beans that use BMT must ensure that any time a transaction has been started, it must also be completed before returning from the method that started it. Failure to do so will result in the container rolling back the transaction automatically and an exception being thrown.

154. ***Do bean-managed transactions get propagated to methods called on another BMT bean?*** One penalty of transactions being managed by the application instead of by the container is that they do not get propagated to methods called on another BMT bean.

155. ***If bean A begins a transaction and then calls Bean B, which is using bean-managed transactions, will transaction be propagated to the method in Bean B?*** If Bean A begins a transaction and then calls Bean B, which is using bean-managed transactions, then the transaction will not get propagated to the method in Bean B

156. ***What happens when a transaction is active when a BMT method is invoked?*** Any time a transaction is active when a BMT method is invoked, the active transaction will be suspended until control returns to the calling method.

157. ***How manually begin and commit container transactions?*** To be able to manually begin and commit container transactions, the application must have an interface that supports it. The UserTransaction interface is the designated object in the JTA that applications can hold on to and invoke to manage transaction boundaries.

158. ***Is an instance of UserTransaction actually the current transaction instance?*** An instance of UserTransaction is not actually the current transaction instance; it is a sort of proxy that provides the transaction API and represents the current transaction.

159. ***Which annotation is used to inject an UserTransaction instance into BMT components?*** A UserTransaction instance can be injected into BMT components by using the @Resource annotation.

160. ***When using dependency lookup, which name is used for UserTransaction?*** When using dependency lookup, it is found in the environment naming context using the reserved name java:comp/UserTransaction.

161. ***Describe UserTransaction Interface.***

		public interface javax.transaction.UserTransaction { 
			public abstract void begin(); 
			public abstract void commit(); 
			public abstract int getStatus(); 
			public abstract void rollback(); 
			public abstract void setRollbackOnly(); 
			public abstract void setTransactionTimeout(int seconds);
		}

162. ***How many transactions can be active at any given time?*** Each JTA transaction is associated with an execution thread, so it follows that no more than one transaction can be active at any given time. So if one transaction is active, the user cannot start another one in the same thread until the first one has committed or rolled back.

163. ***Can UserTransaction suspend a transaction?*** No. Only the container can do this using an internal transaction management API. In this way, multiple transactions can be associated with a single thread, even though only one can ever be active at a time.

164. ***When rollbacks can occur?Rollbacks can occur in several different scenarios:*** 

      1. The setRollbackOnly() method indicates that the current transaction cannot be committed, leaving rollback as the only possible outcome. 
      2. The transaction can be rolled back immediately by calling the rollback() method. 
      3. Alternately, a time limit for the transaction can be set with the setTransactionTimeout() method, causing the transaction to roll back when the limit is reached. The only catch with transaction timeouts is that the time limit must be set before the transaction starts and it cannot be changed once the transaction is in progress.

165. ***How transactional status can be accessed in JTA?*** In JTA every thread has a transactional status that can be accessed through the getStatus() call.

166. ***The return value of getStatus() method.*** The return value of this method is one of the constants defined on the java.transaction.Status interface.

167. ***What will return getStatus if no transaction is active?*** If no transaction is active, then the value returned by getStatus() will be the STATUS_NO_TRANSACTION.

      1. static int 	STATUS_ACTIVE
          A transaction is associated with the target object and it is in the active state.
      2. static int 	STATUS_COMMITTED
          A transaction is associated with the target object and it has been committed.
      3. static int 	STATUS_COMMITTING
          A transaction is associated with the target object and it is in the process of committing.
      4. static int 	STATUS_MARKED_ROLLBACK
          A transaction is associated with the target object and it has been marked for rollback, perhaps as a result of a setRollbackOnly operation.
      5. static int 	STATUS_NO_TRANSACTION
          No transaction is currently associated with the target object.
      6. static int 	STATUS_PREPARED
          A transaction is associated with the target object and it has been prepared.
      7. static int 	STATUS_PREPARING
          A transaction is associated with the target object and it is in the process of preparing.
      8. static int 	STATUS_ROLLEDBACK
          A transaction is associated with the target object and the outcome has been determined to be rollback.
      9. static int 	STATUS_ROLLING_BACK
          A transaction is associated with the target object and it is in the process of rolling back.
      10. static int 	STATUS_UNKNOWN
          A transaction is associated with the target object but its current status cannot be determined.

168. ***What will return getStatus() method if setRollbackOnly() has been called on the current transaction?*** Likewise, if setRollbackOnly() has been called on the current transaction, then the status will be STATUS_MARKED_ROLLBACK until the transaction has begun rolling back.

169. ***Example of using the UserTransaction interface in the servlet.***

		public class ProjectServlet extends HttpServlet { 
	
			@Resource UserTransaction tx; 
			@EJB ProjectService bean;
		
			protected void doPost(HttpServletRequest request, HttpServletResponse response) 
															throws ServletException, IOException { 
				// ... 
				try { 
					tx.begin(); 
					try { 
						bean.assignEmployeeToProject(projectId, empId); 
						bean.updateProjectStatistics(); 
					} finally { 
						tx.commit(); 
					} 
				} catch (Exception e) { 
					// handle exceptions from UserTransaction methods // ... 
				} // ... 
			} 
		}


170. ***Possible clients of a stateless session bean.*** A client of a stateless session bean is any Java EE component that can declare a dependency on the bean. This includes other session beans, message-driven beans, and servlets.

171. ***Can a reference to a stateful session bean be shared between threads?***A reference to a stateful session bean cannot be shared between threads.

172. ***Can we use the @EJB annotation to inject a stateful session bean?***Using the @EJB annotation to inject a stateful session bean is not a good solution.

173. ***Why it's bad idea to inject a stateful session bean into a stateless session bean?*** If @EJB were used to inject a stateful session bean into a stateless session bean where the server had pooled 100 bean instances, there would be 100 stateful session bean instances created as well.

174. ***Is it safe to inject a stateful session bean into another stateful session bean?*** The only time it is ever safe to inject a stateful session bean is into another stateful session bean.

175. ***What is the preferred method for acquiring a stateful session bean instance for a stateless client?*** Dependency lookup is the preferred method for acquiring a stateful session bean instance for a stateless client.

176. ***Example of typical pattern for servlets using stateful session beans.***
		
		@EJB(name="cart", beanInterface=ShoppingCart.class) 
		public class ShoppingCartServlet extends HttpServlet { 
			protected void doPost(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException { 
				HttpSession session = request.getSession(true); 
				ShoppingCart cart = (ShoppingCart) session.getAttribute("cart"); 
				if (cart == null) { 
					try { 
						Context ctx = new InitialContext(); cart = (ShoppingCart) ctx.lookup("java:comp/env/cart"); 
						session.setAttribute("cart", cart); 
					} catch (NamingException e) { 
						throw new ServletException(e); 
					} 
				} 
		}


177. ***How a singleton session bean can be acquired from client?*** From the perspective of a client, using a singleton session bean is similar to using a stateless session bean. It can be acquired via a context lookup or dependency injection and used without any extra steps.

178. ***How to dispose singleton bean?*** No special action is required to dispose of the bean. It will be disposed of automatically when the application is shut down.

179. ***How clients of a message-driven bean invoke business operations?*** As an asynchronous component, clients of a message-driven bean can't directly invoke business operations. Instead they send messages, which are then delivered to the MDB by the messaging system being used. The client needs to know only the format of the message that the MDB is expecting and the messaging destination where the message must be sent.

180. ***When a SOAP message with the attachments specified using the properties is generated?*** Before invoking the first SOAPHandler.

