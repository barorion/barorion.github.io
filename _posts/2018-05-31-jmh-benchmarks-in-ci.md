---
layout: post
title: Continuously running JMH benchmarks
---

I wanted to get early feedback on performance regressions.

We have quite a few JMH benchmarks that cover the most important/sensitive parts of our codebase. I wanted Jenkins to run those JMH benchmarks on each commit, and fail the build when some predefined latency thresholds are exceeded.
Surprisingly, I failed to find something that does it out-of-the-box.

I ended up with the following:

* a @LatencyThreshold annotation, that you can add to any JMH benchmark (method), specifying the maximum allowed latency for this benchmark (in nano seconds).
```java
@Retention(RetentionPolicy.RUNTIME)  
@Target(ElementType.METHOD)  
public @interface LatencyThreshold {  
    boolean enabled() default true;
    
   /**  
    * The maximum allowed latency in nanoseconds.
    */ 
    long thresholdInNanos();  
}
```
* A JUnit (4) test, that scans for benchmarks annotated with @LatencyThreshold, execute each one of them, and (soft) assert on the resulted latency:

```java
public class BenchmarksIntegrationTest {  

private static Reflections reflections;  

@BeforeClass  
public static void init() {  
    // only the specified package will be scanned for benchmarks to execute
    reflections = new Reflections(new ConfigurationBuilder().setUrls(ClasspathHelper.forPackage("my.benchmarks.package"))  
            .setScanners(new MethodAnnotationsScanner()));  
}  

@Test  
public void test() throws Exception {  
    // get all methods annotated with @LatencyThreshold in the declared package
    Set<Method> methods = reflections.getMethodsAnnotatedWith(LatencyThreshold.class);  
    BDDSoftAssertions softAssertions = new BDDSoftAssertions();  
    for (Method method : methods) {  
        LatencyThreshold declaredAnnotation = method.getDeclaredAnnotation(LatencyThreshold.class);  
        if (declaredAnnotation.enabled()) {  
            runWithThreshold(method.getName(), declaredAnnotation.thresholdInNanos(), softAssertions);  
        }  
    }  
    // fail the test is any of the soft assertions failed
    softAssertions.assertAll();  
}  

private void runWithThreshold(String methodName, long threshold, BDDSoftAssertions softAssertions) throws RunnerException {  
 // override some of the benchmark's configuration 
    Options opts = new OptionsBuilder()  
            .include(methodName)  
            .forks(1)  
            .warmupIterations(2)  
            .measurementIterations(2)  
            .timeUnit(TimeUnit.NANOSECONDS)  
            .build();  

    // run this benchmark
    RunResult result = new Runner(opts).runSingle();  

    long score = (int) result.getPrimaryResult().getScore();  
    String label = result.getPrimaryResult().getLabel();  
    Mode mode = result.getParams().getMode();  

    if (Mode.Throughput == mode) {  
       softAssertions.fail(label + ": Throughput mode is not supported");  
    } else {  
       softAssertions.then(score).as(label + ": " + mode + " exceeds predefined threshold").isLessThan(threshold);  
       System.out.println(String.format((score < threshold ? "==OK==" : "==FAILED==") + " %s: %s %sns satisfies predefined threshold %sns", label, mode, score, threshold));  
    }  
  }           
}
```
This is a very simplistic solution, that e.g. does not handle Throughput mode benchmarks. It'd also be much nicer to pack this functionality into a JUnit 4 rule/runner (or even better, a JUnit 5 extension!). But it will have to wait for less pressed up times... ;) 

NB the following dependencies were used:

* [Reflections](https://github.com/ronmamo/reflections) ('org.reflections:reflections') - to scan for methods annotated with @LatencyThreashod

* [AssertJ](http://joel-costigliola.github.io/assertj/) ('org.assertj:assertj-core') - for the soft assertions

* And naturally, JMH: 'org.openjdk.jmh:jmh-core' and 'org.openjdk.jmh:jmh-generator-annprocess'  
