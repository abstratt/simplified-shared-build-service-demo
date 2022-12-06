# Simplified consumption of Shared Build Services

This work proposes a new kind of property called "Service Reference" to simplify the programming model for consuming shared build services. 

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

For Gradle 8+ (release to be confirmed), we intend to release a new model for consuming shared build services (issue). In this new model, all that is required is:

1. declaring a property typed using the service type
2. marking with `@ServiceReference("nameOfService")`

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
1. in the new model, the consumer needs to know the name the service was published under
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

### A task that uses the pre-8+ model

```
./gradlew counter1
```

### A task that uses `@ServiceReference` without providing a service name

```
./gradlew counter2
```


### A task that uses `@ServiceReference("serviceName")`
```
./gradlew counter3
```

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

