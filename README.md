# Burp and JMeter flexing

## Proxy traffic

### macOS env variable

on `macOS`, it is simpler to proxy command line apps - such as Rust, Python, C - using an environment variable:

```bash
export https_proxy=127.0.0.1:8081
export http_proxy=127.0.0.1:8081

# Test it:
curl https://ifconfig.io
```

### macOS Desktop apps

With Safari or Slack, you have to change the macOS Network Proxy settings.

### Proxy OpenSSL

No `invisible proxy` is required to read OpenSSL traffic if you use the `proxy` flag.

`curl -x, --proxy 127.0.0.1:8080 https://httpbin.org/ip`

### Add debug logging, as alternative to proxying

Some AWS libraries can be debugged by setting an environment variable to print network requests. For example:

`RUST_LOG=debug my_rust_app`

## Search saved Burp files

```bash
grep --include=\*.burp -rnw . -e "hotel"


# -r recursive
# -n line number
# -w match whole word
```

## Replay the same requests many times

You can do this with Intruder ( not Repeater, as you might expect ).  

 - Send request to `Intruder`
 - In `Positions` tab, `Clear §`
 - In `Payloads` tab, select:
    - `Payload Type: Null Payment`
    -  Select number of requests to replay

## Enumerate a single user's account email

### Find API

```json
POST /v1/check-email-address HTTP/1.1
Host: foobar.com

{"email":"foo.bar@foobar.com"}
```

### Response

```json
{"registered":false}
```

### Burp Intruder set up

 - Send request to `Intruder`
 - In `Positions` tab, select `Clear §`
 - Then select `Add §` after highlighting `"foo.bar@foobar.com"`
 - In `Payloads` tab, select:
    - `Payload Type: Username Generator`
    - `Payload Options [Username Generator]` add base target email `foo_bar@foobar.com`
    - `Payload Encoding` de-select the `URL encode` box
 - In `Options` tab, select:
    - de-select _"make unmodified baseline request"_
    - In `Attack Results` specify whether to save requests and responses
    - In `grep match` add the line `"registered":true` [ to ensure it is simple to view a successful attack ]

### Optional

You can slow the enumeration by adding a custom `Resource Pool` inside of `Intruder`.  You can delay the time between requests.

## Inject XSS Payload

### Request

```json
POST /v1/final-order HTTP/1.1
Host: foobar.com

{"address":"125 important place"}
```

### Burp Extender

From `Extender` select `BApp Store`. Install `xssValidator`.

### Burp Intruder set up

 - Send request to `Intruder`
 - In `Positions` tab, select `Clear §`
 - Then select `Add §` after highlighting `"125 important place"`
 - In `Payloads` tab, select:
    - `Payload Type: Extension-generated`
    - `Payload Options [Extension-generated]` select `XSS Validator Payloads`
 - In `Options` tab, select:
    - de-select _"make unmodified baseline request"_
    - `Grep – Match section`, and enter the string expected.

## JMeter

### Set a replayed request

`Copy as cURL` from within Firefox Web Developer.

`/Tools/Import from cURL`
Test 1: 5000 requests

Set the `Thread Group`:
   Number of Threads (users): ${__P(threads,10)}
   Ramp-up period (seconds): ${__P(rampup,30)}
   Loop Count: Infinite

Right click on `Thread Group` and select `Add Think Time to Children`.

Select `HTTP Request` and set the `Use KeepAlive`.

Then adjust the `Think Time` as required.

Right click on `Thread Group` and select `Validate`.

### Send Parallel requests

If you want to exhaust a service, parallel requests use a new HTTP client for each request ( which is different from Concurrent requests which uses a single HTTP client).

Import the cURL request (`/Tools/Import from cURL` )

Right click on the imported `HTTP Request`:

- `Add/Listener/View Results in Table`
- `Add/Time/Synchronizing Timer`

On `Synchronizing Timer`, select `Number of Simulated Users to Group by: 10`

Then go to `"View Results by Table"`.  Select Play.

Notice 10 requests sent at once.
