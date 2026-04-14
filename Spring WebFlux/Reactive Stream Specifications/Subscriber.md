Aka Consumer

Will consume the events from the publisher
```
pulbic interface Subscriber<T> {
	public void onSubscribe(Subscription subscription);
	public void onNext(T t);
	public void onError(Throwable throwable);
	public void onComplete();
}
```
