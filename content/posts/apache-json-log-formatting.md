---
title: "JSON Formatting for the Apache Log"
date: 2021-07-22T18:18:53Z
draft: false
---

I'm a big fan of structure logging, having a machine injest logs as JSON speeds up the queryability of the data. Not having to pre-parse the raw log, then convert into JSON is a big win too. So when I had a new application, (an older PHP/CGI website), which is all in Apache I figured I'd bring it into the modern times. Logs are going into AWS Cloudwatch Logs from AWS Fargate, Cloudwatch Logs supports parsing JSON without having to pass in any formats.

This is what I came up with:

It contains the usual fields you'd expect in a standard Apache log. But extras such as: 
* `request_id`: requires `mod_unique_id` enabled. This is present in both Access log and Error log, so you can link the two events. Tracing anyone?
* `request_time`: By far the most useful fields in finding that something is wrong is finding that it's slow.
* `x_forwarded_for`: Since this site is behind an ALB the client IP shows as the LB IP rather than the users.

```
ErrorLogFormat "{ \
\"time\": \"%{%usec_frac}t\", \
\"function\": \"[%-m:%l]\", \
\"process\": \"[pid%P]\", \
\"message\": \"%M\", \
\"request_id\": \"%-{UNIQUE_ID}e\" \
}"

LogFormat "{ \
\"time\": \"%{%Y-%m-%d}t %{%T}t.%{usec_frac}t\", \
\"process\": \"%D\", \
\"filename\": \"%f\", \
\"remoteIP\": \"%a\", \
\"host\": \"%V\", \
\"request\": \"%U\", \
\"query\": \"%q\", \
\"method\": \"%m\", \
\"status\": \"%>s\", \
\"userAgent\": \"%{User-agent}i\", \
\"referer\": \"%{Referer}i\", \
\"request_time\": \"%D\", \
\"request_id\": \"%{UNIQUE_ID}e\", \
\"bytes_sent\": \"%B\", \
\"x_forwarded_for\": \"%{X-Forwarded-For}i\" \
}" json_combined
```

I needed to enable 2 mods for this to work: `mod_remoteip` and `mod_unique_id`.

### Example

A snippet of what sort of thing to expect from this config:
```
{ "time": "2021-07-23 10:44:54.445610", "function": "[mpm_prefork:notice]", "process": "[pid1]", "message": "AH00163: Apache/2.4.38 (Debian) configured -- resuming normal operations", "request_id": "-" }
{ "time": "2021-07-23 10:44:54.445773", "function": "[core:notice]", "process": "[pid1]", "message": "AH00094: Command line: 'apache2 -D FOREGROUND'", "request_id": "-" }
{ "time": "2021-07-23 10:44:59.314098", "function": "[cgi:error]", "process": "[pid18]", "message": "End of script output before headers: broken.cgi", "request_id": "YPqdq8DVJOLdwQumnetHPgAAAAA" }
{ "time": "2021-07-23 10:44:59.310200", "process": "4233", "filename": "/var/www/html/cgi-bin/broken.cgi", "remoteIP": "172.27.0.1", "host": "localhost", "request": "/cgi-bin/broken.cgi", "query": "?foo=bar", "method": "GET", "status": "500", "userAgent": "curl/7.64.1", "referer": "-", "request_time": "4233", "request_id": "YPqdq8DVJOLdwQumnetHPgAAAAA", "bytes_sent": "534", "x_forwarded_for": "-" }
```

### Future
I'm currently looking to feed these logs into Honeycomb.io using AWS Firelens & FluentBit but that's a post for another day.
