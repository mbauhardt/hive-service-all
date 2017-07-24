# Background
You have a big project with dependencies to the hadoop ecosystem and a lot of other dependencies? You want to write Hive Server 2 integration tests ?
You could add the required hive dependencies to your project and start the class `HiveServer2`.
My experience is that the huge tail of *my project dependencies + hadoop dependencies + hive dependencies* are hard to define without to run into jar incompatibilities.

The idea of this project is to have on fat jar which contains all hive classes + hive dependencies jarjar'd into one jar file.


# Usage
Check this project out via 

    git clone git@github.com:mbauhardt/hive-service-all.git
    cd hive-service-all
    # build hive fat jar for cloudera 5.12.0
    ./gradlew clean jar -PhiveVersion=2.6.0-cdh5.12.0

The fat jar is located under *hive-service-all-repackage/build/jarjar/hive-service-all-repackaged-1.1.0-cdh5.12.0.jar*
You can use this jar in your separate project within a flat repository.

    .                                                                                                                                                                                 
    ├── build.gradle
    ├── lib
    │   └── hive-service-all-repackaged-1.1.0-cdh5.12.0.jar
    └── src
        ├── test
            └── java
                └── HiveServer2Test.java

And a build gradle file wil the following repository and dependency definition.

    repositories {
        jcenter()
        mavenCentral()
        maven {
            url "http://central.maven.org/maven2/"
        }
        maven {
            url "http://repo.spring.io/plugins-release/"
        }
        flatDir name: 'localRepository', dirs: 'lib'
    }
    
    dependencies {
    
        compile "com.github.mbauhardt:hive-service-all-repackaged:1.1.0-cdh5.12.0"
        testCompile "junit:junit:4.12"
    
        def hadoopVersion = project.hasProperty('hadoopVersion') ? project.properties.get('hadoopVersion') : '1.1.0'
        println "Resolve dependencies for Hadoop Version ${hadoopVersion}"
        //hadoop
        compile(group: 'org.apache.hadoop', name: 'hadoop-common', version: hadoopVersion)
        compile(group: 'org.apache.hadoop', name: 'hadoop-archives', version: hadoopVersion)
        compile(group: 'org.apache.hadoop', name: 'hadoop-mapreduce-client-core', version: hadoopVersion)
        compile(group: 'org.apache.hadoop', name: 'hadoop-mapreduce-client-common', version: hadoopVersion)
        compile(group: 'org.apache.hadoop', name: 'hadoop-hdfs', version: hadoopVersion)
        compile(group: 'org.apache.hadoop', name: 'hadoop-yarn-api', version: hadoopVersion)
        compile(group: 'org.apache.hadoop', name: 'hadoop-yarn-common', version: hadoopVersion)
        compile(group: 'org.apache.hadoop', name: 'hadoop-yarn-client', version: hadoopVersion)
    }

And a test case like this to start the embedded Hive Server 2.

    import com.github.mbauhardt.org.apache.hadoop.hive.conf.HiveConf;
    import com.github.mbauhardt.org.apache.hive.service.server.HiveServer2;
    import org.junit.Test;
    
    public class HiveServer2Test {
    
        @Test
        public void testStartHiveServer2() throws Exception {
            HiveServer2 hiveServer2 = new HiveServer2();
            hiveServer2.init(new HiveConf());
            hiveServer2.start();
            
            //TBD list/create databases and tables
        }
    }

# Manual Workarounds
Before you can use the jar you have todo the following steps

* Remove `META_INF/MANIFEST.MF` out of the generated jarjar'd fat jar.

Naja, at the end when starting the HiveServer i got the exception

    java.lang.RuntimeException: com.github.mbauhardt.org.apache.hadoop.hive.ql.metadata.HiveException: java.lang.RuntimeException: com.github.mbauhardt.org.apache.hadoop.hive.ql.metadata.HiveException: java.lang.RuntimeException: Unable to instantiate com.github.mbauhardt.org.apache.hadoop.hive.ql.metadata.SessionHiveMetaStoreClient
    	at com.github.mbauhardt.org.apache.hadoop.hive.ql.session.SessionState.setupAuth(SessionState.java:833)
    	at com.github.mbauhardt.org.apache.hadoop.hive.ql.session.SessionState.getAuthorizationMode(SessionState.java:1679)
    	at com.github.mbauhardt.org.apache.hadoop.hive.ql.session.SessionState.isAuthorizationModeV2(SessionState.java:1690)
    	at com.github.mbauhardt.org.apache.hadoop.hive.ql.session.SessionState.applyAuthorizationPolicy(SessionState.java:1738)
    	at com.github.mbauhardt.org.apache.hive.service.cli.CLIService.applyAuthorizationConfigPolicy(CLIService.java:125)
    	at com.github.mbauhardt.org.apache.hive.service.cli.CLIService.init(CLIService.java:111)
    	at com.github.mbauhardt.org.apache.hive.service.CompositeService.init(CompositeService.java:59)
    	at com.github.mbauhardt.org.apache.hive.service.server.HiveServer2.init(HiveServer2.java:119)
    	at HiveServer2Test.testStartHiveServer2(HiveServer2Test.java:14)
    	....
    	Caused by: com.github.mbauhardt.org.datanucleus.exceptions.NucleusUserException: Persistence process has been specified to use a ClassLoaderResolver of name "datanucleus" yet this has not been found by the DataNucleus plugin mechanism. Please check your CLASSPATH and plugin specification.
        	at com.github.mbauhardt.org.datanucleus.NucleusContext.<init>(NucleusContext.java:283)
        	at com.github.mbauhardt.org.datanucleus.NucleusContext.<init>(NucleusContext.java:247)
        	at com.github.mbauhardt.org.datanucleus.NucleusContext.<init>(NucleusContext.java:225)
        	at com.github.mbauhardt.org.datanucleus.api.jdo.JDOPersistenceManagerFactory.<init>(JDOPersistenceManagerFactory.java:416)
        	at com.github.mbauhardt.org.datanucleus.api.jdo.JDOPersistenceManagerFactory.createPersistenceManagerFactory(JDOPersistenceManagerFactory.java:301)
        	at com.github.mbauhardt.org.datanucleus.api.jdo.JDOPersistenceManagerFactory.getPersistenceManagerFactory(JDOPersistenceManagerFactory.java:202) 
       
   Until now, i have no idea how to fix it :/
   
   