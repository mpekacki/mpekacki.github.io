---
layout: post
title:  "Using AutoAnimate in Lightning Web Components"
date:   2024-09-09 19:00:00 +0200
categories: salesforce lwc autoanimate js
---
In this post we will see how to add nicely looking animations to Salesforce's Lightning Web Components using minimal amount of code by using [AutoAnimate](https://autoanimate.style/), a free JavaScript library.

## What does AutoAnimate do?

As their website says, "AutoAnimate is a zero-config, drop-in animation utility that adds smooth transitions to your web app. You can use it with React, Solid, Vue, Svelte, or any other JavaScript application". And indeed, you can also use it with Salesforce. Here's how it will look like when applied to a simple LWC with a sortable list:

<iframe width="100%" height="800" src="https://empathetic-raccoon-u6p71g-dev-ed.trailblaze.my.site.com/autoAnimate/" frameborder="0"></iframe>

## How to use it in LWC

All it took to add these nice animations was to, first, add a class to the parent element for the list elements in the HTML file:

`<div class="animContainer">`

in the JS file, import the library,
```
import autoAnimate from 'c/autoAnimate';
```

define a flag that will make sure that we initialize the animations only once,
```
export default class AnimTodo extends LightningElement {
    aaInitialized = false;
```

and add a `renderedCallback` that will perform the initialization
```
renderedCallback() {
    if (this.aaInitialized) return;
    const animContainer = this.template.querySelector('.animContainer');
    if (!animContainer) return;
    autoAnimate(animContainer);
    this.aaInitialized = true;
}
```

That's it! Now the list is nicely animated every time an element changes position, a new element is created or an existing element is deleted.

## Adding the library to SFDX project

The only other thing we need to do is to add the library to our project (which we imported in `import autoAnimate from 'c/autoAnimate';`). We can do it by adding the library as an LWC module. To do it, let's create a new Lightning Web Component in our project and name it `autoAnimate`. If you're using Illuminated Cloud 2, you can choose `Type: Service` in the new LWC dialog - this way, no HTML file will be created, since it's unneccessary. If you're using VS Code, LWCs are always created with HTML files, but you can just delete the .html file. Now, your new `autoAnimate` component will look like this:

```
import { LightningElement } from 'lwc';

export default class AutoAnimate extends LightningElement {}
```

Since it's just a regular [JS module](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Modules), we can put any valid JS here and `export` whatever we want - and it just happens that AutoAnimate library is also a JS module. So, we can just replace the LWC contents with the code of the library and we'll be able to use it in our Salesforce org. If we take a look at the [.js file of AutoAnimate](https://cdn.jsdelivr.net/npm/@formkit/auto-animate@latest/index.mjs), we can see that it ends with an `export` statement: `export{autoAnimate as default,getTransitionSizes,vAutoAnimate};`. So, if we copy the whole contents and paste them into our newly created LWC .js file, we will have a working JS module with the exports just as we need them. The problem is, we will have to manually keep up with updates to AutoAnimate: if a new version is released, we will have to again manually copy the contents of the .js file into our project.

But, there is a more elegant solution. We can leverage a useful Salesforce feature: [Replace Strings in Code Before Deploying or Packaging](https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/sfdx_dev_ws_string_replace.htm). How can it help us? Well, AutoAnimate can (and usually should) be installed using `npm`. We can install it in our project using `npm install @formkit/auto-animate`. Now, in our project, at `node_modules/@formkit/auto-animate/index.mjs`, we can see the JS file with current version of AutoAnimate (.mjs extension means that it is a JS module, as we mentioned previously). Now we don't have to do any messy copy-pastes to keep the file updated - we have npm and package.json to do the work for us. But, currently LWCs won't be able to use the library, since it is located inside `node_modules`, and we need it as a normal LWC. That's where the replacement feature comes into play. In our `sfdx-project.json`, let's define the replacement to be performed:

```
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