# OPA Built-ins Cheatsheet

## Contents
- Strings
- Regex
- Aggregates
- Objects
- Sets and arrays
- Type checking
- Encoding
- Time
- Units
- Net and HTTP
- Crypto and tokens

## Strings

```rego
concat("/", ["a", "b", "c"])         # "a/b/c"
contains("foobar", "bar")            # true
startswith("foobar", "foo")          # true
endswith("foobar", "bar")            # true
indexof_n("abcabc", "bc")            # [1, 4]
lower("FOO")                         # "foo"
upper("foo")                         # "FOO"
replace("foo.bar", ".", "-")         # "foo-bar"
split("a/b/c", "/")                  # ["a", "b", "c"]
trim_space("  foo  ")                # "foo"
trim_prefix("foobar", "foo")        # "bar"
trim_suffix("foobar", "bar")        # "foo"
sprintf("user %s has %d roles", ["alice", 3])  # formatted string
substring("abcdef", 2, 3)           # "cde"
```

## Regex

```rego
regex.match(`^[a-z]+$`, "hello")     # true
regex.find_n(`\d+`, "a1b22c333", -1) # ["1", "22", "333"]
regex.replace("foo123", `\d+`, "X")  # "fooX"
regex.split(`\s+`, "a b  c")        # ["a", "b", "c"]
```

Use raw strings (backticks) to avoid double-escaping.

## Aggregates

```rego
count([1, 2, 3])        # 3
count({"a", "b"})        # 2
sum([1, 2, 3])           # 6
product([2, 3, 4])       # 24
max([1, 3, 2])           # 3
min([1, 3, 2])           # 1
sort([3, 1, 2])          # [1, 2, 3]
```

## Objects

```rego
object.get(obj, "key", "default")            # safe access with default
object.keys({"a": 1, "b": 2})               # {"a", "b"}
object.remove(obj, {"key1", "key2"})         # remove keys
object.union(obj1, obj2)                      # merge (obj2 wins)
object.filter(obj, {"key1"})                 # keep only listed keys
json.patch(obj, [{"op": "add", "path": "/new", "value": 1}])
```

## Sets and arrays

```rego
# Sets
s := {"a", "b", "c"}
"a" in s                  # membership
s & {"b", "c", "d"}      # intersection: {"b", "c"}
s | {"d"}                 # union: {"a", "b", "c", "d"}
s - {"a"}                 # difference: {"b", "c"}

# Arrays
array.concat([1, 2], [3, 4])    # [1, 2, 3, 4]
array.slice([1, 2, 3, 4], 1, 3) # [2, 3]
array.reverse([1, 2, 3])        # [3, 2, 1]
```

## Type checking

```rego
is_string("foo")          # true
is_number(42)             # true
is_boolean(true)          # true
is_array([1, 2])          # true
is_set({"a"})             # true
is_object({"a": 1})       # true
is_null(null)             # true
type_name("foo")          # "string"
```

## Encoding

```rego
json.marshal({"a": 1})              # "{\"a\":1}"
json.unmarshal("{\"a\":1}")          # {"a": 1}
yaml.marshal({"a": 1})              # "a: 1\n"
yaml.unmarshal("a: 1")              # {"a": 1}
base64.encode("hello")              # "aGVsbG8="
base64.decode("aGVsbG8=")           # "hello"
urlquery.encode("a b")              # "a+b"
urlquery.decode("a+b")              # "a b"
```

## Time

```rego
time.now_ns()                        # current time in nanoseconds
time.parse_rfc3339_ns("2024-01-15T10:30:00Z")
time.parse_duration_ns("1h30m")      # 5400000000000
time.date(time.now_ns())             # [year, month, day]
time.weekday(time.now_ns())          # "Monday"
```

## Units

```rego
units.parse_bytes("1Gi")    # 1073741824
units.parse_bytes("512Mi")  # 536870912
units.parse_bytes("100Ki")  # 102400
```

Useful for comparing Kubernetes resource limits.

## Net and HTTP

```rego
net.cidr_contains("10.0.0.0/8", "10.1.2.3")      # true
net.cidr_intersects("10.0.0.0/8", "10.1.0.0/16")  # true
```

`http.send` for external API calls (not available in all runtimes):
```rego
response := http.send({
    "method": "GET",
    "url": "https://api.example.com/data",
    "headers": {"Authorization": "Bearer token"},
    "raise_error": false,
})
```

## Crypto and tokens

```rego
crypto.sha256("data")               # hex-encoded hash
io.jwt.decode("eyJ...")             # [header, payload, signature]
io.jwt.verify_hs256("eyJ...", secret)
io.jwt.decode_verify("eyJ...", {
    "cert": public_key,
    "iss": "expected-issuer",
})
```
