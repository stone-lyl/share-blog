## useIntersectionObserver
```ts
export function useIntersectionObserver(callback: () => Promise<void>) {
  // Use useLatest to ensure the callback is always the latest
  const callbackRef = useLatest(callback);

  const [loaderChanged$, updateElement] = useCreation(() => {
  // updateElement might be called before subscription, so use BehaviorSubject to replay the most recent call to updateElement. This ensures that subscribers do not miss the mounting event of the sentinel element
    const subject = new BehaviorSubject<HTMLElement | null>(null);

    // updateElement is a react callback ref, called when the dom is created. Through it, one can subscribe to the event that the sentinel element has been created.
    function updateElement(newElement: HTMLElement | null) {
      if (newElement) {
        subject.next(newElement);
      }
    }

    return [subject, updateElement];
  }, []);

  // This stream is responsible for listening to the appearance of DOM elements and triggering the callback function
  const loadMore$ = useCreation(() => {
    return loaderChanged$.pipe(
      // If it is null, this means the sentinel element has not been rendered to the page yet. No need to start listening
      filter(it => !!it),
      // Avoid the same sentinel element being listened to repeatedly
      distinctUntilChanged(),
      // Created event stream switches to the become visible event stream of the sentinel element
      switchMap((el) => observeIntersection(el!)),
    // Share the IntersectionObserver instance between all subscribers
      share(),
      // Ensure that the emitted requests are returned in order.
      concatMap(() => callbackRef.current())
    );
  }, [loaderChanged$]);

  useEffect(() => {
    const sub = loadMore$.subscribe();
    return () => sub.unsubscribe();
  }, [loadMore$]);

  return updateElement;
}
```

**background**:
```tsx
// sentinel element
<div  
  ref={loaderRef}  
  className="loading-spinner h-0.5"  
>  
</div>
```

**before** 
- To prevent ``loadTableData`` from being called multiple times due to re-binding with ``observer.observe(loaderRef.current);`` every time the ``state`` updates, we'll address this issue. 
- everything was executed synchronously on the frontend, we didn't consider the sequence of asynchronous requests

![before](../assets/Pasted%20image%20240411231800.png)

**after**  implementing server-side pagination and the requests are asynchronous, needing to tackle duplicate requests and manage the order of requests. I'll also address the issue of repeatedly binding the sentinel event listener.

![after](../assets/Pasted%20image%20240411231811.png)

- implement a strategy for deduplicating data.
-  We'll resolve the issue of incorrect request sequencing, such as the scenario ``requestA -> requestB -> responseB -> responseA``. 

## about useLatest

```ts
export const useDataStoryEvent = (handler: EventHandler) => {  
  // Use useLatest to ensure the callback is always the latest
  // making sure the dependencies of `loadTableData` (like loading, offset, etc) are current.
  const handlerRef = useLatest(handler);  
  
  useEffect(() => {  
    const subscription = eventManager.on(handlerRef.current);  
    return () => subscription.unsubscribe();  
  }, [handler, handlerRef]);  
}
```
