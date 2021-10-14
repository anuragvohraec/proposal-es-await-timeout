# Proposition for an additonal syntax to Javascript:  await[timeoutInMs]
Usage for promise:
```js
let someExpectedValue = await[3000] getSomeValueFromAPromise(); 
```
Here a pomise will either resolve before 3000 millisecond, or will get a timeout error (Do not fear errors!, its an intended timeout error, only comes in picture when timeout is mentioned)

Usage for async for await loops
```js
for await[2500](let s of someAsynGeneratorOrStream){
//do something
}
```
Here again this loop will finish by at most 2500 milliseconds.

# Detailed problem description and solution

We all are aware with usage of **await** of a promise: it basically commands code to wait for a promise to resolve or reject..... but wait until upto when? Indefinitely actually!
As of now any asynchronous promise based code's eventual destiny is at mercy of the asynchronous source. 

The asyncronous source has full power to keep all resource on stack of an async process engaged on RAM, and developer seems to have no control on it, as async source can decide when should it resolve (or never resolve) a promise, there by engaging everything on RAM.

Consider this code:
```js
let someReallyBigItemOnRAM = getSomeBulkyValue();
let res = await someReallyTimeConsumingAsyncFunction(someReallyBigItemOnRAM);
```
Now in this `someReallyTimeConsumingAsyncFunction` can take really long time to return or say never return and keep `someReallyBigItemOnRAM` on RAM engaged on RAM forever!

To overcome this problem, a JS developer must have contol over await. A new code will look something like this:
```js
let someReallyBigItemOnRAM = getSomeBulkyValue();
try{
let res = await[1500] someReallyTimeConsumingAsyncFunction(someReallyBigItemOnRAM);
}catch(e){
  //try catch is used as await[timeInMs] can cause a timeoutError, which needs to be caught
  console.error(e);
}
```
Such an await will await for at most 1500 ms, else it will generate an timeout error.
**NOTE**: It is assured that if used without timeout the `await` will exactly behave as it has always has behaved, so no old code is ever going to fail due to this new enhancement. User will still be able to use `await` without timeout.

Now one proposition that comes to mind is usage of `Promise.race` to simulate what is intended here:
```js
let timeout = (time)=>new Promise(res=>setTimeout(res,time));
let someReallyBigItemOnRAM = getSomeBulkyValue();
let res = Promise.race([timeout(1500),someReallyTimeConsumingAsyncFunction(someReallyBigItemOnRAM)]);
```
But Promise.race has a flaw which do not suffice the requirement.
It  will though ignore the value returned by `someReallyTimeConsumingAsyncFunction` function, if its not finished before timeout, but it do not interrupts its execution. Indeed your code will never exit and neither the `someReallyBigItemOnRAM` will be released until the promise of `someReallyTimeConsumingAsyncFunction` is resolved. Your virtually have no control on `someReallyBigItemOnRAM` now. It is at mercy of async source when they want to release it!

# Async for await loops
Consider this code:
```js
for await(let s of anAsyncGeneratorOrStream){
//do some thing here
}
//once loop finish do shomething after wards
```
Again the `anAsyncGeneratorOrStream` has full power to keep this loop running for ever with developer having no control. As the source is asynchronous, it can send data at interva of its own will, and can take forever to complete if it wants.
However if we have an `await[timeInMs]` syntax available as well with regular await:
```js
try{
  for await[3000](let s of anAsyncGeneratorOrStream){
  //do some thing here
  }
}catch(e){
//catch time out error if above code throws it
}
//once loop finish do shomething after wards
```
We can be assured that we will get out of such a loop by at most 3000 milliseconds.
A much better control in hands of a developer.

Again there are codes to simulate this kind of timeout loops using `Promise.race`, but as like before `Promise.race` will ignore the value returned by LongRunning async code but will not stop it from holding RAM and values on stack until async promise it had is finished, even though we intended to ignore such timed out values.

# Why is it required/matter ?
1. Better control on developer end, rather than at mercy of asynchronouse function.
2. Can give much more better understanding of how long can a particular line can at most take and can help pin point bottleneck in the code.
3. Is very simple to implement, as the code simply generate Timeout error. `try/catch` and `async/await` are part of JS. An `await[timeInMs]` is possible source of a timeout Error, and hence compiler can pre warn user about potential timeout points in the code.

# What are the fears, and they indeed are not to worry
Argument: A code can't be made to break/interrupted in between, it can cause potential resource leaks. That is some resource which were supposed to clean up but were interrupted by timeout error, will be in leak stage.
Consider this problem (**Problem 1**):
```js
async function doLongRunningTask() {
  const connection = await getConnectionFromPool()
  const { error, resource } = await connection.fetchResource()
  connection.release()

  if (error) throw error
  return resource
} 
```
If such a code is interrupted before a call to `connection.release()` is made, then it will evetually cause leak.
```js
await[3000] doLongRunningTask();//its feared that this new syntax can cause leaks inside long running task, as if it takes too long it will raise an error and will not get time to call connection.release()
```

But it should be noted that developer has deliberatly written `await[timeInMs]` , and user knows that it will cause an error to be raised.
**When something is deliberate, all repercussions, are not unexpected, they are intended outcomes.**

User can create such **deliberate** problem, by writing a code as such to the same problem without using await[timeInMs]:
**(example 1)**
```js
//deliberate attempt to mess up some one's api code:
let t = getConnectionFromPool;
getConnectionFromPool = ()=>setTimeout(a=>throw "error",100); return t();

async function doLongRunningTask() {
  const connection = await getConnectionFromPool()
  const { error, resource } = await connection.fetchResource()
  connection.release()

  if (error) throw error
  return resource
} 
```
Both will have same effect and , are done deliberately, and hence user knows what is about to come.
An api that intend to do a **must have clean up** , would have rather written code as such.
```js
async function doLongRunningTask() {
let connection;  
try{
 //now any where any exception occurs, or an interrupt exception is thrown, or time out error is throw in middle, all clean up will still take place.
  }catch(e){
     if(connection) connection.release();
  }
} 
```
They have written the code as discussed in previous example (Problem 1), as may be thats what they wanted, as thats what the code does which they have written! (As it allows people to mess it up anyway even if await[timeOutInMs] is not there in place, as explianed in example 1).

This new syntax indeed gives a better control to developer to mandate him to wrap such a code with  try catch:
```js
try{
await[3000] doLongRunningTask();//a try catch as this line can possible throw timeout error or any other error within from function even without timeout
}catch(e){
//you actually get a chance to clean up manually if something goes wrong.
}
```

# Context
I was desining a consensus algorithm, where each particpant needs to send their response via websocket. As the response from each of the particpant can come anychrnously, the framework I use for websocketing provide this via asynchronous stream, which is then dealt by me using `for await ...of` loop.
```js
for await(let data of asyncStreamFromChannelOfConsensusResponse){
//collect thier response
}
//do something after response are recived.
```
Now this problem can't be left at the mercy of consensus participant, if one never sends any response, the code will run forever.
So a need for such an `await` with timeout aroused.

Its pretty simple, its very clear what it intends to do:  `await for x amount of timeat most` === `await[x] somePromise.
People have alwasy wanted cntrol on cancelling a promise and this is one such way.

I hope that other people find it usefull and a good feature to have in lovely Javascript!

# Comments are welcomed
Please raise issues or support with this new syntax.
Cheers!
