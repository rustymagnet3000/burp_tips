# Burp, JMeter, AB, HAProxy, cURL tips

<!-- TOC depthfrom:2 depthto:3 withlinks:true updateonsave:true orderedlist:false -->

- [Proxy traffic](#proxy-traffic)
    - [macOS env variable](#macos-env-variable)
    - [macOS Desktop apps](#macos-desktop-apps)
    - [Proxy OpenSSL](#proxy-openssl)
    - [Invisble proxying](#invisble-proxying)
    - [Add debug logging, as alternative to proxying](#add-debug-logging-as-alternative-to-proxying)
- [Burp](#burp)
    - [Search Burp files](#search-burp-files)
    - [Replay requests](#replay-requests)
    - [Replay requests turbo](#replay-requests-turbo)
    - [Enumeration](#enumeration)
    - [Inject XSS Payload](#inject-xss-payload)
- [JMeter](#jmeter)
    - [Set a replayed request](#set-a-replayed-request)
    - [Send Parallel requests](#send-parallel-requests)
- [Apache Bench](#apache-bench)
    - [load test a container](#load-test-a-container)
    - [Verbose flag to verify HTTP response code](#verbose-flag-to-verify-http-response-code)
- [haproxy](#haproxy)
    - [Install](#install)
    - [Run](#run)
    - [Validate config file](#validate-config-file)
    - [Example Proxy Pass all data](#example-proxy-pass-all-data)
    - [Example remove Cookies and add header](#example-remove-cookies-and-add-header)
    - [Replace user-agent](#replace-user-agent)
- [cURL](#curl)

<!-- /TOC -->

## Proxy traffic

### macOS env variable

on `macOS`, it is simpler to proxy command line apps - such as Rust, Python, C - using an environment variable:

```bash
export https_proxy=127.0.0.1:8081

# Test it:
curl https://ifconfig.io

unset https_proxy
```

### macOS Desktop apps

With Safari or Slack, you have to change the macOS Network Proxy settings.

### Proxy OpenSSL

No `invisible proxy` is required to read OpenSSL traffic if you use the `proxy` flag.

```bash
# original
curl https://httpbin.org/ip

# proxied
curl -x, --proxy 127.0.0.1:8080 https://httpbin.org/ip

# proxied
openssl s_client -connect httpbin.org:443 -proxy 127.0.0.1:8080
```

### Invisble proxying

For proxy unaware clients via Burp on macOS.

```bash
echo "[*]Invisible proxy script starting..";

get_forwarding_status () {
    forwarding_status="$(sysctl net.inet.ip.forwarding)"
    if [ "${forwarding_status: -1}" -eq 1 ]; then
        echo "Forwarding already on"
    else
        sudo sysctl -w net.inet.ip.forwarding=1
        echo "Turned on forwarding"
    fi
    unset forwarding_status
}

set_port_forwarding_rules () {
    sudo pfctl -s nat &> ~/fifo.txt

    if grep -q '80 -> 127.0.0.1 port 8080' ~/fifo.txt && grep -q '443 -> 127.0.0.1 port 8080' ~/fifo.txt; then
        echo "-> Port Forwarding rules already on"
    else
        echo "rdr pass inet proto tcp from any to any port { 80 443 } -> 127.0.0.1 port 8080" | sudo pfctl -ef -
        echo "Port Forwarding rules added"
    fi
    echo "[*]Removing temporary file";
    if rm ~/fifo.txt ; then echo "Removed fifo.txt" ; fi
}

while getopts ": aAcC" opt; do
case $opt in
        [aA])
            echo "[*]CHECK FOR KERNAL FORWARDING";
            get_forwarding_status;
            set_port_forwarding_rules;
            echo "[*]SCRIPT COMPLETE";
        exit 0;;

    [cC]) echo "[*]CLEAN_UP";
            sudo pfctl -F all -f /etc/pf.conf;
            sudo sysctl -w net.inet.ip.forwarding=0;
            echo "[*]SCRIPT COMPLETE";
        exit 0;;

    \?) echo "[!]Invalid option: -$OPTARG" >&2;exit 0;;
esac
done
echo "[!]Enter [-a] add [-c] clean Proxy Rules";
```

### Add debug logging, as alternative to proxying

Some AWS libraries can be debugged by setting an environment variable to print network requests. For example:

`RUST_LOG=debug my_rust_app`

Or:

`RUST_LOG=rusoto,hyper=debug`

## Burp

### Search Burp files

```bash
grep --include=\*.burp -rnw . -e "hotel"


# -r recursive
# -n line number
# -w match whole word
```

### Replay requests

#### Same requests many times

You can do this with Intruder ( not Repeater, as you might expect ).  

 - Send request to `Intruder`
 - In `Positions` tab, `Clear §`
 - In `Payloads` tab, select:
    - `Payload Type: Null Payment`
    -  Select number of requests to replay

### Replay requests (turbo)

`Turbo Intruder` is a `Burp Suite extension` for sending large numbers of HTTP requests when you require extreme speed.

### Enumeration

#### Find API

```json
POST /check-account
Host: foobar.com

{"email":"foo.bar@foobar.com"}
```

#### Response

```json
{"registered":false}
```

#### Burp Intruder - Username Generator

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

#### Burp Intruder - Brute Forcer

 - < same as above steps>
 - In `Payloads` tab, select:
  - `Payload Type: Brute Forcer`
    - Select the `Character Set`
    - Select the `min length` and `max length`
  

> You can slow the enumeration attempt to avoid `Rate Limits` by adding a custom `Resource Pool` inside of `Intruder`.  You can delay the time between requests.

### Inject XSS Payload

#### Request

```json
POST /v1/final-order HTTP/1.1
Host: foobar.com

{"address":"125 important place"}
```

#### Burp Extender

From `Extender` select `BApp Store`. Install `xssValidator`.

#### Burp Intruder set up

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

## Apache Bench

### load test a container

```bash

    -n: Number of requests
    -c: Number of concurrent requests
    -H: Add header
    —r: flag to not exit on socket receive errors
    -k: Use HTTP KeepAlive feature
    -p: File containing data to POST
    -T: Content-type header to use for POST/PUT data,


#GET with Header
ab -n 100 -c 10 -H "Accept-Encoding: gzip, deflate" -rk https://0.0.0.0:4000/
#POST
ab -n 100 -c 10 -p data.json -T application/json -rk https://0.0.0.0:4000/
```

### Verbose flag to verify HTTP response code

```bash
ab \
	-v 4 \
	-n 1 \
	-c 1 \
	-p foobar_body.json \
	-T application/json \
	-H "Authorization:Bearer xxxxxxx" \
	-rk ${FOOBAR_TEST_URL}
```

## haproxy

### Install

```bash
brew install haproxy
brew info haproxy
haproxy -v
```

### Run

```bash
brew services start haproxy
brew services stop haproxy

# verbose
sudo haproxy -f haproxy.cfg -V

# silent
sudo haproxy -f haproxy.cfg
```

### Validate config file

`haproxy -c -f haproxy.cfg`

### Example Proxy Pass all data

```js
// haproxy.cfg
// https://www.haproxy.com/blog/haproxy-configuration-basics-load-balance-your-servers/

defaults
  mode http
  timeout client 10s
  timeout connect 5s
  timeout server 10s 
  timeout http-request 10s

frontend myfrontend
  bind 127.0.0.1:8080
  default_backend myservers

backend myservers
  server server1 127.0.0.1:8000
```

### Example remove Cookies and add header

```js
// https://www.haproxy.com/documentation/hapee/latest/traffic-routing/rewrites/rewrite-requests/
defaults
  mode http
  timeout client 10s
  timeout connect 5s
  timeout server 10s
  timeout http-request 10s

frontend myfrontend
  bind 127.0.0.1:8080
  acl h_xff_exists req.hdr(X-Forwarded-For) -m found
  http-request add-header X-Forwarded-For %[src] unless h_xff_exists
  default_backend myservers

backend myservers
  acl at_least_one_cookie req.cook_cnt() gt 0
  http-request del-header Cookie if at_least_one_cookie
  server server1 127.0.0.1:8000

```

### Replace user-agent

```bash
# http://cbonte.github.io/haproxy-dconv/2.0/configuration.html#1.2.2

http-request replace-header User-Agent curl foo

# applied to:
User-Agent: curl/7.47.0

# outputs:
User-Agent: foo
```

## cURL

```bash
#generate a random cookie string
curl 127.0.0.1:8080 --cookie "CUSTOMER_COOKIE=$(openssl rand -hex 4)"

#get all DockerHub images from a company
curl -s "https://hub.docker.com/v2/repositories/someCompany/?page_size=100" | jq -r '.results|.[]|.name'
```
