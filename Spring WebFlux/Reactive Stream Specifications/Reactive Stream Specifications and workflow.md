It has only 4 interfaces:
1. [[Publisher]]
2. [[Subscriber]]
3. [[Subscription]]
4. [[Processor]]

Workflow:
1. Subscriber will invoke subscribe method of the publisher
2. Publisher will send the subscription event to the subscriber, confirming the subscription is successful
3. Subscriber will call the request method from the subscription interface to get the data from the publisher (n number of request can be sent)
4. Publisher will send the data stream to the subscriber by invoking the onNext method for n times
5. Once all records are published, publisher would invoke onComplete method of the subscriber and if there is any error onError would be invoked

## Reactive Programming Library
- Reactor
- RxJava
- Jdk9 Reactive Stream

