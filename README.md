# Bun Tricks

A collection of useful Bun tricks

## Avoid TS global type incompatibilities

Bun recommends installing `@types/bun` on [their TypeScript docs page](https://bun.sh/docs/typescript). For Bun-only projects, this makes sense, and is a low-configuration way to add types for `Bun` globals and imports of modules with `bun:` prefixes.

However, some projects may include files with multiple runtime environments - for example, most files using Node.js and some files using Bun. In this case, the types of globals can lead to type incompatibilities, with errors such as the following when attempting to use `setTimeout()` in Node.js code:

```
error TS2739: Type 'Timer' is missing the following properties from type 'Timeout': refresh, [Symbol.dispose]
```

To avoid these type incompatibilities, uninstall `@types/bun` and install `bun-types`:

```bash
npm remove @types/bun
npm add --save-dev bun-types
```

And add an import of `bun-types` at the top of each file which should use the Bun environment:

`index.test.ts`

```ts
import 'bun-types';
import { expect, test } from 'bun:test';

test('2 + 2', () => {
  expect(2 + 2).toBe(4);
});
```

Alternative to the `import`, using a triple-slash directive:

```ts
/// <reference types="bun-types" />

import { expect, test } from 'bun:test';

test('2 + 2', () => {
  expect(2 + 2).toBe(4);
});
```
