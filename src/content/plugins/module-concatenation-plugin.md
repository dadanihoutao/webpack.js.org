---
title: ModuleConcatenationPlugin
contributors:
  - skipjack
  - TheLarkInn
related:
  - webpack 3: Official Release!!
---

In the past, one of webpack’s trade-offs when bundling was that each module in your bundle would be wrapped in individual function closures. These wrapper functions made it slower for your JavaScript to execute in the browser. In comparison, tools like Closure Compiler and RollupJS ‘hoist’ or concatenate the scope of all your modules into one closure and allow for your code to have a faster execution time in the browser.

This plugin will enable the same concatenation behavior in webpack.

``` js
new webpack.optimize.ModuleConcatenationPlugin()
```

> Scope Hoisting is specifically a feature made possible by ECMAScript Module syntax. Because of this webpack may fallback to normal bundling based on what kind of modules you are using, and [other conditions](https://medium.com/webpack/webpack-freelancing-log-book-week-5-7-4764be3266f5).


## Optimization Bailouts

As the article explains, webpack attempts to achieve partial scope hoisting. It will merge modules into a single scope but cannot do so in every case. If webpack cannot merge a module, the two alternatives are Prevent and Root. Prevent means the module must be in its own scope. Root means a new module group will be created. The following conditions determine the outcome:

Condition                                     | Outcome
--------------------------------------------- | --------
Non ES6 Module                                | Prevent
Imported By Non Import                        | Root   
Imported From Other Chunk                     | Root   
Imported By Multiple Other Module Groups      | Root   
Imported With `import()`                      | Root   
Affected By `ProvidePlugin` Or Using `module` | Prevent
HMR Accepted                                  | Root   
Using `eval()`                                | Prevent
In Multiple Chunks                            | Prevent
`export * from "cjs-module"`                  | Prevent


### Module Grouping Algorithm

The following pseudo JavaScript explains the algorithm:

```js
modules.forEach(module => {
  const group = new ModuleGroup({
    root: module
  });
  module.dependencies.forEach(dependency => {
    tryToAdd(group, dependency);
  });
  if (group.modules.length > 1) {
    orderedModules = topologicalSort(group.modules);
    concatenatedModule = new ConcatenatedModule(orderedModules);
    chunk.add(concatenatedModule);
    orderedModules.forEach(groupModule => {
      chunk.remove(groupModule);
    });
  }
});

function tryToAdd(group, module) {
  if (group.has(module)) {
    return true;
  }
  if (!hasPreconditions(module)) {
    return false;
  }
  const nextGroup = group;
  const result = module.dependents.reduce((check, dependent) => {
    return check && tryToAdd(nextGroup, dependent);
  }, true);
  if (!result) {
    return false;
  }
  module.dependencies.forEach(dependency => {
    tryToAdd(group, dependency);
  });
  group.merge(nextGroup);
  return true;
}
```


### Debugging Optimization Bailouts

When using the webpack CLI, the `--display-optimization-bailout` flag will display bailout reasons. When using the webpack config, just add the following to the `stats` object:

```js
{
  ...stats,
  // Examine all modules
  maxModules: Infinity,
  // Display bailout reasons
  optimizationBailout: true
}
```