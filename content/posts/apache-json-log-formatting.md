---
title: "JSON Formatting for the Apache Log"
date: 2021-07-23T18:18:53Z
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
\"request_id\": \" %{UNIQUE_ID}e \" \
}"

LogFormat "{ \
\"time\":\"%{%Y-%m-%d}tT%{%T}t.%{msec_frac}tZ\", \
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

### Bugs
`request_id` in the error log needs some spaces around the variable. Or for some reason the quote disappear and you have invalid JSON
`time`: Should be an fix but need to format the time in the same way between the error log and access log

### Future
I'm currently looking to feed these logs into Honeycomb.io using AWS Firelens & FluentBit but that's a post for another day.
