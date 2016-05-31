## Wildfly Poc ##


### concurrency poc ###

explain:

	在JAVA EE环境下，以前没有办法从容器里面拿到一个自己想创建的线程池，
	或者说，自己创建的线程池不会被服务器管理。现在Java EE 7实现了。应用程序可以独自独立的线程池。
	wildfly standaone.xml文件中增加被容器管理的concurrency service subsystem 模块

		<subsystem xmlns="urn:jboss:domain:ee:4.0">
            <spec-descriptor-property-replacement>false</spec-descriptor-property-replacement>
            <concurrent>
                <context-services>
                    <context-service name="default" jndi-name="java:jboss/ee/concurrency/context/default" use-transaction-setup-provider="true"/>
                </context-services>
                <managed-thread-factories>
                    <managed-thread-factory name="default" jndi-name="java:jboss/ee/concurrency/factory/default" context-service="default"/>
                </managed-thread-factories>
                <managed-executor-services>
                    <managed-executor-service name="default" jndi-name="java:jboss/ee/concurrency/executor/default" context-service="default" hung-task-threshold="60000" keepalive-time="5000"/>
                </managed-executor-services>
                <managed-scheduled-executor-services>
                    <managed-scheduled-executor-service name="default" jndi-name="java:jboss/ee/concurrency/scheduler/default" context-service="default" hung-task-threshold="60000" keepalive-time="3000"/>
                </managed-scheduled-executor-services>
            </concurrent>
            <default-bindings context-service="java:jboss/ee/concurrency/context/default" datasource="java:jboss/datasources/ExampleDS" managed-executor-service="java:jboss/ee/concurrency/executor/default" managed-scheduled-executor-service="java:jboss/ee/concurrency/scheduler/default" managed-thread-factory="java:jboss/ee/concurrency/factory/default"/>
        </subsystem>
	
	代码中直接注入使用，这里不写name，默认也是注入DefaultManagedExecutorService。而且在wildfly里面，只能注入这个，因为这边xsd中定义，不能添加自己的concurrent。

	@Resource(name = "DefaultManagedExecutorService")
	private ManagedExecutorService managedExecutorService;
	
demo:
	
	BooksellerServlet.java

	@WebServlet("/booksellerServlet")
	public class BooksellerServlet extends HttpServlet{

		@Resource(name = "DefaultManagedExecutorService")
		private ManagedExecutorService managedExecutorService;

		protected void doGet(HttpServletRequest request,
			HttpServletResponse response) throws ServletException, IOException {
			int i=0;
			while(i<30){
				managedExecutorService.execute(new WorkTask()); //跟J2se里面使用就抑郁了
				i++;
			}
		}
	}


### EJB war poc ###


explain:
	
	现在如果要使用ejb，也可以打包成一个war包，不需要打包成一个ejb.jar。ejb 跟普通service类一样。可以告别ear的打包了。


	
### websocket ###

explain:

	web socket 是Java EE7 中新增的一个亮点。以后如果要新建一个socket，可以直接使用web socket。当然，这个是web socket的其中一个用途。
	它实现了浏览器与服务器全双工通信(full-duplex)。
	浏览器中通过http仅能实现单向的通信,comet可以一定程度上模拟双向通信,但效率较低,并需要服务器有较好的支持。
	可以很好的为实时通讯服务。

	@ServerEndpoint(“/hello”)
	public class HelloEndpoint {
		@OnOpen
		public void open(Session session, EndpointConfig conf) throws
		IOException {
			session.getBasicRemote().sendText(“Hi!”);
		}
	}


demo:

	ChatServlet.java
	
	@WebServlet("chatServlet")
	public class ChatServlet extends HttpServlet {
		protected void doGet(HttpServletRequest request,
			HttpServletResponse response) throws ServletException, IOException {
			WebSocketContainer container = ContainerProvider.getWebSocketContainer();
		 	String uri = "ws://localhost:" + request.getLocalPort() + request.getContextPath() + "/chat";
		 	System.out.println("Connecting to " + uri);
         	try {
				Session session = container.connectToServer(ChatClient.class, URI.create(uri));
				session.getBasicRemote().sendText("send you call!"); //发给服务器的消息
			} catch (DeploymentException e) {
				e.printStackTrace();
			}
		}
	}
	
	//客户端，可以页面做客户端
	@ClientEndpoint
	public class ChatClient {
		public static String response;
		@OnOpen
		public void onOpen(Session session) {
			System.out.println("on open....");
		}

		@OnMessage
		public void processMessage(String message) {
			System.out.println("server message="+message);
		}
	}

	// server端
	@ServerEndpoint("/chat")
	public class CommunctionChat {

    	@OnOpen
    	public void onOpen(Session session) {
        	System.out.println("New connection with client: {0}"+session.getId());
    	}
    
    	@OnMessage
    	public String onMessage(String message, Session session) {
        	System.out.println("New message from Client [{0}]: {1}"+new Object[] {session.getId(), message});
       		System.out.println(message);
        	return "Server received [" + message + "]";
    	}
    
    	@OnClose
    	public void onClose(Session session) {
        	System.out.println("Close connection for client: {0}"+session.getId());
    	}
    
    	@OnError
    	public void onError(Throwable exception, Session session) {
        	System.out.println("Error for client: {0}"+session.getId());
    	}
	}

运行

	15:53:58,065 INFO  [stdout] (default task-2) Connecting to ws://localhost:8080/poc7/chat

	15:53:58,114 INFO  [stdout] (default task-4) on open....

	15:53:58,115 INFO  [stdout] (default task-5) New connection with client: {0}NhzIqvQXbphGMjEA1wdWPh7WbL0P-zbOwrA5pmDM

	15:53:58,132 INFO  [stdout] (default task-6) New message from Client [{0}]: {1}[Ljava.lang.Object;@1bca941

	15:53:58,136 INFO  [stdout] (default task-6) send you call!

	15:53:58,141 INFO  [stdout] (default task-7) message=Server received [send you call!]


### 异步EJB 	###
		//触发异步ejb的入口，相当于测试入口
		@WebServlet("calculatorServlet")
		public class CalculatorServlet extends HttpServlet {

		@EJB
		private CalculatorBean calcBean;

		protected void doGet(HttpServletRequest request,
				HttpServletResponse response) throws ServletException, IOException {
			StringBuilder sb = new StringBuilder();
			sb.append("Begin addition calculation: " + new Date() + "<br/>");
			Future<Integer> addFuture = calcBean.asyncAdd(sb, 1, 1);
			return ;
			}
		}


	@Stateless
	@Asynchronous //异步annotation
	public class CalculatorBean {
		//必须实现异步方法
		public Future<Integer> asyncAdd(StringBuilder sb, int num1, int num2) {
			sb.append("start running asyncAdd in thread "
					+ Thread.currentThread().getName() + "<br/>");
			try {
				Thread.sleep(10000);
			} catch (InterruptedException e) {
				System.out.println("Error occured while trying to make this thread asleep.");
				e.printStackTrace();
			}
			int result = num1 + num2;
			sb.append("Addition calculation is finished: " + new Date() + "<br/>");

			sb.append("Finished running asyncAdd in thread "
					+ Thread.currentThread().getName() + "<br/>");
			System.out.println(sb.toString());
			return new AsyncResult<Integer>(result);
		}
	}


### Batch ###

	
