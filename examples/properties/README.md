## properties

this example prevalidates the authentication header via the usage of properties fetched from the proxy (i.e. Envoy metadata here).

### message on clients
```
curl localhost:18000/anything/one -v
*   Trying 127.0.0.1:18000...
* Connected to localhost (127.0.0.1) port 18000 (#0)
> GET /one HTTP/1.1
> Host: localhost:18000
> User-Agent: curl/7.82.0
> Accept: */*
> 
< HTTP/1.1 401 Unauthorized
< date: Mon, 31 Oct 2022 00:53:01 GMT
< server: envoy
< content-length: 0
< 

curl localhost:18000/anything/one -v -H 'cookie: value'
*   Trying 127.0.0.1:18000...
* Connected to localhost (127.0.0.1) port 18000 (#0)
> GET /one HTTP/1.1
> Host: localhost:18000
> User-Agent: curl/7.82.0
> Accept: */*
> cookie: value
> 
< HTTP/1.1 200 OK
< content-length: 13
< content-type: text/plain
< date: Mon, 31 Oct 2022 00:54:59 GMT
< server: envoy
< x-envoy-upstream-service-time: 0
< 
example body

curl localhost:18000/anything/two -v
*   Trying 127.0.0.1:18000...
* Connected to localhost (127.0.0.1) port 18000 (#0)
> GET /two HTTP/1.1
> Host: localhost:18000
> User-Agent: curl/7.82.0
> Accept: */*
> 
< HTTP/1.1 401 Unauthorized
< date: Mon, 31 Oct 2022 00:53:30 GMT
< server: envoy
< content-length: 0
< 

curl localhost:18000/anything/two -v -H 'authorization: token'
*   Trying 127.0.0.1:18000...
* Connected to localhost (127.0.0.1) port 18000 (#0)
> GET /two HTTP/1.1
> Host: localhost:18000
> User-Agent: curl/7.82.0
> Accept: */*
> authorization: token
> 
< HTTP/1.1 200 OK
< content-length: 13
< content-type: text/plain
< date: Mon, 31 Oct 2022 00:53:52 GMT
< server: envoy
< x-envoy-upstream-service-time: 0
< 
example body

curl localhost:18000/anything/three -v
*   Trying 127.0.0.1:18000...
* Connected to localhost (127.0.0.1) port 18000 (#0)
> GET /three HTTP/1.1
> Host: localhost:18000
> User-Agent: curl/7.82.0
> Accept: */*
> 
< HTTP/1.1 200 OK
< content-length: 13
< content-type: text/plain
< date: Mon, 31 Oct 2022 00:55:27 GMT
< server: envoy
< x-envoy-upstream-service-time: 0
< 
example body
```

### message on Envoy
```
wasm log: auth header is "cookie"
wasm log: 2 finished
wasm log: auth header is "cookie"
wasm log: 3 finished
wasm log: auth header is "authorization"
wasm log: 2 finished
wasm log: auth header is "authorization"
wasm log: 3 finished
wasm log: no auth header for route
wasm log: 4 finished
```

## Using proxywasm.GetProperty()

This example demonstrates how to use `proxywasm.GetProperty()` to access various Envoy attributes from within a Wasm filter.

### Official Documentation

For a complete list of available attributes, refer to the official Envoy documentation:
- **[Envoy Attributes Reference](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/advanced/attributes)**

### Key Attributes Demonstrated

#### 1. Route Metadata (Available in OnHttpRequestHeaders)
```go
// Access route-level metadata configured in envoy.yaml
auth, err := proxywasm.GetProperty([]string{
    "xds",
    "route_metadata",
    "filter_metadata",
    "envoy.filters.http.wasm",
    "auth",
})
```

#### 2. Cluster Metadata (Available after routing)
```go
// Access cluster-level metadata - requires "xds" prefix
myValue, err := proxywasm.GetProperty([]string{
    "xds",
    "cluster_metadata",
    "filter_metadata",
    "foo",
    "my_value",
})
```

For map-type metadata, use `GetPropertyMap()`:
```go
metadataMap, err := proxywasm.GetPropertyMap([]string{
    "xds",
    "cluster_metadata",
    "filter_metadata",
    "foo",
})
for _, kv := range metadataMap {
    key, value := kv[0], kv[1]
    proxywasm.LogInfof("metadata[%s] = %s", string(key), string(value))
}
```

#### 3. Other Useful Attributes
```go
// Cluster name
clusterName, _ := proxywasm.GetProperty([]string{"xds", "cluster_name"})

// Upstream address
upstreamAddr, _ := proxywasm.GetProperty([]string{"upstream", "address"})

// Request path
requestPath, _ := proxywasm.GetProperty([]string{"request", "path"})
```

### Attribute Availability by Lifecycle Phase

| Attribute | OnHttpRequestHeaders | OnHttpResponseHeaders | OnHttpResponseBody | OnHttpStreamDone |
|-----------|---------------------|----------------------|-------------------|-----------------|
| `request.*` | ✅ | ✅ | ✅ | ✅ |
| `route_metadata.*` | ✅ | ✅ | ✅ | ✅ |
| `xds.cluster_name` | ❌ | ✅ | ✅ | ✅ |
| `xds.cluster_metadata.*` | ❌ | ✅ | ✅ | ✅ |
| `upstream.address` | ❌ | ✅ | ✅ | ✅ |

**Note**: Cluster-related attributes are only available after routing decision is made, which happens after `OnHttpRequestHeaders`.

### Important: XDS Prefix Requirement

When accessing cluster-related attributes, you **must** use the `xds` prefix:

- ✅ Correct: `["xds", "cluster_metadata", "filter_metadata", "foo", "my_value"]`
- ❌ Wrong: `["cluster_metadata", "filter_metadata", "foo", "my_value"]`

### Configuration Example

In your `envoy.yaml`, configure cluster metadata like this:

```yaml
clusters:
  - name: httpbin
    connect_timeout: 30s
    type: LOGICAL_DNS
    metadata:
      filter_metadata:
        foo:                    # namespace
          my_value: "1234"      # accessible via xds.cluster_metadata.filter_metadata.foo.my_value
          my_map:               # map type - use GetPropertyMap()
            k1: v1
            k2: v2
```
