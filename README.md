# Bun Tricks

A collection of useful Bun tricks

## Avoid TS global type incompatibilities

Bun recommends installing `@types/bun` on [their TypeScript docs page](https://bun.sh/docs/typescript). For Bun-only projects, this makes sense, and is a low-configuration way to add types for `Bun` globals and imports of modules with `bun:` prefixes.

However, some projects may include files with multiple runtime environments - for example, most files using Node.js and some files using Bun. In this case, the types of globals can lead to type incompatibilities, with errors such as the following when attempting to use `setTimeout()` in Node.js code:

```
error TS2739: Type 'Timer' is missing the following properties from type 'Timeout': refresh, [Symbol.dispose]
```

To avoid these type incompatibilities, uninstall [`@types/bun`](https://www.npmjs.com/package/@types/bun) and install [`bun-types`](https://www.npmjs.com/package/bun-types):

```bash
npm remove @types/bun
npm add --save-dev bun-types
```

And add a triple-slash directive referencing `bun-types` at the top of each file which should use the Bun environment:

`index.test.ts`

```ts
/// <reference types="bun-types" />

import { expect, test } from 'bun:test';

test('2 + 2', () => {
  expect(2 + 2).toBe(4);
});
```

## `bun test`: Smoke test for server start

Bun's [built-in test runner](https://bun.sh/docs/cli/test) can be used to create zero-dependency tests written in TypeScript - not only unit tests, but also integration / end to end tests for command line interface programs. This is useful for creating smoke tests for programs such as dev servers, testing that the stdout and stderr match a snapshot:

`__tests__/smokeTestDevServer.test.ts`

```ts
/// <reference types="bun-types" />

import { setTimeout } from 'node:timers/promises';
import { $, spawn } from 'bun';
import { expect, test } from 'bun:test';

test(
  'Dev server smoke test',
  async () => {
    console.log('Starting dev server...');
    const devServer = spawn('pnpm dev'.split(' '), {
      env: {
        ...process.env,
        // TODO: Add any environment variables you need
      },
      stdout: 'pipe',
      stderr: 'pipe',
    });

    console.log('Starting read of stdout and stderr...');
    const stdoutPromise = new Response(devServer.stdout).text();
    const stderrPromise = new Response(devServer.stderr).text();

    console.log('Waiting for 8 seconds for server to finish booting...');
    await setTimeout(8000);

    console.log('Killing child process...');
    devServer.kill();

    // Kill all other Node.js processes started by spawning `pnpm dev`,
    // needed only for GitHub Actions workflows test runs
    // https://github.com/oven-sh/bun/issues/11892#issuecomment-2170104825
    if (process.env.GITHUB_ACTIONS) {
      try {
        await $`killall node`;
      } catch {
        // Swallow error
      }
    }

    console.log('Waiting for process to exit...');
    await devServer.exited;

    console.log('Waiting for read of stdout and stderr to complete...');
    const [stdout, stderr] = await Promise.all([stdoutPromise, stderrPromise]);

    console.log('Expecting stdout and stderr to match snapshots...');
    expect({ stdout, stderr }).toMatchSnapshot();
  },
  { timeout: 9000 },
);
```

This can also be run automatically on every push using GitHub Actions:

`.github/workflows/ci.yml`

```yml
name: CI
on: [push]
jobs:
  ci:
    name: CI
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: oven-sh/setup-bun@v1
        with:
          bun-version: latest
      - name: Checkout
        uses: actions/checkout@v4
      - run: bun install
      - name: Smoke test for dev server
        run: bun test __tests__/smokeTestDevServer.test.ts
        timeout-minutes: 1
```
