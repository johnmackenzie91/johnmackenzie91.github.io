+++
title = 'Explicit or Exception? My thoughts on Error Handling'
date = 2024-12-03T10:14:05Z
+++

In computer programming, explicit error handling and exception error handling are two approaches to signaling an error or problem during the runtime of a computer program.

Here is a breakdown of the two;

## Explicit Error Handling
This approach requires that errors are returned as values from a method, usually as an object or code. The calling code then makes the decision on how to react based on the returned value or object.

### Advantages:

* Visibility and predictability: It makes error-handling logic explicit and predictable since the error is directly returned by the method.
* Encourages thoughtful handling: This approach encourages developers to think about error cases explicitly at every point in the program and respond appropriately.
* Works well in procedural contexts: In procedural code, where execution flows step by step, explicit error handling aligns naturally as each operation is clearly defined.

### Disadvantages:
* Repetitiveness: It can become repetitive, requiring constant checks for returned errors.


## Exception Error handling
This approach uses a specialised exception mechanism built into the language to signal and handle errors. Errors are "thrown" or "raised" and are expected to be caught by a try/catch block in the calling code.

### Advantages:
* Improved readability: Separating error-handling logic from the main program logic makes the code cleaner and easier to follow, especially in complex systems.
* Centralised handling: Exceptions can be handled at a higher level in the program, reducing repetitive error-handling code.

### Disadvantages:

* Implicit error signaling: It’s not always clear which methods can throw errors, making it easier for exceptions to go unhandled.
* Runtime surprises: If exceptions are not properly caught, they can propagate and crash the program unexpectedly.


## Final Thoughts

In a procedural context explicit error handling wins hands down.  With explicit error returns, I can immediately see which method returns an error, and I can decide there an then how I wish to response to said error. Unlike exception handling, where errors are often thrown far from where they are caught, explicit error handling keeps everything close to the source, making the flow of execution predictable and transparent. It eliminates the need for a distant catch blocks and avoids the complexity of error propagation, allowing for more maintainable and understandable code. In procedural code, error handling should be explicit, local, and as simple as possible to keep the code clean and easy to maintain.

In Object-Oriented Programming, I again prefer explicit error handling because it allows me to make clear, conscious decisions on how to handle errors at each level; whether to retry an operation, take an alternative flow, log or ignore the error completely. One could argue that handling errors at every level creates clutter, I would say that each level in a service chain often requires different strategies to address specific failure conditions or at the very least allows one to add additional context to error before handing it back the the calling code. The "clutter" created by these error checks should be kept to a minimum by using patterns to avoid code duplication.

For asynchronous programming I prefer throwing exceptions. My main experience with asynchronous programming comes from either a typescript component or running asynchronous go routines. Asynchronous code is decentralised by its very nature and because of this I find it advantageous to have the execution handling in a centralised place over sending an error on a channel and ensure that everything is listening as it should.