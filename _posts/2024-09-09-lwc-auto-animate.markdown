---
layout: post
title:  "Using AutoAnimate in Lightning Web Components"
date:   2024-09-09 19:00:00 +0200
categories: salesforce lwc autoanimate js
---
In this post we will see how to add nicely looking animations to Salesforce's Lightning Web Components using minimal amount of code by using [AutoAnimate](https://autoanimate.style/), a free JavaScript library.

## What does AutoAnimate do?

As their website says, "AutoAnimate is a zero-config, drop-in animation utility that adds smooth transitions to your web app. You can use it with React, Solid, Vue, Svelte, or any other JavaScript application". And indeed, you can also use it with Salesforce. Here's how it will look like when applied to a simple LWC with a sortable list:

<video controls width="100%">
  <source src="/media/todo1.mp4" type="video/mp4"/>
</video>

Or with to a search and sort component:

<video controls width="100%">
  <source src="/media/search1.mp4" type="video/mp4"/>
</video>

## How to use it in LWC

All it took to add these nice animations was to, first, add a class (`animContainer` in this case) to the parent element for the list elements in the HTML file:

```html
<div class="animContainer">
    <template for:each={items} for:item="item">
        <div key={item.id}>
            {item.name}
        </div>
    </template>
</div>
```

in the JS file, import the library,
```javascript
import autoAnimate from 'c/autoAnimate';
```

define a flag that will make sure that we initialize the animations only once,
```javascript
export default class AnimTodo extends LightningElement {
    aaInitialized = false;
```

and add a `renderedCallback` that will perform the initialization
```javascript
renderedCallback() {
    if (this.aaInitialized) return; // if already initialized, do nothing
    const animContainer = this.template.querySelector('.animContainer'); // get animation container
    if (!animContainer) return; // container is not yet created, return and wait for another renderedCallback
    autoAnimate(animContainer); // initialize animations using AutoAnimate
    this.aaInitialized = true; // set the flag to true so we don't initialize again
}
```

That's it! Now the list is nicely animated every time an element changes position, a new element is created or an existing element is deleted.

## Adding the library to SFDX project

The only other thing we need to do is to add the library (which we imported in `import autoAnimate from 'c/autoAnimate';`) to our project. We can do it by adding the library as an LWC module. To do it, let's create a new Lightning Web Component in our project and name it `autoAnimate`. If you're using Illuminated Cloud 2, you can choose `Type: Service` in the new LWC dialog - this way, no HTML file will be created, since it's unneccessary. If you're using VS Code, LWCs are always created with HTML files, but you can just delete the .html file. Now, your new `autoAnimate` component will look like this:

```javascript
import { LightningElement } from 'lwc';

export default class AutoAnimate extends LightningElement {}
```

Since it's just a regular [JS module](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Modules), we can put any valid JS here and `export` whatever we want - and it just happens that AutoAnimate library is also a JS module. So, we can just replace the LWC contents with the code of the library and we'll be able to use it in our Salesforce org. If we take a look at the [.mjs file of AutoAnimate](https://cdn.jsdelivr.net/npm/@formkit/auto-animate@latest/index.mjs) (.mjs extension means that it is a JS module), we can see that it ends with an `export` statement: `export{autoAnimate as default,getTransitionSizes,vAutoAnimate};`. So, if we copy the whole contents and paste them into our newly created LWC .js file, we will have a working JS module with the exports just as we need them. We can go ahead with this approach and it will work, but there is a better way.

### Syncing using npm and string replacement
The problem with the previous approach is that we will have to manually keep up with updates to AutoAnimate: if a new version is released, we will have to again manually copy the contents of the .mjs file into our project.

But, there is a more elegant solution. We can leverage a useful Salesforce feature: [Replace Strings in Code Before Deploying or Packaging](https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/sfdx_dev_ws_string_replace.htm). How can it help us? Well, AutoAnimate can (and usually should) be installed using `npm`. We can install it in our project using `npm install @formkit/auto-animate`. Now, in our project, at `node_modules/@formkit/auto-animate/index.mjs`, we can see the JS file with current version of AutoAnimate. Now we don't have to do any messy copy-pastes to keep the file updated - we have npm and package.json to do the work for us. But, currently LWCs won't be able to use the library, since it is located inside `node_modules`, and we need it as a normal LWC. That's where the replacement feature comes into play. In our `sfdx-project.json`, let's define the replacement to be performed:

```javascript
{
  "packageDirectories": [
    {
      "path": "force-app",
      "default": true
    }
  ],
  "name": "autoanimate",
  "namespace": "",
  "sfdcLoginUrl": "https://login.salesforce.com",
  "sourceApiVersion": "61.0",
  "replacements": [
    {
      "filename": "force-app/main/default/lwc/autoAnimate/autoAnimate.js",
      "stringToReplace": "replaceMeWith@LibraryFromNodeModules",
      "replaceWithFile": "node_modules/@formkit/auto-animate/index.mjs"
    }
  ]
}
```
and in `force-app\main\default\lwc\autoAnimate\autoAnimate.js`, replace the whole contents of the file with just this single string:
`replaceMeWith@LibraryFromNodeModules`

Now, our LWC module will be kept in sync with `npm` automatically! One more thing: apparently, the replacements don't work for all deployment methods. I didn't test it thoroughly yet, but it seems that if we just deploy our LWC library using a VS Code command, the replacement will not be performed. But if we use `sf project deploy start` - it will. Of course, the latter command will deploy the whole project, not just a single component. To avoid mistakes, we put `@` inside our replacement string - this makes the unaltered file contain an invalid JavaScript, so it will not be deployed using a method that does not perform replacements and instead it will produce an error. If we just used `replaceMeWithLibraryFromNodeModules`, it would deploy even without replacement, since this string is technically a valid Javascript (albeit useless).

## Limitations related to LWC

So far everything looks good, but there are some caveats related to how LWCs work.

First, the library will only work in orgs that have [Lightning Web Security (LWS)](https://developer.salesforce.com/docs/platform/lwc/guide/security-lwsec-intro.html) enabled. If your org doesn't use it, it uses Locker Service instead, and that means that any third-party libraries are not allowed to modify the DOM, so the animations will not work.

Second, even if LWS is enabled, not all DOM can animated - only standard HTML tags that you explicitly define in your LWC. In particular, any standard LWC components such as `lightning-button-group` or `lightning-data-table` will not work with AutoAnimate because their DOM is protected by LWS and can't be touched by any third-party code. For example, you may be tempted to animate `lightning-layout`:

```html
<lightning-layout class="animContainer"> <!-- This will not work -->
    <lightning-layout-item for:each={items} for:item="item" key={item.id}>
        {item.name}
    </lightning-layout-item>
</lightning-layout>
```
Unfortunately, this will not work. LWS protects the DOM of `lightning-` components. AutoAnimate won't be able to access or modify it.

However, we can achieve a similar effect in another way. We can use normal HTML tags and [SLDS classes](https://www.lightningdesignsystem.com/utilities/grid/):

```html
<div class="slds-grid slds-gutters animContainer">
  <div class="slds-col slds-size_1-of-3" for:each={items} for:item="item" key={item.id}>
    <span>{item.name}</span>
  </div>
</div>
```
Now we have full access and control over the DOM and the library will be able to animate it.