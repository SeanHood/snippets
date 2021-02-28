---
title: "Opentelemetry OTLP Structure"
date: 2021-02-28T12:28:37Z
draft: false
---

I've been finddling around with OpenTelemetery for a while, I wrote the [SeanHood/laravel-opentelemetry](https://github.com/SeanHood/laravel-opentelemetry) package which used the Zipkin Exporter to get data out. However, recently Honeycomb.io added support for ingesting OTLP over GRPC nativly.

This support isn't quite there yet [opentelemetry-php#230](https://github.com/open-telemetry/opentelemetry-php/pull/230) but I wanted to see how far off it was. Information/docs on this layer of OpenTelemetry is incredibly sparse, you're mostly down to reading source code.

Anyway, in all of this I wasn trying to understand the structure of how a Span relates to an array of ResourceSpans so I came up with this psudocode:

```
ExportTraceServiceRequest(
    resource_spans: ResourceSpans(
        resource: Resource (
            attributes
        )
        instrumentation_library_spans: InstrumentationLibrarySpans (
            instrumentation_library: InstrumentationLibrary(
                name: 'my-instrumentation'
                version: '0.0.1'
            )
            spans: Span(
                span_id: 'xxx'
                trace_id: 'xxx'
                ...
            )
        )
    )
)
```

It might not be correct, let me know if I'm off but it helped me visualise what the protocol was expecting.

I figured this out by reading the opentelemetry-ruby source code [ref](https://github.com/open-telemetry/opentelemetry-ruby/blob/main/exporter/otlp/lib/opentelemetry/exporter/otlp/exporter.rb#L232-L257). They aren't there yet with GRPC but have things in place for OTLP/HTTP.
