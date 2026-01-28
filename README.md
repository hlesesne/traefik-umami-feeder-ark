# Traefik Umami Feeder (Ark Fork)

A Traefik middleware plugin that sends pageview events to [Umami Analytics](https://umami.is/) - **with custom header capture support**.

This is a fork of [astappiev/traefik-umami-feeder](https://github.com/astappiev/traefik-umami-feeder) v1.4.1 with the following enhancement:

## New Feature: Capture Request Headers

The `captureHeaders` configuration option allows you to capture request headers and include them in the Umami event data. This is useful for tracking authenticated user information injected by forward auth proxies like oauth2-proxy.

### Configuration

```yaml
# In your middleware configuration
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: umami-feeder
  namespace: observability
spec:
  plugin:
    umami-feeder-ark:
      umamiHost: "http://umami.observability.svc.cluster.local:3000"
      websites:
        "ai.samaritanark.cloud": "your-website-id"

      # NEW: Capture authentication headers
      captureHeaders:
        "X-Auth-Request-User": "user"
        "X-Auth-Request-Preferred-Username": "username"
        "X-Auth-Request-Email": "email"
        "X-Auth-Request-Department": "department"
        "X-Auth-Request-Title": "title"
        "X-Auth-Request-Groups": "groups"
```

### How It Works

When a request contains headers matching the keys in `captureHeaders`, their values are stored in the Umami event's `data` field using the mapped name. For example:

- Request header `X-Auth-Request-User: jsmith`
- Configuration: `"X-Auth-Request-User": "user"`
- Result: Event data includes `{"user": "jsmith"}`

This data is stored in Umami's `event_data` table and can be queried for analytics.

### Example Grafana Query

```sql
SELECT
  ed.string_value AS username,
  COUNT(*) AS pageviews,
  COUNT(DISTINCT we.session_id) AS sessions
FROM website_event we
JOIN event_data ed ON ed.website_event_id = we.event_id
WHERE ed.data_key = 'username'
  AND we.created_at >= NOW() - INTERVAL '7 days'
GROUP BY 1
ORDER BY 2 DESC
```

## Traefik Configuration

### Static Configuration (HelmChartConfig for k3s)

```yaml
apiVersion: helm.cattle.io/v1
kind: HelmChartConfig
metadata:
  name: traefik
  namespace: kube-system
spec:
  valuesContent: |-
    experimental:
      plugins:
        umami-feeder-ark:
          moduleName: github.com/samaritan-ark/traefik-umami-feeder-ark
          version: v1.0.0
```

## All Configuration Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `enabled` | bool | `true` | Enable/disable the plugin |
| `debug` | bool | `false` | Enable debug logging |
| `queueSize` | int | `1000` | Event queue buffer size |
| `batchSize` | int | `20` | Events per API batch request |
| `batchMaxWait` | duration | `5s` | Max wait before flushing batch |
| `umamiHost` | string | required | Umami instance URL |
| `umamiToken` | string | | API token for auto website discovery |
| `umamiUsername` | string | | Username for token retrieval |
| `umamiPassword` | string | | Password for token retrieval |
| `umamiTeamId` | string | | Team ID for website scoping |
| `websites` | map | | Manual hostname → website ID mapping |
| `createNewWebsites` | bool | `false` | Auto-create websites via API |
| `trackErrors` | bool | `false` | Track HTTP error responses |
| `trackAllResources` | bool | `false` | Track all requests (not just pages) |
| `trackExtensions` | []string | | Custom file extensions to track |
| `ignoreUserAgents` | []string | | User agents to exclude |
| `ignoreURLs` | []string | | URL regex patterns to exclude |
| `ignoreHosts` | []string | | Hostnames to exclude |
| `ignoreIPs` | []string | | IPs/CIDRs to exclude |
| `headerIp` | string | `X-Real-IP` | Header for client IP extraction |
| **`captureHeaders`** | map | | **NEW: Headers to capture as event data** |

## License

MIT License - see [LICENSE](LICENSE)

## Credits

Original plugin by [Oleksandr Stappiev](https://github.com/astappiev)
