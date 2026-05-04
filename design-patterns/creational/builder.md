<p><a target="_blank" href="https://app.eraser.io/workspace/VyiauoQEKe7w5Cosobo0" id="edit-in-eraser-github-link"><img alt="Edit in Eraser" src="https://firebasestorage.googleapis.com/v0/b/second-petal-295822.appspot.com/o/images%2Fgithub%2FOpen%20in%20Eraser.svg?alt=media&amp;token=968381c8-a7e7-472a-8ed6-4a6626da5501"></a></p>

**Intent:** Construct complex objects step by step, separating construction from representation.

**Use when:** An object requires many configuration parameters (especially optional ones) and a multi-argument constructor would be unreadable and error-prone.

### Real-World Example: Building a Notification Request
java

```java
import java.time.LocalDateTime;
import java.util.ArrayList;
import java.util.List;

public class NotificationRequest {
    // Required
    private final String recipient;
    private final String message;
    // Optional with defaults
    private final String channel;
    private final String priority;
    private final boolean retryOnFailure;
    private final int maxRetries;
    private final LocalDateTime scheduledAt;
    private final List<String> tags;

    private NotificationRequest(Builder builder) {
        this.recipient       = builder.recipient;
        this.message         = builder.message;
        this.channel         = builder.channel;
        this.priority        = builder.priority;
        this.retryOnFailure  = builder.retryOnFailure;
        this.maxRetries      = builder.maxRetries;
        this.scheduledAt     = builder.scheduledAt;
        this.tags            = List.copyOf(builder.tags);
    }

    @Override
    public String toString() {
        return String.format(
                "NotificationRequest { recipient='%s', channel='%s', priority='%s', " +
                "retry=%s, maxRetries=%d, scheduledAt=%s, tags=%s, message='%s' }",
                recipient, channel, priority, retryOnFailure, maxRetries,
                scheduledAt, tags, message);
    }

    public static class Builder {
        private final String recipient;
        private final String message;
        private String channel   = "EMAIL";
        private String priority  = "NORMAL";
        private boolean retryOnFailure = false;
        private int maxRetries   = 0;
        private LocalDateTime scheduledAt;
        private final List<String> tags = new ArrayList<>();

        public Builder(String recipient, String message) {
            if (recipient == null || recipient.isBlank())
                throw new IllegalArgumentException("Recipient must not be blank.");
            if (message == null || message.isBlank())
                throw new IllegalArgumentException("Message must not be blank.");
            this.recipient = recipient;
            this.message = message;
        }

        public Builder viaChannel(String channel) {
            this.channel = channel;
            return this;
        }

        public Builder withPriority(String priority) {
            this.priority = priority;
            return this;
        }

        public Builder retryOnFailure(int maxRetries) {
            this.retryOnFailure = true;
            this.maxRetries = maxRetries;
            return this;
        }

        public Builder scheduledAt(LocalDateTime scheduledAt) {
            this.scheduledAt = scheduledAt;
            return this;
        }

        public Builder withTag(String tag) {
            this.tags.add(tag);
            return this;
        }

        public NotificationRequest build() {
            return new NotificationRequest(this);
        }
    }
}

// Usage
public class Main {
    public static void main(String[] args) {
        // Minimal — required fields only
        NotificationRequest simple = new NotificationRequest.Builder(
                "user@example.com", "Your order has shipped!")
                .build();

        // Full configuration — reads like a sentence
        NotificationRequest urgentSms = new NotificationRequest.Builder(
                "+91-9876543210", "OTP: 482910")
                .viaChannel("SMS")
                .withPriority("HIGH")
                .retryOnFailure(3)
                .withTag("otp")
                .withTag("security")
                .build();

        // Scheduled report notification
        NotificationRequest scheduled = new NotificationRequest.Builder(
                "admin@company.com", "Weekly report is ready")
                .viaChannel("EMAIL")
                .withPriority("LOW")
                .scheduledAt(LocalDateTime.of(2026, 5, 5, 9, 0))
                .withTag("report")
                .build();

        System.out.println(simple);
        System.out.println(urgentSms);
        System.out.println(scheduled);
    }
}
```




<!--- Eraser file: https://app.eraser.io/workspace/VyiauoQEKe7w5Cosobo0 --->