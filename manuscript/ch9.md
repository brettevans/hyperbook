# Hyperapp: functional programming in your browser with 500 lines of code
# Chapter 9: Runtime optimisation

## Breaking from purity

"Effects/subscriptions as data" taken to the extreme means no side-effects in the userland code. 
```setTimeout``` becomes an effect, ````setInterval```` becomes a subscription. HTTP calls become effects, SSE become subscriptions. 

What about things like ```console.log``` or ```Math.random()```? 
We can wrap them inside effects, but sometimes it's more convenient to use them directly. 
Our next requirement is to add unique identifiers to the posts.

Put this ```guid``` function based on ```Math.random()``` into your code:
```javascript
const guid = () => {
    return 'xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx'.replace(/[xy]/g, function (c) {
        var r = Math.random() * 16 | 0, v = c == 'x' ? r : (r & 0x3 | 0x8);
        return v.toString(16);
    });
}
```

Modify ```AddPost``` action to generate ```id``` for all new posts:
```javascript
    const newPost = {
      id: guid(),
      username: "anonymous",
      body: state.currentPostText,
    };
```

## Optimising long lists of items 

Our application keeps adding new posts to the top of the list. 
When Hyperapp sees a new item in the new list and compares it with the first item in the old list they differ. 
Same with the second and third and all the other items. 

![Figure: Lists without keys require unnecessary Virtual DOM computations](images/nokeys.jpg)

The old items got shifted by one. We know it, but the algorithm for the Virtual DOM diffing doesn't. 
To maintain a stable list item identity between the renders add a **key** attribute.

![Figure: Lists with keys skip unnecessary Virtual DOM computations](images/keys.jpg)

With the extra hint from the **key** attribute Hyperapp avoids re-rendering the items that got shifted by one.

Modify ```listItem``` view function to include the ```key``` attribute:
```javascript
const listItem = (post) => html`
  <li key=${post.id}>
    ...
  </li>
`;
```
Usually the best candidate for the ```key``` value is a stable identifier (e.g. guid). 
Don't use post content because it may not be unique. Don't use array index as it's not stable over re-renders.

Since ```key``` attribute is not visible in the generated DOM you can also add ```data-key``` attribute for debugging purposes:
```javascript
const listItem = (post) => html`
  <li key=${post.id} data-key=${post.id}>
    ...
  </li>
`;
```
Hyperapp internally uses ```key``` and a developer uses ```data-key```.

## Finding runtime performance bottlenecks

Switch ```LoadLatestPosts``` to fetch 1000 items when the application starts:
```javascript
const LoadLatestPosts = Http({
  url: "https://hyperapp-api.herokuapp.com/api/post?limit=1000",
  action: SetPosts,
});
```

Start typing a new post. 
On every typed charater Hyperapp has to do a Virtual DOM diffing of the entire page it controls. 
With a fast machine the typing delay may not even be noticeable. Hyperapp is often fast enough without extra optimisation. 

Go to your DevTools and slow down your CPU to make the impact of large DOM tree updates noticeable.

![Figure: Slowing down CPU in DevTools](images/slowcpu.png)

Record CPU profile while typing the text. It should show significant time spent on JS execution.
I recommend to profile in the **Incognito mode** which usually has most browser extensions disabled.
Those extensions may significantly impact performance results. 

![Figure: Impact of slow CPU on runtime performance](images/slow-cpu-impact.png)

Zoom in on the slowest part, which is the widest yellow box in the flame chart:

![Figure: Zooming in on performance bottleneck](images/zoomin.png)

```render``` function seems to be our bottleneck. But the ```render``` function belongs to Hyperapp so keep looking for the code that you wrote. 
Just below the ```render``` function you should see a ```view``` function invoking ```listItem```  repeatedly. 
The source of our bottleneck is the ```listItem``` function invoked multiple times when we type a new post.
Excessive ```listItem``` calls result in multiple ```patch``` function calls. It's Hyperapp patching physical DOM tree to keep
up with your changes. 

## Optimizing large DOM trees 

You want to avoid the unnecessary computation of the post list items when typing a new post text. 

Extract ```postList``` view fragment:
```javascript
const postList = ({ posts }) => html`
  <ul>
    ${posts.map(listItem)}
  </ul>
`;
```
Use it in the ```view``` function:
```javascript
{postList({ posts: state.posts })}
```

Import ```Lazy``` function from Hyperapp:
```javascript
import { h, app, Lazy } from "./web_modules/hyperapp.js";
```
```Lazy``` wraps view fragments that need to be optimized.

Decorate ```postList``` with ```Lazy```:
```javascript
const lazyPostList = ({posts}) => Lazy({view: postList, posts});
```
```Lazy``` expects a ```view``` to optimize (```postList```) and other properties (```posts```) that will be passed to the ```postList```.
The optimization implemented by ```Lazy``` is  called **memoization**. 
```Lazy``` remembers the input and output of the previous ```postList``` invocation. 
If you call it again with the same input the ```postList``` doesn't compute anything and ```lazyPostList``` returns previously saved result.

Replace ```postList``` with ```lazyPostList```:
```javascript
${lazyPostList({posts: state.posts})}
```

Verify performance profile again.

![Figure: Optimized view profile with shorter JS blocking time](images/optimized.png)

Most key presses should generate a performance profile with much shorter JS blocking time.

```Lazy``` is the last part of Hyperapp API you need to learn.  There's nothing more. Congratulations!
After this section remove the ```limit``` query param from the API url.

In the remaining chapters I'll cover extra topics that are not part of Hyperapp core, but are still useful for day to day development.
You will learn about:
* testing
* routing
* integrating with 3rd party libraries
* rendering on the server