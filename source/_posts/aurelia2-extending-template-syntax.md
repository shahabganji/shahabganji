Hello and welcome back, last month Rob Eisenberg [announced](https://aurelia.io/blog/2019/10/31/aurelia-vnext-2019-fall-update/) about Aurelia 2; one of the amazing parts to me was the [extensibility](https://aurelia.io/blog/2019/10/31/aurelia-vnext-2019-fall-update/#extensibility). Let's dig deeper than just a notice :wink: ;)

* # [attributePattern](https://github.com/aurelia/aurelia/blob/master/packages/jit/src/attribute-pattern.ts#L443)

This special decorator is one of the extension points and will allow to define your very own binding syntax, such as `value.bind` in the Aurelia framework. it receives a params array of [AttributePatternDefinition](https://github.com/aurelia/aurelia/blob/master/packages/jit/src/attribute-pattern.ts#L4).

<!-- more -->

```js
export function attributePattern(...patternDefs: AttributePatternDefinition[]){
    // omitted code
}
```

```js
export interface AttributePatternDefinition {
  pattern: string;
  symbols: string;
}
```

so a simple usage would look like `@attributePattern({ pattern: 'foo@PART', symbols: '@' })`, but what are each property and how should they be configured?

* ### _pattern_: Here you define the pattern of your new syntax in terms of a very special keyword, `PART`. That's essentially the equivalent of this regex: `(.+)`, said [Fred Kleuver](https://github.com/fkleuver).

* ### _symbols_: In symbols you put anything that should not be included in part extraction, anything that makes your syntax more readable but plays no role but separator. **e.g.** in `value.bind` syntax, the `symbols` is `.` which sits there just in terms of more readability, and does not play a role in detecting `parts` of the syntax. Another example, `@attributePattern({ pattern: 'foo@PART', symbols: '@' })` on `foo@bar` would give you the parts `foo` and `bar`, but if you omitted symbols then it would give you the parts `foo@` and `bar`



This attribute should be on top of a class, and that class should have methods which their name matches the `pattern` property of each pattern you have passed to the `attributePattern`. Consider the following example:

```js
@attributePattern({ pattern: '[(PART)]', symbols: '[()]' })
export class AngularTwoWayBindingAttributePattern {

    public ['[(PART)]'](rawName: string, rawValue: string, parts: string[]): AttrSyntax {
        return new AttrSyntax(rawName, rawValue, parts[0], 'two-way');
    }

}
```

we have defined the Angular two-way binding pattern, `[(PART)]`, the symbols are `[()]` which behave s a syntax sugar for us; the public method defined in the body of the class has the same exact name as the pattern defined, this method also accepts three parameters, `rawName`, `rawValue`, and `parts`; it is most likely that somewhere in out views, html part of the application, we use the above syntax like `<input [(value)]="message">`, right?, then the above mentioned parameters will respectively have the values, `[(value)]`, `message`, and an array of `PARTS` which in this case is a single-item array with `value` as its only element; based on this information we could decide which Aurelia `AttrSyntax` to map to our customized syntax, here we used `part[0]` which is `value` to be bound with `two-way` binding of Aurelia.

Let's have another example, you do remember `ref` and its friends like `ref.view-model` in Aurelia 1, don't you? okay in Angular, there is a syntax which is more concise than what exists in Aurelia, `#`. For instance, `ref="uploadInput"` has `#uploadInput` equivalent in Angular. How do you think we could implement such syntax in Aurelia 2? You're right, by creating a class which has been decorated with a proper `@attributePattern` decorator :wink:

```js
@attributePattern(
    { pattern: '#PART', symbols: '#' }
)
export class SharpRefAttributePattern {
    
    public ['#PART'](rawName: string, rawValue: string, parts: string[]): AttrSyntax {
        return new AttrSyntax(rawName, parts[0], 'element', 'ref');
    }

}
```

Given the above example and the implementation the parameters would have values like the following:

* rawName: "#uploadInput"
* rawValue: "" , an empty string
* parts: ["uploadInput"]

then we use these values to tell Aurelia to bound the `ref` with the element and by `uploadInput` identifier. If we want to extend the syntax for `ref.view-model="uploadVM"`, for example, we could just add another patter to the existing class: 

```js
@attributePattern(
    { pattern: 'PART#PART', symbols: '#' }, // e.g. view-model#uploadVM
    { pattern: '#PART', symbols: '#' }      // e.g. #uploadInput
)
export class SharpRefAttributePattern {

    public ['PART#PART'](rawName: string, rawValue: string, parts: string[]): AttrSyntax {
        return new AttrSyntax(rawName, parts[1], parts[0], 'ref');
    }

    public ['#PART'](rawName: string, rawValue: string, parts: string[]): AttrSyntax {
        return new AttrSyntax(rawName, parts[0], 'element', 'ref');
    }
    
}
```

It is up to you to decide how each `PART` will be taken into play, super easy. I think this is one of the great features of Aurelia 2, although you might be comfortable with the current syntax, having the opportunity to define what might match the taste of your team members is always a plus.

One thing left! you cannot use these customization until you have registered them, to do so, got to the `main.ts`, or `.js` based on how you created your app, file and add lines like the following: 

```js
import { AngularTwoWayBindingAttributePattern, SharpRefAttributePattern } from './my-sample';

// omitted code
    
Aurelia.register(
    JitHtmlConfiguration, 
    
    // omitted code

    AngularTwoWayBindingAttributePattern, 
    SharpRefAttributePattern,
).app(MyApp).start();

```

this way you globalized a syntax for the application, if you desire to have a special syntax only within a special component try something like: 

```js
@customElement({ 
        name: 'foo',
        template, 
        dependencies: [
            AttributePattern.define(class { ['!PART'](n, v, [t]) { return new AttrSyntax(n, v, t, 'bind') } }
        ] 
})  
```
<br />

I hope you've enjoyed this article and you've found it fruitful for your Aurelia 2 applications, have a great day and enjoy coding.