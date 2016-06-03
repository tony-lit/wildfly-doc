## wildfly Suspend, Resume and Graceful shutdown 代码解析 ##

### request controller ###

wildfly新增了request controller功能，实现模块：wildfly-request-controller ， 他的作用 suspend 服务器，resume服务器，Graceful shutdown服务器。

首先来说说Graceful shutdown： jboss7之前的版本，都有优雅的停止这一说法，当执行stop的时候，先把在active的请求一个个处理完后，再停止。但jboss 7 ，直接使用stop，他是直接 immediately shutdown。

可以看 http://172.17.254.218:3000/issues/984


suspend 服务器：服务器停止服务，如果里面还有active 线程，处理完。如果用了mod_cluster，就告知apache这台已经停止，不要发新请求过来。有新请求过来，直接reject

resume 服务器：从suspend状态到active状态，可以接受新的请求。


### 代码流向 ###

![](http://i.imgur.com/t8HlmXE.png)

* suspend：
	
	从cli那边得到suspend消息，调用。RequestController的suspend函数。

		   public synchronized void suspended(ServerActivityCallback requestCountListener) {
        		this.paused = true;
        		listenerUpdater.set(this, requestCountListener);

        		if (activeRequestCountUpdater.get(this) == 0) {
            		if (listenerUpdater.compareAndSet(this, requestCountListener, null)) {
                		requestCountListener.done();
            		}
        		}
    		}

	然后调用SuspendController 的suspend。

		public synchronized void suspend(long timeoutMillis) {
        if (timeoutMillis > 0) {
            ServerLogger.ROOT_LOGGER.suspendingServer(timeoutMillis);
        } else {
            ServerLogger.ROOT_LOGGER.suspendingServerWithNoTimeout();
        }
        state = State.PRE_SUSPEND;
        //we iterate a copy, in case a listener tries to register a new listener
        for(OperationListener listener: new ArrayList<>(operationListeners)) {
            listener.suspendStarted();
        }
        outstandingCount = activities.size();
        if (outstandingCount == 0) {
            handlePause();
        } else {
            CountingRequestCountCallback cb = new CountingRequestCountCallback(outstandingCount, new ServerActivityCallback() {
                @Override
                public void done() {
                    state = State.SUSPENDING;
                    for (ServerActivity activity : activities) {
                        activity.suspended(SuspendController.this.listener);
                    }
                }
            });

            for (ServerActivity activity : activities) {
                activity.preSuspend(cb);
            }
            timer = new Timer();  //suspend 有timeout参数，如果超过这个时间还没有处理完，就直接强挂起，如果没有，就会无期限等下去
            if (timeoutMillis > 0) {
                timer.schedule(new TimerTask() {
                    @Override
                    public void run() {
                        timeout();
                    }
                }, timeoutMillis);
            }
        }
    }


* resume：

	从cli传来resume命令，RequestController的resume函数

	   	public synchronized void resume() {
        		this.paused = false;
       		 ServerActivityCallback listener = listenerUpdater.get(this);
        		if (listener != null) {
            		listenerUpdater.compareAndSet(this, listener, null);
        		}
       		 while (!taskQueue.isEmpty() && (activeRequestCount < maxRequestCount || maxRequestCount < 0)) {
            		runQueuedTask(false);
        		}
    		}


	然后调用SuspendController 的resume。

		public synchronized void resume() {
        if (state == State.RUNNING) {
            return;
        }
        ServerLogger.ROOT_LOGGER.resumingServer();
        if (timer != null) {
            timer.cancel();
            timer = null;
        }
        for(OperationListener listener: new ArrayList<>(operationListeners)) {
            listener.cancelled();
        }
        for (ServerActivity activity : activities) {
            try {
                activity.resume();
            } catch (Exception e) {
                ServerLogger.ROOT_LOGGER.failedToResume(activity);
            }
        }
        state = State.RUNNING;
    }

* suspend状态时请求。

	![](http://i.imgur.com/O3lqOJH.png)
	
	请求到ControlPoint 的beginRequest函数
		
		 public RunResult beginRequest() throws Exception {
        	if (paused) {
           		return RunResult.REJECTED;
        	}
        	if(trackIndividualControlPoints) {
            	activeRequestCountUpdater.incrementAndGet(this);
        	}
        	RunResult runResult = controller.beginRequest(false);
        	if (runResult == RunResult.REJECTED) {
            	decreaseRequestCount();
        	}
        	return runResult;
   	 	}	

然后调用RequestController的beginRequest函数

		 	RunResult beginRequest(boolean force) {
        		int maxRequests = maxRequestCount;
        		int active = activeRequestCountUpdater.get(this);
        		boolean success = false;
        		while ((maxRequests <= 0 || active < maxRequests) && (!paused || force)) {
            		if (activeRequestCountUpdater.compareAndSet(this, active, active + 1)) {
                		success = true;
               		 break;
            		}
            		active = activeRequestCountUpdater.get(this);
        		}
        		if (success) {
            		//re-check the paused state
            		//this is necessary because there is a race between checking paused and updating active requests
            		//if this happens we just call requestComplete(), as the listener can only be invoked once it does not
            		//matter if it has already been invoked
            		if(!force && paused) {
                		requestComplete();
                		return RunResult.REJECTED;
            		}
            		return RunResult.RUN;
        		} else {
           		 	return RunResult.REJECTED;
        			}
    			}


maxRequests =-1，success =false，然后返回rejected











