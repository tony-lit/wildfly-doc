## wildfly 10 standalone 调试##

###wildfly 代码简单说明###

首先迁出wildfly 10源码，会发现启动的Main函数不在wildfly 10的源码里。

wildfly 只是一个骨架，把各个干活的project 的jar dependency 引入进来。


启动主函数：org.jboss.as.server.Main 函数。在wildfly-server 项目里面，也就是wildfly-core里面。

在github里面下载 wildfly-server，jboss-msc。

jboss-msc是一个底层容器，全部的服务，都会由msc容器来启动停止。

wildfly-core就是wildfly核心模块，容器最简单的启动，停止，就在这个里面。

###wildfly如何调试###

1. 下载完wildfly-core、jboss-msc，在workspacke中import进入。
2. 新建server,新建wildfly 10.0.0.Final.
3. 在Main函数中打断点，然后在standalone.conf.bat 里面，把远程调试打开。
4. 然后debug wildfly server。就会跳入debug了。

###wildfly启动##

如图：

 ![](http://i.imgur.com/ibTo3oh.png)


1. 从JBoss Modules的Main函数入口，然后中间ServerEnvironmentService 拿到启动的参数。

2. Main函数中，把环境参数弄好，config文件读进来，然后传入BootstrapImpl的bootstrap 函数，再调用internalBootstrap

		private AsyncFuture<ServiceContainer> internalBootstrap(final Configuration configuration, final List<ServiceActivator> extraServices) {
			
		}
3. 从BootstrapImpl 调用ApplicationServerService，applicationServerService作为一个service装入MSC中。

   		public synchronized void start(final StartContext context) throws StartException {
				......
 				serviceTarget.addService(Services.JBOSS_PRODUCT_CONFIG_SERVICE, new ValueService<ProductConfig>(productConfigValue))
                .setInitialMode(ServiceController.Mode.ACTIVE)
                .install();
		}
	调用start函数开始启动。都是装入msc容器，状态改为Mode.ACTIVE。OSGI中表示激活。
	wildfly自己实现了一个osgi容器。

4.ApplicationServerService调用serverservice的start，把一些依赖的server都弄成service，装入msc。

5.然后返回到AbstractControllerService 继续执行

		final Thread bootThread = new Thread(null, new Runnable() {
            public void run() {
                try {
                    try {
                        boot(new BootContext() {
                            public ServiceTarget getServiceTarget() {
                                return target;
                            }
                        });
                    } finally {
                        processState.setRunning();
                    }
                } catch (Throwable t) {
                    container.shutdown();
                    if (t instanceof StackOverflowError) {
                        ROOT_LOGGER.errorBootingContainer(t, bootStackSize, BOOT_STACK_SIZE_PROPERTY);
                    } else {
                        ROOT_LOGGER.errorBootingContainer(t);
                    }
                } finally {
                    bootThreadDone();
                }

            }
        }, "Controller Boot Thread", bootStackSize);
        bootThread.start();

	把service启动起来。

