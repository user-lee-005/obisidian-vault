Core Features:
- Async and non blocking
- Functional style code
- Data flow as Event driven stream
- Backpressure on streams

Functional Style Code:
```
public Mono<T> someOperation() { 
	return repo.findById().map().then(Mono.just(T))
}
```
`Mono -> Single Data, Flux -> Collection of data`

Data flow as event driven stream
- Unlike the traditional rest service, there is always events getting posted from the publisher
- All the subscribers get the data when an event published by the publisher
- Majorly used for streaming the data
- Connection is always open

Backpressure:
1. Returning a large volume of data may not be handled for traditional rest api or throw oom
2. Backpressure --> In reactive programming, if huge data comes in, the publisher is informed that huge data is coming in first let me process the data that i have, after this next set comes in
3. We could add limitation for data to the db driver

Advantages:
1. Optimal CPU Usage
2. No downtime
3. Flow of execution can be concurrent
4. Streaming of data