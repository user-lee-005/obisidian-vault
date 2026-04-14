Aka Producer

Publisher is a data source who will always publish event

```
public interface Publisher<T> {
	public void subscribe(Subscriber<? super T> subscriber);
}
```
