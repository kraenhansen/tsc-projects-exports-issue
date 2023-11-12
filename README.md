# Resolving a "subpath" export 

Building a cross platform package, I want to leverage TypeScript to:
- write runtime / platform independent code (located in `./src/common`).
- multiple runtime specific entry-points, being conditionally exported (see `./src/browser` as an example).

## Objective #1 (use `noResolve` and `types` to lock down types)

One important technique I use to achieve the above, is enabling the `noResolve` compiler option, to avoid platform specific globals (such as those declared by `@types/node`) being automatically included in the project. To explicitly allow the types of individual dependencies I use the `types` option to enumerate the allowed types.

## Objective #2 (use subpath export to reference "common" code)

From each of the runtime specific entry-points I want to import and re-export the "common" platform independent code.
A package [subpath export](https://nodejs.org/api/packages.html#subpath-exports) seems ideal to expose an entrypoint for this common code, in a way that makes it importable from an external dependent package as well as when importing internally from the runtime specific files.

## Objective #3 (use TypeScript Project References)

I use the [TypeScript Project References](https://www.typescriptlang.org/docs/handbook/project-references.html) in an attempt to optimize build times and compile the project using a single invocation of the TypeScript compiler. I've configured this using a root `tsconfig.json` which doesn't `include` anything, but `references` to the individual runtime specific projects, which in turn all have `references` to the `common` project.

As a side-note: When investigating this, I've also tried hoisting the "common" project to the root project to avoid multiple projects referencing the same common project, but it doesn't affect the issue I'm experiencing.

## Issue #1 (Cannot find type definition file for 'my-lib/common')

Follow these steps to reproduce the issue I'm experiencing:
- `npm install`
- `npm run build` (which runs `tsc --build` in the root project)

This yields the following error:

```
error TS2688: Cannot find type definition file for 'my-lib/common'.
  The file is in the program because:
    Entry point of type library 'my-lib/common' specified in compilerOptions
```

Running the same command with `--traceResolution` yields:

```
======== Resolving module 'my-lib/common' from '/Users/kraen.hansen/Projects/tsc-projects-exports-issue/src/browser/index.ts'. ========
Explicitly specified module resolution kind: 'NodeNext'.
Resolving in CJS mode with conditions 'require', 'types', 'node'.
File '/Users/kraen.hansen/Projects/tsc-projects-exports-issue/src/browser/package.json' does not exist according to earlier cached lookups.
File '/Users/kraen.hansen/Projects/tsc-projects-exports-issue/src/package.json' does not exist according to earlier cached lookups.
File '/Users/kraen.hansen/Projects/tsc-projects-exports-issue/package.json' exists according to earlier cached lookups.
Entering conditional exports.
Matched 'exports' condition 'types'.
Using 'exports' subpath './common' with target './dist/common/index.d.ts'.
File '/Users/kraen.hansen/Projects/tsc-projects-exports-issue/src/common/index.ts' exists - use it as a name resolution result.
Resolved under condition 'types'.
Exiting conditional exports.
Resolving real path for '/Users/kraen.hansen/Projects/tsc-projects-exports-issue/src/common/index.ts', result '/Users/kraen.hansen/Projects/tsc-projects-exports-issue/src/common/index.ts'.
======== Module name 'my-lib/common' was successfully resolved to '/Users/kraen.hansen/Projects/tsc-projects-exports-issue/src/common/index.ts' with Package ID 'my-lib/src/common/index.ts@0.1.0'. ========
======== Resolving type reference directive 'my-lib/common', containing file '/Users/kraen.hansen/Projects/tsc-projects-exports-issue/src/browser/__inferred type names__.ts', root directory '/Users/kraen.hansen/Projects/tsc-projects-exports-issue/src/browser/node_modules/@types,/Users/kraen.hansen/Projects/tsc-projects-exports-issue/src/node_modules/@types,/Users/kraen.hansen/Projects/tsc-projects-exports-issue/node_modules/@types,/Users/kraen.hansen/Projects/node_modules/@types,/Users/kraen.hansen/node_modules/@types,/Users/node_modules/@types,/node_modules/@types'. ========
Resolving with primary search path '/Users/kraen.hansen/Projects/tsc-projects-exports-issue/src/browser/node_modules/@types, /Users/kraen.hansen/Projects/tsc-projects-exports-issue/src/node_modules/@types, /Users/kraen.hansen/Projects/tsc-projects-exports-issue/node_modules/@types, /Users/kraen.hansen/Projects/node_modules/@types, /Users/kraen.hansen/node_modules/@types, /Users/node_modules/@types, /node_modules/@types'.
Directory '/Users/kraen.hansen/Projects/tsc-projects-exports-issue/src/browser/node_modules/@types' does not exist, skipping all lookups in it.
Directory '/Users/kraen.hansen/Projects/tsc-projects-exports-issue/src/node_modules/@types' does not exist, skipping all lookups in it.
Directory '/Users/kraen.hansen/Projects/tsc-projects-exports-issue/node_modules/@types' does not exist, skipping all lookups in it.
Directory '/Users/kraen.hansen/Projects/node_modules/@types' does not exist, skipping all lookups in it.
Directory '/Users/kraen.hansen/node_modules/@types' does not exist, skipping all lookups in it.
Directory '/Users/node_modules/@types' does not exist, skipping all lookups in it.
Directory '/node_modules/@types' does not exist, skipping all lookups in it.
Looking up in 'node_modules' folder, initial location '/Users/kraen.hansen/Projects/tsc-projects-exports-issue/src/browser'.
Searching all ancestor node_modules directories for preferred extensions: Declaration.
Directory '/Users/kraen.hansen/Projects/tsc-projects-exports-issue/src/browser/node_modules' does not exist, skipping all lookups in it.
Directory '/Users/kraen.hansen/Projects/tsc-projects-exports-issue/src/node_modules' does not exist, skipping all lookups in it.
Directory '/Users/kraen.hansen/Projects/tsc-projects-exports-issue/node_modules/@types' does not exist, skipping all lookups in it.
Directory '/Users/kraen.hansen/Projects/node_modules' does not exist, skipping all lookups in it.
Directory '/Users/kraen.hansen/node_modules' does not exist, skipping all lookups in it.
Directory '/Users/node_modules' does not exist, skipping all lookups in it.
Directory '/node_modules' does not exist, skipping all lookups in it.
======== Type reference directive 'my-lib/common' was not resolved. ========
```

I interpret the above as follows: The `import` in `./src/browser/index.ts` resolves `my-lib/common` as correctly, but the `"types": ["my-lib/common"]` in `./src/browser/tsconfig.json` was not resolved.

## Workaround #1

To verify the interpretation, I removed the `types` property and add `"noResolve": false` in `./src/browser/tsconfig.json`, which builds the project successfully! ... but that violates objective #1 😿

## Workaround #2

As an attempt to help the compiler locate the types for `my-lib/common`, I've tried creating a `./typings` directory in the root, adding it to `typeRoots` of the root project's `./tsconfig.json` and creating a symbolic link from `./typings/my-lib` to the root. This yields the following resolution trace:

```
======== Resolving type reference directive 'my-lib/common', containing file '/Users/kraen.hansen/Projects/tsc-projects-exports-issue/src/browser/__inferred type names__.ts', root directory '/Users/kraen.hansen/Projects/tsc-projects-exports-issue/typings'. ========
Resolving with primary search path '/Users/kraen.hansen/Projects/tsc-projects-exports-issue/typings'.
File '/Users/kraen.hansen/Projects/tsc-projects-exports-issue/typings/my-lib/common.d.ts' does not exist.
Resolving type reference directive for program that specifies custom typeRoots, skipping lookup in 'node_modules' folder.
======== Type reference directive 'my-lib/common' was not resolved. ========
```

I interpret this as the subpath export resolution isn't considered for this, as the compiler is looking for `my-lib/common.d.ts` instead of following the `types` entry of the `./common` subpath declared in `my-lib`'s `package.json`.

This is in itself very unfortunate. But playing along in the hopes to get this resolved, I tried creating a `./common.d.ts` in the root, re-exporting `./dist/common`, which ultimately yields:

```
======== Resolving module 'my-lib/common' from '/Users/kraen.hansen/Projects/tsc-projects-exports-issue/src/browser/index.ts'. ========
Explicitly specified module resolution kind: 'NodeNext'.
Resolving in CJS mode with conditions 'require', 'types', 'node'.
File '/Users/kraen.hansen/Projects/tsc-projects-exports-issue/src/browser/package.json' does not exist according to earlier cached lookups.
File '/Users/kraen.hansen/Projects/tsc-projects-exports-issue/src/package.json' does not exist according to earlier cached lookups.
File '/Users/kraen.hansen/Projects/tsc-projects-exports-issue/package.json' exists according to earlier cached lookups.
Entering conditional exports.
Matched 'exports' condition 'types'.
Using 'exports' subpath './common' with target './dist/common/index.d.ts'.
File '/Users/kraen.hansen/Projects/tsc-projects-exports-issue/src/common/index.ts' exists - use it as a name resolution result.
Resolved under condition 'types'.
Exiting conditional exports.
Resolving real path for '/Users/kraen.hansen/Projects/tsc-projects-exports-issue/src/common/index.ts', result '/Users/kraen.hansen/Projects/tsc-projects-exports-issue/src/common/index.ts'.
======== Module name 'my-lib/common' was successfully resolved to '/Users/kraen.hansen/Projects/tsc-projects-exports-issue/src/common/index.ts' with Package ID 'my-lib/src/common/index.ts@0.1.0'. ========
======== Resolving type reference directive 'my-lib/common', containing file '/Users/kraen.hansen/Projects/tsc-projects-exports-issue/src/browser/__inferred type names__.ts', root directory '/Users/kraen.hansen/Projects/tsc-projects-exports-issue/typings'. ========
Resolving with primary search path '/Users/kraen.hansen/Projects/tsc-projects-exports-issue/typings'.
File '/Users/kraen.hansen/Projects/tsc-projects-exports-issue/typings/my-lib/common.d.ts' exists - use it as a name resolution result.
Resolving real path for '/Users/kraen.hansen/Projects/tsc-projects-exports-issue/typings/my-lib/common.d.ts', result '/Users/kraen.hansen/Projects/tsc-projects-exports-issue/common.d.ts'.
======== Type reference directive 'my-lib/common' was successfully resolved to '/Users/kraen.hansen/Projects/tsc-projects-exports-issue/common.d.ts', primary: true. ========
```

While this seem like a valid workaround for the type issue, it still yields the following build error:

```
src/browser/index.ts:1:26 - error TS6305: Output file '/Users/kraen.hansen/Projects/tsc-projects-exports-issue/dist/common/index.d.ts' has not been built from source file '/Users/kraen.hansen/Projects/tsc-projects-exports-issue/src/common/index.ts'.
```

I tried deleting the `./dist` directory in an attempt to prune stale `.tsbuildinfo` artifacts, but the error persists.
