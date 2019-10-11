# Aurelia Component Communication, Leverage Services, Part 1

_Published by [shahabganji](https://shahabganji.me) on January 24, 2019_


Hello and welcome! In this post I want to discover how you can take advantage of services to make components of your application talk to each other. This is inspired by what Mrs. Deborah Kurata explained [here](https://www.pluralsight.com/courses/angular-component-communication) and I have tried to adapt it to the world of Aurelia. I have created a [Github repository](https://github.com/shahabisblogging/AureliaComponentCommunicationViaServices/tree/start) so that you can use it to follow up with this post. I am a great fan of typescript which is used in the sample, besides I also used [aurelia-toolbelt](https://github.com/aurelia-toolbelt/aurelia-toolbelt) as the UI library with the `aurelia-cli` configured with built-in bundler and no tests. Shall we start? Let’s do it!

## A component needs to talk to itself

There are times that you need a component to keep her state, as such it can use it in the near future when it is required. To start this part you can clone the repository and run the the following commands to start the project.

```bash
- git clone --branch start https://github.com/shahabganji/AureliaComponentCommunicationViaServices.git

- yarn install
- au run --watch
```

Bear in mind that you should run the latter two commands within the project’s directory. When you ran the project you should see a welcome screen with a navigation bar at the top, clicking on the `contacts` menu, you’ll be navigated to the contact page with a grid of contacts, and a filtering text box, play around with filtering and you’ll see that the list will be filtered; however if you navigate away and return back, you’ll see that your component have lost its latest state. Some times we need our components to retain their state for further use. At such situations we can use an intermediary service to keep track of states for use, this service acts as a property holder for our component. To see it in action create a class under the `src/contacts/services/` and name it `contact-filter-bag`.

<script src="https://gist.github.com/shahabganji/84e5ec3f10dc9a22bbb0e6bf62777166.js?file=contact-filter-bag.ts"></script>

Should we want such a simple service do a magic for us, we have to import it and inject an instance via the constructor in the `main` component, you can find it under the path `src/contacts/routes/components/main` and change lines 17 and 20 to use the injected instance instead of `_criteria` field.

<script src="https://gist.github.com/shahabganji/84e5ec3f10dc9a22bbb0e6bf62777166.js?file=contact-main.ts"></script>

Now let’s try what would happen if we search and navigate away, then return back and you’ll see the component restores/retains its latest state. How it works now and not before? Well, components are not `singleton` in Aurelia, thus when you navigate away Aurelia disposes the old instance of the component, and when you return back it creates a **new instance**, services **are** `singleton` by default though. That means, when aurelia creates the new instance of our `main` component and observes that it requires a service as a dependency, it will inject the previously created service and _**not a new one**_, since we have bound our `criteria` property to the `filter` property of the service, it retrieves the latest value that was saved by the old component.

## Components inform each other


You can take similar approach to let one component become aware of changes in another. Checkout to the commit of this section by running the following command in the repository

```bash
git checkout inter-components
```

First of all we need to define a property, `currentContact`, for the `ContactsInMemoryService`, this is our intermediary service, now we should change `delete`, `add`, and `get` methods to update this newly added property properly. After applying the changes the service should be similar to this one:

<script src="https://gist.github.com/shahabganji/84e5ec3f10dc9a22bbb0e6bf62777166.js?file=contacts-service.ts"></script>


It’s time to change the contact details and contact list components, in the contact list component which is `src/contacts/components/contact-list`, each item has a click event bound to `selectContact` function and passing the current clicked contact as a parameter, the only thing we need to do is to add the following line in the function, bear in mind to inject the `ContactsInMemoryService` as a dependency via `constructor`, use `inject` or `autoinject` decorators.

```ts
private selectContact(selected: IContact) {
    this.contactService.currentContact = selected;
}
```

For the details component, `src/contacts/routes/components/details`, you just need to change the simple property to a `get/set` property of javascript in which you return the `currentContact` property of the service

<script src="https://gist.github.com/shahabganji/84e5ec3f10dc9a22bbb0e6bf62777166.js?file=contact-details.ts"></script>

<script src="https://gist.github.com/shahabganji/84e5ec3f10dc9a22bbb0e6bf62777166.js?file=contact-details.html"></script>


Easy peasy!? This approach works because the service properties are used in the properties of our component that are bound to elements in the template of that component, `${contact.family}, ${contact.name}`, thus, the change detection mechanism of Aurelia detects the changes in the properties of the services and updates all the dependent parts of the application.

However, if we want to be notified of changes in the code and not in the `templates/views` this approach is far from useful, that’s where `EventAggregator` and `aurelia-store` come into play. We will discover those two approaches in part two of the same subject, and I hope you have found this part fruitful. Have a great day and enjoy coding. :smile:




