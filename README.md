# Tips to get Burp flexing

## Enumerate a single user's account email

### Find unauthenticated API

### Request

```json
POST /v1/check-email HTTP/1.1
Host: foobar.com

{"email_address":"foo.bar@foobar.com"}
```

### Response

```json
{"registered":false}
```

### Burp Intruder

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