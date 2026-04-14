Subscription represents the unique relationship between the subscriber and the publisher

```
public interface Subscription {
	public void request(long n);
	public void cancel();
}
```
