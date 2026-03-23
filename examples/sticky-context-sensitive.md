# Sticky Context Sensitive Slots

The [openclaw-sticky-context](https://github.com/jamebobob/openclaw-sticky-context) plugin
supports a `sensitive: true` flag on slots. Sensitive slots are automatically redacted
before being injected into lower-trust agents (like your social-* agents).

## Example: Identity Anchor

Your main agent needs operational details (IP, location, infrastructure) to do its job.
Your social-* agents should NOT see these details.

### Setting the slot

```
sticky_set("identity-anchor", "I'm [name]. AI on [server]. Human: [operator], [location]. Primary channel: [channel].", { pinned: true, sensitive: true, priority: 90 })
```

### What the main agent sees (unredacted)

```
I'm Assistant. AI on HomeServer (10.0.0.100). Human: Alex (he/him), Portland OR. Primary channel: Telegram.
```

### What social-* agents see (redacted)

```
I'm Assistant. AI on HomeServer ([REDACTED]). Human: Alex (he/him), [REDACTED]. Primary channel: Telegram.
```

## Redaction Patterns

The default redaction targets:
- IPv4 addresses (`10.x.x.x`, `172.16.x.x`, etc.)
- API keys and tokens (common patterns)
- SSH connection strings
- Port numbers in sensitive contexts

**Important limitation:** Redaction uses regex patterns, not semantic understanding.
A city name like "Portland" won't be caught by the regex. If your identity anchor
contains sensitive terms that aren't pattern-matchable, either:
- Don't include them in the slot (reference a file instead)
- Use a generic version ("a city in the Pacific Northwest")

This is why the privacy framework uses multiple layers. Sticky-context redaction
handles pattern-matchable secrets. The privacy protocol handles known sensitive
terms via grep. Pool routing handles stored memories. No single layer catches
everything.

## Other Sensitive Slots

Common candidates for `sensitive: true`:
- Infrastructure details (IPs, ports, hostnames)
- API configuration (keys, endpoints)
- Personal operator details (address, phone, schedule)
- Security configuration (SSH, firewall rules)

Slots without `sensitive: true` are injected identically to all agents.
