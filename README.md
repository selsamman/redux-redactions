# redux-redactions
## Reactions simplifies actions and reducers:
* Actions and reducers defined right next to each other.
* No need for action type strings to bind actions and reducers
* No need to worry about not-mutating the state
* No need to wire together a reducer heirarchy for your state tree.
* In fact you don't define reducers at all
* You just define functions that return a state slice and redux-redactions will take care of merging that into the state.
## Usage

Add reactions to your project
```
npm install --save redux-redactions
```

> This is still a work in progres yet to be incorporated into a serious React project.


Define your Reactions.  Reactions are a combination of reducers and actions:
```
var todoList = {
    AddItem: {
        action:
            (text) => ({text: text}),
        state: [{
            slice: ['domain', 'nextId'],
            set:    (action, state, nextId) => nextId + 1
        },{
            slice: ['domain', 'todoList'],
            append:    (action, state) => ({text: action.text, id: state.domain.nextId, completed: false})
        },{
            slice: ['app', 'filter'],
            set:    (action, state, filter) => filter.filter === 'SHOW_ACTIVE' ? filter.filter : 'SHOW_ALL'
        }]},
    DeleteItem: {
        action:
            (idToDelete) => ({id: idToDelete}),
        state: [{
            slice: ['domain', 'todoList', (action, state, item) => action.id == item.id],
            delete:  true
        }]},
    ToggleItem: {
        action:
            (idToToggle) => ({id: idToToggle}),
        state: [{
            slice: ['domain', 'todoList', (action, state, item) => action.id == item.id],
            assign:    (action, state, item) => ({completed: !item.completed})
        }]},
    FilterList: {
        action:
            (filter) => ({filter: filter}),
        state: [{
            slice: ['app', 'filter'],
            set:    (action, state, filter) => action.filter
        }]}
};
```
Add your reaction definitons as reactions:
```
import {Reactions} from 'redux-redactions';
Reactions.addReactions(todoList);
```
Connect them to redux:
```
const createStoreWithMiddleware = applyMiddleware(thunk)(createStore);
var state = {
    domain: {
        todoList: [
        ],
        nextId: 0
    },
    app: {filter: 'SHOW_ALL'}
};
const store = createStoreWithMiddleware(Reactions.reduce, state);
```
Dispatch them:
```
store.dispatch(Reactions.actions.AddItem("First Item"));
```
You can easily write tests your reactions and ensure that not only do they do what you want them to do but that they don't mutate other parts of the state.  The stateChanges member function returns a string which describes which state slices have changed: 
```
        let oldState = state;
        store.dispatch(Reactions.actions.AddItem("First Item"));
        expect(state.domain.todoList.length).toEqual(1);
        expect(state.domain.todoList[0].text).toEqual("First Item");
        expect(state.domain.todoList[0].completed).toEqual(false);
        expect(state.domain.nextId).toEqual(1);
        expect(state.domain.todoList instanceof Array).toEqual(true);
        expect(Reactions.stateChanges(state, oldState)).toEqual('domain;domain.todoList;domain.nextId;app;');

```

##Anatomy of a Reaction

A reaction is a definition of both an action and a reaction.  Each reaction is defined as a property where the property name is the reaction type.  
```
var todoList = {
    AddItem: {
```
The property contains further properties that
* define the function that returns an action you can dispatch:
```
        action:
            (idToToggle) => ({id: idToToggle}),
```
* define the state that will be affected by the action and how that state will be changed: 
```
        state: [{
            slice: ['domain', 'todoList', (action, state, item) => action.id == item.id],
            assign:    (action, state, item) => ({completed: !item.completed})
        }]},
```
The state definition describes the specific slice of the tree that will be modified. It is an array that defines each element of the state heirarchy.  Any state properties that are arrays or hashes can have a function that will be called for each instance and returns true to selected that particular item.  It is passed the action, the top level state, the item being compared and the index of the array (or object key). In this example it is **_domain.todoList[x]_**, where **_x_** is the todoList item that matches **_action.id == item.id_**.  You also define a property that describe what the reducer should do: :
* **assign: (action, state, item)** - a function that will return properties to be merged into a copy of the state (similar to Object.assign) 
* **set: (action, state, item)** - a function returning a new value for that slice of the state
* **append: (action, state, item)** - a function returning a new value to be concatenated to this slice of the state which must be an array.  Similar to Array.concat.
* **delete: true** - returns undefined for the new state.  Use for array elements which are to be deleted since any elements set to null or undefined will be removed from the array.

Assign, set and append functions have these arguments:
* **action** - the action object returned from the action function
* **state** - the root of the state heirarchy
* **item** - the particular slice of the state heirarchy as defined by the slice property

##State Composition

Although your reactions may be written to be aware of the entire state graph you might actually want to have them be independent of where they fit into the state of a large application. 
 
 For example you might have multiple todo Lists and select a  'current one' or you might have two specific todoLists active at the same time.  In all cases your reactions should not need to know anything other than what they need to manage a single todoList.
 
Let's say you wanted to have multiple todoLists with one active at a time. Your state might look like this:
```
       var state = {
            currentListIndex: 0,
            domain: {
                lists: [{
                    todoList: [
                ],
                nextId: 0
            }]},
            app: {
                lists: [{filter: 'SHOW_ALL'}]
            }
        };
 
 ```
 In the examples we have adopted the convention of dividing state into domain which reflects the data itself for a todoList and app which represents the workings of the application that manages the todoList.

 Your todoList reactions need not know about the fact that domain is now going to be structured into an array of todoLists.  You accomplish this by 'mapping' domain and app such that the reactions only need to know about a domain and app that represent a single todoList.  This is done with a state map:
 
 ```
 var stateMap = {
     app: ['app', 'lists', (action, state, list, index) => index == state.currentListIndex],
     domain: ['domain', 'lists', (action, state, list, index) => index == state.currentListIndex]
 }
 ```
 You add the reactions along with the state map:
 ```
 
 Reactions.addReactions(todoList, stateMap);
 ````
This does two things when reducing:
* Substitutes the 'app' and 'domain' slice elements for the ones specified in the state map such that all the original actions will apply to the correct todoList.  
* Substitutes the 'app' and 'domain' properties in the state passed to the reaction functions where state is passed such that they point to the correct todoList. 
 
This is great if you want to have multiple todoLists and you want to simply set the current one but what if you actually have multiple active todoLists on your page.  Your state might look like this:
 ```
 var state = {
     domain: {
         list1: {
             todoList: [
             ],
             nextId: 0
         },
         list2: {
             todoList: [
             ],
             nextId: 0
         }
     },
     app: {
         list1: {filter: 'SHOW_ALL'},
         list2: {filter: 'SHOW_ALL'}
     }
 };
```
Now you need to dispatch separate actions for each todoLists to ensure the correct list's state is updated.  So each todoList gets it's own state map:
 ```
 var stateMap1 = {
     app: ['app', 'list2'],
     domain: ['domain', 'list1']
 }
var stateMap1 = {
    app: ['app', 'list2'],
    domain: ['domain', 'list2']
}
```
And you can now connect each state map to the same set of actions by passing a group name when you add each set of reactions: 
 ```
  Reactions.addReactions(todoList, stateMap1, 'list1');
  Reactions.addReactions(todoList, stateMap2, 'list2');
 ```
You now have two sets of actions and cna refer to them as:
```
    Reactions.actionGroup.list1.AddItem
    Reactions.actionGroup.list2.AddItem
````
This makes it easy to connect up a component that deals with one list or the other:
```
var todoList1 = connect(
    state => ({domain: {todoList: state.app.list1}, app: {todoList: state.app.list1),
    dispatch => ({actions: bindActionCreators(Reactions.actionGroup.list1)
);
var todoList2 = connect(
   state => ({domain: {todoList: state.app.list2}, app: {todoList: state.app.list2),
    dispatch => ({actions: bindActionCreators(Reactions.actionGroup.list1)
);
````
In effect what we have created is a structure for reducers and actions that is parallel to the mechanism you would normally use in connecting a component to state -- that is setting the state props to reflect just the part of the state relevant to the component instance.