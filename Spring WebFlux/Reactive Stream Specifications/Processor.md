Processor represents a processing stage -- which is both subscriber and publisher and must obey the contracts of the both

```
public interface Processor<T, R> extends Subscriber<T>, Publisher<R> {}
```
