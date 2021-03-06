# ha_gateway
This is a tiny home automation REST gateway. It currently controls:

1. My LED strips using [ledenet_api](http://github.com/sidoh/ledenet_api).

Please see [this blog post](http://blog.christophermullins.net/2015/10/17/cheap-alternative-to-phillips-hue-led-strip/) for more details on the overall setup.

## Using

You can start or stop the server with the provided scripts in `./bin`. Configure the port and device binding in `./bin/run.sh`.

`bin/run.sh` will start the process in the foreground. `bin/start` will daemonize the process, redirecting output to `log/ha_gateway.log`. `bin/stop` kills a daemonized process.

## Security

This server uses [HMAC](https://en.wikipedia.org/wiki/Hash-based_message_authentication_code) signatures to verify that the caller is authorized. A shared secret is used to sign a random parameter and the timestamp. A valid request must include the following headers:

1. `X-Signature-Payload`: a random string. I used UUIDs.
2. `X-Signature-Timestamp`: a UNIX epoch timestamp.
3. `X-Signature`: the HMAC signature of `payload + timestamp`.

The value of `X-Signature` is checked against the computed signature from the other headers. If this signature is not present or does not match, the server returns a `403: Unauthorized` error.

To prevent reply attacks, the timestamp specified in `X-Signature-Timestamp` must be no older than 20 seconds. This requires that servers involved have up to date clocks.

The shared secret should be placed in `./config/hmac.key`. Be sure that there are no spaces, newlines, or EOFs in the file! Generate a random one like this:

```bash
head -c 4096 /dev/urandom | md5sum | cut -d' ' -f1 | tr -d '\r\n' > config/hmac.key
```

## Endpoint

This server starts on port 8000. Supported endpoints:

1. `/leds`, responds to `POST` commands.

## Supported parameters

### POST /leds

1. `r`, `g`, `b` (all must be present to have an effect). Sets RGB value for LEDs. Range for each parameter should be [0,255].
2. `status`. Sets on/off status. Supported values are "on" and "off".
3. `level`. Sets the level/luminosity. Converts current color to HSL, adjusts level, and re-converts to RGB. Range should be [0,100].

#### Example

The following example was executed with security features disabled (commented out):

```
$ curl -v -X POST -d'r=100' -d'g=0' -d'b=0' -d'status=on' http://localhost:8000/leds
* Hostname was NOT found in DNS cache
*   Trying ::1...
* connect to ::1 port 8000 failed: Connection refused
*   Trying 127.0.0.1...
* Connected to localhost (127.0.0.1) port 8000 (#0)
> POST /leds HTTP/1.1
> User-Agent: curl/7.35.0
> Host: localhost:8000
> Accept: */*
> Content-Length: 23
> Content-Type: application/x-www-form-urlencoded
>
* upload completely sent off: 23 out of 23 bytes
< HTTP/1.1 200 OK
< Content-Type: text/html;charset=utf-8
< X-XSS-Protection: 1; mode=block
< X-Content-Type-Options: nosniff
< X-Frame-Options: SAMEORIGIN
< Content-Length: 17
<
* Connection #0 to host localhost left intact
{"success": true}%
```
