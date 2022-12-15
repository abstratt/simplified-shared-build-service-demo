# Simplified Consumption of Shared Build Services

This work proposes a new kind of property called "Service Reference" to simplify the programming model for consuming [shared build services](https://docs.gradle.org/current/userguide/build_services.html). 

This repository includes a simple project to illustrate the feature.

## Existing programming model

Up to Gradle 7.6, the consumption of shared build services required:

1. declaring a property typed using the service type
2. marking it as `@Internal`
3. explicitly declaring the task uses said service
4. explicitly setting the property value or convention with the reference to the service (as returned by service registration)

```groovy
Provider<CountingService> serviceProvider = gradle.sharedServices.registerIfAbsent("counter", 
CountingService) {
    // ... 
}

abstract class Consumer extends DefaultTask {
    // 2
    @Internal 
    // 1
    abstract Property<CountingService> getCounter() 

    @TaskAction
    def go() {
        counter.get().increment()
    }
}

task consumerTask(type: Consumer){
    // 3
    usesService(serviceProvider)
    // 4
    counter = serviceProvider
}

```

## Proposed programming model

For Gradle 8+ (release to be confirmed), we intend to release a new model for consuming shared build services. In this new model, all that is required is:

1. declaring a property typed using the service type
2. marking with `@ServiceReference("nameOfService")` or `@ServiceReference` (if only one service exists with the consumed type)

```groovy
// no need to keep the service reference around
gradle.sharedServices.registerIfAbsent("counter", 
    CountingService) {
    // ...
}

abstract class Consumer extends DefaultTask {
    // 2
    @ServiceReference(”counter”)
    // 1
    abstract Property<CountingService> getCounter()

    @TaskAction
    def go() {
        counter.get().increment()
    }
}

task consumerTask(type: Consumer) {
    // no further configuration required
}

```

### Comparing both models

1. in the new model, there is no need to store and refer to the provider returned by service registration.
1. in the new model, the consumer needs to provide the name the service was published under, if more than one service will be registered with the consumed type
1. in the new model, properties declared as service references may be declared as `@Optional`. If optional, an unresolved service reference will not lead to a validation error, but code needs to be written so it can handle a missing service (e.g. `if (counter.isPresent()) { ... }`).

### Service references without a name

A variation of the new model that we also support allows applying the `@ServiceReference` annotation _without_ providing a service name. In that case, the only difference to the 7.6 model is that it no longer requires calling `Task#usesService(...)`. 

### Known issues

At time of writing, using the new model leads to services being instantiated at the beginning of execution, even if not used ([issue 22996](https://github.com/gradle/gradle/issues/22996)). 

## Using the demo project

### A task that fails to declare usage of a shared build service
```
./gradlew counter0
```

which produces a warning as this project has `STABLE_CONFIGURATION_CACHE` [enabled](https://docs.gradle.org/current/userguide/configuration_cache.html#config_cache:stable).

```
> Task :counter0
Build service 'countingService' is being used by task ':counter0' without the corresponding declaration via 'Task#usesService'. This behavior has been deprecated. This will fail with an error in Gradle 9.0. Declare the association between the task and the build service using 'Task#usesService'. Consult the upgrading guide for further information: ...
        at build_2qdxf8j9h5z0lh1qwyrrybs3d$_run_closure3$_closure9.doCall(/Users/rafael/sources/samples/demos/simplified-shared-build-service-demo/build.gradle:57)
        (Run with --stacktrace to get the full stack trace of this deprecation warning.)
service: created with value = 42
service: value is 43
service: value is 44
service: closed with value 44

BUILD SUCCESSFUL in 354ms
1 actionable task: 1 executed

```

### A task that uses the pre-8+ model

```
./gradlew counter1
```

which just works, but has a lot of ceremony.

### A task that uses `@ServiceReference` without providing a service name

```
./gradlew counter2
```

which works, as there is only one service registered with the property type.

### A task that uses `@ServiceReference` with an ambiguous type

```
./gradlew counter3
```

Which fails, as there are multiple services registered with the specified property type, and hence it is not possible to automatically resolve the reference.

```
Configuration cache is an incubating feature.
Calculating task graph as no configuration cache is available for tasks: count3

FAILURE: Build failed with an exception.

* What went wrong:
Cannot resolve service by type for type 'BaseCountingService' when there are two or more instances. Please also provide a service name. Instances found: altCountingService: SubCountingService2, countingService: SubCountingService1.

* Try:
> Run with --stacktrace option to get the stack trace.
> Run with --info or --debug option to get more log output.
> Run with --scan to get full insights.

* Get more help at https://help.gradle.org

BUILD FAILED in 340ms
Configuration cache entry stored.
```

### A task that uses `@ServiceReference` with an ambiguous type but explicitly assigns a service

```
./gradlew counter4
```

which works, as when a value is set (or configured as convention), no automatic resolution is required.

### A task that uses `@ServiceReference("serviceName")`

```
./gradlew counter5
```

which works, as when a name is provided, it is fine if multiple services with the same type are registered.

## Considered Alternatives

### Using standard @Inject and @Named annotations instead of a new annotation 

Pros:

1. @Inject and @Named are standard

Cons:

1. We still need an annotation for marking a property as a service reference holder (when the user does not assign a name), so it does not free us from having to teach users about it
1. Three annotations (@ServiceReference + @Inject + @Named("name")) instead of only @ServiceReference("name")
1. The semantics for @ServiceReference("name") is not exactly the same as for @Inject. For example:
    1. It is a lazy resolution, there is no guarantee a value is available, and it won't fail unless the task attempts to (retrieve and) use the service
    1. We have no use case to support construction-time population of the property

Note that support for @Inject/@Named could be added in the future, with no impact to the rest of the design and limited impact to implementation. 

## Additional reading

### Design spec

The design spec (including comments from reviewers) is available [here](https://docs.google.com/document/d/15dlxPKI0SZvmwgoMLtdGjlr4DbXp1VqMOzXpLEEWgj4/edit#).

### Related Github tickets

* https://github.com/gradle/gradle/issues/16168
* https://github.com/gradle/gradle/issues/16170

