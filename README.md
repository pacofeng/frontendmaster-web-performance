https://gist.github.com/stevekinney/fe401ffb8b2b7279e56dd165b272f0c3

## Rough outline:
* JavaScript performance: Write code that runs faster, later, or not at all.
* Rendering performance: It turns out most of our JavaScript happens in the browser, which has its own performance concerns.
* Load performance: Until the user actually gets the page, there isn’t much to optimize.

## JavaScript performance
* Overview
![Test Image 1](https://github.com/pacofeng/frontendmaster-web-performance/blob/master/AF5324F0-964B-420B-8B31-E45A9BB9741C.png)
* Most browsers use just-in-time (JIT) compilation: it happens moments before execution on client’s machine
* Parsing:
    * Two parsing phases
        * Eager (full parse): Scan through the top-level scope, parse all the code you see that's actually doing something.
        * Lazy (pre-parse): Do the minimum now, we will parse it later. Skip things like function declarations and classes for now, we will parse them when we need them.
    * Utility library to help: optimize-js
    * Avoid nested functions
```
function sumOfSquares(x, y) {
    // This will repeatedly be parsed.
    function square(n) {
        return n * n;
    }

    return square(x) + square(y);
}
```

* After it’s parsed, it’s turned into an abstract syntax tree. We have gone from a big long string of text to an actual data structure representing our code.
    * Three things the engine can help in this phase:
        * Speculative optimization
            * How does it work?
                * We use an interpreter because the optimizing complier is slow to get started.
                * It needs some information before it knows what work it can either optimize or skip out on all together.
                * The interpreter starts gathering feedback about what is sees as the function is used.
            * The optimizing compiler optimizes for what it has seen. If it sees something new, that’s problematic.
                * Monomorphic: This is all I know and all that I have seen. I can get incredibly fast at this one thing.
                * Polymorphic: I have seen a few shapes before. Let me just check to see which one and then I will go do the fast thing.
                * Megamorphic: I have sen things. A lot of things. I am not particularly specialized. Sorry.
            * Takeaways:
                * Turbofan is able to optimize your code in substantial ways if you pass it consistent values
        * Hidden classes for dynamic lookups
            * How does it work?
                * This object could be anything, so look at the rule book and figure this out.
                * Computers are good at looking stuff up repeatedly, but also good at remembering things.
            * Takeaways:
                * Initialize your properties at creation.
                * Initialize them in the same order.
                * Try not to modify them after the fact.
                * Maybe just use TypeScript or Flow so you don’t have to worry about these things?
        * Function inlining
```
for (let i = 0; i < 5; i++) {
    console.log('hi');          => console.log('hi');console.log('hi');console.log('hi');console.log('hi');console.log('hi');
}
```

## Rendering performance
* Overview:
![Test Image 1](https://github.com/pacofeng/frontendmaster-web-performance/blob/master/901B2011-92E7-4E68-9711-ADD86E95ABE1.png)
* Render tree:
    * It has a one-to-one mapping with the visible objects on the page
        * No hidden object (display: none)
        * Include pseudo element (:after, :before)
    * There might be multiple rule apply to single element
        * Figuring out which rules apply to which elements — What styles apply to an element.
        * Figuring out what is the end result of an element with multiple rules — CSS specificity.
    * Takeaways
        * Use simple selectors whenever possible, consider BEM.
        * Reduce the effected elements.
        * Reduce the amount of unused CSS that you are shipping.
        * Reduce the number of styles that effect a given element.
* Layout (Reflow): Look at the elements and figure out where they go on the page.
    * Whenever the geometry of an element changes, the browser has to reflow the page
    * A reflow is a blocking operation, everything else stops.
    * Examples: 
        * Resize window
        * Change font/content
        * Add or remove a stylesheet/class/element
    * How to avoid?
        * Change classes at the lowest levels of DOM tree.
        * Avoid repeatedly modifying inline styles.
        * Trade smoothness for speed if animate in JavaScript
        * Avoid table layout
        * Batch DOM manipulation
        * Debounce window resize events
    * Layout thrashing (Forced synchronous layout): 
        * It occurs when JavaScript violently writes, then reads, from the DOM, multiple times causing document reflows.
            1. Read: The browser wants to get you the most up to date answer, so it goes and does a style and layout check.
            2. Then write: The browser knew it was going to have to change stuff.
            3. Then read again: You ask for some info about the geometry of an element, so browser stops your Javascript and reflows the page in order to get you an answer.
        * Solution: Separate reading from writing 
        * Utility library to help: fastdom
        * Takeaways
            * Don’t mix reading layout properties and writing them.
            * If you can change the visual appearance of an element by adding a CSS class. Do that and you will avoid accidental trashing.
* Paint: After knowing what things look like and where they go, draw some pixels to the screen.
    * Whenever you change something other than opacity or a CSS transform, you are going to trigger a paint.
    * When we do a paint, the browser tells every element on the page to draw a picture of itself
    * Triggering a layout will always trigger a paint, but change something like colors it doesn’t trigger reflow, just repaint.
    * Two Threads:
        * The renderer thread: Main thread, all JavaScript, parsing html and CSS, style calculation 
            * CPU intensive
        * The compositor thread: Draws bitmaps to the screen via the GPU
            * GPU intensive
            * When painting, we create bitmaps for the elements, put them onto layers, and prepare shaders for animation if necessary.
            * After painting, the bitmaps are shared with a thread on the GPU to do the actual compositing.
    * Takeaways: Let the compositor thread handle the painting, if you want to be fast, then offload whatever you can to the less-busy thread.
* Composite Layers: You might end up painting on multiple layers, but you will eventually need to combine them.
    * Layers are an optimization that the browser does for you under the hood.
    * What kind of stuff gets its own layer?
        * The root object of the page.
        * Objects that have specific CSS positions.
        * Objects with CSS transforms.
        * Objects that have overflow.
    * You can give the browser hints using `will-change` property
        * Using layer is a trade off
            1. Managing layers takes a certain amount of work on the browser’s behalf.
            2. Each layer needs to be kept in the shared memory between main thread and composite thread.
        * If user is interacting with constantly, add it to the CSS, otherwise do it with JavaScript.
        * Clean up after yourself, remove `will-change` when it’s not going to change anymore.

## Load performance
* Bandwidth vs Latency
    * Bandwidth: how much stuff you can fit through the tube per second
    * Latency: how long it takes to get to the other end of the tube
* TCP: 
    * focuses on reliability
        * Packets are delivered in the correct order.
        * Packets are delivered without errors.
        * Client acknowledges each packet.
        * Unreliable connections are handled well.
        * Not overload the network.
    * Initial window size is 14kb, if you get files under 14kb means you can get everything in the first window.
* Cache-Control
    * only affect these HTTP methods
        1. GET
        2. OPTIONS
        3. HEAD 
    * Cache-control header types
        * no-store: The browser gets a new version every time.
        * no-cache: You can store a copy, but can not use it without checking with server. 
        * max-age: Tell the browser not to bother if whatever asset it has less than a certain number of seconds old.
        * s-maxage: 
            * We want CSS and JavaScripts to be cached by browser.
            * We would like the CDN to cache the HTML — s-maxage is for CDNs only.
        * immutable
* Lazy-loading and pre-loading
* HTTP/2
    * New features:
        * Fully multiplexed - send multiple requests in parallel.
        * Allows servers to proactively push responses to clients.
    * Some best fracties become not-so-good
        * Concatenating all JS and CSS into single file still useful?
        * What about inlining images as data urls in CSS?

## Build
* Utility library: purifycss
* Babel
    * @babel/preset-env: allows you to use the latest JavaScript without worrying syntax transforms and setup target environment
    * @babel/plugin-transform-react-remove-prop-types: remove PropTypes
    * babel-plugin-transform-react-class-to-function
    * @babel/plugin-transform-react-inline-elements: transform to react elements 
    * @babel/plugin-transform-react-constant-elements: hoisting React elements to the highest possible scope, preventing multiple unnecessary reinstantiations.

## Others are not covered:
* Server-side rendering
* Image performance
* Loading web fonts
* Progressive web applications





#################################################################################################

```
const { performance, PerformanceObserver } = require('perf_hooks');

// SETUP 🏁
let iterations = 1e7;
const a = 1;
const b = 2;

const add = (x, y) => x + y;

// 🔚 SETUP
performance.mark('start');

// EXERCISE 💪
// %NeverOptimizeFunction(add);
while (iterations--) {
    add(a, b);
}

// add('a', 'b')

// iterations = 1e7;
//     while (iterations--) {
//     add(a, b);
// }

// 🔚 EXERCISE
performance.mark('end');

const obs = new PerformanceObserver((list, observer) => {
    console.log(list.getEntries()[0]);
    performance.clearMarks();
    observer.disconnect();
});
obs.observe({ entryTypes: ['measure'] });
performance.measure('My Special Benchmark', 'start', 'end');
```

* node benchmark.js    
```
PerformanceEntry {
    name: 'My Special Benchmark',
    entryType: 'measure',
    startTime: 32.887708,
    duration: 8.209448
}
```

* node --trace-opt benchmark.js | grep add
    
```
// mark add function for optimized recompilation
[marking 0x0546cffbdd09 <JSFunction add (sfi = 0x5462aed7ce1)> for optimized recompilation, reason: small function]
// compile the add function using TurboFan
[compiling method 0x0546cffbdd09 <JSFunction add (sfi = 0x5462aed7ce1)> using TurboFan]
// optimizing the add function
[optimizing 0x0546cffbdd09 <JSFunction add (sfi = 0x5462aed7ce1)> - took 0.569, 0.670, 0.009 ms]
// complete the optimization
[completed optimizing 0x0546cffbdd09 <JSFunction add (sfi = 0x5462aed7ce1)>]
```

* if we uncomment the string concatenation code after the while loop, then: run node --trace-opt --trace-deopt benchmark.js | grep add
  * passing different type of argument into function cause slow down
  * the optimizing compiler optimizes for what it has seen. If it sees something new, that is problematic.
        
```
[marking 0x0b02c3b5c2b1 <JSFunction add (sfi = 0xb02aec97ce1)> for optimized recompilation, reason: small function]
[compiling method 0x0b02c3b5c2b1 <JSFunction add (sfi = 0xb02aec97ce1)> using TurboFan]
[optimizing 0x0b02c3b5c2b1 <JSFunction add (sfi = 0xb02aec97ce1)> - took 0.607, 0.725, 0.008 ms]
[completed optimizing 0x0b02c3b5c2b1 <JSFunction add (sfi = 0xb02aec97ce1)>]
12: 0x0b02c3b5c2b1 ; rcx 0x0b02c3b5c2b1 <JSFunction add (sfi = 0xb02aec97ce1)>
0x7ffeefbfec40: [top + 64] <- 0x0b02c3b5c2b1 <JSFunction add (sfi = 0xb02aec97ce1)> ; stack parameter (input #12)
// we are getting de-optimizing
[deoptimizing (DEOPT eager): begin 0x0b02c3b5c2b1 <JSFunction add (sfi = 0xb02aec97ce1)> (opt #0) @0, FP to SP delta: 24, caller sp: 0x7ffeefbfec08]
reading input frame add => bytecode_offset=0, args=3, height=0, retval=0(#0); inputs:
0: 0x0b02c3b5c2b1 ; [fp - 16] 0x0b02c3b5c2b1 <JSFunction add (sfi = 0xb02aec97ce1)>
translating interpreted frame add => bytecode_offset=0, variable_frame_size=8, frame_size=80
0x7ffeefbfebd0: [top + 24] <- 0x0b02c3b5c2b1 <JSFunction add (sfi = 0xb02aec97ce1)> ; function (input #0)
// we are getting de-optimizing
[deoptimizing (eager): end 0x0b02c3b5c2b1 <JSFunction add (sfi = 0xb02aec97ce1)> @0 => node=0, pc=0x0001008e0d60, caller sp=0x7ffeefbfec08, took 0.168 ms]
```
* if we uncomment the %NeverOptimizeFunction(add), then run node --allow-natives-syntax benchmark.js 
```
PerformanceEntry {
    name: 'My Special Benchmark',
    entryType: 'measure',
    startTime: 49.227924,
    duration: 149.088589
}
```

