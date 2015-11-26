#写给初学者的flux入门

“放弃吧！”
在我挣扎着学习Flux时，最希望听到某个人这样对我说。它一点都不简单直接，缺少文档，还有很多需要完善的地方。

这篇文章就是献给那些和我一样不怎么聪明的家伙，作为学习ReactJS后的补充。

*我该使用Flux吗？

如果你的应用需要处理动态的数据，那么你可能需要。
If your application deals with dynamic data then yes, you should probably use Flux.
如果只是一些静态的页面也没有状态，并且你不需要保存或者更新数据，那就离Flux远点，它不会给你更多的好处。
If your application is just static views that don't share state, and you never save nor update data, then no, Flux won't give you any benefit.
*为什么需要Flux
Why Flux?

坦白的说，Flux有点复杂。那为什么要引入Flux增加复杂性呢？
Humor me for a moment, because Flux is a moderately complicated idea. Why should you add complexity?

90的iOS应用的数据是用表格形式呈现的。iOS的工具箱里有很多好用的视图和数据模型，帮助开发者更轻松的开发。
90% of iOS applications are data in a table view. The iOS toolkit has well defined views and data models that make application development easy.

但是对于前端（HTML,Javascript,CSS）,我们并没有这些东西。相反我们面临一个很严重的问题：没有人知道如何构建一个前端应用。
这行我干了很多年了，从来没看到过“最佳实践”。各种“库”倒是学了不少。jQuery？Angular？Backbone？现实的问题是，数据流仍然困扰着大多数人。
On the Front End™ (HTML, Javascript, CSS), we don't even have that. Instead we have a big problem: No one knows how to structure a Front End™ application. I've worked in this industry for years, 
and "best practices" are never taught. Instead, "libraries" are taught. jQuery? Angular? Backbone? The real problem, data flow, still eludes us.

What is Flux?

Flux is a word made up to describe "one way" data flow with very specific events and listeners. There's no Flux library, but you'll need the Flux Dispatcher, and any Javascript event library.

The official documentation is written like someone's stream of conciousness, and is a bad place to start. Once you get Flux in your head, though, it can help fill in the gaps.

Don't try to compare Flux to Model View Controller (MVC) architecture. Drawing parallels will only be confusing.

Let's dive in! I will explain concepts in order and build on them one at a time.

1. Your Views "Dispatch" "Actions"

A "dispatcher" is essentially an event system. It broadcasts events and registers callbacks. There is only ever one, global dispatcher. You should use the Facebook Dispatcher Library. It's very easy to instantiate:

var AppDispatcher = new Dispatcher();  
Let's say your application has a "new" button that adds an item to a list.

<button onClick={ this.createNewItem }>New Item</button>  
What happens on click? Your view dispatches a very specific event, with the event name and new item data:

createNewItem: function( evt ) {

    AppDispatcher.dispatch({
        eventName: 'new-item',
        newItem: { name: 'Marco' } // example data
    });

}
2. Your "Store" Responds to Dispatched Events

Like Flux, "store" is just a word Facebook made up. For our application, we need a specific collection of logic and data for the list. This describes our store. We'll call it a ListStore.

A store is a singleton, meaning you probably shouldn't declare it with new. The ListStore is a global object:

// Global object representing list data and logic
var ListStore = {

    // Actual collection of model data
    items: [],

    // Accessor method we'll use later
    getAll: function() {
        return this.items;
    }

};
Your store then responds to the dispatched event:

var ListStore = …

AppDispatcher.register( function( payload ) {

    switch( payload.eventName ) {

        case 'new-item':

            // We get to mutate data!
            ListStore.items.push( payload.newItem );
            break;

    }

    return true; // Needed for Flux promise resolution

}); 
This is traditionally how Flux does dispatch callbacks. Each payload contains an event name and data. A switch statement decides specific actions.

 Key Concept: A store is not a model. A store contains models.

 Key concept: A store is the only thing in your application that knows how to update data. This is the most important part of Flux. The event we dispatched doesn't know how to add or remove items.

If, for example, a different part of your application needed to keep track of some images and their metadata, you'd make another store, and call it ImageStore. A store represents a single "domain" of your application. If your application is large, the domains will probably be obvious to you already. If your application is small, you probably only need one store.

Only your stores are allowed to register dispatcher callbacks! Your views should never call AppDispatcher.register. The dispatcher only exists to send messages from views to stores. Your views will respond to a different kind of event.

3. Your Store Emits a "Change" Event

We're almost there! Now that your data is definitely changed, we need to tell the world.

Your store emits an event, but not using the dispatcher. This is confusing, but it's the Flux way. Let's give our store the ability to trigger events. If you're using MicroEvent.js this is easy:

MicroEvent.mixin( ListStore );  
Then let's trigger our change event:

        case 'new-item':

            ListStore.items.push( payload.newItem );

            // Tell the world we changed!
            ListStore.trigger( 'change' );

            break;
 Key Concept: We don't pass the newest item when we trigger. Our views only care that something changed. Let's keep following the data to understand why.

4. Your View Responds to the "Change" Event

Now we need to display the list. Our view will completely re-render when the list changes. That's not a typo.

First, let's listen for the change event from our ListStore when the component "mounts," which is when the component is first created:

componentDidMount: function() {  
    ListStore.bind( 'change', this.listChanged );
},
For simplicity's sake, we'll just call forceUpdate, which triggers a re-render.

listChanged: function() {  
    // Since the list changed, trigger a new render.
    this.forceUpdate();
},
Don't forget to clean up your event listeners when your component "unmounts," which is when it goes back to hell:

componentWillUnmount: function() {  
    ListStore.unbind( 'change', this.listChanged );
},
Now what? Let's look at our render function, which I've purposely saved for last.

render: function() {

    // Remember, ListStore is global!
    // There's no need to pass it around
    var items = ListStore.getAll();

    // Build list items markup by looping
    // over the entire list
    var itemHtml = items.map( function( listItem ) {

        // "key" is important, should be a unique
        // identifier for each list item
        return <li key={ listItem.id }>
            { listItem.name }
          </li>;

    });

    return <div>
        <ul>
            { itemHtml }
        </ul>

        <button onClick={ this.createNewItem }>New Item</button>

    </div>;
}
We've come full circle. When you add a new item, the view dispatches an action, the store responds to that action, the store updates, the store triggers a change event, and the view responds to the change event by re-rendering.

But here's a problem: we're re-rendering the entire view every time the list changes! Isn't that horribly inefficient?!

Nope.

Sure, we'll call the render function again, and sure, all the code in the render function will be re-run. But React will only update the real DOM if your rendered output has changed. Your render function is actually generating a "virtual DOM", which React compares to the previous output of render. If the two virtual DOMs are different, React will update the real DOM with only the difference.

 Key Concept: When store data changes, your views shouldn't care if things were added, deleted, or modified. They should just re-render entirely. React's "virtual DOM" diff algorithm handles the heavy lifting of figuring out which real DOM nodes changed. This makes your life much simpler, and will lower your blood pressure.

One More Thing: What The Hell Is An "Action Creator"?

Remember, when we click our button, we dispatch a specific event:

AppDispatcher.dispatch({  
    eventName: 'new-item',
    newItem: { name: 'Samantha' }
});
Well, this can get pretty repetitious to type if many of your views need to trigger this event. Plus, all of your views need to know the specific object format. That's lame. Flux suggests an abstraction, called action creators, which just abstracts the above into a function.

ListActions = {

    add: function( item ) {
        AppDispatcher.dispatch({
            eventName: 'new-item',
            newItem: item
        });
    }

};
Now your view can just call ListActions.add({ name: '...' }); and not have to worry about dispatched object syntax.

Unanswered Questions

All Flux tells us how to do is manage data flow. It doesn't answer these questions:

How do you load and save data to the server?
How do you handle communication between components with no common parent?
What events library should I use? Does it matter?
Why hasn't Facebook released all of this as a library?
Should I use a model layer like Backbone as the model in my store?
The answer to all of these questions is: have fun!

PS: Don't Use forceUpdate

I used forceUpdate for simplicty's sake. The correct way for your component to read store data is to copy data into state and read from this.state in the render function. You can see how this works in the TodoMVC example.

When the component first loads, store data is copied into state. When the store updates, the data is re-copied in its entirety. This is better, because internally, forceUpdate is synchronous, while setState is more efficient.

That's It!

For additional resources, check out the Example Flux Application provided by Facebook. Hopefully after this article, the files in the js/ folder will be easier to understand.

The Flux Documentation contains some helpful nuggets, buried deep in layers of poorly accessible writing.

For an example of organizing components with Flux, see the follow up ReactJS Controller View Pattern.

If this post helped you understand Flux, consider following me on Twitter.
