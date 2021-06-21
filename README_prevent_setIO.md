
# README (prevent setIO)

### Introduction

Here we summarize the steps performed for preventing CoreNLP to redirect System.out and System.err .  
When we want to use CoreNLP as a dependency in other projects, the original behaviour is conflicting with the set of runtime permissions enforced by Java Security Manager in those projects.  
The raised exception states:   

```
Caused by: java.security.AccessControlException: access denied ("java.lang.RuntimePermission" "setIO")
......
at edu.stanford.nlp.pipeline.AnnotationPipeline.<clinit>(AnnotationPipeline.java:30) ~[?:?]
```

See details on risks of using setIO in "Permissions in the Java Development Kit (JDK)" at https://docs.oracle.com/javase/7/docs/technotes/guides/security/permissions.html   



### Github operations

Fork repo: https://github.com/stanfordnlp/CoreNLP   

Create branch: `no_setIO_Redwood`   

Clone branch locally:  

```
git clone -b no_setIO_Redwood --single-branch https://github.com/camelia-c/CoreNLP.git
```

Check remote repo:

```
git remote show origin


* remote origin
  Fetch URL: https://github.com/camelia-c/CoreNLP.git
  Push  URL: https://github.com/camelia-c/CoreNLP.git
  HEAD branch: no_setIO_Redwood
  Remote branch:
    no_setIO_Redwood tracked
  Local branch configured for 'git pull':
    no_setIO_Redwood merges with remote no_setIO_Redwood
  Local ref configured for 'git push':
    no_setIO_Redwood pushes to no_setIO_Redwood (fast-forwardable)

```


### Inspect source code (in NetBeans IDE)

Find all setIO operations:

**System.setOut**

```
[ColumnDataClassifierITest.java at line 50, column 5]      System.setOut(outPrint);
[ColumnDataClassifierITest.java at line 55, column 5]      System.setOut(oldOut);
[TestThreadedCRFClassifier.java at line 77, column 7]        System.setOut(new PrintStream(System.out, true, "UTF-8"));
[BuildLexicalizedParserITest.java at line 180, column 5]      System.setOut(teeOutPS);
[BuildLexicalizedParserITest.java at line 222, column 5]      System.setOut(originalOut);
[LexicalizedParserCharacterEncodingITest.java at line 54, column 5]      System.setOut(ps);
[MulticoreMaxentTaggerITest.java at line 83, column 5]      System.setOut(outPrint);
[MulticoreMaxentTaggerITest.java at line 97, column 5]      System.setOut(outPrint);
[MulticoreMaxentTaggerITest.java at line 109, column 5]      System.setOut(oldOut);
[LexicalizedParserClient.java at line 173, column 5]      System.setOut(new PrintStream(System.out, true, "utf-8"));
[LexicalizedParserServer.java at line 299, column 5]      System.setOut(new PrintStream(System.out, true, "utf-8"));
[CountClosedTags.java at line 238, column 5]      System.setOut(new PrintStream(System.out, true, "UTF-8"));
[GrammaticalStructureConversionUtils.java at line 587, column 7]        System.setOut(new PrintStream(System.out, true, encoding));
[Redwood.java at line 249, column 7]        System.setOut(new RedwoodPrintStream(STDOUT, realSysOut));
[Redwood.java at line 251, column 7]        System.setOut(realSysOut);
[Redwood.java at line 264, column 5]      System.setOut(realSysOut);
[SegDemo.java at line 37, column 5]      System.setOut(new PrintStream(System.out, true, "utf-8"));
```

**System.setErr**

```
[ColumnDataClassifierITest.java at line 51, column 5]      System.setErr(errPrint);
[ColumnDataClassifierITest.java at line 56, column 5]      System.setErr(oldErr);
[TestThreadedCRFClassifier.java at line 78, column 7]        System.setErr(new PrintStream(System.err, true, "UTF-8"));
[BuildLexicalizedParserITest.java at line 181, column 5]      System.setErr(teeErrPS);
[BuildLexicalizedParserITest.java at line 223, column 5]      System.setErr(originalErr);
[MulticoreMaxentTaggerITest.java at line 84, column 5]      System.setErr(errPrint);
[MulticoreMaxentTaggerITest.java at line 98, column 5]      System.setErr(errPrint);
[MulticoreMaxentTaggerITest.java at line 110, column 5]      System.setErr(oldErr);
[LexicalizedParserClient.java at line 174, column 5]      System.setErr(new PrintStream(System.err, true, "utf-8"));
[LexicalizedParserServer.java at line 300, column 5]      System.setErr(new PrintStream(System.err, true, "utf-8"));
[CountClosedTags.java at line 239, column 5]      System.setErr(new PrintStream(System.err, true, "UTF-8"));
[Redwood.java at line 254, column 7]        System.setErr(new RedwoodPrintStream(STDERR, realSysErr));
[Redwood.java at line 256, column 7]        System.setErr(realSysErr);
[Redwood.java at line 265, column 5]      System.setErr(realSysErr);
```

One method causing trouble is "captureSystemStreams" in class Redwood, so we track it down:  

```
[Redwood.java at line 247, column 25]    protected static void captureSystemStreams(boolean captureOut, boolean captureErr){
[Redwood.java at line 1491, column 13]      Redwood.captureSystemStreams(true, true);
[RedwoodConfiguration.java at line 70, column 31]        tasks.add(() -> Redwood.captureSystemStreams(true, Redwood.realSysErr == System.err));
[RedwoodConfiguration.java at line 72, column 31]        tasks.add(() -> Redwood.captureSystemStreams(Redwood.realSysOut == System.out, true));
[RedwoodConfiguration.java at line 81, column 31]        tasks.add(() -> Redwood.captureSystemStreams(false, Redwood.realSysErr == System.err));
[RedwoodConfiguration.java at line 83, column 31]        tasks.add(() -> Redwood.captureSystemStreams(Redwood.realSysOut == System.out, false));
```

Its usage is in methods "capture" and "restore" in RedwoodConfiguration, as well as in the "main" of Redwood.  
In RedwoodConfiguration, the "capture" method is called from within "parse(Properties props)", which is called from within "apply(Properties props)".  


### Implement the modifications
  
Then see them in a terminal with git diff:

```
cd CoreNLP

git diff -- '*.java'

diff --git a/src/edu/stanford/nlp/util/logging/Redwood.java b/src/edu/stanford/nlp/util/logging/Redwood.java
index 946a6e8..f432eae 100644
--- a/src/edu/stanford/nlp/util/logging/Redwood.java
+++ b/src/edu/stanford/nlp/util/logging/Redwood.java
@@ -603,13 +603,13 @@ public class Redwood  {
    * If SLF4J is in the code's classpath
    */
   static {
-    RedwoodConfiguration config = RedwoodConfiguration.minimal();
+    RedwoodConfiguration config = RedwoodConfiguration.empty2(); // .minimal();
     try {
       MetaClass.create("org.slf4j.LoggerFactory").createInstance();
       MetaClass.create("edu.stanford.nlp.util.logging.SLF4JHandler").createInstance();
       config = RedwoodConfiguration.slf4j();
     } catch (Exception ignored) { }
-    config.apply();
+    //config.apply();
   }
 
   /**
diff --git a/src/edu/stanford/nlp/util/logging/RedwoodConfiguration.java b/src/edu/stanford/nlp/util/logging/RedwoodConfiguration.java
index fbe2057..068cb8d 100644
--- a/src/edu/stanford/nlp/util/logging/RedwoodConfiguration.java
+++ b/src/edu/stanford/nlp/util/logging/RedwoodConfiguration.java
@@ -42,14 +42,20 @@ public class RedwoodConfiguration  {
    */
   private LinkedList<Runnable> tasks = new LinkedList<>();
 
-  private OutputHandler outputHandler = Redwood.ConsoleHandler.out();
+  private OutputHandler outputHandler;
   private File defaultFile = new File("/dev/null");
   private int channelWidth = 0;
 
   /**
    * Private constructor to prevent use of "new RedwoodConfiguration()"
    */
-  protected RedwoodConfiguration(){}
+  protected RedwoodConfiguration(int... args){
+      //Variable number of arguments, including no  args
+      if (args.length == 0){        
+          //original behaviour
+          this.outputHandler = Redwood.ConsoleHandler.out();
+      }
+  }
 
   /**
    * Apply this configuration to Redwood
@@ -431,7 +437,18 @@ public class RedwoodConfiguration  {
   public static RedwoodConfiguration empty(){
     return new RedwoodConfiguration().clear();
   }
-
+  
+  /**
+   * An empty Redwood configuration - new behaviour.
+   * Note that without a Console Handler, Redwood will not print anything
+   * @return An empty Redwood Configuration object.
+   */

```

### Build and install jar in Maven local repository (in a terminal)

**Gradle**

Check version in Gradle wrapper of the CoreNLP project:

```
cd CoreNLP

./gradlew --version                                                                                       

------------------------------------------------------------
Gradle 3.2
------------------------------------------------------------

Build time:   2016-11-14 12:32:59 UTC
Revision:     5d11ba7bc3d79aa2fbe7c30a022766f4532bbe0f

Groovy:       2.4.7
Ant:          Apache Ant(TM) version 1.9.6 compiled on June 29 2015
JVM:          1.8.0_101 (Oracle Corporation 25.101-b13)
OS:           Linux 4.4.0-210-generic amd64

```

Clean and build:

```
./gradlew --configure-on-demand -x check clean build                                                      


Starting a Gradle Daemon, 1 incompatible and 2 stopped Daemons could not be reused, use --status for details
Configuration on demand is an incubating feature.                                                                                                           
:clean                                                                                                                                                      
:compileJava                                                                                                                                                
Note: Some input files use or override a deprecated API.
Note: Recompile with -Xlint:deprecation for details.
Note: Some input files use unchecked or unsafe operations.
Note: Recompile with -Xlint:unchecked for details.
:processResources                                                                                                                                           
:classes                                                                                                                                                    
:jar                                                                                                                                                        
:startScripts                                                                                                                                               
:distTar                                                                                                                                                    
:distZip                                                                                                                                                    
:assemble                                                                                                                                                   
:build

BUILD SUCCESSFUL

```

**Install jar in local Maven repository (to be used by other projects):**

```
cd CoreNLP

mvn install:install-file -Dfile=build/libs/CoreNLP-4.2.2.jar -DgroupId=edu.stanford.nlp -DartifactId=stanford-corenlp -Dversion=4.2.2 -Dpackaging=jar


[INFO] Scanning for projects...
Downloading: https://repo.maven.apache.org/maven2/org/codehaus/mojo/build-helper-maven-plugin/1.7/build-helper-maven-plugin-1.7.pom
Downloaded: https://repo.maven.apache.org/maven2/org/codehaus/mojo/build-helper-maven-plugin/1.7/build-helper-maven-plugin-1.7.pom (6 KB at 0.9 KB/sec)
Downloading: https://repo.maven.apache.org/maven2/org/codehaus/mojo/build-helper-maven-plugin/1.7/build-helper-maven-plugin-1.7.jar
Downloaded: https://repo.maven.apache.org/maven2/org/codehaus/mojo/build-helper-maven-plugin/1.7/build-helper-maven-plugin-1.7.jar (32 KB at 176.6 KB/sec)
[INFO]                                                                         
[INFO] ------------------------------------------------------------------------
[INFO] Building Stanford CoreNLP 4.2.2
[INFO] ------------------------------------------------------------------------
[INFO] 
[INFO] --- maven-install-plugin:2.5.2:install-file (default-cli) @ stanford-corenlp ---
[INFO] pom.xml not found in CoreNLP-4.2.2.jar
[INFO] Installing /home/camelia/github_repos/CoreNLP/build/libs/CoreNLP-4.2.2.jar to /home/camelia/.m2/repository/edu/stanford/nlp/stanford-corenlp/4.2.2/stanford-corenlp-4.2.2.jar
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------

```

### Commit and push to remote repo

```
cd CoreNLP

git status


On branch no_setIO_Redwood
Your branch is up-to-date with 'origin/no_setIO_Redwood'.
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git checkout -- <file>..." to discard changes in working directory)

	modified:   src/edu/stanford/nlp/util/logging/Redwood.java
	modified:   src/edu/stanford/nlp/util/logging/RedwoodConfiguration.java

Untracked files:
  (use "git add <file>..." to include in what will be committed)

	.gradle/
	build/

no changes added to commit (use "git add" and/or "git commit -a")



git add src/edu/stanford/nlp/util/logging/Redwood.java
git add src/edu/stanford/nlp/util/logging/RedwoodConfiguration.java

git commit -m "changed Redwood to prevent setIO"

git push origin
```





  


