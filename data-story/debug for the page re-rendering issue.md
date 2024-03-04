## problem

- **reproduce**: Open the playground and add some nodes. After a while, the page will become increasingly slower, regardless of the operation you perform.
- **video**:
![[before-performance-issue.mp4]]

## How to debug a performance issue

1. I used the `Profiler` in `React` dev tools and found that the `Workbench` component is constantly re-rendering. The `Profiler` indicated that this is due to the third hook (`useStore`) in the component changing

![a](../assets/Pasted%20image%2020240304171949.png)

2. I suspected that some state in the `Workbench` was changing, so I added `useWhyDidYouUpdate` to investigate what was constantly updating. ([Refer to useWhyDidYouUpdate from ahooks](https://ahooks.js.org/hooks/use-why-did-you-update/))

```ts
useWhyDidYouUpdate('Workbench', { nodes, edges, onNodesChange, onEdgesChange, connect, onInit, openNodeModalId, setOpenNodeModalId, traverseNodes });
```
3. I discovered that the `nodes` were continuously updating when I was using the dev tools.
![[Pasted image 20240304172625.png]]
4. I compared the `nodes` content `json` in 'from' and 'to', they were the same, leading me to guess that *the reference might have changed*.

5. Then, I looked for operations in `useStore` that could change the `node`, namely `onNodesChange`. I logged it and found that changes in `dimensions` caused the `nodes` to change.
![[Pasted image 20240304172917.png]]
6. The `onNodesChange` method is only used in the `Workbench` component.

7. So, I debugged in `onNodesChange`, and through the call stack, I found that `updateNodeInternals` was consistently calling `RAF`, causing constant rendering.
![[Pasted image 20240304172936.png]]
8. I checked the [source code](https://github.com/xyflow/xyflow/blob/main/packages/react/src/hooks/useUpdateNodeInternals.ts#L28) of `xyflow` and found that `updateNodeInternals` would call the reference of `updateNodeDimensions`. 

9. Upon further investigation of `updateNodeDimensions`, I found that regardless of whether the `node` has actual changes, the `nodes` would eventually be updated.  

 **reason**: *The 9* is the reason why the page (`Workbench Component`) keeps re-rendering.
## how to solve

Check out the code in the following link:
- [nodeSettingsModal.tsx](https://github.com/ajthinking/data-story/pull/148/files#diff-7412d8e2510cfe1e26efe2ebd14ddce0921562bb598b34189e0af5a5516c0038)
- [Workbench](https://github.com/ajthinking/data-story/pull/148/files#diff-fe13771085a0ed345d22c33230dca46c186ad01a0d5fe5e468b407f7a4ed9c11)

PR:  https://github.com/ajthinking/data-story/pull/148/files#
