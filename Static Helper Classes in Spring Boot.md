# Static Helper Classes in Spring Boot

A beginner's guide to what they are, why we use them, and the testing headaches they cause.

---

## What is a Helper Class?

A helper class is a utility class that groups together reusable logic — things like formatting strings, validating inputs, or calculating values. Rather than duplicating that logic across multiple services, you centralise it in one place.

```java
public class PirateHelper {

    public static boolean hasValidCrew(int crewCount) {
        return crewCount >= 3;
    }

    public static String formatShipName(String name) {
        return "The " + name.trim();
    }
}
```

---

## Why Do We Make Helper Methods Static?

Helper methods tend to be **stateless** — they take some input, return some output, and don't need to remember anything between calls. Making them `static` reflects that. It means the method belongs to the class itself, not to any particular instance of it.

The practical benefit is that you can call them directly without creating an object first:

```java
// Clean — no object needed
boolean valid = PirateHelper.hasValidCrew(5);

// Pointless — why create an object just to call one method?
PirateHelper helper = new PirateHelper();
boolean valid = helper.hasValidCrew(5);
```

---

## Do We Need to Register Them With Spring?

No. Static helper classes sit **outside of Spring's world** entirely. You don't need `@Component`, `@Service`, or any other annotation. Spring manages beans — objects it creates, wires together, and injects. A static helper isn't an object you ever instantiate, so there's nothing for Spring to manage.

You just call them directly, from anywhere:

```java
@Service
public class PirateService {

    public void sailAway(int crew) {
        if (PirateHelper.hasValidCrew(crew)) { // called directly, no injection
            // set sail!
        }
    }
}
```

---

## Why You Sometimes See `@UtilityClass`

If you've browsed Spring Boot codebases you may have spotted a Lombok annotation called `@UtilityClass` on some helper classes:

```java
import lombok.experimental.UtilityClass;

@UtilityClass
public class PirateHelper {

    public boolean hasValidCrew(int crewCount) { // no `static` keyword needed
        return crewCount >= 3;
    }

    public String formatShipName(String name) {
        return "The " + name.trim();
    }
}
```

### What Does Lombok's `@UtilityClass` Actually Do?

At compile time, Lombok transforms your class in three ways:

**1. Makes all methods `static` automatically** You don't need to write `static` on every method — Lombok adds it for you.

**2. Makes the constructor `private`** This prevents anyone from accidentally doing `new PirateHelper()`. Since the class is purely a collection of static methods, there's no valid reason to ever create an instance.

**3. Makes the class `final`** Stops anyone subclassing it.

### Why Do We Want Those Things?

Without `@UtilityClass`, nothing stops a developer from doing this:

```java
PirateHelper helper = new PirateHelper(); // compiles fine, but makes no sense
```

The class has no state, no instance methods — the object is completely useless. `@UtilityClass` makes that mistake a **compile error** instead, and signals clearly to anyone reading the code: _this class is never meant to be instantiated_.

It's essentially a way of encoding the intent of the class into the code itself.

---

## The Spock / Groovy Testing Problem

This is where static helpers become a headache.

When you write Spock tests in Groovy, you can mock a Spring bean effortlessly:

```groovy
def pirateService = Mock(PirateService)
pirateService.doSomething(_) >> "mocked result"
```

This works because Spock creates a **fake proxy object** that stands in for the real one. When your code calls a method on it, Spock intercepts the call and returns whatever you told it to.

But that only works on **objects**. A static method isn't called on an object — it's called directly on the class, resolved at compile time. Spock has nothing to intercept.

So if `PirateService` calls a static helper:

```java
if (PirateHelper.hasValidCrew(crew)) { ... }
```

You cannot stub that out in Spock:

```groovy
// ❌ This does not work — Spock can't mock static methods
PirateHelper.hasValidCrew(_) >> true
```

The real `hasValidCrew` will always run in your test, whether you like it or not.

### What Can You Do About It?

**If you own the code** — the simplest fix is to not use a static helper at all. Convert it to a `@Component`, inject it into your service via the constructor, and Spock can mock it like anything else. If you were going to write a workaround anyway, you may as well have done this from the start.

**If you can't change the static class** (legacy code, third-party library) — you can wrap it in a thin `@Component` that delegates to it:

```java
@Component
public class PirateHelperWrapper {
    public boolean hasValidCrew(int crewCount) {
        return PirateHelper.hasValidCrew(crewCount);
    }
}
```

Inject the wrapper into your service instead, and now Spock can mock the wrapper — the static method underneath never gets called in tests.

**If the method is truly simple and pure** (a straightforward calculation with no side effects) — sometimes it's fine to just let it run for real in your test and not bother mocking it at all.

---

## Summary

- Helper classes group reusable, stateless logic in one place
- Methods are made `static` so you can call them without instantiating anything
- They don't need Spring annotations — they live outside Spring's world entirely
- `@UtilityClass` (Lombok) enforces this by auto-adding `static`, making the constructor private, and preventing subclassing
- The downside: Spock cannot mock static methods, so if business logic depends on them, testing gets awkward
- The cleanest long-term solution is usually a `@Component` instead — injectable, mockable, and Spring-idiomatic