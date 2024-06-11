## The problem

In a workflow-style data processing system, if all the data sources are granted-to-complete, how do we ensure that the execution of a processing task can reach its complete state?

## Terms

| Name                   | Description                                                                                                                                                                                                                   |
| ---------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Data Stream            | An observable stream of async data emitted by something.                                                                                                                                                                       |
| SourceNode (Abbr. S)   | A node that creates granted-to-complete Data Stream for the next node in the Flow, which is usually the start point of a data processing task.                                                                                |
| `OperatorNode` (Abbr. O) | A node that observes Data Stream from the previous node in the Flow, and creates granted-to-complete Data Stream for the next node; which is usually used to transform data, fetch data from API using the incoming data, etc. |
| WatcherNode (Abbr. W)  | A node that observes Data Stream from the previous node, usually used to inspect the data processing state, e.g., logging, and reporting progress.                                                                         |
| Flow                   | A list consists of SourceNode, `OperatorNode`, and WatcherNode, it must start with SourceNode and end with WatcherNode (S, O....O, W) or completely consists of `OperatorNode`(O, O, .... O).                                    |
| Diagram                | A set of Flows                                                                                                                                                                                                                |
|                        |                                                                                                                                                                                                                               |

## Define the complement

### S
The S creates a granted-to-complete data stream, when the data stream completes, the S itself also reaches its complete state.

The data stream S created is consumed by the next node (e.g., O, W) in the Flow list.
```
type S<T> = DataStream<T>
```

### O

In the Flow list, every incoming data from the previous node, makes O create a new granted-to-complete data-stream for the next node. Therefore, O is stateless, it doesn't have a complete state, it propagates the complete state from the previous node to the next node.

```ts
type O<A, B> = DataStream<A> => DataStream<B>
```

As you can see, the O is actually a function that takes data stream in and out.
### W
In the Flow list, W consumes data from the previous node, when the previous node comes to the complete state, the W reaches to its complete state.

```ts
type W<T> = DataStream<T> => void
```

### Flow

#### Stateful flow (S, O, .... O, W)

If flow starts with an S and ends with a W, it is a stateful flow.

When the last W, comes to a complete state, the Flow reaches a complete state.

The complete state is propagated from the beginning S to the last W via many Os.

```ts
type StatefullFlow<A, B> = {
  source: S<A>;
  operators: O[];
  watcher: W<B>;
}
```
#### Stateless flow (O, O, .... O, O)

The stateless flow behaves like an O, it creates granted-to-complete data stream based on the incoming data.

```ts
type StatelessFlow = O[];
```
### Diagram 

When every flow in the Diagram completes, the diagram reaches a complete state, which means the current running data process task is completed.

To have a granted-to-complete Diagram, we need to make sure:
1. All S, O in the diagram are correctly implemented to create granted-to-complete data stream, which can be verified by doing code reviews and writing unit tests for these nodes' implementation
2. A method to prevent an *Infinite Flow*

### Flow vs Diagram
- `Diagram` can transform multiple composed `Flows`.
- A `Diagram` can be seen as a directed graph containing startPoints and endPoints, which can be transformed to a flow through the following code.

```js
const stateful = new Set<Flow>();
const stateless= new Set<Flow>();

for startPoint of Diagram.startPoints:
  for endPoint of Diagram.endPoints:
    for flow of findFlows(startPoint, endPoint): 
      if (isStatefull(flow)):
        statefull.add(flow)
      else
        stateless.add(flow)
```
## The Infinite Flow

For the `OperatorNode's`, we can have a compose function, that connects the 2 `OperatorNode`, and returns a newly composed `OperatorNode`.

```ts
function compose(op1: O<A, B>, op2: O<B,C>): O<A, C>;
```


This enables us to convert any *Stateless Flow* into an `OperatorNode`.

We can also have a delegate function for `OperatorNode`:

```ts
function delegate(operatorRef: ()=> O<A, B>): O<A, B>;
```

It creates a new O, which creates the data stream using the returned O of operatorRef when it receives incoming data from the previous node.

With the help of compose and delegate, it is possible to reuse stateless flow in the diagram:

![image](https://github.com/stone-lyl/share-blog/assets/20497176/772a33d3-76c7-4544-b73b-d5f4e22bbf5e)

Since the reuse is based on reference, it is possible to make reference circles in the diagram, which may lead to infinite processing. 

### Branch

To avoid this problem, we need to add the branching ability to `OperatorNode`.

```ts
function branch(selector: (data: T) => O<T,T>): O<T, T>
```

branch creates a new O, when data arrives at O, it sends the data to the O' returned by selector(data)  , then pipe the output of O' to the next node.

If there is no need to do branching for some data, the selector function can return an identity O:

```ts
function identity: O<T, T>
```

It just outputs what it received.

With the help of branch and identity, we can now add an exit branch for a circular flow, but the exit branch can not be verified until running the data processing task.

```js
const filterNode = branch(item => {
  if(item > 4) {
    return nodeA;
  }
  return identity;
})
```

To avoid infinite processing at runtime, we need to implement a *delegation depth detection* at runtime.

### Delegation Depth Detection

The delegation depth actually means "How many times does the data pass through a `DelegateNode`", it simply comes to an idea of attaching a `delegateCount` property to the data inside the data stream. When data is passed through a `DelegateNode` created by the delegate, the `delegateCount` increases.

When the `delegateCount` reaches a limit, let's say, 200k , the `DelegateNode` should drop the data and write a warning log.

## Answer to the beginning question

1. Make sure all S, O are correctly implemented to create granted-to-complete data stream
2. Use compose, and delegate to allow circular reference in the Diagram
3. Use a branch to add an exit branch to the circular flow
4. Use delegate to avoid infinite-traveling data in the flow at runtime
