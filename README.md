# JavaScript Module Systems Explained

## CommonJS

> The primary reason for its creation was a major lack of commonly accepted forms of JavaScript module units which could be reusable in environments different from that provided by conventional web browsers running JavaScript scripts. <sup>[1]</sup>

### Recognization

- `module.exports`
- `require()`

## AMD and UMD

The CommonJS problem is browsers don't support CommonJS. And that's [why AMD](https://requirejs.org/docs/whyamd.html)<sup>[2]</sup> came in.
And later on when npm and NodeJS got more popular, programmers want to reuse browser side AMD modules inside NodeJS so they invented UMD <sup>[3]</sup>.

> using AMD style while still making modules that can be used in Node and installed via npm without extra dependencies to set up the full AMD API

```
// AMD example:
define('myModule', ['dep1', 'dep2'], function (dep1, dep2) {
    //Define the module value by returning a value.
    return function () {};
});

// UMD exmaple: https://github.com/umdjs/umd/blob/master/templates/returnExports.js
(function (root, factory) {
    if (typeof define === 'function' && define.amd) {
        // AMD. Register as an anonymous module.
        define(['b'], factory);
    } else if (typeof module === 'object' && module.exports) {
        // Node. Does not work with strict CommonJS, but
        // only CommonJS-like environments that support module.exports,
        // like Node.
        module.exports = factory(require('b'));
    } else {
        // Browser globals (root is window)
        root.returnExports = factory(root.b);
    }
}(typeof self !== 'undefined' ? self : this, function (b) {
    // Use b in some fashion.

    // Just return a value to define the module export.
    // This example returns an object, but the module
    // can return a function as the exported value.
    return {};
}));
```

## ECMAScript Module (ESM)

To end the string of re-inventing wheels, starting from ECMAScript 2015 (ES6),
ES module is defined as JavaScript language spec, meaning no matter where you JavaScript runs,
it should support ESM. It's available in [all modern browsers](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Modules). 

For NodeJS starting from 12.17.0:
  - support ESM but require `.mjs` extension or `"type": "module"` in package.json.
  - assume `.js` CommonJS module, obviously.
  - also support dynamic (async) `import()` in both CommonJS and ESModule. 
  - no `require.xxx` in ES module but only `require()`, which **always** assume CommonJS module.

### import / export

#### Easy case

```
// StringValidator.ts
export interface StringValidator {
  isAcceptable(s: string): boolean;
}

// ZipCodeValidator.ts
export const numberRegexp = /^[0-9]+$/;
export class ZipCodeValidator implements StringValidator {
  isAcceptable(s: string) {
    return s.length === 5 && numberRegexp.test(s);
  }
}

// re-export without creating local variables
export { ZipCodeValidator } from "./ZipCodeValidator";
```

#### Rename

```
// rename and export
export { ZipCodeValidator as mainValidator };

// import and rename before usage
import { ZipCodeValidator as ZCV } from "./ZipCodeValidator";
```

### import * / export *

```
// re-export and combine exports
export * from "./StringValidator";
export * from "./ZipCodeValidator"; 

// import all exports of module as local (wrapper) variable
// NOTE: wrapper != module.exports in my-module (explain it later)
import * as wrapper from "./my-module";
```

Named `export *`

```
export * as utilities from "./utilities";
// the specific name is requried during import
import { utilities } from "./index";
```

### export `default`

- `default` export is optional
- only one `default` export per module
- **require special on word import statement**: no "as xxx"

```
// type definition in foo.mjs
export const HTTP_OK = 200
export default class Foo {}

// type consumption in client.mjs
import Foo from './foo'
import { HTTP_OK } from './foo'
```

The `default` export actually exports an `module.exports` object with extra `default` field,
containing the type specified in the statement.
One can observe this by compiling the code of foo.mjs above as TypeScript.

### TypeScript Only: `export =` and `import = require()`

Because TypeScript compiler will error if one `import * as Mod from './mod.js`, which use `module.exports`:
> This module can only be referenced with ECMAScript imports/exports by turning on the 'esModuleInterop' flag and referencing its default export.

During execution, `import * as Mod from './mod.js`, the Mod will become a 'Module' object instead of `module.exports`. According to TypeScript document:

>Default exports are meant to act as a replacement for this behavior; however, the two are incompatible. TypeScript supports export = to model the traditional CommonJS and AMD workflow.<p/>
The export = syntax specifies a single object that is exported from the module. This can be a class, interface, namespace, function, or enum.

One cannot `export =` and `export default` at same time (TypeScript cannot compile), however [there is a trick](https://remarkablemark.org/blog/2020/05/05/typescript-export-commonjs-es6-modules/):
```
myModule.default = myModule
export = myModule
```


### TypeScript --esModuleInterop flag

This flag resolves the above problem, by changing the TypeScript compiler behavior in 2 ways<sup>[7]</sup>:

1. a namespace import like `import * as moment from "moment"` acts the same as  
  `const moment = require("moment")`
2. a default import like `import moment from "moment"` acts the same as  
  `const moment = require("moment").default`.

The second change is compiler sugar which generates helper functions.  
While **the first change actually breaks the language spec**, so TypeScript make this flag `false` by default.

### Special cases

```
// Import a module for side-effects only
import "./my-module.js";

// useful when creating a types only TypeScript module
import type { APIResponseType } from "./api";
```

## References
1. https://en.wikipedia.org/wiki/CommonJS
2. https://en.wikipedia.org/wiki/Asynchronous_module_definition
3. https://github.com/umdjs/umd#amd-with-simple-nodecommonjs-adapter
4. https://www.typescriptlang.org/docs/handbook/modules.html
5. https://nodejs.org/docs/latest-v12.x/api/esm.html
6. https://www.typescriptlang.org/docs/handbook/namespaces-and-modules.html
7. https://www.typescriptlang.org/tsconfig#esModuleInterop
