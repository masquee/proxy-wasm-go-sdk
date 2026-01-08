
## metrics

this example creates simple request counter with prometheus tags.

```
$ curl localhost:10000 -v -H "my-custom-header: foo" 

$ curl -s 'localhost:9901/stats/prometheus'| grep proxy
# TYPE custom_header_value_counts counter
custom_header_value_counts{value="foo",reporter="wasmgosdk"} 1
```
