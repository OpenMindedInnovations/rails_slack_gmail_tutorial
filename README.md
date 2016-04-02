### Rails + Gmail Extension = Awesome

I was playing with this really cool app called [Streak](https://www.streak.com/) that brings a CRM (Customer Relationship Manager) into your gmail. It impressed me how much it improved my gmail, so I decided to experiment with something like this myself. I found the Streak team made [InboxSDK](https://www.inboxsdk.com/), which allows you to easily make gmail extensions on your own. I also wanted gmail interactions like adding a todo to save to my web apps so I integrated it with a Rails API to test this out.

In this blog, I'll be detailing how I built a minimalistic gmail "Todo App" extension using the following tools outlined below:

1. [Rails](http://rubyonrails.org/)
2. [InboxSDK](https://www.inboxsdk.com/)
3. [Kefir](https://rpominov.github.io/kefir/)
4. [Extensionizr](http://extensionizr.com)
5. [jQuery](https://jquery.com/)

Our app will allow users to add email threads to their todo lists and mark those emails as "complete". The gmail extension will hit the rails api every time a todo is updated or created.

### Creating Our Rails App
Here we'll just be needing a simple todo model and controller that creates a todo item and updates the checked state of the todo. We'll have actions for creating todos, updating todos, and listing todos.

```sh
$ rails new TodoApp
$ cd TodoApp
$ bundle install
$ rails g resource Todo item:string description:string checked:boolean

```

```ruby
# config/routes.rb

Rails.application.routes.draw do
  resources :todos
end
```

```ruby
# app/controllers/todos_controller.rb

class TodosController < ApplicationController
  respond_to :json

  before_filter :find_todo, only: [:update]

  def index
      @todos = Todo.all
      respond_with(@todos)
  end

  def create
    @todo = Todo.create(todo_params)
    # respond with the todo object if the todo saves successfully
    if @todo.save
      respond_with(@todo)
    else
      respond_with(nil, @message = "Error while creating Todo")
    end
  end

  def update
    # respond with the todo object if the todo updates successfully
    if @todo.update(todo_params)
      respond_with(@todo)
    else
      respond_with(nil, @message = "Todo Update Failed")
    end
  end

  private
  def todo_params
    params.require(:todo).permit(
      :item,
      :checked,
      :description
    )
  end

  def find_todo
    @todo = Todo.find(params[:id])
  end

end

```

### Enable CORS

Our gmail extension needs to send api calls to our rails app from a different origin, so we need to enable CORS.

```ruby
# Gemfile

gem "rack-cors", :require => "rack/cors"
```

```ruby
# config/application.rb

module TodoApp
  class Application < Rails::Application
    config.active_record.raise_in_transactional_callbacks = true

    config.middleware.insert_before 0, "Rack::Cors" do
      allow do
        origins '*','chrome-extension://ibfdlibfkoemoeapebafdgnhpngjahho'
        resource '*', :headers => :any, :methods => [:get, :post, :options, :delete, :put, :patch], credentials: true
      end
    end
  end
end

```

### Enable HTTPS
All gmail extensions require connections to be https, so I hosted the app on heroku to quickly enable this. I'm not going to cover how to launch an app to heroku, but you can follow this [guide](https://devcenter.heroku.com/articles/getting-started-with-rails4). 

You could also set up SSL on your local machine, but it was much easier to just host on heroku.


### Setting up your gmail extension

There is a tool called [Extensionizr](http://extensionizr.com/!#{"modules":["hidden-mode","with-bg","with-persistent-bg","no-options","no-override"],"boolean_perms":[],"match_ptrns":[]}) that sets up an initial chrome extension for you to develop on. Go to that link and get the boilerplate that we'll be modifying.

We'll be making chrome extension using inboxSDK that automatically connects a todo label to each email and allows us to add email threads as a todo to rails

###### To get more info on Gmail extension and to get a gmail extension boilerplate visit the following links

###### We will be building a page_action chrome extension because we want to be able to inject custom styles and javascript to pages or a specific page

###### A quick approach to setting a chrome extension project is <a href="http://extensionizr.com/!#{"modules":["page-mode","with-bg","with-persistent-bg","no-options","no-override","inject-css","inject-js"],"boolean_perms":[],"match_ptrns":[]}"> Extensionizr</a>

###### Just select

--> 000.png <-----

###### Since we only want our extension to be active when the user visits his or gmail account we will be editing our

src/bg/background.js

###### by adding

```
chrome.runtime.onInstalled.addListener(function() {
    // Replace all rules ...
    chrome.declarativeContent.onPageChanged.removeRules(undefined, function() {
        // With a new rule ...
        chrome.declarativeContent.onPageChanged.addRules([
            {
                // That fires when a page's URL contains a 'mail.google.com' ...
                conditions: [
                    new chrome.declarativeContent.PageStateMatcher({
                        pageUrl: { urlContains: 'mail.google.com' },
                    })
                ],
                // And shows the extension's page action.
                actions: [ new chrome.declarativeContent.ShowPageAction() ]
            }
        ]);
    });
});


```

###### #Since will be using InboxSDK a library created by Streak which gives us the ability to add UI (like labels, side navigation, compose button details) to the gmail interface, we will also be using kefir a stream api that allows us to listen to changes and emit actions based on these changes.... We will also be using kefir as a wrapper around some of our Ui components to allow us to emit changes to them based on changes in Todo Item state. I'll personally advise to read more on Streak, InboxSDK and Streams (Kefir). Since we can't go into details in any of them at the moment.

So download the inboxsdk.js && kefir.min.js file and put it in

src/inject
folder, so we have our project dir has

-----> 001.png <-----

At this point we have all the libraries we want to inject and the first thing we will want to do is start working in gmail but unfortunately gmail doesn't allow us this free access. We have to explicitly state in our manifest.json file the files we want to inject into the page... we can easily do that by adding

```

  "content_scripts": [

    {
      "matches": [
        "https://mail.google.com/*"
      ],
      "css": [
        "src/inject/inject.css"
      ]
    },
    {
      "matches": [
        "https://mail.google.com/*"
      ],
      "js": [
        "js/jquery/jquery.min.js",
        "js/inboxsdk/inboxsdk.js",
        "js/kefir.min.js",
        "src/inject/inject.js"
      ]
    }
  ]


// so our new manifest.json file is

{
  "name": "Todo Gmail App",
  "version": "0.0.1",
  "manifest_version": 2,
  "description": "This extension was created with the awesome extensionizr.com",
  "homepage_url": "http://extensionizr.com",
  "icons": {
    "16": "icons/icon48.png",
    "48": "icons/icon48.png",
    "128": "icons/icon128.png"
  },
  "default_locale": "en",
  "background": {
    "scripts": [
      "src/bg/background.js"
    ],
    "persistent": true
  },
  "options_page": "src/options/index.html",
  "page_action": {
    "default_icon": "icons/icon19.png",
    "default_title": "page action demo",
    "default_popup": "src/page_action/page_action.html"
  },
  "chrome_url_overrides": {
    "newtab": "src/override/override.html"
  },
  "permissions": [
    "bookmarks",
    "clipboardRead",
    "clipboardWrite",
    "contextMenus",
    "cookies",
    "history",
    "notifications",
    "tabs",
    "geolocation",
    "declarativeContent"

  ],
  "omnibox": {
    "keyword": "extensionizr"
  },
  "content_scripts": [

    {
      "matches": [
        "https://mail.google.com/*"
      ],
      "css": [
        "src/inject/inject.css"
      ]
    },
    {
      "matches": [
        "https://mail.google.com/*"
      ],
      "js": [
        "js/jquery/jquery.min.js",
        "js/inboxsdk/inboxsdk.js",
        "js/kefir.min.js",
        "src/inject/inject.js"
      ]
    }
  ]
}


```

to the manifest.json. This injected files are called content scripts and can be either js or css files of any size.


SO LETS MOVE STRAIGHT INTO GMAIL AND BUILDING OUR TODO App

The first thing we will want to do is edit our inject.js and load the InboxSDK... One cool thing about this library is that it lets us just download a very small file and the main InboxSDK file that does all the work is loaded asynchronously which means we don't really need to worry about re-downloading and repacking whenever they make changes to the library

src/inject/inject.js

```
chrome.extension.sendMessage({}, function(response) {

    InboxSDK.load('1', 'Hello World!').then(function(sdk){

        // all the codes pertaining to the inboxsdk should be found here

    });

});


```


First thing we will be creating a Todo Sidebar dropdown component, this component will be the parent to all our todo items created. We will be using the InboxSDK NavItem #put link here to do this, so we will add

```
var todoItem = sdk.NavMenu.addNavItem({
          name: "Todos",
          iconUrl: "https://i.imgur.com/52FWtfw.png"
      });

```


to src/inject/inject.js


Now what we want to do is add a button to all the email threads in gmail, when the user clicks this button the particular email thread ID is gotten and the email description which we then use to create a todo item.

So to do that we are going to be using the inboxSdk threadRowView; but we will have to register a threadRowViewHandlder, using the <code>sdk.List</code> which expects a callback where it passes us a threadRowView for us to manipulate the threadRowView..

src/inject/inject.js

```
sdk.Lists.registerThreadRowViewHandler(function(threadRowView){

  // perform all list manipulation here

});


```

Using the <code>threadRowView</code> passed to us lets add button to all the email threads, when this button is clicked the we send the email it is attached to our Rails app. This will be the default state of icon on email threads that have not been added as todo. We are creating the button has a stream because we will want to listen and update its state.


```
var threadBtnEmmiter;

var threadBtnStream = Kefir.stream(function(inEmitter){
    threadBtnEmmiter = inEmitter;
    return function(){}; //we need to return a function that gets called when the stream ends
});

threadRowView.addButton(threadBtnStream);





```
------> 003.png <------

Before we can send the email thread to the rails app, we will want to know if this thread is already a todo or not. So to do that we will have an array of all the todos on the client side and also a function to check if the email thread clicked is in the array or not.

```
chrome.extension.sendMessage({}, function(response) {

    InboxSDK.load('1', 'Hello World!').then(function(sdk){

        var array_of_todos = []; //object type { thread_id, item, checked }

        var todoItem = sdk.NavMenu.addNavItem({
            name: "Todos",
            iconUrl: "https://i.imgur.com/52FWtfw.png"
        });

        function checkIfIsTodo(array,thread_id){
           if(array.length == 0){
             return false;
           } else {
             for(var i = 0; i < array.length; i++){
               if(array[i].description == thread_id){
                 return true;
               }
             }
           }
         }

        sdk.Lists.registerThreadRowViewHandler(function(threadRowView){

          var threadBtnEmmiter;

          var threadBtnStream = Kefir.stream(function(inEmitter){
              threadBtnEmmiter = inEmitter;
              return function(){}; //we need to return a function that gets called when the stream ends
          });

          threadRowView.addButton(threadBtnStream);

          // emit default state

          threadBtnEmmiter.emit({
              iconUrl:"http://pontifolio.com/img/grey-image.jpg",
              onClick:function(event){

                  var _threadID = event.threadRowView._threadRowViewDriver._cachedThreadID;
                  var isTodo = checkIfIsTodo(array_of_todos,threadRowView.getThreadID());

                  if(!isTodo){
                      if(threadRowView.getThreadID() == _threadID){
                          $.ajax({
                              url:"https://afternoon-ocean-92308.herokuapp.com/todos/",
                              type:"POST",
                              data:{
                                  todo : { item:threadRowView.getSubject(), checked:false, description:_threadID}
                              },
                              success:function(response){

                                  sdk.Widgets.showModalView({
                                      el:"<h3>New todo Item Created</h3>",
                                      title:"Add Todo"
                                  });
                                  array_of_todos.push(response)
                              }
                          });

                      }

                  } else {

                      // delete from todo list when we uncheck by
                      // send ajax request to server to update todo
                      for(var i = 0; i < array_of_todos.length; i++){
                          if(array_of_todos[i].description == _threadID){
                              var cache_item = array_of_todos[i];
                              cache_item.checked = !array_of_todos[i].checked;
                              array_of_todos.splice(i,1);
                              array_of_todos.push(cache_item)
                              $.ajax({
                                  url:"https://afternoon-ocean-92308.herokuapp.com/todos/"+cache_item.id,
                                  type:"DELETE",
                                  data:{
                                      todo : { item:cache_item.item, checked:true, description:cache_item.description}
                                  },
                                  success:function(response){

                                      sdk.Widgets.showModalView({
                                          el:"<h3>New todo Item Updated</h3>",
                                          title:"Edit Todo"
                                      });

                                  }
                              });

                              break;
                          }
                      }
                  }

              }
          })


        })





  });

});


```


If the email thread is not a todo we send the thread details to the rails app else we update the todo checked state in the array and send an update request to rails...


```
chrome.extension.sendMessage({}, function(response) {

    InboxSDK.load('1', 'Hello World!').then(function(sdk){

        var array_of_todos = []; //object type { thread_id, item, checked }

        var todoItem = sdk.NavMenu.addNavItem({
            name: "Todos",
            iconUrl: "https://i.imgur.com/52FWtfw.png"
        });

        function checkIfIsTodo(array,thread_id){
            if(array.length == 0){
              return false;
            } else {
              for(var i = 0; i < array.length; i++){
                if(array[i].description == thread_id){
                  return true;
                }
              }
            }
          }

        sdk.Lists.registerThreadRowViewHandler(function(threadRowView){

          var threadBtnEmmiter;

          var threadBtnStream = Kefir.stream(function(inEmitter){
              threadBtnEmmiter = inEmitter;
              return function(){}; //we need to return a function that gets called when the stream ends
          });

          threadRowView.addButton(threadBtnStream);

          threadBtnEmmiter.emit({
              iconUrl:"http://pontifolio.com/img/grey-image.jpg",
              onClick:function(event){

                  var _threadID = event.threadRowView._threadRowViewDriver._cachedThreadID;
                  var isTodo = checkIfIsTodo(array_of_todos,threadRowView.getThreadID());

                  if(!isTodo){
                      if(threadRowView.getThreadID() == _threadID){
                          $.ajax({
                              url:"https://afternoon-ocean-92308.herokuapp.com/todos/",
                              type:"POST",
                              data:{
                                  todo : { item:threadRowView.getSubject(), checked:false, description:_threadID}
                              },
                              success:function(response){

                                  sdk.Widgets.showModalView({
                                      el:"<h3>New todo Item Created</h3>",
                                      title:"Add Todo"
                                  });
                                  array_of_todos.push(response)
                              }
                          });

                      }

                  } else {

                      // delete from todo list when we uncheck by
                      // send ajax request to server to update todo
                      for(var i = 0; i < array_of_todos.length; i++){
                          if(array_of_todos[i].description == _threadID){
                              var cache_item = array_of_todos[i];
                              cache_item.checked = !array_of_todos[i].checked;
                              array_of_todos.splice(i,1);
                              array_of_todos.push(cache_item)
                              $.ajax({
                                  url:"https://afternoon-ocean-92308.herokuapp.com/todos/"+cache_item.id,
                                  type:"DELETE",
                                  data:{
                                      todo : { item:cache_item.item, checked:true, description:cache_item.description}
                                  },
                                  success:function(response){

                                      sdk.Widgets.showModalView({
                                          el:"<h3>New todo Item Updated</h3>",
                                          title:"Edit Todo"
                                      });

                                  }
                              });

                              break;
                          }
                      }
                  }

              }
          })



        })





  });

});


```
------> 004.png <------

But heck nothing has changed, the user doesn't know which email thread is a todo item or not.... We'll its time we add labels to each email thread that is a todo item... To do this we will be using Kefir a stream library which allows emit changes to the threadRowView based on events(document, window, promise, array). But in our use case we will be listening to changes in the state of the array using a new Array.observe function... This function allows us to listen to changes in the state (size, content) of an array and perform an action on this change.

```

var emitter; //variable name to hoist the emitter to
var stream = Kefir.stream(function(inEmitter){
    emitter = inEmitter;
    return function(){}; //we need to return a function that gets called when the stream ends
});
threadRowView.addLabel(stream);

// pass the array to listen to
Array.observe(array_of_todos, function(changes){
    console.log(changes);
    for(var i = 0; i < array_of_todos.length; i++){
        var _threadID = threadRowView._threadRowViewDriver._cachedThreadID;
        if(_threadID == array_of_todos[i].description){
          // if the todo item is checked then remove any label
          // and add a todo is Completed label
            if(array_of_todos[i].checked == true){
                emitter.emit(null);
                emitter.emit({
                    title:"Todo Completed",
                    foregroundColor:"#fff",
                    backgroundColor:"#91c661"
                })

                threadBtnEmmiter.emit(null);
                threadBtnEmmiter.emit({
                    iconUrl:"http://pontifolio.com/img/green-image.jpg",
                    onClick:function(event){

                        var _threadID = event.threadRowView._threadRowViewDriver._cachedThreadID;
                        var isTodo = checkIfIsTodo(array_of_todos,threadRowView.getThreadID());

                        if(!isTodo){
                            if(threadRowView.getThreadID() == _threadID){
                                $.ajax({
                                    url:"https://afternoon-ocean-92308.herokuapp.com/todos/",
                                    type:"POST",
                                    data:{
                                        todo : { item:threadRowView.getSubject(), checked:false, description:_threadID}
                                    },
                                    success:function(response){

                                        sdk.Widgets.showModalView({
                                            el:"<h3>New todo Item Created</h3>",
                                            title:"Add Todo"
                                        });
                                        array_of_todos.push(response)
                                    }
                                });

                            }

                        } else {

                            // delete from todo list when we uncheck by
                            // send ajax request to server to update todo
                            for(var i = 0; i < array_of_todos.length; i++){
                                if(array_of_todos[i].description == _threadID){
                                    var cache_item = array_of_todos[i];
                                    cache_item.checked = !array_of_todos[i].checked;
                                    array_of_todos.splice(i,1);
                                    array_of_todos.push(cache_item)
                                    $.ajax({
                                        url:"https://afternoon-ocean-92308.herokuapp.com/todos/"+cache_item.id,
                                        type:"DELETE",
                                        data:{
                                            todo : { item:cache_item.item, checked:true, description:cache_item.description}
                                        },
                                        success:function(response){

                                            sdk.Widgets.showModalView({
                                                el:"<h3>New todo Item Updated</h3>",
                                                title:"Edit Todo"
                                            });

                                        }
                                    });

                                    break;
                                }
                            }
                        }

                    }
                })



            } else {
              // else add a todo item label

              threadBtnEmmiter.emit(null);
              threadBtnEmmiter.emit({
                  iconUrl:"https://encrypted-tbn3.gstatic.com/images?q=tbn:ANd9GcR6t41ErQxx0y1rApv207bM3LznQVdvOILrYy-XTUVg3JpxGvRn",
                  onClick:function(event){

                      var _threadID = event.threadRowView._threadRowViewDriver._cachedThreadID;
                      var isTodo = checkIfIsTodo(array_of_todos,threadRowView.getThreadID());

                      if(!isTodo){
                          if(threadRowView.getThreadID() == _threadID){
                              $.ajax({
                                  url:"https://afternoon-ocean-92308.herokuapp.com/todos/",
                                  type:"POST",
                                  data:{
                                      todo : { item:threadRowView.getSubject(), checked:false, description:_threadID}
                                  },
                                  success:function(response){

                                      sdk.Widgets.showModalView({
                                          el:"<h3>New todo Item Created</h3>",
                                          title:"Add Todo"
                                      });
                                      array_of_todos.push(response)
                                  }
                              });

                          }

                      } else {

                          // delete from todo list when we uncheck by
                          // send ajax request to server to update todo
                          for(var i = 0; i < array_of_todos.length; i++){
                              if(array_of_todos[i].description == _threadID){
                                  var cache_item = array_of_todos[i];
                                  cache_item.checked = !array_of_todos[i].checked;
                                  array_of_todos.splice(i,1);
                                  array_of_todos.push(cache_item)
                                  $.ajax({
                                      url:"https://afternoon-ocean-92308.herokuapp.com/todos/"+cache_item.id,
                                      type:"DELETE",
                                      data:{
                                          todo : { item:cache_item.item, checked:true, description:cache_item.description}
                                      },
                                      success:function(response){

                                          sdk.Widgets.showModalView({
                                              el:"<h3>New todo Item Updated</h3>",
                                              title:"Edit Todo"
                                          });

                                      }
                                  });

                                  break;
                              }
                          }
                      }

                  }
              })



                emitter.emit({
                    title:"Todo Item",
                    foregroundColor:"#fff",
                    backgroundColor:"#bdbdbd"
                })
            }
        }
    }
})


```
------> 005.png <--------

```
chrome.extension.sendMessage({}, function(response) {

    InboxSDK.load('1', 'Hello World!').then(function(sdk){

        var array_of_todos = []; //object type { thread_id, item, checked }

        var todoItem = sdk.NavMenu.addNavItem({
            name: "Todos",
            iconUrl: "https://i.imgur.com/52FWtfw.png"
        });

        function checkIfIsTodo(array,thread_id){
           if(array.length == 0){
             return false;
           } else {
             for(var i = 0; i < array.length; i++){
               if(array[i].description == thread_id){
                 return true;
               }
             }
           }
         }

        sdk.Lists.registerThreadRowViewHandler(function(threadRowView){


          threadBtnEmmiter.emit({
              iconUrl:"http://pontifolio.com/img/grey-image.jpg",
              onClick:function(event){

                  var _threadID = event.threadRowView._threadRowViewDriver._cachedThreadID;
                  var isTodo = checkIfIsTodo(array_of_todos,threadRowView.getThreadID());

                  if(!isTodo){
                      if(threadRowView.getThreadID() == _threadID){
                          $.ajax({
                              url:"https://afternoon-ocean-92308.herokuapp.com/todos/",
                              type:"POST",
                              data:{
                                  todo : { item:threadRowView.getSubject(), checked:false, description:_threadID}
                              },
                              success:function(response){

                                  sdk.Widgets.showModalView({
                                      el:"<h3>New todo Item Created</h3>",
                                      title:"Add Todo"
                                  });
                                  array_of_todos.push(response)
                              }
                          });

                      }

                  } else {

                      // delete from todo list when we uncheck by
                      // send ajax request to server to update todo
                      for(var i = 0; i < array_of_todos.length; i++){
                          if(array_of_todos[i].description == _threadID){
                              var cache_item = array_of_todos[i];
                              cache_item.checked = !array_of_todos[i].checked;
                              array_of_todos.splice(i,1);
                              array_of_todos.push(cache_item)
                              $.ajax({
                                  url:"https://afternoon-ocean-92308.herokuapp.com/todos/"+cache_item.id,
                                  type:"DELETE",
                                  data:{
                                      todo : { item:cache_item.item, checked:true, description:cache_item.description}
                                  },
                                  success:function(response){

                                      sdk.Widgets.showModalView({
                                          el:"<h3>New todo Item Updated</h3>",
                                          title:"Edit Todo"
                                      });

                                  }
                              });

                              break;
                          }
                      }
                  }

              }
          })


            var emitter; //variable name to hoist the emitter to
            var stream = Kefir.stream(function(inEmitter){
                emitter = inEmitter;
                return function(){}; //we need to return a function that gets called when the stream ends
            });
            threadRowView.addLabel(stream);

            // pass the array to listen to
            Array.observe(array_of_todos, function(changes){
                console.log(changes);
                for(var i = 0; i < array_of_todos.length; i++){
                    var _threadID = threadRowView._threadRowViewDriver._cachedThreadID;
                    if(_threadID == array_of_todos[i].description){
                      // if the todo item is checked then remove any label
                      // and add a todo is Completed label
                        if(array_of_todos[i].checked == true){
                            emitter.emit(null);
                            emitter.emit({
                                title:"Todo Completed",
                                foregroundColor:"#fff",
                                backgroundColor:"#91c661"
                            })

                            threadBtnEmmiter.emit(null);
                            threadBtnEmmiter.emit({
                                iconUrl:"http://pontifolio.com/img/green-image.jpg",
                                onClick:function(event){

                                    var _threadID = event.threadRowView._threadRowViewDriver._cachedThreadID;
                                    var isTodo = checkIfIsTodo(array_of_todos,threadRowView.getThreadID());

                                    if(!isTodo){
                                        if(threadRowView.getThreadID() == _threadID){
                                            $.ajax({
                                                url:"https://afternoon-ocean-92308.herokuapp.com/todos/",
                                                type:"POST",
                                                data:{
                                                    todo : { item:threadRowView.getSubject(), checked:false, description:_threadID}
                                                },
                                                success:function(response){

                                                    sdk.Widgets.showModalView({
                                                        el:"<h3>New todo Item Created</h3>",
                                                        title:"Add Todo"
                                                    });
                                                    array_of_todos.push(response)
                                                }
                                            });

                                        }

                                    } else {

                                        // delete from todo list when we uncheck by
                                        // send ajax request to server to update todo
                                        for(var i = 0; i < array_of_todos.length; i++){
                                            if(array_of_todos[i].description == _threadID){
                                                var cache_item = array_of_todos[i];
                                                cache_item.checked = !array_of_todos[i].checked;
                                                array_of_todos.splice(i,1);
                                                array_of_todos.push(cache_item)
                                                $.ajax({
                                                    url:"https://afternoon-ocean-92308.herokuapp.com/todos/"+cache_item.id,
                                                    type:"DELETE",
                                                    data:{
                                                        todo : { item:cache_item.item, checked:true, description:cache_item.description}
                                                    },
                                                    success:function(response){

                                                        sdk.Widgets.showModalView({
                                                            el:"<h3>New todo Item Updated</h3>",
                                                            title:"Edit Todo"
                                                        });

                                                    }
                                                });

                                                break;
                                            }
                                        }
                                    }

                                }
                            })



                        } else {
                          // else add a todo item label

                          threadBtnEmmiter.emit(null);
                          threadBtnEmmiter.emit({
                              iconUrl:"https://encrypted-tbn3.gstatic.com/images?q=tbn:ANd9GcR6t41ErQxx0y1rApv207bM3LznQVdvOILrYy-XTUVg3JpxGvRn",
                              onClick:function(event){

                                  var _threadID = event.threadRowView._threadRowViewDriver._cachedThreadID;
                                  var isTodo = checkIfIsTodo(array_of_todos,threadRowView.getThreadID());

                                  if(!isTodo){
                                      if(threadRowView.getThreadID() == _threadID){
                                          $.ajax({
                                              url:"https://afternoon-ocean-92308.herokuapp.com/todos/",
                                              type:"POST",
                                              data:{
                                                  todo : { item:threadRowView.getSubject(), checked:false, description:_threadID}
                                              },
                                              success:function(response){

                                                  sdk.Widgets.showModalView({
                                                      el:"<h3>New todo Item Created</h3>",
                                                      title:"Add Todo"
                                                  });
                                                  array_of_todos.push(response)
                                              }
                                          });

                                      }

                                  } else {

                                      // delete from todo list when we uncheck by
                                      // send ajax request to server to update todo
                                      for(var i = 0; i < array_of_todos.length; i++){
                                          if(array_of_todos[i].description == _threadID){
                                              var cache_item = array_of_todos[i];
                                              cache_item.checked = !array_of_todos[i].checked;
                                              array_of_todos.splice(i,1);
                                              array_of_todos.push(cache_item)
                                              $.ajax({
                                                  url:"https://afternoon-ocean-92308.herokuapp.com/todos/"+cache_item.id,
                                                  type:"DELETE",
                                                  data:{
                                                      todo : { item:cache_item.item, checked:true, description:cache_item.description}
                                                  },
                                                  success:function(response){

                                                      sdk.Widgets.showModalView({
                                                          el:"<h3>New todo Item Updated</h3>",
                                                          title:"Edit Todo"
                                                      });

                                                  }
                                              });

                                              break;
                                          }
                                      }
                                  }

                              }
                          })


                            emitter.emit({
                                title:"Todo Item",
                                foregroundColor:"#fff",
                                backgroundColor:"#bdbdbd"
                            })

                        }
                    }
                }
            })


        })





  });

});


```

Its high time we loaded all the todos created from rails and add them to our array of todos. This way the moment the user visits his/her gmail account all the labels are added to each email thread

```
// fetch all the todos from the api and add to array of todos
$.ajax({
    url:"https://afternoon-ocean-92308.herokuapp.com/todos/",
    type:"GET",
    success:function(response){

        for(var i = 0; i < response.length; i++){
            array_of_todos.push(response[i])
            // add each todo item to the NavItem
        }


    }
});


```


------> 005.png <-------

```
chrome.extension.sendMessage({}, function(response) {

    InboxSDK.load('1', 'Hello World!').then(function(sdk){

        var array_of_todos = []; //object type { thread_id, item, checked }

        var todoItem = sdk.NavMenu.addNavItem({
            name: "Todos",
            iconUrl: "https://i.imgur.com/52FWtfw.png"
        });

        function checkIfIsTodo(array,thread_id){
           if(array.length == 0){
             return false;
           } else {
             for(var i = 0; i < array.length; i++){
               if(array[i].description == thread_id){
                 return true;
               }
             }
           }
         }

         sdk.Lists.registerThreadRowViewHandler(function(threadRowView){


           threadBtnEmmiter.emit({
               iconUrl:"http://pontifolio.com/img/grey-image.jpg",
               onClick:function(event){

                   var _threadID = event.threadRowView._threadRowViewDriver._cachedThreadID;
                   var isTodo = checkIfIsTodo(array_of_todos,threadRowView.getThreadID());

                   if(!isTodo){
                       if(threadRowView.getThreadID() == _threadID){
                           $.ajax({
                               url:"https://afternoon-ocean-92308.herokuapp.com/todos/",
                               type:"POST",
                               data:{
                                   todo : { item:threadRowView.getSubject(), checked:false, description:_threadID}
                               },
                               success:function(response){

                                   sdk.Widgets.showModalView({
                                       el:"<h3>New todo Item Created</h3>",
                                       title:"Add Todo"
                                   });
                                   array_of_todos.push(response)
                               }
                           });

                       }

                   } else {

                       // delete from todo list when we uncheck by
                       // send ajax request to server to update todo
                       for(var i = 0; i < array_of_todos.length; i++){
                           if(array_of_todos[i].description == _threadID){
                               var cache_item = array_of_todos[i];
                               cache_item.checked = !array_of_todos[i].checked;
                               array_of_todos.splice(i,1);
                               array_of_todos.push(cache_item)
                               $.ajax({
                                   url:"https://afternoon-ocean-92308.herokuapp.com/todos/"+cache_item.id,
                                   type:"DELETE",
                                   data:{
                                       todo : { item:cache_item.item, checked:true, description:cache_item.description}
                                   },
                                   success:function(response){

                                       sdk.Widgets.showModalView({
                                           el:"<h3>New todo Item Updated</h3>",
                                           title:"Edit Todo"
                                       });

                                   }
                               });

                               break;
                           }
                       }
                   }

               }
           })


             var emitter; //variable name to hoist the emitter to
             var stream = Kefir.stream(function(inEmitter){
                 emitter = inEmitter;
                 return function(){}; //we need to return a function that gets called when the stream ends
             });
             threadRowView.addLabel(stream);

             // pass the array to listen to
             Array.observe(array_of_todos, function(changes){
                 console.log(changes);
                 for(var i = 0; i < array_of_todos.length; i++){
                     var _threadID = threadRowView._threadRowViewDriver._cachedThreadID;
                     if(_threadID == array_of_todos[i].description){
                       // if the todo item is checked then remove any label
                       // and add a todo is Completed label
                         if(array_of_todos[i].checked == true){
                             emitter.emit(null);
                             emitter.emit({
                                 title:"Todo Completed",
                                 foregroundColor:"#fff",
                                 backgroundColor:"#91c661"
                             })

                             threadBtnEmmiter.emit(null);
                             threadBtnEmmiter.emit({
                                 iconUrl:"http://pontifolio.com/img/green-image.jpg",
                                 onClick:function(event){

                                     var _threadID = event.threadRowView._threadRowViewDriver._cachedThreadID;
                                     var isTodo = checkIfIsTodo(array_of_todos,threadRowView.getThreadID());

                                     if(!isTodo){
                                         if(threadRowView.getThreadID() == _threadID){
                                             $.ajax({
                                                 url:"https://afternoon-ocean-92308.herokuapp.com/todos/",
                                                 type:"POST",
                                                 data:{
                                                     todo : { item:threadRowView.getSubject(), checked:false, description:_threadID}
                                                 },
                                                 success:function(response){

                                                     sdk.Widgets.showModalView({
                                                         el:"<h3>New todo Item Created</h3>",
                                                         title:"Add Todo"
                                                     });
                                                     array_of_todos.push(response)
                                                 }
                                             });

                                         }

                                     } else {

                                         // delete from todo list when we uncheck by
                                         // send ajax request to server to update todo
                                         for(var i = 0; i < array_of_todos.length; i++){
                                             if(array_of_todos[i].description == _threadID){
                                                 var cache_item = array_of_todos[i];
                                                 cache_item.checked = !array_of_todos[i].checked;
                                                 array_of_todos.splice(i,1);
                                                 array_of_todos.push(cache_item)
                                                 $.ajax({
                                                     url:"https://afternoon-ocean-92308.herokuapp.com/todos/"+cache_item.id,
                                                     type:"DELETE",
                                                     data:{
                                                         todo : { item:cache_item.item, checked:true, description:cache_item.description}
                                                     },
                                                     success:function(response){

                                                         sdk.Widgets.showModalView({
                                                             el:"<h3>New todo Item Updated</h3>",
                                                             title:"Edit Todo"
                                                         });

                                                     }
                                                 });

                                                 break;
                                             }
                                         }
                                     }

                                 }
                             })



                         } else {
                           // else add a todo item label

                           threadBtnEmmiter.emit(null);
                           threadBtnEmmiter.emit({
                               iconUrl:"https://encrypted-tbn3.gstatic.com/images?q=tbn:ANd9GcR6t41ErQxx0y1rApv207bM3LznQVdvOILrYy-XTUVg3JpxGvRn",
                               onClick:function(event){

                                   var _threadID = event.threadRowView._threadRowViewDriver._cachedThreadID;
                                   var isTodo = checkIfIsTodo(array_of_todos,threadRowView.getThreadID());

                                   if(!isTodo){
                                       if(threadRowView.getThreadID() == _threadID){
                                           $.ajax({
                                               url:"https://afternoon-ocean-92308.herokuapp.com/todos/",
                                               type:"POST",
                                               data:{
                                                   todo : { item:threadRowView.getSubject(), checked:false, description:_threadID}
                                               },
                                               success:function(response){

                                                   sdk.Widgets.showModalView({
                                                       el:"<h3>New todo Item Created</h3>",
                                                       title:"Add Todo"
                                                   });
                                                   array_of_todos.push(response)
                                               }
                                           });

                                       }

                                   } else {

                                       // delete from todo list when we uncheck by
                                       // send ajax request to server to update todo
                                       for(var i = 0; i < array_of_todos.length; i++){
                                           if(array_of_todos[i].description == _threadID){
                                               var cache_item = array_of_todos[i];
                                               cache_item.checked = !array_of_todos[i].checked;
                                               array_of_todos.splice(i,1);
                                               array_of_todos.push(cache_item)
                                               $.ajax({
                                                   url:"https://afternoon-ocean-92308.herokuapp.com/todos/"+cache_item.id,
                                                   type:"DELETE",
                                                   data:{
                                                       todo : { item:cache_item.item, checked:true, description:cache_item.description}
                                                   },
                                                   success:function(response){

                                                       sdk.Widgets.showModalView({
                                                           el:"<h3>New todo Item Updated</h3>",
                                                           title:"Edit Todo"
                                                       });

                                                   }
                                               });

                                               break;
                                           }
                                       }
                                   }

                               }
                           })


                             emitter.emit({
                                 title:"Todo Item",
                                 foregroundColor:"#fff",
                                 backgroundColor:"#bdbdbd"
                             })

                         }
                     }
                 }
             })


         })



        // fetch all the todos from the api and add to array of todos
        $.ajax({
            url:"https://afternoon-ocean-92308.herokuapp.com/todos/",
            type:"GET",
            success:function(response){

                for(var i = 0; i < response.length; i++){
                    array_of_todos.push(response[i])
                    // add each todo item to the NavItem
                }


            }
        });



  });

});


```

Phewwww, we are almost there. So whats left? We just need to add each todo item to the NavItem (Todo) in our side bar as children components... To do this we will create navItems on ajax load of the all the todo items from rails..

We also will need to keep track of all the navItems, so that we can edit their properties later on.
This time around we won't be using streams but will be keeping all the navItems created in an array_of_navitems..

```
var array_of_navitems = []; // object type{navItem, item, description, checked}
// fetch all the todos from the api and add to array of todos
// create navItems from response
$.ajax({
    url:"https://afternoon-ocean-92308.herokuapp.com/todos/",
    type:"GET",
    success:function(response){

        for(var i = 0; i < response.length; i++){
            array_of_todos.push(response[i])
            var newItem = todoItem.addNavItem({
                name:response[i].item,
                iconUrl:(!response[i].checked ? "http://www.dotnetcart.com/demov4/Styles/images/icons/icon-pricetable-false.png" : "https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcRLOGxjHDWICicw_XZ5mvtyu-h9_W7QcrB131SQsG453Y-zzlT2")
            })
            array_of_navitems.push({
                navItem:newItem,
                item:response[i].item,
                description:response[i].description,
                checked:response[i].checked
            })
        }


    }
});


```
----> 007.png <------

```
chrome.extension.sendMessage({}, function(response) {

    InboxSDK.load('1', 'Hello World!').then(function(sdk){

        var array_of_todos = []; //object type { thread_id, istem, checked }
        var array_of_navitems = []; // object type{navItem, item, description, checked}

        var todoItem = sdk.NavMenu.addNavItem({
            name: "Todos",
            iconUrl: "https://i.imgur.com/52FWtfw.png"
        });

        function checkIfIsTodo(array,thread_id){
           if(array.length == 0){
             return false;
           } else {
             for(var i = 0; i < array.length; i++){
               if(array[i].description == thread_id){
                 return true;
               }
             }
           }
         }

         sdk.Lists.registerThreadRowViewHandler(function(threadRowView){


           threadBtnEmmiter.emit({
               iconUrl:"http://pontifolio.com/img/grey-image.jpg",
               onClick:function(event){

                   var _threadID = event.threadRowView._threadRowViewDriver._cachedThreadID;
                   var isTodo = checkIfIsTodo(array_of_todos,threadRowView.getThreadID());

                   if(!isTodo){
                       if(threadRowView.getThreadID() == _threadID){
                           $.ajax({
                               url:"https://afternoon-ocean-92308.herokuapp.com/todos/",
                               type:"POST",
                               data:{
                                   todo : { item:threadRowView.getSubject(), checked:false, description:_threadID}
                               },
                               success:function(response){

                                   sdk.Widgets.showModalView({
                                       el:"<h3>New todo Item Created</h3>",
                                       title:"Add Todo"
                                   });
                                   array_of_todos.push(response)
                               }
                           });

                       }

                   } else {

                       // delete from todo list when we uncheck by
                       // send ajax request to server to update todo
                       for(var i = 0; i < array_of_todos.length; i++){
                           if(array_of_todos[i].description == _threadID){
                               var cache_item = array_of_todos[i];
                               cache_item.checked = !array_of_todos[i].checked;
                               array_of_todos.splice(i,1);
                               array_of_todos.push(cache_item)
                               $.ajax({
                                   url:"https://afternoon-ocean-92308.herokuapp.com/todos/"+cache_item.id,
                                   type:"DELETE",
                                   data:{
                                       todo : { item:cache_item.item, checked:true, description:cache_item.description}
                                   },
                                   success:function(response){

                                       sdk.Widgets.showModalView({
                                           el:"<h3>New todo Item Updated</h3>",
                                           title:"Edit Todo"
                                       });

                                   }
                               });

                               break;
                           }
                       }
                   }

               }
           })


             var emitter; //variable name to hoist the emitter to
             var stream = Kefir.stream(function(inEmitter){
                 emitter = inEmitter;
                 return function(){}; //we need to return a function that gets called when the stream ends
             });
             threadRowView.addLabel(stream);

             // pass the array to listen to
             Array.observe(array_of_todos, function(changes){
                 console.log(changes);
                 for(var i = 0; i < array_of_todos.length; i++){
                     var _threadID = threadRowView._threadRowViewDriver._cachedThreadID;
                     if(_threadID == array_of_todos[i].description){
                       // if the todo item is checked then remove any label
                       // and add a todo is Completed label
                         if(array_of_todos[i].checked == true){
                             emitter.emit(null);
                             emitter.emit({
                                 title:"Todo Completed",
                                 foregroundColor:"#fff",
                                 backgroundColor:"#91c661"
                             })

                             threadBtnEmmiter.emit(null);
                             threadBtnEmmiter.emit({
                                 iconUrl:"http://pontifolio.com/img/green-image.jpg",
                                 onClick:function(event){

                                     var _threadID = event.threadRowView._threadRowViewDriver._cachedThreadID;
                                     var isTodo = checkIfIsTodo(array_of_todos,threadRowView.getThreadID());

                                     if(!isTodo){
                                         if(threadRowView.getThreadID() == _threadID){
                                             $.ajax({
                                                 url:"https://afternoon-ocean-92308.herokuapp.com/todos/",
                                                 type:"POST",
                                                 data:{
                                                     todo : { item:threadRowView.getSubject(), checked:false, description:_threadID}
                                                 },
                                                 success:function(response){

                                                     sdk.Widgets.showModalView({
                                                         el:"<h3>New todo Item Created</h3>",
                                                         title:"Add Todo"
                                                     });
                                                     array_of_todos.push(response)
                                                 }
                                             });

                                         }

                                     } else {

                                         // delete from todo list when we uncheck by
                                         // send ajax request to server to update todo
                                         for(var i = 0; i < array_of_todos.length; i++){
                                             if(array_of_todos[i].description == _threadID){
                                                 var cache_item = array_of_todos[i];
                                                 cache_item.checked = !array_of_todos[i].checked;
                                                 array_of_todos.splice(i,1);
                                                 array_of_todos.push(cache_item)
                                                 $.ajax({
                                                     url:"https://afternoon-ocean-92308.herokuapp.com/todos/"+cache_item.id,
                                                     type:"DELETE",
                                                     data:{
                                                         todo : { item:cache_item.item, checked:true, description:cache_item.description}
                                                     },
                                                     success:function(response){

                                                         sdk.Widgets.showModalView({
                                                             el:"<h3>New todo Item Updated</h3>",
                                                             title:"Edit Todo"
                                                         });

                                                     }
                                                 });

                                                 break;
                                             }
                                         }
                                     }

                                 }
                             })



                         } else {
                           // else add a todo item label

                           threadBtnEmmiter.emit(null);
                           threadBtnEmmiter.emit({
                               iconUrl:"https://encrypted-tbn3.gstatic.com/images?q=tbn:ANd9GcR6t41ErQxx0y1rApv207bM3LznQVdvOILrYy-XTUVg3JpxGvRn",
                               onClick:function(event){

                                   var _threadID = event.threadRowView._threadRowViewDriver._cachedThreadID;
                                   var isTodo = checkIfIsTodo(array_of_todos,threadRowView.getThreadID());

                                   if(!isTodo){
                                       if(threadRowView.getThreadID() == _threadID){
                                           $.ajax({
                                               url:"https://afternoon-ocean-92308.herokuapp.com/todos/",
                                               type:"POST",
                                               data:{
                                                   todo : { item:threadRowView.getSubject(), checked:false, description:_threadID}
                                               },
                                               success:function(response){

                                                   sdk.Widgets.showModalView({
                                                       el:"<h3>New todo Item Created</h3>",
                                                       title:"Add Todo"
                                                   });
                                                   array_of_todos.push(response)
                                               }
                                           });

                                       }

                                   } else {

                                       // delete from todo list when we uncheck by
                                       // send ajax request to server to update todo
                                       for(var i = 0; i < array_of_todos.length; i++){
                                           if(array_of_todos[i].description == _threadID){
                                               var cache_item = array_of_todos[i];
                                               cache_item.checked = !array_of_todos[i].checked;
                                               array_of_todos.splice(i,1);
                                               array_of_todos.push(cache_item)
                                               $.ajax({
                                                   url:"https://afternoon-ocean-92308.herokuapp.com/todos/"+cache_item.id,
                                                   type:"DELETE",
                                                   data:{
                                                       todo : { item:cache_item.item, checked:true, description:cache_item.description}
                                                   },
                                                   success:function(response){

                                                       sdk.Widgets.showModalView({
                                                           el:"<h3>New todo Item Updated</h3>",
                                                           title:"Edit Todo"
                                                       });

                                                   }
                                               });

                                               break;
                                           }
                                       }
                                   }

                               }
                           })


                             emitter.emit({
                                 title:"Todo Item",
                                 foregroundColor:"#fff",
                                 backgroundColor:"#bdbdbd"
                             })

                         }
                     }
                 }
             })


         })

        // fetch all the todos from the api and add to array of todos
        // create navItems from response
        $.ajax({
            url:"https://afternoon-ocean-92308.herokuapp.com/todos/",
            type:"GET",
            success:function(response){

                for(var i = 0; i < response.length; i++){
                    array_of_todos.push(response[i])
                    var newItem = todoItem.addNavItem({
                        name:response[i].item,
                        iconUrl:(!response[i].checked ? "http://www.dotnetcart.com/demov4/Styles/images/icons/icon-pricetable-false.png" : "https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcRLOGxjHDWICicw_XZ5mvtyu-h9_W7QcrB131SQsG453Y-zzlT2")
                    })
                    array_of_navitems.push({
                        navItem:newItem,
                        item:response[i].item,
                        description:response[i].description,
                        checked:response[i].checked
                    })
                }


            }
        });


  });

});


```

So now lets update the sidebar list of todo whenever the user updates the todo item in the thread view.
Since whenever the user updates the todo item in the threadRowView we update the array_of_todos, we can simply observe a change in that and update the navitems respectively

```
   function shouldUpdateNavItem(array_of_navItems,todo_object){
       /**
        This function iterates throught the array of navitems looking for a nav item that has the same property has the todo list item given to is
        if the description of both are the same (note description is the email thread_id which is unique)
           we check for the equality of the navitem item checked property with the todo_object checked property
           if they are equal we delete the navItem and return true

        We also return true if the description is different and if the navitems is empty

        This is done because a navItem doesn't expose an update function but only a add child or remove function
        so to reduce adding multiple navItem with the same thread_id (description) to the sidebar whenever the user changes the state of a todo
        item we delete the previous one and add a new one... something like using es6 Object.assign({},old_val, new_val)..

       */
        if(array_of_navItems.length > 0){

           for(var item = 0; item < array_of_navitems.length; item++){

               if(array_of_navItems[item].description == todo_object.description){
                   if(array_of_navItems[item].checked != todo_object.checked){
                       console.log(array_of_navItems[item]);
                       array_of_navItems[item].navItem.remove();
                       return true;
                   }
               } else if(array_of_navItems[item].description != todo_object.description){
                   // this condition also checks if the item is not even in the list of navItem and return true
                   console.log("its a new item all together");
                   return true;
               }
           }
       } else {
           return true;
       }

   }

   // observe the array of todos and when it changes
   // we add new todo items to the list of todos
        Array.observe(array_of_todos, function(changes){
            // the SDK has been loaded, now do something with it!
                console.log(changes);
            for(var i = 0; i < array_of_todos.length; i++){
                var shouldUpdate = shouldUpdateNavItem(array_of_navitems,array_of_todos[i]);
                if(shouldUpdate){
                    var newItem = todoItem.addNavItem({
                        name:array_of_todos[array_of_todos.length - 1].item,
                        iconUrl:(!array_of_todos[array_of_todos.length - 1].checked ? "http://www.dotnetcart.com/demov4/Styles/images/icons/icon-pricetable-false.png" : "https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcRLOGxjHDWICicw_XZ5mvtyu-h9_W7QcrB131SQsG453Y-zzlT2")
                    });
                    array_of_navitems.push({
                        navItem:newItem,
                        item:array_of_todos[array_of_todos.length - 1].item,
                        description:array_of_todos[array_of_todos.length - 1].description,
                        checked:array_of_todos[array_of_todos.length - 1].checked
                    });
                    break;
                }

            }

        })


```

```
chrome.extension.sendMessage({}, function(response) {

    InboxSDK.load('1', 'Hello World!').then(function(sdk){

        var array_of_todos = []; //object type {thread_id, item, checked}
        var array_of_navitems = []; // object type{navItem, item, description, checked}

        var todoItem = sdk.NavMenu.addNavItem({
            name: "Todos",
            iconUrl: "https://i.imgur.com/52FWtfw.png"
        });

       function checkIfIsTodo(array,thread_id){
          if(array.length == 0){
            return false;
          } else {
            for(var i = 0; i < array.length; i++){
              if(array[i].description == thread_id){
                return true;
              }
            }
          }
        }

       function shouldUpdateNavItem(array_of_navItems,todo_object){
           /**
            This function iterates throught the array of navitems looking for a nav item that has the same property has the todo list item given to is
            if the description of both are the same (note description is the email thread_id which is unique)
               we check for the equality of the navitem item checked property with the todo_object checked property
               if they are equal we delete the navItem and return true

            We also return true if the description is different and if the navitems is empty

            This is done because a navItem doesn't expose an update function but only a add child or remove function
            so to reduce adding multiple navItem with the same thread_id (description) to the sidebar whenever the user changes the state of a todo
            item we delete the previous one and add a new one... something like using es6 Object.assign({},old_val, new_val)..

           */
            if(array_of_navItems.length > 0){

               for(var item = 0; item < array_of_navitems.length; item++){

                   if(array_of_navItems[item].description == todo_object.description){
                       if(array_of_navItems[item].checked != todo_object.checked){
                           console.log(array_of_navItems[item]);
                           array_of_navItems[item].navItem.remove();
                           return true;
                       }
                   } else if(array_of_navItems[item].description != todo_object.description){
                       // this condition also checks if the item is not even in the list of navItem and return true
                       console.log("its a new item all together");
                       return true;
                   }
               }
           } else {
               return true;
           }

       }



       // fetch all the todos from the api and add to array of todos
        $.ajax({
            url:"https://afternoon-ocean-92308.herokuapp.com/todos/",
            type:"GET",
            success:function(response){

                for(var i = 0; i < response.length; i++){
                    array_of_todos.push(response[i])
                    var newItem = todoItem.addNavItem({
                        name:response[i].item,
                        iconUrl:(!response[i].checked ? "http://www.dotnetcart.com/demov4/Styles/images/icons/icon-pricetable-false.png" : "https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcRLOGxjHDWICicw_XZ5mvtyu-h9_W7QcrB131SQsG453Y-zzlT2")
                    })
                    array_of_navitems.push({
                        navItem:newItem,
                        item:response[i].item,
                        description:response[i].description,
                        checked:response[i].checked
                    })
                }


            }
        });

        // observe the array of todos and when it changes
        // we add new todo items to the list of todos
        Array.observe(array_of_todos, function(changes){
            // the SDK has been loaded, now do something with it!
                console.log(changes);
            for(var i = 0; i < array_of_todos.length; i++){
                var shouldUpdate = shouldUpdateNavItem(array_of_navitems,array_of_todos[i]);
                if(shouldUpdate){
                    var newItem = todoItem.addNavItem({
                        name:array_of_todos[array_of_todos.length - 1].item,
                        iconUrl:(!array_of_todos[array_of_todos.length - 1].checked ? "http://www.dotnetcart.com/demov4/Styles/images/icons/icon-pricetable-false.png" : "https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcRLOGxjHDWICicw_XZ5mvtyu-h9_W7QcrB131SQsG453Y-zzlT2")
                    });
                    array_of_navitems.push({
                        navItem:newItem,
                        item:array_of_todos[array_of_todos.length - 1].item,
                        description:array_of_todos[array_of_todos.length - 1].description,
                        checked:array_of_todos[array_of_todos.length - 1].checked
                    });
                    break;
                }

            }

        })

        sdk.Lists.registerThreadRowViewHandler(function(threadRowView){


          threadBtnEmmiter.emit({
              iconUrl:"http://pontifolio.com/img/grey-image.jpg",
              onClick:function(event){

                  var _threadID = event.threadRowView._threadRowViewDriver._cachedThreadID;
                  var isTodo = checkIfIsTodo(array_of_todos,threadRowView.getThreadID());

                  if(!isTodo){
                      if(threadRowView.getThreadID() == _threadID){
                          $.ajax({
                              url:"https://afternoon-ocean-92308.herokuapp.com/todos/",
                              type:"POST",
                              data:{
                                  todo : { item:threadRowView.getSubject(), checked:false, description:_threadID}
                              },
                              success:function(response){

                                  sdk.Widgets.showModalView({
                                      el:"<h3>New todo Item Created</h3>",
                                      title:"Add Todo"
                                  });
                                  array_of_todos.push(response)
                              }
                          });

                      }

                  } else {

                      // delete from todo list when we uncheck by
                      // send ajax request to server to update todo
                      for(var i = 0; i < array_of_todos.length; i++){
                          if(array_of_todos[i].description == _threadID){
                              var cache_item = array_of_todos[i];
                              cache_item.checked = !array_of_todos[i].checked;
                              array_of_todos.splice(i,1);
                              array_of_todos.push(cache_item)
                              $.ajax({
                                  url:"https://afternoon-ocean-92308.herokuapp.com/todos/"+cache_item.id,
                                  type:"DELETE",
                                  data:{
                                      todo : { item:cache_item.item, checked:true, description:cache_item.description}
                                  },
                                  success:function(response){

                                      sdk.Widgets.showModalView({
                                          el:"<h3>New todo Item Updated</h3>",
                                          title:"Edit Todo"
                                      });

                                  }
                              });

                              break;
                          }
                      }
                  }

              }
          })


            var emitter; //variable name to hoist the emitter to
            var stream = Kefir.stream(function(inEmitter){
                emitter = inEmitter;
                return function(){}; //we need to return a function that gets called when the stream ends
            });
            threadRowView.addLabel(stream);

            // pass the array to listen to
            Array.observe(array_of_todos, function(changes){
                console.log(changes);
                for(var i = 0; i < array_of_todos.length; i++){
                    var _threadID = threadRowView._threadRowViewDriver._cachedThreadID;
                    if(_threadID == array_of_todos[i].description){
                      // if the todo item is checked then remove any label
                      // and add a todo is Completed label
                        if(array_of_todos[i].checked == true){
                            emitter.emit(null);
                            emitter.emit({
                                title:"Todo Completed",
                                foregroundColor:"#fff",
                                backgroundColor:"#91c661"
                            })

                            threadBtnEmmiter.emit(null);
                            threadBtnEmmiter.emit({
                                iconUrl:"http://pontifolio.com/img/green-image.jpg",
                                onClick:function(event){

                                    var _threadID = event.threadRowView._threadRowViewDriver._cachedThreadID;
                                    var isTodo = checkIfIsTodo(array_of_todos,threadRowView.getThreadID());

                                    if(!isTodo){
                                        if(threadRowView.getThreadID() == _threadID){
                                            $.ajax({
                                                url:"https://afternoon-ocean-92308.herokuapp.com/todos/",
                                                type:"POST",
                                                data:{
                                                    todo : { item:threadRowView.getSubject(), checked:false, description:_threadID}
                                                },
                                                success:function(response){

                                                    sdk.Widgets.showModalView({
                                                        el:"<h3>New todo Item Created</h3>",
                                                        title:"Add Todo"
                                                    });
                                                    array_of_todos.push(response)
                                                }
                                            });

                                        }

                                    } else {

                                        // delete from todo list when we uncheck by
                                        // send ajax request to server to update todo
                                        for(var i = 0; i < array_of_todos.length; i++){
                                            if(array_of_todos[i].description == _threadID){
                                                var cache_item = array_of_todos[i];
                                                cache_item.checked = !array_of_todos[i].checked;
                                                array_of_todos.splice(i,1);
                                                array_of_todos.push(cache_item)
                                                $.ajax({
                                                    url:"https://afternoon-ocean-92308.herokuapp.com/todos/"+cache_item.id,
                                                    type:"DELETE",
                                                    data:{
                                                        todo : { item:cache_item.item, checked:true, description:cache_item.description}
                                                    },
                                                    success:function(response){

                                                        sdk.Widgets.showModalView({
                                                            el:"<h3>New todo Item Updated</h3>",
                                                            title:"Edit Todo"
                                                        });

                                                    }
                                                });

                                                break;
                                            }
                                        }
                                    }

                                }
                            })



                        } else {
                          // else add a todo item label

                          threadBtnEmmiter.emit(null);
                          threadBtnEmmiter.emit({
                              iconUrl:"https://encrypted-tbn3.gstatic.com/images?q=tbn:ANd9GcR6t41ErQxx0y1rApv207bM3LznQVdvOILrYy-XTUVg3JpxGvRn",
                              onClick:function(event){

                                  var _threadID = event.threadRowView._threadRowViewDriver._cachedThreadID;
                                  var isTodo = checkIfIsTodo(array_of_todos,threadRowView.getThreadID());

                                  if(!isTodo){
                                      if(threadRowView.getThreadID() == _threadID){
                                          $.ajax({
                                              url:"https://afternoon-ocean-92308.herokuapp.com/todos/",
                                              type:"POST",
                                              data:{
                                                  todo : { item:threadRowView.getSubject(), checked:false, description:_threadID}
                                              },
                                              success:function(response){

                                                  sdk.Widgets.showModalView({
                                                      el:"<h3>New todo Item Created</h3>",
                                                      title:"Add Todo"
                                                  });
                                                  array_of_todos.push(response)
                                              }
                                          });

                                      }

                                  } else {

                                      // delete from todo list when we uncheck by
                                      // send ajax request to server to update todo
                                      for(var i = 0; i < array_of_todos.length; i++){
                                          if(array_of_todos[i].description == _threadID){
                                              var cache_item = array_of_todos[i];
                                              cache_item.checked = !array_of_todos[i].checked;
                                              array_of_todos.splice(i,1);
                                              array_of_todos.push(cache_item)
                                              $.ajax({
                                                  url:"https://afternoon-ocean-92308.herokuapp.com/todos/"+cache_item.id,
                                                  type:"DELETE",
                                                  data:{
                                                      todo : { item:cache_item.item, checked:true, description:cache_item.description}
                                                  },
                                                  success:function(response){

                                                      sdk.Widgets.showModalView({
                                                          el:"<h3>New todo Item Updated</h3>",
                                                          title:"Edit Todo"
                                                      });

                                                  }
                                              });

                                              break;
                                          }
                                      }
                                  }

                              }
                          })


                            emitter.emit({
                                title:"Todo Item",
                                foregroundColor:"#fff",
                                backgroundColor:"#bdbdbd"
                            })

                        }
                    }
                }
            })


        })





  });

});

```


