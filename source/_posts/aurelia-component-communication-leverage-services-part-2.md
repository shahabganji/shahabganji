# Aurelia Component Communication, Leverage Services, Part 2

_Published by [shahabganji](https://shahabganji.me) on March 24, 2019_




Hello guys, I hope you have found [part 1](http://shahabganji.me/2019/01/24/aurelia-component-communication-leverage-services-part-1/) of this article useful, we used properties of our services to store the states we required in our application. However, in the previous part there was two problems, first, the properties using services’ properties are dirty checked, and that’s not good at all, we can add `@computedFrom` attribute on top of them to inform Aurelia that these properties rely on changes of other properties so the change management mechanism will handle it properly, In vNext that is not necessary as the framework will detect them automatically. Next, there are times that we may not use properties in the views, if so, then our components won’t get notified when the properties they depend upon change, that’s where `EventAggregator` and `aurelia-store` come to play. Let’s find out more about them.

<!-- more -->

## Event Aggregator

[EventAggregator](https://aurelia.io/docs/tutorials/creating-a-contact-manager/#adding-pubsub-messaging) provides a `publisher/subscriber` mechanism within your application. That’s totally self-explanatory, a component registers for some events, say `contact-created`, and when another component fires that event the subscription, the former component, will be notified and can get an instance of data provided by the publishing component, publisher. Let’s see it in action. You can clone the project by running the following command:

```bash
git clone -b event-aggregator https://github.com/shahabganji/AureliaComponentCommunicationViaServices.git
```

Inject an instance of `EventAggregator` class from `aurelia-event-aggregator` module to the `contacts-service`:

```ts
constructor(private ea: EventAggregator) { }
```

Now we can publish messages in our service by using the injected instance, add a method named `selectContact` and add the following:

```ts
selectContact(selected: IContact) {
    this.ea.publish('selected-contact-changed', selected);
}
```

The first argument is the event name, it can also be a class, and the second one is the payload meant to be sent to the subscriber, now change other methods of this service where `currentContact` was set, to call this new method, we need to make other changes in the dependent components, `contact-list` and `details`.


In `contact-list` where the `currentContac` of `conatcts-service` were set, just call the new method and that’s done. For the other components, `ContactDetails` for instance, inject an `EventAggregator` instance, and add subscribe using that instance, the code should look like the following section:


<script src="https://gist.github.com/shahabganji/84e5ec3f10dc9a22bbb0e6bf62777166.js?file=ea-contact-details.ts"></script>


In the above code, at lines 13 to 15 we have subscribed to the `selected-contact-changed` event, thus as the second parameter we got an `IContact` sent by the publisher. It is also worth to notice that in the `deactivate` lifecycle the subscription have been `disposed` to prevent memory leaks, that is what you should always keep in mind.


Run the project and select a contact from the list and you will see the details on the right. There is one problem, however, try to add or edit a contact and when you are back you won’t see the newly added or changed contact being selected. Can you guess why? That’s because we have subscribed to the changes on the constructor of the details component, and when we publish the event the `details` component have not been created yet, so there is no active subscription.


## [Aurelia Store](https://aurelia.io/docs/plugins/store#introduction)

Aurelia store plugin is built on top of [Rxjs](https://rxjs-dev.firebaseapp.com/), it will hold a state or a snapshot of all data of your application at a specific moment. These states can be modified only by developer-defined actions registered within the store, these actions get the current state as the first parameter and other optional parameters to help manipulate the state. They preferably should not mutate the current state or we will lose advanced features such as time-traveling.


To use aurelia store plugin like any other aurelia plugin we first need to download the `npm` package and register the plugin.

```bash
yarn add aurelia-store
```

To register the plugin you may require to pass an initial state of the application:

```ts
aurelia.use.plugin( 'aurelia-store' , { initialStateObject } );
```

Next, there are two approaches, either you want to subscribe to changes to the state, or you want to manipulate the state via the actions defined and registered earlier. The former can be done by simply using `@connectTo` decorator provided by `aurelia-store`, and the latter is just by using a `dispatch` method of the sore to call the actions. Let’s see them in action. First clone the project for this section:

```bash
git clone -b aurelia-store https://github.com/shahabganji/AureliaComponentCommunicationViaServices.git 
```


Create two files, `state.ts` to hold the definition and initial state of the application, and `actions.ts` for the actions which we can use to alter the state with.

<script src="https://gist.github.com/shahabganji/84e5ec3f10dc9a22bbb0e6bf62777166.js?file=state.ts"></script>

<script src="https://gist.github.com/shahabganji/84e5ec3f10dc9a22bbb0e6bf62777166.js?file=action.ts"></script>

Well, in the state file we just defined the `state` structure and exported an initial state; in the `actions` file you can see that we have defined an action named `selectContactAction`, it takes the current state as the first parameter and a `IContact` as the second parameter, it makes a shallow copy of the current state and set the selected contact on that, then returns the new state, just that simple. We can now change parts of our code to use the store functionality.


Change the `contact-service` file to look like the following:


<script src="https://gist.github.com/shahabganji/84e5ec3f10dc9a22bbb0e6bf62777166.js?file=aus-contact-service.ts"></script>


There are two important lines in the above code, at `line 7` we have just registered the previously defined action within the store so it will be permitted to dispatch that action to manipulate the state of the store/application next at `line 11`. We’re done with the part of altering the state; it’s quite simple to subscribe for the changes in our components too. First in the `details` component add the `connectTo` decorator on top of the component’s class, this will automatically add a property to your class named `state` and it will handle the subscription and its disposal for you, by default that would be don on `bind` and `unbind` hooks.

```ts
get contact(): IContact {
    return this.state.contacts.selected;
}
private state: IState;
```

Add the above code to the details component to eliminate any further change in your `html` file. To the same for the `contact-list` component and just change the html part to update the selected contact based on the new `state` property, check `line 3`.

<script src="https://gist.github.com/shahabganji/84e5ec3f10dc9a22bbb0e6bf62777166.js?file=contact-list.html"></script>


You can now run the application and see that everything works seamlessly. I leave it as a practice for you to move the array of contacts to the store.

There’s still a problem with our application demo, it will lose its state if you hit `F5`, happily, `aurelia-store` plugin has a solution for it, Middleware!



## Conclusion

There are several approaches to handle how various components in your application store their states and communicate with each other; each of which brings merits and demerits and we should make a decision based on the scenarios and trade-offs, these approaches range from a property bag to event-aggregator and aurelia-store. I wish you a great day and best of luck :wink: