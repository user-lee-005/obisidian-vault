## Mono
- Create Mono objects using Mono.just()
- Mono can handle only one element
- `Mono<T> mono = Mono.just(T);`
- Mono is a publisher
- According to the flow call the subscribe method first, so the method where the mono is declared we need to use `mono.subscribe()`
- Inside subscriber we put our function that needs the data
- `.log()` --> This is used for debugging the steps of the reactive programming `Mono.just().log()`

## Flux
- Create flux objects using `Flux.just("", "", "");`
- `.concatWithValues` --> This would add data to the Flux object created

## OnError call
- In Mono when there is exception, the data would not be returned
- In Flux when there is exception, the data before the exception would be returned and after the exception the data would not be returned

## Server
- Tomcat server is not used, Netty is used. This runs the event loop

## Controller Changes
- In normal controller, we just have the Annotation GetMapping, `@GetMapping("/")`
- For reactive programming we need to use mediaType along with the path so that the data would be streaming rather than going in bulk after everything is processed, `@GetMapping("/", produces = MediaType.TEXT_EVENT_STREAM_VALUE)`
- This is backed by the Event Loop given by [[Netty]] (default server for webflux given by spring boot) 