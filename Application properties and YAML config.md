# Spring Boot — Application Properties & YAML Config

So your app needs to connect to a database. Or call an external API. Or know what port to run on.

You _could_ hardcode those values directly in your Java code. But then every developer on your team would need different code for their local setup. And you'd have to change the code every time you deployed to a different environment. And if you ever accidentally committed an API key to GitHub... well. That's a bad day.

This is exactly what **application properties** are for. They let you store configuration _outside_ your code — in a dedicated file that's easy to change, easy to share with your team, and easy to keep secrets out of.

Let's build up from scratch.

---

## What kind of things go in here?

Typically things that change between environments, or that you just don't want buried in your Java code. Common examples:

- What port your app runs on
- Your database URL and credentials
- The base URL of an external API you're calling
- How verbose your logging should be
- Feature flags (is this feature turned on or off?)
- Business config like a timeout value, a max result count, a currency code

The rule of thumb: if it's a value that _could_ change without the logic of your code changing, it probably belongs in config.

---

## Your first key-value pair

Spring Boot looks for a file called `application.properties` in `src/main/resources`. It's created automatically when you start a new project. Open it up and add this:

```properties
person.name=Alice
```

That's it. A key (`person.name`), an equals sign, and a value (`Alice`). Everything in a properties file follows this same flat format.

---

## Reading that value in your code

Now — how do we actually use it? Spring gives us the `@Value` annotation. Put it on a field and Spring will inject the matching config value before your app starts handling requests:

```java
@Service
public class GreetingService {

    @Value("${person.name}")
    private String name;

    public String greet() {
        return "Hello, " + name + "!";
    }
}
```

The `${person.name}` syntax tells Spring: _go look up the key `person.name` in your config and put it here_. When the app starts, Spring finds the field, resolves the value, and sets it. By the time `greet()` is ever called, `name` is already `"Alice"`.

---

## Adding more values

Let's say we need a few more bits of config:

```properties
person.name=Alice
person.favouriteThing=Coffee
person.homeTown=Portsmouth
```

We can pull each one in the same way:

```java
@Service
public class GreetingService {

    @Value("${person.name}")
    private String name;

    @Value("${person.favouriteThing}")
    private String favouriteThing;

    @Value("${person.homeTown}")
    private String homeTown;
}
```

This works — but look at how much repetition there is. Three annotations, three string literals, all slightly different. Add five more properties and it becomes a real chore. And if you ever typo a key (`${person.favoriteThing}` vs `${person.favouriteThing}`) Spring won't catch it until runtime.

There's a better way.

---

## Switch to YAML — same config, more readable

Rename `application.properties` to `application.yml`. Spring picks it up automatically, nothing else changes.

YAML uses indentation to express structure. Instead of repeating the `person.` prefix on every line, you nest the keys:

```yaml
# application.properties           →    # application.yml
person.name=Alice                        person:
person.favouriteThing=Coffee               name: Alice
person.homeTown=Portsmouth                 favouriteThing: Coffee
                                           homeTown: Portsmouth
```

Same data. But in YAML it's immediately obvious these three values belong together under `person`. As your config grows — database settings, external APIs, logging — the structure makes it far easier to scan and understand at a glance.

> Your existing `@Value("${person.name}")` annotations keep working with no changes. The dot-notation maps directly onto the YAML hierarchy.

---

## Don't want @Value everywhere? Make a config object instead

`@Value` is handy for a single field, but once you're injecting several related values across your codebase it gets messy. A cleaner approach is `@ConfigurationProperties` — it maps a whole group of YAML values onto a single Java class:

```java
@ConfigurationProperties(prefix = "person")
@Configuration
public class PersonProperties {

    private String name;
    private String favouriteThing;
    private String homeTown;

    // getters and setters...
}
```

Spring sees `prefix = "person"` and automatically maps:

- `person.name` → `setName()`
- `person.favouriteThing` → `setFavouriteThing()`
- `person.homeTown` → `setHomeTown()`

Then inject the object wherever you need it:

```java
@Service
public class GreetingService {

    private final PersonProperties person;

    public GreetingService(PersonProperties person) {
        this.person = person;
    }

    public String greet() {
        return "Hello " + person.getName() + " from " + person.getHomeTown() + "!";
    }
}
```

One injection, all the values. And because it's a proper Java object, your IDE can autocomplete `person.get...` — no more hoping you spelled the key right.

---

## The problem with committing your YAML

Your `application.yml` gets pushed to Git. That's intentional — you want your team to share the same config.

But what about secrets? API keys, database passwords, credentials for third-party services?

**Never put those in a file that goes to Git.** Even in a private repo. Even "just temporarily". Once a secret is in Git history it's very difficult to fully remove.

The answer is **environment variables**. Instead of putting the secret value in your YAML, you put a placeholder:

```yaml
cheese-api:
  api-key: ${CHEESE_API_KEY}
```

When Spring starts up and sees `${CHEESE_API_KEY}`, it doesn't look for that value in the file — it looks for an environment variable with that name on the machine running the app. Your YAML is safe to commit. The secret never lives in it.

---

## How does Spring find environment variables?

When Spring resolves a `${}` placeholder, it checks a list of sources in priority order:

1. Command-line arguments
2. OS environment variables
3. A `.env` file (if you've set one up — see below)
4. `application.yml` / `application.properties`

So `${CHEESE_API_KEY}` causes Spring to ask: _is there an environment variable called `CHEESE_API_KEY`?_ If yes, use it. If not, keep looking. If nothing's found anywhere, the app fails to start with a clear error telling you exactly which property is missing.

---

## Using a .env file locally

Setting environment variables manually in your terminal every time gets old fast. The more convenient approach is a **`.env` file** — a plain text file in your project root listing your local values:

```bash
# .env
CHEESE_API_KEY=sk_test_abc123
DB_PASSWORD=localpassword
```

Two things to do straight away:

**1. Add `.env` to your `.gitignore`:**

```
# .gitignore
.env
```

Do this before you put anything sensitive in it.

**2. Add the `spring-dotenv` dependency** so Spring knows to load it:

```xml
<!-- pom.xml -->
<dependency>
    <groupId>me.paulschwarz</groupId>
    <artifactId>spring-dotenv</artifactId>
    <version>4.0.0</version>
</dependency>
```

```groovy
// build.gradle
implementation 'me.paulschwarz:spring-dotenv:4.0.0'
```

That's all. `spring-dotenv` registers itself as a Spring property source — the same mechanism used by `application.yml` and OS environment variables — and loads your `.env` file at startup. Your `${CHEESE_API_KEY}` placeholder will resolve to whatever value is in `.env`.

One more useful habit: commit a `.env.example` with the same keys but fake values, so anyone cloning the repo knows what they need to set up:

```bash
# .env.example  ← safe to commit
CHEESE_API_KEY=your_key_here
DB_PASSWORD=your_password_here
```

In production you won't use `.env` at all — you set environment variables directly in your hosting environment (AWS, Heroku, Railway, etc). Your code doesn't care where the value came from. `${CHEESE_API_KEY}` works the same either way.

---

## A practical example — a REST client backed by config

Let's bring everything together. Your app calls an external **Cheese API** to look up cheese details. The client needs a base URL, an API key, and a timeout. The base URL and timeout are fine to commit — they're not sensitive. The API key is a secret.

Here's the YAML:

```yaml
# application.yml
cheese-api:
  base-url: https://api.cheesehub.com
  api-key: ${CHEESE_API_KEY}           # secret — not stored here, comes from .env
  timeout-seconds: 10
```

And the `.env` on your local machine:

```bash
# .env  ← never committed
CHEESE_API_KEY=sk_test_brie_abc123
```

Now create a `@ConfigurationProperties` class to hold these values as a proper object:

```java
@ConfigurationProperties(prefix = "cheese-api")
@Configuration
public class CheeseApiProperties {

    private String baseUrl;
    private String apiKey;
    private int timeoutSeconds;

    public String getBaseUrl() { return baseUrl; }
    public void setBaseUrl(String baseUrl) { this.baseUrl = baseUrl; }

    public String getApiKey() { return apiKey; }
    public void setApiKey(String apiKey) { this.apiKey = apiKey; }

    public int getTimeoutSeconds() { return timeoutSeconds; }
    public void setTimeoutSeconds(int timeoutSeconds) { this.timeoutSeconds = timeoutSeconds; }
}
```

Spring maps `cheese-api.base-url` → `setBaseUrl()`, `cheese-api.api-key` → `setApiKey()`, and so on. By the time your app starts, `CheeseApiProperties` is fully populated — including `apiKey`, which Spring resolved by looking up `CHEESE_API_KEY` in your `.env` file.

Now inject it into your REST client:

```java
@Service
public class CheeseApiClient {

    private final CheeseApiProperties config;

    public CheeseApiClient(CheeseApiProperties config) {
        this.config = config;
    }

    public CheeseDetails getCheese(String cheeseId) {
        return WebClient.builder()
                .baseUrl(config.getBaseUrl())
                .defaultHeader("Authorization", "Bearer " + config.getApiKey())
                .build()
                .get()
                .uri("/cheeses/{id}", cheeseId)
                .retrieve()
                .bodyToMono(CheeseDetails.class)
                .timeout(Duration.ofSeconds(config.getTimeoutSeconds()))
                .block();
    }
}
```

The _logic_ of how to call the API lives in the code. The _details_ — which URL, which key, what timeout — live in config. To point the app at a different Cheese API in production, or rotate the API key, you change config. The code stays untouched.

---

## What you'll typically see in a real application.yml

Here's a fuller YAML covering the categories that come up on almost every Spring Boot project:

```yaml
# ─── App Identity ─────────────────────────────────────────────────────────────
spring:
  application:
    name: my-app                  # appears in logs and monitoring dashboards


# ─── Server ───────────────────────────────────────────────────────────────────
server:
  port: 8082                      # default is 8080 — change to avoid port clashes
  servlet:
    context-path: /api/v1         # prefixes every controller automatically
  tomcat:
    connection-timeout: 5000      # drop slow clients after 5 seconds


# ─── Database ─────────────────────────────────────────────────────────────────
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/my_db
    username: myuser
    password: ${DB_PASSWORD}      # secret — never hardcoded
  jpa:
    hibernate:
      ddl-auto: validate          # checks schema matches your entities at startup
    show-sql: false               # flip to true locally to see every SQL query


# ─── External APIs ────────────────────────────────────────────────────────────
cheese-api:
  base-url: https://api.cheesehub.com
  api-key: ${CHEESE_API_KEY}      # secret — resolved from .env locally, env var in prod
  timeout-seconds: 10
  retry-attempts: 3


# ─── Logging ──────────────────────────────────────────────────────────────────
logging:
  level:
    root: WARN                    # quiet by default — warnings and errors only
    io.myapp: DEBUG               # your own code — verbose
    org.springframework.web: INFO # see incoming HTTP requests
    org.hibernate.SQL: DEBUG      # see the SQL being run
  file:
    name: logs/app.log


# ─── Custom Business Config ───────────────────────────────────────────────────
myapp:
  features:
    cheese-search-enabled: true
    premium-pairings-enabled: false   # not launched yet — code ships, feature stays off

  catalogue:
    low-stock-threshold: 5
    max-results-per-search: 50
    currency: GBP

  cache:
    cheese-list-ttl-minutes: 60
```

**`server.port`** — change from the default 8080 to avoid clashes when running multiple services locally at the same time.

**`server.servlet.context-path`** — setting this to `/api/v1` means every controller in your app is automatically prefixed with that path. You never have to write it in your `@RequestMapping` annotations. Great for versioning your API cleanly.

**`spring.jpa.hibernate.ddl-auto`** — controls what Hibernate does to your database schema at startup. Use `validate` in production (checks your entities match the schema and fails fast if not). Use `create-drop` locally (wipes and recreates the schema every restart — great for dev, catastrophic in prod).

**`logging.level`** — tune verbosity per package. Your own code at `DEBUG`, everything else at `WARN`. Keeps logs useful without drowning in Spring framework noise.

**Feature flags** — a boolean in YAML is the simplest possible feature toggle. Ship the code, keep the flag off. When you're ready, flip it. No redeploy needed.

**Business config** — values like `low-stock-threshold` or `max-results-per-search` feel like constants, but putting them in YAML means changing them is a one-line config update rather than a code change, PR, review, and full deployment cycle.

---

## The mental model

```
application.yml           → shared config, committed to Git
.env                      → local secrets, never committed (.gitignore it)
${VAR_NAME}               → tells Spring to look for an environment variable
spring-dotenv             → teaches Spring to read your .env file
@Value                    → inject a single config value into a field
@ConfigurationProperties  → map a group of config values onto a Java class
```

Config belongs outside your code. Secrets belong outside your repo. ☕