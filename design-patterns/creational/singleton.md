<p><a target="_blank" href="https://app.eraser.io/workspace/FSy5zL1etbdg9ghjBBp9" id="edit-in-eraser-github-link"><img alt="Edit in Eraser" src="https://firebasestorage.googleapis.com/v0/b/second-petal-295822.appspot.com/o/images%2Fgithub%2FOpen%20in%20Eraser.svg?alt=media&amp;token=968381c8-a7e7-472a-8ed6-4a6626da5501"></a></p>

**Intent:** Ensure a class has exactly one instance and provide a global access point to it.

**Use when:** You need a single, shared, stateful resource — like a config manager, thread pool, or registry.

### Real-World Example: Application Configuration Registry
java

```java
import java.util.HashMap;
import java.util.Map;

// Enum-based Singleton — safest approach (JVM-guaranteed, thread-safe, serialization-safe)
public enum AppConfig {
    INSTANCE;

    private final Map<String, String> config = new HashMap<>();

    AppConfig() {
        System.out.println("[Config] Loading configuration (runs ONCE)...");
        config.put("app.env", "production");
        config.put("max.connections", "100");
        config.put("retry.attempts", "3");
        config.put("feature.dark-mode", "true");
    }

    public String get(String key) {
        return config.getOrDefault(key, "");
    }

    public int getInt(String key) {
        return Integer.parseInt(config.getOrDefault(key, "0"));
    }

    public boolean isEnabled(String featureKey) {
        return Boolean.parseBoolean(config.get(featureKey));
    }
}

// Usage — same instance everywhere, loaded once
public class NotificationService {
    public void send(String message) {
        int retries = AppConfig.INSTANCE.getInt("retry.attempts");
        System.out.printf("Sending '%s' with %d retry attempts%n", message, retries);
    }
}

public class Main {
    public static void main(String[] args) {
        // All three calls return the SAME instance
        System.out.println(AppConfig.INSTANCE.get("app.env"));
        System.out.println(AppConfig.INSTANCE.getInt("max.connections"));
        new NotificationService().send("Welcome!");
    }
}
```
**Output:**

```
[Config] Loading configuration (runs ONCE)...
production
100
Sending 'Welcome!' with 3 retry attempts
```




<!--- Eraser file: https://app.eraser.io/workspace/FSy5zL1etbdg9ghjBBp9 --->