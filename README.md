## InWasm - Inline WebAssembly for Typescript.

InWasm is a small bundler for inline standalone wasm libraries. It compiles and bundles
the wasm source code inplace. Example with Typescript:
```typescript
// src/xy.wasm.ts
import { InWasm, OutputMode, OutputType } from 'inwasm';

const getAdderInstance = InWasm({
  name: 'adder',
  type: OutputType.INSTANCE,
  mode: OutputMode.SYNC,
  srctype: 'C',
  exports: {
    add: (a: number, b: number) => 0
  },
  code: `
    int add(int a, int b) {
      return a + b;
    }`
});
const adderInstance = getAdderInstance();

// use the wasm instance:
console.log(adderInstance.exports.add(23, 42));
```

Before we can actually use the JS script generated by the TS compiler, we need to
run another build step to also compile the wasm source code:
```bash
# compile wasm source, may pull SDKs (depends on config settings)
inwasm lib/xy.wasm.js
# finally run the module, should output 65
node lib/xy.wasm.js
```
It is important to note, that the compile script will overwrite the original JS script (inplace rewrite).
This is normally not an issue for TS projects (already has separated source files).
If directly applied to JS sources, it will "destroy" your source files - currently you have to do
a manual copy step with all nasty consequences for module imports and such. The inplace rewrite may be lifted
in the future once the story, how to deal with plain JS sources, got sorted out.

The final JS file can be further processed or distributed as any other JS script.


### How does it work internally?

The compile script `inwasm` loads the JS files as normal module (currently no ESM support!)
and catches errors thown by `InWasm` to get a hold of the callstack and the source code positions.
Next it evaluates the provided wasm definition (thats the literal object argument of `InWasm`)
and calls the compiler backend for the `srctype` with the content of `code`.
If the compilation succeeds, the generated wasm binary gets base64 encoded and wrapped
into a runtime definition, which finally replaces the original wasm definition.

At runtime `InWasm` returns a getter for the requested output type and mode. The getter approach helps
to lift the initialization burden of the wasm types from the loading phase (lazy eval).
Furthermore the getter memorizes decoded bytes and wasm modules (act as static singletons).


### What is `InWasm` good for?

**TL;DR:** Tiny standalone wasm helpers down to just one exposed function. Anything bigger - probably not,
unless you love to write tons of glue code yourself.

The main purpose of the library is to provide an easy way for embedding small standalone wasm helpers.
Note that beside the wasm bootstrapping there is no glue code provided, thus it cannot replace more
complex solutions with additional JS bridging output (like emscripten with its runtime extensions,
wasm-bindgen etc).

Due to missing glue code, things are very bare metal and you have to provide access and type wrappers
yourself (e.g. directly reading from wasm memory segments). This "embedded programming style" may sound
like a big drawback, but in fact TS/JS is often fast enough to get most things done in a timely fashion,
and just lacks proper performance at a certain point of some data conversion.
Thats where `InWasm` can provide a much faster drop-in alternative to a pure TS/JS implementation,
normally with much smaller package footprint than a bigger wasm integration. `InWasm` itself is at
~800 Bytes bundled, plus base64 string data of the generated binaries. Of course this further depends on
the amount of your own glue code.

Sidenote: Technically it would be possible to create a similar embedding experience for
bigger wasm integrations. Well, that is questionable for several reasons, e.g. wasm files
will grow really big, and inlining those in JS with base64 is a bad idea. Furthermore it would need
proper interfacing of the additional JS glue code, which is definitely out of scope for this library.

For bigger wasm integrations you are better served with the high level interfaces offered by emscripten
and/or rust with wasm-bindgen.


### Supported Types and Modes

Source Types (`srctype`):
- `'C'` - emscripten C, compiled as standalone wasm
- `'C++'` - emscripten C++, compiled as standalone wasm
- `'Clang-C'` - using clang from emscripten SDK
- `'Clang-C++'` - using clang from emscripten SDK
- `'Zig'` - preinstalled or autoinstall, compiled as freestanding
- `'wat'` - compiled with wat2wasm
- `'Rust'` - must be preinstalled currently with `cargo` in PATH
- `'custom'` - any custom build script

... TODO: document srctype extensions on wasm definitions ...


Output Types (`type`):
- `BYTES` - Uint8Array (raw wasm bytes), typed as `IWasmBytes<T extends IWasmDefinition>`
- `MODULE` - WebAssembly.Module, typed as `IWasmModule<T extends IWasmDefinition>`
- `INSTANCE` - WebAssembly.Instance, typed as `IWasmInstance<T extends IWasmDefinition>`

Output Modes (`mode`):
- `SYNC` - Getter bootstraps output type synchronously and returns it directly.
- `ASYNC` - Getter bootstraps output type with asynchronous interfaces and returns a promise
  resolving to the output type (e.g. `Promise<IWasmInstance<T>>`).

Note that SYNC output mode works only reliable in NodeJS and a web worker context,
as browsers may restrict synchronous wasm module or instance creation in the main context.


### Types & Type Inference

... WIP, types & inference patterns are still subject to change ...


### `InWasm` Coding Restrictions

Due to the way the compile script `inwasm` works by partial execution and a macro like source code replacement,
there are some coding restrictions:
- In general modules containing `InWasm` calls should be endpoints in the module dependency tree
  (not containing complicated imports itself, no cycling imports). They may not import other modules
  containing `InWasm` calls, or compilation will fail.
- All `InWasm` occurences must execute on import of the JS file. This can easily be achieved by
  always declaring them on top level respectively always reachable from there (also allowed in nested
  non-branching structures). Declaring them behind conditional branching will not work,
  if not all branches containing `InWasm` calls are guaranteed to execute from import.
- `InWasm` may not be indirected, it must be called for every given wasm definition as `InWasm({...})`.
  Doing the following will **not** work: `const a = (def) => InWasm(def); a(def1); a(def2);`.
- The wasm definition argument must be provided as a real object literal `{...}` to `InWasm`,
  e.g. `InWasm({... definition details go here ...})`. Using an object identifier like
  `const def = {...}; InWasm(def);` will not work. While technically not needed, this is currently
  enforced by the compile script to keep proper TS type inference working.


### Config Options

... WIP, more config settings yet to come, env overrides still partially broken ...

With a file `inwasm.config.js` in your project root you can configure some settings of `inwasm`:

```javascript
// default - autoinstall zig and emsdk
// (no config file needed, if you are good to go with these)
module.exports = {
  zig: {
    version: 'master',      // pulled sdk version
    store: 'project'        // where to store: 'project' or 'inwasm' folder
  },
  emsdk: {
    version: 'latest',
    store: 'project'
  },
};
// alternatively with custom paths to preinstalled sdks
module.exports = {
  zig: {
    binary: '/path/to/zig'  // zig binary path (use '$PATH' or 'zig' if in PATH)
  },
  emsdk: {
    path: '/path/to/emsdk'  // emsdk installed elsewhere
  },
};
```
For on-the-fly config overrides it is possible to use env variables,
where the name of the env variable is derived from the config object keys:
```bash
# config.zig.version --> INWASM_ZIG_VERSION
# config.emsdk.path  --> INWASM_EMSDK_PATH
# and so forth ...

# example: set custom zig binary override
INWASM_ZIG_BINARY=/path/to/zig inwasm lib/*wasm.js
```


### Development

The source repo contains two node package folders:
- `/inwasm` - inwasm package with cli tool and definitions
- `/testproject` - main test package for different compiler runners/SDKs

Since `/testproject` depends on `/inwasm`, initialize in this order:
```bash
git clone https://github.com/jerch/inwasm.git

# setup inwasm first
cd inwasm/inwasm
npm install

# then the testproject
cd ../testproject
npm install
```

### TODO

- ESM support
- better config, option to write to different file
- individual runner config options with proper TS typing
- better docs
- tests, tests, tests
- windows support


### State

Early alpha, use at your own risk. Currently working on Linux,
prolly also on macos (did just a preliminary test). No windows support yet.
