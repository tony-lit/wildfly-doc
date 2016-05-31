## wildfly 打包 ##

* 需要maven 版本提高Maven 3.2.5，如果打包环境没有，它自己会去互联网下载这个版本，然后放到它的tool文件夹中。
* jdk 8
* 执行mvn install -Drelease=true -DskipTests

打包过程中报错：
https://developer.jboss.org/message/956900?et=watches.email.thread#956900
然后说是没装git
我看log，确实会去服务器上git比对一下。
就去环境装git，没有root用户，源码编译开始装git，全是泪。。。。。

然后没用~~~

然后git下了打包代码，这么大的wildfly，还有几个project写代码用于打包的。

在报错的点：ZipFileSubsystemInputStreamSources.addAllSubsystemFileSourcesFromZipFile(ZipFileSubsystemInputStreamSources.java:67)

插入代码（到底什么file，解压不了）：

	public void addAllSubsystemFileSourcesFromZipFile(File file) throws IOException {
    	System.out.println("file=================="+file+"  file name="+file.getName());
        try (ZipFile zip = new ZipFile(file)) {
            // extract subsystem template and schema, if present
            if (zip.getEntry("subsystem-templates") != null) {
                Enumeration<? extends ZipEntry> entries = zip.entries();
                while (entries.hasMoreElements()) {
                    ZipEntry entry = entries.nextElement();
                    if (!entry.isDirectory()) {
                        String entryName = entry.getName();
                        if (entryName.startsWith("subsystem-templates/")) {
                            addSubsystemFileSource(entryName.substring("subsystem-templates/".length()), file, entry);
                        }
                    }
                }
            }
        }
    } 

然后打包这个修改过的打包工具。先得注释掉checkstyle，写了System.out.println这种东西，checkstyle肯定过不了，就不check了。

在wildfly-build-tools-parent.pom 注释掉checkstyle
	
	<executions>
    	<execution>
        	<id>check-style</id>
            <phase>compile</phase>
            <goals>
            	<goal>checkstyle</goal>
            </goals>
        </execution>    
    </executions>

日志：
	
	file==================/home/dd_mgmas/ylgao/.m2/repository/org/fusesource/jansi/jansi/1.11/jansi-1.11.jar  file name=jansi-1.11.jar
	file==================/home/dd_mgmas/ylgao/.m2/repository/org/wildfly/wildfly-cmp/10.0.0.Final/wildfly-cmp-10.0.0.Final.jar  file name=wildfly-cmp-10.0.0.Final.jar
	file==================/home/dd_mgmas/ylgao/.m2/repository/org/jboss/metadata/jboss-metadata-ear/10.0.0.Final/jboss-metadata-ear-10.0.0.Final.jar  file name=jboss-metadata-ear-10.0.0.Final.jar
	file==================/home/dd_mgmas/ylgao/.m2/repository/org/hibernate/javax/persistence/hibernate-jpa-2.1-api/1.0.0.Final/hibernate-jpa-2.1-api-1.0.0.Final.jar  file name=hibernate-jpa-2.1-api-1.0.0.Final.jar
	file==================/home/dd_mgmas/ylgao/.m2/repository/org/hibernate/hibernate-validator/5.2.3.Final/hibernate-validator-5.2.3.Final.jar  file name=hibernate-validator-5.2.3.Final.jar
	file==================/home/dd_mgmas/ylgao/.m2/repository/org/jboss/spec/javax/transaction/jboss-transaction-api_1.2_spec/1.0.0.Final/jboss-transaction-api_1.2_spec-1.0.0.Final.jar  file name=jboss-transaction-api_1.2_spec-1.0.0.Final.jar
	file==================/home/dd_mgmas/ylgao/.m2/repository/org/wildfly/wildfly-jsf/10.0.0.Final/wildfly-jsf-10.0.0.Final.jar  file name=wildfly-jsf-10.0.0.Final.jar
	file==================/home/dd_mgmas/ylgao/.m2/repository/org/jboss/hal/release-stream/2.8.19.Final/release-stream-2.8.19.Final-resources.jar  file name=release-stream-2.8.19.Final-resources.jar
	[INFO] ------------------------------------------------------------------------
	[INFO] BUILD FAILURE
	[INFO] ------------------------------------------------------------------------
	[INFO] Total time: 36.959 s
	[INFO] Finished at: 2016-05-27T14:25:16+08:00
	[INFO] Final Memory: 88M/2881M
	[INFO] ------------------------------------------------------------------------
	[ERROR] Failed to execute goal org.wildfly.build:wildfly-server-provisioning-maven-plugin:1.1.0.Final:build (server-provisioning) on project wildfly-build: Execution server-provisioning of goal org.wildfly.build:wildfly-server-provisioning-maven-plugin:1.1.0.Final:build failed: java.lang.RuntimeException: java.util.zip.ZipException: error in opening zip file -> [Help 1]
	org.apache.maven.lifecycle.LifecycleExecutionException: Failed to execute goal org.wildfly.build:wildfly-server-provisioning-maven-plugin:1.1.0.Final:build (server-provisioning) on project wildfly-build: Execution server-provisioning of goal org.wildfly.build:wildfly-server-provisioning-maven-plugin:1.1.0.Final:build failed: java.lang.RuntimeException: java.util.zip.ZipException: error in opening zip file
        at org.apache.maven.lifecycle.internal.MojoExecutor.execute(MojoExecutor.java:212)
        at org.apache.maven.lifecycle.internal.MojoExecutor.execute(MojoExecutor.java:153)
        at org.apache.maven.lifecycle.internal.MojoExecutor.execute(MojoExecutor.java:145)
        at org.apache.maven.lifecycle.internal.LifecycleModuleBuilder.buildProject(LifecycleModuleBuilder.java:116)


最后一个release-stream-2.8.19.Final-resources.jar 解压不了，去本地maven仓库看，奇怪，这个jar只有1K，解压不了。

但我是在新环境，重新配的maven，这个jar都是拉下来的，如果没拉成功，就应该已经停止运行下去了。
在本地仓库里面删除这个jar，重新打包，还是只下载了1K。
手工copy这个包去仓库。

打包终于成功了。