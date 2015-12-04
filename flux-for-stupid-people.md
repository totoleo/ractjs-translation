#写给初学者的flux入门

“放弃吧！”  
在我挣扎着学习Flux时，最希望听到某个人这样对我说。它一点都不简单直接，缺少文档，还有很多需要完善的地方。  

这篇文章就是献给那些和我一样不怎么聪明的家伙，作为学习ReactJS后的补充。  

*我该使用Flux吗？

如果你的应用需要处理动态的数据，那么你可能需要。  

如果只是一些静态的页面也没有状态，并且你不需要保存或者更新数据，那就离Flux远点，它不会给你更多的好处。  

*为什么需要Flux  


坦白的说，Flux有点复杂。那为什么要引入Flux增加复杂性呢？  


90的iOS应用的数据是用表格形式呈现的。iOS的工具箱里有很多好用的视图和数据模型，帮助开发者更轻松的开发。  


但是对于前端（HTML,Javascript,CSS）,我们并没有这些东西。相反我们面临一个很严重的问题：没有人知道如何构建一个前端应用。  
这行我干了很多年了，从来没看到过“最佳实践”。各种“库”倒是学了不少。jQuery？Angular？Backbone？现实的问题是，数据流仍然困扰着大多数人。  

Flux是个什么东东？


“Flux”用来描述绑定特定事件(events)和监听器(listeners)的单向数据流。没有特定的Flus库，但是你需要一个Flux Dispatcher和任何一个Javascript的事件库。

官方的文档属于“意识流”派的风格，不适合新手。如果你已经对Flux足够了解，它将帮助你补上某些确实的东西。

不要拿Flux和MVC做比较。   
不要拿Flux和MVC做比较。  
不要拿Flux和MVC做比较。  
重要的事情要说三遍。
那两个不相干的东西做比较，会使你更困惑。    

下面是我们开工！下面我会一步步的讲解、演示那些概念。  

1.视图(Views) Dispatch 动作(Actions)    

Dispatcher 本质上是一个基于事件的系统。它负责分发事件和注册事件的处理函数。只有一个全局的Dispatcher。推荐使用Facebook提供的Dispatcher库，它很方便使用:  

    var AppDispatcher = new Dispatcher();
    
假设你的应用有一个 new 按钮去给一个列表增加一个新的项目。   

    <button onClick={ this.createNewItem }>New Item</button>  
    
当你点击完后会发生什么？首先，视图会发出一个特殊的事件，其中包括事件的名称以及相关数据：    

    createNewItem: function( evt ) {

        AppDispatcher.dispatch({
          eventName: 'new-item',
            newItem: { name: 'Marco' } // example data
        });
    }
    
2.“Store”会响应一个被分发的的事件   

像Flux一样，这个也是Facebook自己造的一个词。对于我们的应用来讲，我们需要一个描述逻辑和数据的地方，这就是 “Store”。现在我们整一个 ListStore。

Store 是单例对象，这就意味着你不应该通过new来声明它。ListStore是一个全局的对象：    

    // Global object representing list data and logic
    var ListStore = {

        // Actual collection of model data
        items: [],

        // Accessor method we'll use later
        getAll: function() {
         return this.items;
        }

    };
    
这个Store需要相应事件，就像下面一样去注册一个需要响应的事件：   

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

这是Flux处理事件的典型方式：每个payload都包含事件的名称和数据；而后通过一个switch语句去决定执行那个动作。


 关键点: Store不是Model，Store包含一个Model

 关键点: Store是唯一知道如何更新组件数据的。这是Flux中最重要的一点。分发的事件对如何添加和删除项。

例如，如果应用的其它地方需要跟踪一些图片和相关的metadata，你就需要新建一个Store，比如叫ImageStore。一个Store代表了应用中一个独立的概念。
如果你的应用很大，它们可能对你来说显而易见。但是如果只是一个小应用，或许你只需要一个Store就够了。

只有Store才允许向Dispatcher注册回调！   
只有Store才允许向Dispatcher注册回调！   
只有Store才允许向Dispatcher注册回调！   
视图不应该去调用 AppDispatcher.register。Dispatcher应该只用来将事件从View分发到Store。视图可能会相应多个事件。


3.从Store触发Change事件 

好了，终于到重点了！数据已经完全改变，我们需要告诉所有人。

现在你的Store触发需要触发一个事件，但是我们并没有使用dispatcher。是不是很困惑？但Flux就是这么干的。现在我们需要赋予Store触发事件的能力。如果你使用MicroEvent，就可以像这样: 

    MicroEvent.mixin( ListStore );
    
接下来我们需要触发这个change事件：  

    case 'new-item':

        ListStore.items.push( payload.newItem );

        // Tell the world we changed!
        ListStore.trigger( 'change' );

        break;

 关键点：我们并没有将最新的数据添加到事件中。因为视图只关心是不是有东西发生改变。想知道为什么吗？继续跟踪数据吧^_^。

4.视图去响应(Respond) Change事件    

现在我们需要显示这个列表。视图此时会整体重新渲染(re-render)。对，你没看错，就是这样！   

首先，我们需要在ListStore中，设置监听component首次加载的事件，就是说component刚被创建时：   

    componentDidMount: function() {
        ListStore.bind( 'change', this.listChanged );
    },
    
简单起见，我们只调用 foreUpdate,此时会触发重绘。

    listChanged: function() {
        // Since the list changed, trigger a new render.
        this.forceUpdate();
    },
    
别忘了在component被删除时移除事件的监听器： 

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
