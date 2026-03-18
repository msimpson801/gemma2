# The Strategy Pattern with Bean Map Injection

> _You've got five music services and a conditional spaghetti soup. There's a better way._

---

## The Problem Nobody Talks About

Picture this: you're building a feature that needs to delegate work to different implementations depending on some runtime value — a genre, a payment provider, a notification channel. Your first instinct? A fat `if-else` block, or maybe a `switch`. It works. It's also the kind of code that makes your future self sigh audibly.

```java
// 😬 Please don't do this
@Service
public class MusicPlayer {
    private final FunkMusicService funkMusicService;
    private final GrungeMusicService grungeMusicService;
    private final PopMusicService popMusicService;
    // ...a new field every time a genre is added

    public MusicPlayer(FunkMusicService funkMusicService,
                       GrungeMusicService grungeMusicService,
                       PopMusicService popMusicService) {
        this.funkMusicService = funkMusicService;
        this.grungeMusicService = grungeMusicService;
        this.popMusicService = popMusicService;
    }

    public List<String> getSongs(String genre) {
        if (genre.equals("funk")) {
            return funkMusicService.getSongs();
        } else if (genre.equals("grunge")) {
            return grungeMusicService.getSongs();
        } else if (genre.equals("pop")) {
            return popMusicService.getSongs();
        }
        // ...and another branch every sprint
        throw new IllegalArgumentException("Unknown genre: " + genre);
    }
}
```

Every new genre means touching this method. Every new service means a new injected field cluttering your class. This violates the Open/Closed Principle — and more practically, it's just annoying to maintain.

Spring Boot has a quiet, elegant solution that most devs walk right past.

---

## The Interface: Your Contract

Start with a clean interface. Every music service agrees to this deal:

```java
public interface MusicService {
    List<String> getSongs();
    String getGenre();
}
```

Note that `getGenre()` method — it'll pull its weight later.

Now your implementations:

```java
@Service
public class FunkMusicService implements MusicService {
    public List<String> getSongs() { return List.of("Give Up the Funk", "Super Freak"); }
    public String getGenre() { return "funk"; }
}

@Service
public class GrungeMusicService implements MusicService {
    public List<String> getSongs() { return List.of("Smells Like Teen Spirit", "Black Hole Sun"); }
    public String getGenre() { return "grunge"; }
}

@Service
public class PopMusicService implements MusicService {
    public List<String> getSongs() { return List.of("Billie Jean", "Bad"); }
    public String getGenre() { return "pop"; }
}
```

Three services. Three `@Service` annotations. Zero wiring ceremony.

---

## The Magic: Spring's Bean Map Injection

Here's the trick Spring doesn't advertise loudly enough:

> **If you inject `Map<String, MyInterface>`, Spring will auto-populate it with every bean that implements that interface — keyed by bean name.**

```java
@Service
public class MusicPlayer {
    private final Map<String, MusicService> musicServices;

    public MusicPlayer(Map<String, MusicService> musicServices) {
        this.musicServices = musicServices;
    }
}
```

That's it. Spring detects all `MusicService` implementations and hands you a map keyed by bean name — but more on why that's a problem in a moment.

No `@Qualifier`. No five separate injected fields. No XML (mercifully). Spring just... does it.

---

## The Key Mismatch Problem

Once Spring injects the map, you'll hit a practical snag. The default keys are the camelCase bean names — the full class name with a lowercase first letter:

|Key|Value|
|---|---|
|`"funkMusicService"`|`FunkMusicService` instance|
|`"grungeMusicService"`|`GrungeMusicService` instance|
|`"popMusicService"`|`PopMusicService` instance|

But your controller is probably passing in something clean like `"funk"` or `"pop"`:

```java
// Request comes in as: GET /songs?genre=funk
public List<String> getSongs(@RequestParam String genre) {
    return musicPlayer.getSongs(genre); // genre = "funk"
}
```

`musicServices.get("funk")` returns `null` — because the key is actually `"funkMusicService"`. The map lookup silently fails.

You have two options to fix this.

---

## Approach 1: Re-key the Map in the Constructor

You can remap the keys using each service's own `getGenre()` method right in the constructor:

```java
@Service
public class MusicPlayer {
    private final Map<String, MusicService> servicesByGenre;

    public MusicPlayer(Map<String, MusicService> musicServices) {
        this.servicesByGenre = musicServices.entrySet().stream()
            .collect(Collectors.toMap(
                entry -> entry.getValue().getGenre(), // ← your clean key
                Map.Entry::getValue
            ));
    }

    public List<String> getSongs(String genre) {
        MusicService service = servicesByGenre.get(genre);
        if (service == null) throw new IllegalArgumentException("Unknown genre: " + genre);
        return service.getSongs();
    }
}
```

Now calling the player is clean and extensible:

```java
musicPlayer.getSongs("funk");   // ["Give Up the Funk", "Super Freak"]
musicPlayer.getSongs("grunge"); // ["Smells Like Teen Spirit", "Black Hole Sun"]
```

Adding a new genre? Write a new `@Service` class. **Touch nothing else.** The map self-updates at startup. Your `play()` method doesn't care.

---

## Approach 2: Name the Bean Directly with `@Service("name")` ✅ Preferred

The cleaner fix is to just tell Spring what key to use at the bean definition level, so the map is already correct on injection — no transformation needed:

```java
@Service("funk")   // ← becomes the map key
public class FunkMusicService implements MusicService { ... }

@Service("grunge")
public class GrungeMusicService implements MusicService { ... }

@Service("pop")
public class PopMusicService implements MusicService { ... }
```

Now the injected map already has clean keys — no stream, no transformation:

```java
@Service
public class MusicPlayer {
    private final Map<String, MusicService> musicServices;

    public MusicPlayer(Map<String, MusicService> musicServices) {
        this.musicServices = musicServices; // keys: "funk", "grunge", "pop"
    }
}
```

The injected map already has the right keys — `"funk"`, `"grunge"`, `"pop"` — matching exactly what a controller would pass in. No stream, no transformation, no mismatch. This is the preferred approach when your keys are fixed strings.

---

## Approach 3: `@Bean` in a `@Configuration` Class

When you need more control — third-party classes, conditional logic, constructor arguments — reach for `@Configuration`:

```java
@Configuration
public class MusicConfig {
    @Bean("funk")
    public MusicService funkMusicService() {
        return new FunkMusicService(/* custom args */);
    }
}
```

Same injection pattern, more instantiation power.

---

## Which Approach Should You Use?

|Approach|Best When|
|---|---|
|**Re-key in constructor**|Key logic lives in the service itself (e.g. `getGenre()`) — keeps the contract self-describing|
|**`@Service("name")`**|Keys are fixed, simple strings — cleanest and most readable|
|**`@Bean` in `@Configuration`**|You need control over instantiation, or classes are from external libraries|

---

## The Strategy Pattern + Spring DI

This is the **Strategy Pattern** — a classic OOP design pattern where you define a family of algorithms (or behaviours) behind a common interface, and swap between them at runtime. Spring's bean map injection is a natural fit: it acts as the strategy registry, and you get it for free.

This scales beautifully to real-world scenarios:

- **Payment processors** — `StripeService`, `PaypalService`, `BraintreeService` all implementing `PaymentService`
- **Notification channels** — `EmailNotifier`, `SmsNotifier`, `PushNotifier` all implementing `Notifier`
- **Export formats** — `PdfExporter`, `CsvExporter`, `ExcelExporter` all implementing `ReportExporter`

The pattern is the same. The ceremony is minimal. The extension story is great.

---

## Quick Reference

```java
// 1. Define the interface
public interface MusicService {
    List<String> getSongs();
    String getGenre();
}

// 2. Implement it (once per strategy)
@Service("funk")
public class FunkMusicService implements MusicService {
    public List<String> getSongs() { return List.of("Give Up the Funk", "Super Freak"); }
    public String getGenre() { return "funk"; }
}

// 3. Inject the map — Spring handles the rest
@Service
public class MusicPlayer {
    private final Map<String, MusicService> musicServices;

    public MusicPlayer(Map<String, MusicService> musicServices) {
        this.musicServices = musicServices;
    }

    public List<String> getSongs(String genre) {
        return musicServices.get(genre).getSongs();
    }
}
```

No more `if-else`. No more touching the dispatcher every time a new service appears. Just clean, open-for-extension, closed-for-modification Spring code — and a DJ that never runs out of genres. 🎶

---

_Written for Spring Boot 3.x. Bean map injection has been supported since Spring 4, so you're probably already covered._