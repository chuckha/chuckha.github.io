---
title: "Decoupling Reconciler Code"
date: 2023-02-13T09:15:27-08:00
draft: true
---

In the context of Kubernetes, and very specifically, [controller-runtime][1], reconciler code is responsible for operating on objects in order to bring reality into alignment with the incoming request. In the case of a reconciler that manages a Kubernetes cluster, a request might be "scale my cluster to 10 nodes" and the reconciler would know how to do this. But the request might also be "10 minutes has passed, if something in reality has changed, we need to bring that into alignment with our desired state".

Phrased another way, a user can submit a desired state and the reconciler makes that desired state a reality (or errors).

However, I've seen lots of reconcilers with way too much responsibility. They become difficult to test due to lack of explicit control flow. 

In my obsession with writing maintainable code (feel free to define this however you want, that's a separate post entirely), I have been thinking of ways to write controller-runtime controllers in a more maintainable way.

My current thinking here is to completely forget the reconciler at first. Build the code that interacts with your system in vertical slices. Wrap the whole thing up for ease of use in a `struct`. Now that you have defined all of the desired ways your system can change, you can focus on writing the reconciler. It has the ability to call this struct and all that remains is to implement the reconciler that makes decisions. It looks at the application running, sees that it does not match the desired specification and calls the correct methods on the thing initially written.


[1]: https://github.com/kubernetes-sigs/controller-runtime