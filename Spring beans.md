# 🌍 Spring Beans: A Beginner's Guide (Featuring the World's Simplest Country API)

Let's build something tangible. A tiny API about countries — you hit an endpoint, you get country info back. Simple. But along the way, you'll see exactly how Spring Boot's bean system works, from the moment the app starts to the moment it shuts down.

---

## What Even Is a Bean?

A bean is just a fancy word for **an object that Spring manages for you**.

In normal Java, if you need an object, you create it yourself:

```java
CountryService countryService = new CountryService();
```

But in Spring, you don't do that. You just annotate your class, and Spring creates it, stores it, and hands it to whoever needs it.

```java
@Service
public class CountryService {
    public String describe(String name) {
        return name + " is a wonderful country.";
    }
}
```

That `@Service` annotation is your way of saying: _"Spring, please manage this class as a bean."_

---

## The Application Context: Spring's Registry

Spring keeps all its beans in something called the **Application Context** — a central registry of every object the app needs to run.

Every bean — the `CountryService`, the `CountryRepository`, the `CountryController` — lives in there. When any part of the app needs something, it doesn't create a new one. It asks the Application Context.

```
📦 The Application Context
   ├── CountryRepository   (talks to the database)
   ├── CountryService      (contains the business logic)
   └── CountryController   (handles HTTP requests)
```

---

## One Instance for Everyone: Singleton Scope

By default, Spring only creates **one** `CountryService` for the entire application.

Even if multiple controllers need a `CountryService`, they all share the **same** instance. Spring doesn't create a new one each time — it reuses the one already in the Application Context.

This is called **singleton scope**, and it's Spring's default behaviour.

```java
// Both get the SAME CountryService instance — cities need country data too
@RestController
public class CountryController {
    private final CountryService countryService; // same instance
}

@RestController
public class CityController {
    private final CountryService countryService; // same instance
}
```

Why? Because creating objects is expensive. One instance, shared everywhere.

---

## Dependency Injection: Wiring It All Together

`CountryService` can't work alone — it needs a `CountryRepository` to fetch country data from the database.

```java
@Service
public class CountryService {
    private final CountryRepository countryRepository;

    public CountryService(CountryRepository countryRepository) {
        this.countryRepository = countryRepository;
    }

    public Country getByName(String name) {
        return countryRepository.findByName(name);
    }
}
```

Notice: **nobody calls `new CountryService(repository)` anywhere**. So where does the `CountryRepository` come from?

Spring figures it out. When creating `CountryService`, it looks at the constructor and thinks:

> _"This needs a `CountryRepository`. I have one in my Application Context. Let me pass it in."_

This is **dependency injection** — Spring wires the pieces together so you don't have to.

---

## Component Scanning: How Spring Discovers Everything

At startup, Spring scans your project for annotated classes. This is called **component scanning** — Spring walks your packages and registers anything it finds:

|Annotation|Typical use|
|---|---|
|`@Component`|Generic bean|
|`@Service`|Business logic|
|`@Repository`|Database access|
|`@Controller` / `@RestController`|HTTP request handling|

```java
@Repository
public class CountryRepository {
    public Country findByName(String name) {
        // imagine this hits a real database
        return new Country(name, "Unknown capital", 0);
    }
}
```

---

## The Full Bean Lifecycle

Here's everything that happens from `main()` to shutdown:

```
1. 🔍 COMPONENT SCAN
   Spring finds @Service, @Repository, @RestController etc.
   "Found CountryRepository! Found CountryService! Found CountryController!"

2. 📋 BEAN DEFINITIONS REGISTERED
   Spring plans which beans to create — not yet, just noted.

3. 🌱 INSTANTIATION
   Spring works out the dependency order and creates beans.
   CountryRepository first (no dependencies).
   Then CountryService (needs CountryRepository).
   Then CountryController (needs CountryService).

4. 💉 DEPENDENCY INJECTION
   Spring passes each bean what it needs via the constructor.
   Everything is wired together.

5. ✅ READY
   All beans are live in the Application Context.
   The API is up. Requests can come in. 🎉

6. 🛑 SHUTDOWN
   App closes. Beans are destroyed.
```

---

## Putting It All Together

```java
// Talks to the database
@Repository
public class CountryRepository {
    public Country findByName(String name) {
        return new Country(name, "Some Capital", 1_000_000);
    }
}

// Contains the business logic
@Service
public class CountryService {
    private final CountryRepository countryRepository;

    public CountryService(CountryRepository countryRepository) {
        this.countryRepository = countryRepository;
    }

    public String describe(String name) {
        Country c = countryRepository.findByName(name);
        return c.getName() + " has a population of " + c.getPopulation();
    }
}

// Handles HTTP requests
@RestController
public class CountryController {
    private final CountryService countryService;

    public CountryController(CountryService countryService) {
        this.countryService = countryService;
    }

    @GetMapping("/country/{name}")
    public String getCountry(@PathVariable String name) {
        return countryService.describe(name);
    }
}
```

Spring scans, finds all three, figures out that `CountryRepository` → `CountryService` → `CountryController` is the right creation order, injects everything, and registers them in the Application Context.

A request comes in for `/country/France`. Spring routes it to `CountryController`, which calls `CountryService`, which calls `CountryRepository`. Data flows back up the chain.

You never called `new` once.

---

## The Big Three to Remember

- 🫘 **A bean** is an object Spring manages for you
- 📦 **The Application Context** is Spring's registry of all beans
- 💉 **Dependency injection** is how Spring wires beans together automatically

That's the heart of Spring. Everything else — security, JPA, caching, scheduling — is just more beans. 🌍