## `bun init`

Scaffold an empty project with the interactive `bun init` command.

```bash
$ bun init
bun init helps you get started with a minimal project and tries to
guess sensible defaults. Press ^C anytime to quit.

package name (quickstart):
entry point (index.ts):

Done! A package.json file was saved in the current directory.
 + index.ts
 + .gitignore
 + tsconfig.json (for editor auto-complete)
 + README.md

To get started, run:
  bun run index.ts
```

Press `enter` to accept the default answer for each prompt, or pass the `-y` flag to auto-accept the defaults.

## `bun create`

Template a new Bun project with `bun create`.

```bash
$ bun create <template> <destination>
```

{% callout %}
**Note** — You don’t need `bun create` to use Bun. You don’t need any configuration at all. This command exists to make getting started a bit quicker and easier.
{% /callout %}

A template can take a number of forms:

```bash
$ bun create <template>         # an official template (remote)
$ bun create <username>/<repo>  # a GitHub repo (remote)
$ bun create <local-template>   # a custom template (local)
```

Running `bun create` performs the following steps:

- Download the template (remote templates only)
- Copy all template files into the destination folder. By default Bun will _not overwrite_ any existing files. Use the `--force` flag to overwrite existing files.
- Install dependencies with `bun install`.
- Initialize a fresh Git repo. Opt out with the `--no-git` flag.
- Run the template's configured `start` script, if defined.

### Official templates

The following official templates are available.

```bash
bun create next ./myapp
bun create react ./myapp
bun create svelte-kit ./myapp
bun create elysia ./myapp
bun create hono ./myapp
bun create kingworld ./myapp
```

Each of these corresponds to a directory in the [bun-community/create-templates](https://github.com/bun-community/create-templates) repo. If you think a major framework is missing, please open a PR there. This list will change over time as additional examples are added. To see an up-to-date list, run `bun create` with no arguments.

```bash
$ bun create
Welcome to bun! Create a new project by pasting any of the following:
  <list of templates>
```

{% callout %}
⚡️ **Speed** — At the time of writing, `bun create react app` runs ~11x faster on a M1 Macbook Pro than `yarn create react-app app`.
{% /callout %}

### GitHub repos

A template of the form `<username>/<repo>` will be downloaded from GitHub.

```bash
$ bun create ahfarmer/calculator ./myapp
```

Complete GitHub URLs will also work:

```bash
$ bun create github.com/ahfarmer/calculator ./myapp
$ bun create https://github.com/ahfarmer/calculator ./myapp
```

Bun installs the files as they currently exist current default branch (usually `main` or `master`). Unlike `git clone` it doesn't download the commit history or configure a remote.

### Local templates

{% callout %}
**⚠️ Warning** — Unlike remote templates, running `bun create` with a local template will delete the entire destination folder if it already exists! Be careful.
{% /callout %}
Bun's templater can be extended to support custom templates defined on your local file system. These templates should live in one of the following directories:

- `$HOME/.bun-create/<name>`: global templates
- `<project root>/.bun-create/<name>`: project-specific templates

{% callout %}
**Note** — You can customize the global template path by setting the `BUN_CREATE_DIR` environment variable.
{% /callout %}

To create a local template, navigate to `$HOME/.bun-create` and create a new directory with the desired name of your template.

```bash
$ cd $HOME/.bun-create
$ mkdir foo
$ cd foo
```

Then, create a `package.json` file in that directory with the following contents:

```json
{
  "name": "foo"
}
```

You can run `bun create foo` elsewhere on your file system to verify that Bun is correctly finding your local template.

{% table %}

---

- `postinstall`
- runs after installing dependencies

---

- `preinstall`
- runs before installing dependencies

<!-- ---

- `start`
- a command to auto-start the application -->

{% /table %}

Each of these can correspond to a string or array of strings. An array of commands will be executed in order. Here is an example:

```json
{
  "name": "@bun-examples/simplereact",
  "version": "0.0.1",
  "main": "index.js",
  "dependencies": {
    "react": "^17.0.2",
    "react-dom": "^17.0.2"
  },
  "bun-create": {
    "preinstall": "echo 'Installing...'", // a single command
    "postinstall": ["echo 'Done!'"], // an array of commands
    "start": "bun run echo 'Hello world!'"
  }
}
```

When cloning a template, `bun create` will automatically remove the `"bun-create"` section from `package.json` before writing it to the destination folder.

### Reference

#### CLI flags

{% table %}

- Flag
- Description

---

- `--force`
- Overwrite existing files

---

- `--no-install`
- Skip installing `node_modules` & tasks

---

- `--no-git`
- Don’t initialize a git repository

---

- `--open`
- Start & open in-browser after finish

{% /table %}

#### Environment variables

{% table %}

- Name
- Description

---

- `GITHUB_API_DOMAIN`
- If you’re using a GitHub enterprise or a proxy, you can customize the GitHub domain Bun pings for downloads

---

- `GITHUB_API_TOKEN`
- This lets `bun create` work with private repositories or if you get rate-limited

{% /table %}

{% details summary="How `bun create` works" %}

When you run `bun create ${template} ${destination}`, here’s what happens:

IF remote template

1. GET `registry.npmjs.org/@bun-examples/${template}/latest` and parse it
2. GET `registry.npmjs.org/@bun-examples/${template}/-/${template}-${latestVersion}.tgz`
3. Decompress & extract `${template}-${latestVersion}.tgz` into `${destination}`

   - If there are files that would overwrite, warn and exit unless `--force` is passed

IF GitHub repo

1. Download the tarball from GitHub’s API
2. Decompress & extract into `${destination}`

   - If there are files that would overwrite, warn and exit unless `--force` is passed

ELSE IF local template

1. Open local template folder
2. Delete destination directory recursively
3. Copy files recursively using the fastest system calls available (on macOS `fcopyfile` and Linux, `copy_file_range`). Do not copy or traverse into `node_modules` folder if exists (this alone makes it faster than `cp`)

4. Parse the `package.json` (again!), update `name` to be `${basename(destination)}`, remove the `bun-create` section from the `package.json` and save the updated `package.json` to disk.
   - IF Next.js is detected, add `bun-framework-next` to the list of dependencies
   - IF Create React App is detected, add the entry point in /src/index.{js,jsx,ts,tsx} to `public/index.html`
   - IF Relay is detected, add `bun-macro-relay` so that Relay works
5. Auto-detect the npm client, preferring `pnpm`, `yarn` (v1), and lastly `npm`
6. Run any tasks defined in `"bun-create": { "preinstall" }` with the npm client
7. Run `${npmClient} install` unless `--no-install` is passed OR no dependencies are in package.json
8. Run any tasks defined in `"bun-create": { "preinstall" }` with the npm client
9. Run `git init; git add -A .; git commit -am "Initial Commit";`

   - Rename `gitignore` to `.gitignore`. NPM automatically removes `.gitignore` files from appearing in packages.
   - If there are dependencies, this runs in a separate thread concurrently while node_modules are being installed
   - Using libgit2 if available was tested and performed 3x slower in microbenchmarks

{% /details %}


## Troubleshooting

### Bun not running on an M1 (or Apple Silicon)

If you see a message like this

> [1] 28447 killed bun create next ./test

It most likely means you’re running Bun’s x64 version on Apple Silicon. This happens if Bun is running via Rosetta. Rosetta is unable to emulate AVX2 instructions, which Bun indirectly uses.

The fix is to ensure you installed a version of Bun built for Apple Silicon.

### error: Unexpected

If you see an error like this:

![image](https://user-images.githubusercontent.com/709451/141210854-89434678-d21b-42f4-b65a-7df3b785f7b9.png)

It usually means the max number of open file descriptors is being explicitly set to a low number. By default, Bun requests the max number of file descriptors available (which on macOS, is something like 32,000). But, if you previously ran into ulimit issues with, e.g., Chokidar, someone on The Internet may have advised you to run `ulimit -n 8096`.

That advice unfortunately **lowers** the hard limit to `8096`. This can be a problem in large repositories or projects with lots of dependencies. Chokidar (and other watchers) don’t seem to call `setrlimit`, which means they’re reliant on the (much lower) soft limit.

To fix this issue:

1. Remove any scripts that call `ulimit -n` and restart your shell.
2. Try again, and if the error still occurs, try setting `ulimit -n` to an absurdly high number, such as `ulimit -n 2147483646`
3. Try again, and if that still doesn’t fix it, open an issue

### Unzip is required

Unzip is required to install Bun on Linux. You can use one of the following commands to install `unzip`:

#### Debian / Ubuntu / Mint

```sh
$ sudo apt install unzip
```

#### RedHat / CentOS / Fedora

```sh
$ sudo dnf install unzip
```

#### Arch / Manjaro

```sh
$ sudo pacman -S unzip
```

#### OpenSUSE

```sh
$ sudo zypper install unzip
```

### bun install is stuck

Please run `bun install --verbose 2> logs.txt` and send them to me in Bun's discord. If you're on Linux, it would also be helpful if you run `sudo perf trace bun install --silent` and attach the logs.


Bun.js focuses on performance, developer experience, and compatibility with the JavaScript ecosystem.

## HTTP Requests

```ts
// http.ts
export default {
  port: 3000,
  fetch(request: Request) {
    return new Response("Hello World");
  },
};

// bun ./http.ts
```

| Requests per second                                                    | OS    | CPU                            | Bun version |
| ---------------------------------------------------------------------- | ----- | ------------------------------ | ----------- |
| [260,000](https://twitter.com/jarredsumner/status/1512040623200616449) | macOS | Apple Silicon M1 Max           | 0.0.76      |
| [160,000](https://twitter.com/jarredsumner/status/1511988933587976192) | Linux | AMD Ryzen 5 3600 6-Core 2.2ghz | 0.0.76      |

{% details summary="See benchmark details" %}
Measured with [`http_load_test`](https://github.com/uNetworking/uSockets/blob/master/examples/http_load_test.c) by running:

```bash
$ ./http_load_test  20 127.0.0.1 3000
```

{% /details %}

## File System

`cat` clone that runs [2x faster than GNU cat](https://twitter.com/jarredsumner/status/1511707890708586496) for large files on Linux

```js
// cat.js
import { resolve } from "path";
import { write, stdout, file, argv } from "bun";

const path = resolve(argv.at(-1));

await write(
  // stdout is a Blob
  stdout,
  // file(path) returns a Blob - https://developer.mozilla.org/en-US/docs/Web/API/Blob
  file(path),
);
```

Run this with `bun cat.js /path/to/big/file`.

## Reading from standard input

```ts
// As of Bun v0.3.0, console is an AsyncIterable
for await (const line of console) {
  // line of text from stdin
  console.log(line);
}
```

## React SSR

```js
import { renderToReadableStream } from "react-dom/server";

const dt = new Intl.DateTimeFormat();

export default {
  port: 3000,
  async fetch(request: Request) {
    return new Response(
      await renderToReadableStream(
        <html>
          <head>
            <title>Hello World</title>
          </head>
          <body>
            <h1>Hello from React!</h1>
            <p>The date is {dt.format(new Date())}</p>
          </body>
        </html>,
      ),
    );
  },
};
```

Write to stdout with `console.write`:

```js
// no trailing newline
// works with strings and typed arrays
console.write("Hello World!");
```

There are some more examples in the [examples](./examples) folder.

PRs adding more examples are very welcome!

## Fast paths for Web APIs

Bun.js has fast paths for common use cases that make Web APIs live up to the performance demands of servers and CLIs.

`Bun.file(path)` returns a [`Blob`](https://developer.mozilla.org/en-US/docs/Web/API/Blob) that represents a lazily-loaded file.

When you pass a file blob to `Bun.write`, Bun automatically uses a faster system call:

```js
const blob = Bun.file("input.txt");
await Bun.write("output.txt", blob);
```

On Linux, this uses the [`copy_file_range`](https://man7.org/linux/man-pages/man2/copy_file_range.2.html) syscall and on macOS, this becomes `clonefile` (or [`fcopyfile`](https://developer.apple.com/library/archive/documentation/System/Conceptual/ManPages_iPhoneOS/man3/copyfile.3.html)).

`Bun.write` also supports [`Response`](https://developer.mozilla.org/en-US/docs/Web/API/Response) objects. It automatically converts to a [`Blob`](https://developer.mozilla.org/en-US/docs/Web/API/Blob).

```js
// Eventually, this will stream the response to disk but today it buffers
await Bun.write("index.html", await fetch("https://example.com"));
```


Let's write a simple HTTP server using the built-in `Bun.serve` API. First, create a fresh directory.

```bash
$ mkdir quickstart
$ cd quickstart
```

Run `bun init` to scaffold a new project. It's an interactive tool; for this tutorial, just press `enter` to accept the default answer for each prompt.

```bash
$ bun init
bun init helps you get started with a minimal project and tries to
guess sensible defaults. Press ^C anytime to quit.

package name (quickstart):
entry point (index.ts):

Done! A package.json file was saved in the current directory.
 + index.ts
 + .gitignore
 + tsconfig.json (for editor auto-complete)
 + README.md

To get started, run:
  bun run index.ts
```

Since our entry point is a `*.ts` file, Bun generates a `tsconfig.json` for you. If you're using plain JavaScript, it will generate a [`jsconfig.json`](https://code.visualstudio.com/docs/languages/jsconfig) instead.

## Run a file

Open `index.ts` and paste the following code snippet, which implements a simple HTTP server with [`Bun.serve`](/docs/api/http).

```ts
const server = Bun.serve({
  port: 3000,
  fetch(req) {
    return new Response(`Bun!`);
  },
});

console.log(`Listening on http://localhost:${server.port}...`);
```

Run the file from your shell.

```bash
$ bun index.ts
Listening at http://localhost:3000...
```

Visit [http://localhost:3000](http://localhost:3000) to test the server. You should see a simple page that says "Bun!".

## Run a script

Bun can also execute `"scripts"` from your `package.json`. Add the following script:

```json-diff
  {
    "name": "quickstart",
    "module": "index.ts",
    "type": "module",
+   "scripts": {
+     "start": "bun run index.ts"
+   },
    "devDependencies": {
      "bun-types": "^0.4.0"
    }
  }
```

Then run it with `bun run start`.

```bash
$ bun run start
  $ bun run index.ts
  Listening on http://localhost:4000...
```

{% callout %}
⚡️ **Performance** — `bun run` is roughly 28x faster than `npm run` (6ms vs 170ms of overhead).
{% /callout %}

## Install a package

Let's make our server a little more interesting by installing a package. First install the `figlet` package and its type declarations. Figlet is a utility for converting strings into ASCII art.

```bash
$ bun add figlet
$ bun add -d @types/figlet # TypeScript users only
```

Update `index.ts` to use `figlet` in the `fetch` handler.

```ts-diff
+ import figlet from "figlet";

  const server = Bun.serve({
    fetch() {
+     const body = figlet.textSync('Bun!');
+     return new Response(body);
-     return new Response(`Bun!`);
    },
    port: 3000,
  });
```

Restart the server and refresh the page. You should see a new ASCII art banner.

```txt
  ____              _
 | __ ) _   _ _ __ | |
 |  _ \| | | | '_ \| |
 | |_) | |_| | | | |_|
 |____/ \__,_|_| |_(_)
```


Bun uses [a fork](https://github.com/oven-sh/WebKit) of WebKit with a small number of changes.

It's important to periodically update WebKit for many reasons:

- Security
- Performance
- Compatibility
- …and many more.

To upgrade, first find the commit in **Bun's WebKit fork** (not Bun!) between when we last upgraded and now.

```bash
$ cd src/bun.js/WebKit # In the WebKit directory! not bun
$ git checkout $COMMIT
```

This is the main command to run:

```bash
$ git pull https://github.com/WebKit/WebKit.git main --no-rebase --allow-unrelated-histories -X theirs
```

Then, you will likely see some silly merge conflicts. Fix them and then run:

```bash
# You might have to run this multiple times.
$ rm -rf WebKitBuild

# Go to Bun's directory! Not WebKit.
cd ../../../../
make jsc-build-mac-compile
```

Make sure that JSC's CLI is able to load successfully. This verifies that the build is working.

You know this worked when it printed help options. If it complains about symbols, crashes, or anything else that looks wrong, something is wrong.

```bash
src/bun.js/WebKit/WebKitBuild/Release/bin/jsc --help
```

Then, clear out our bindings and regenerate the C++<>Zig headers:

```bash
make clean-bindings headers builtins
```

Now update Bun's bindings wherever there are compiler errors:

```bash
# It will take awhile if you don't pass -j here
make bindings -j10
```

This is the hard part. It might involve digging through WebKit's commit history to figure out what changed and why. Fortunately, WebKit contributors write great commit messages.


Bun is an all-in-one toolkit for JavaScript and TypeScript apps. It ships as a single executable called `bun​`.

At its core is the _Bun runtime_, a fast JavaScript runtime designed as a drop-in replacement for Node.js. It's written in Zig and powered by JavaScriptCore under the hood, dramatically reducing startup times and memory usage.

```bash
$ bun run index.tsx  # TS and JSX supported out of the box
```

​​The `bun​` command-line tool also implements a test runner, script runner, and Node.js-compatible package manager, all significantly faster than existing tools and usable in existing Node.js projects with little to no changes necessary.

```bash
$ bun run start                 # run the `start` script
$ bun install <pkg>​             # install a package
$ bun build ./index.tsx         # bundle a project for browsers
$ bun test                      # run tests
$ bunx cowsay "Hello, world!"   # execute a package
```

{% callout type="note" %}
**​​Bun is still under development.** Use it to speed up your development workflows or run simpler production code in resource-constrained environments like serverless functions. We're working on more complete Node.js compatibility and integration with existing frameworks. Join the [Discord](https://bun.sh/discord) and watch the [GitHub repository](https://github.com/oven-sh/bun) to keep tabs on future releases.
{% /callout %}

Get started with one of the quick links below, or read on to learn more about Bun.

{% block className="gap-2 grid grid-flow-row grid-cols-1 md:grid-cols-2" %}
{% arrowbutton href="/docs/installation" text="Install Bun" /%}
{% arrowbutton href="/docs/quickstart" text="Do the quickstart" /%}
{% arrowbutton href="/docs/cli/install" text="Install a package" /%}
{% arrowbutton href="/docs/templates" text="Use a project template" /%}
{% arrowbutton href="/docs/bundler" text="Bundle code for production" /%}
{% arrowbutton href="/docs/api/http" text="Build an HTTP server" /%}
{% arrowbutton href="/docs/api/websockets" text="Build a Websocket server" /%}
{% arrowbutton href="/docs/api/file-io" text="Read and write files" /%}
{% arrowbutton href="/docs/api/sqlite" text="Run SQLite queries" /%}
{% arrowbutton href="/docs/cli/test" text="Write and run tests" /%}
{% /block %}

## What is a runtime?

JavaScript (or, more formally, ECMAScript) is just a _specification_ for a programming language. Anyone can write a JavaScript _engine_ that ingests a valid JavaScript program and executes it. The two most popular engines in use today are V8 (developed by Google) and JavaScriptCore (developed by Apple). Both are open source.

### Browsers

But most JavaScript programs don't run in a vacuum. They need a way to access the outside world to perform useful tasks. This is where _runtimes_ come in. They implement additional APIs that are then made available to the JavaScript programs they execute. Notably, browsers ship with JavaScript runtimes that implement a set of Web-specific APIs that are exposed via the global `window` object. Any JavaScript code executed by the browser can use these APIs to implement interactive or dynamic behavior in the context of the current webpage.

<!-- JavaScript runtime that exposes  JavaScript engines are designed to run "vanilla" JavaScript programs, but it's often JavaScript _runtimes_ use an engine internally to execute the code and implement additional APIs that are then made available to executed programs.
JavaScript was [initially designed](https://en.wikipedia.org/wiki/JavaScript) as a language to run in web browsers to implement interactivity and dynamic behavior in web pages. Browsers are the first JavaScript runtimes. JavaScript programs that are executed in browsers have access to a set of Web-specific global APIs on the `window` object. -->

### Node.js

Similarly, Node.js is a JavaScript runtime that can be used in non-browser environments, like servers. JavaScript programs executed by Node.js have access to a set of Node.js-specific [globals](https://nodejs.org/api/globals.html) like `Buffer`, `process`, and `__dirname` in addition to built-in modules for performing OS-level tasks like reading/writing files (`node:fs`) and networking (`node:net`, `node:http`). Node.js also implements a CommonJS-based module system and resolution algorithm that pre-dates JavaScript's native module system.

<!-- Bun.js prefers Web API compatibility instead of designing new APIs when possible. Bun.js also implements some Node.js APIs. -->

Bun is designed as a faster, leaner, more modern replacement for Node.js.

<!-- ## Why a new runtime?

Bun is designed as a faster, leaner, more modern replacement for Node.js. Node.js is burdened by ingrained performance issues, backwards compatibility concerns, and slow development velocity—inevitable issues for a project of its age and magnitude. -->

## Design goals

Bun is designed from the ground-up with today's JavaScript ecosystem in mind.

- **Speed**. Bun processes start [4x faster than Node.js](https://twitter.com/jarredsumner/status/1499225725492076544) currently (try it yourself!)
- **TypeScript & JSX support**. You can directly execute `.jsx`, `.ts`, and `.tsx` files; Bun's transpiler converts these to vanilla JavaScript before execution.
- **ESM & CommonJS compatibility**. The world is moving towards ES modules (ESM), but millions of packages on npm still require CommonJS. Bun recommends ES modules, but supports CommonJS.
- **Web-standard APIs**. Bun implements standard Web APIs like `fetch`, `WebSocket`, and `ReadableStream`. Bun is powered by the JavaScriptCore engine, which is developed by Apple for Safari, so some APIs like [`Headers`](https://developer.mozilla.org/en-US/docs/Web/API/Headers) and [`URL`](https://developer.mozilla.org/en-US/docs/Web/API/URL) directly use [Safari's implementation](https://github.com/oven-sh/bun/blob/HEAD/src/bun.js/bindings/webcore/JSFetchHeaders.cpp).
- **Node.js compatibility**. In addition to supporting Node-style module resolution, Bun aims for full compatibility with built-in Node.js globals (`process`, `Buffer`) and modules (`path`, `fs`, `http`, etc.) _This is an ongoing effort that is not complete._ Refer to the compatibility page for the current status.

Bun is more than a runtime. The long-term goal is to be a cohesive, infrastructural toolkit for building apps with JavaScript/TypeScript, including a package manager, transpiler, bundler, script runner, test runner, and more.

<!-- - tsconfig.json `"paths"` is natively supported, along with `"exports"` in package.json
- `fs`, `path`, and `process` from Node.js are partially implemented
- Web APIs like [`fetch`](https://developer.mozilla.org/en-US/docs/Web/API/fetch), [`Response`](https://developer.mozilla.org/en-US/docs/Web/API/Response), [`URL`](https://developer.mozilla.org/en-US/docs/Web/API/URL) and more are built-in
- [`HTMLRewriter`](https://developers.cloudflare.com/workers/runtime-apis/html-rewriter/) makes it easy to transform HTML in Bun.js
- `.env` files automatically load into `process.env` and `Bun.env`
- top level await -->


[TOML](https://toml.io/) is a minimal configuration file format designed to be easy for humans to read.

Bun implements a TOML parser with a few tweaks designed for better interoperability with INI files and with JavaScript.

### ; and # are comments

In Bun-flavored TOML, comments start with `#` or `;`

```ini
# This is a comment
; This is also a comment
```

This matches the behavior of INI files.

In TOML, comments start with `#`

```toml
# This is a comment
```

### String escape characters

Bun-flavored adds a few more escape sequences to TOML to work better with JavaScript strings.

```
# Bun-flavored TOML extras
\x{XX}     - ASCII           (U+00XX)
\u{x+}     - unicode         (U+0000000X) - (U+XXXXXXXX)
\v         - vertical tab

# Regular TOML
\b         - backspace       (U+0008)
\t         - tab             (U+0009)
\n         - linefeed        (U+000A)
\f         - form feed       (U+000C)
\r         - carriage return (U+000D)
\"         - quote           (U+0022)
\\         - backslash       (U+005C)
\uXXXX     - unicode         (U+XXXX)
\UXXXXXXXX - unicode         (U+XXXXXXXX)
```


Bun ships as a single executable that can be installed a few different ways.

{% callout %}
**Windows users** — Bun does not currently provide a native Windows build. We're working on this; progress can be tracked at [this issue](https://github.com/oven-sh/bun/issues/43). In the meantime, use one of the installation methods below for Windows Subsystem for Linux.

**Linux users** — The `unzip` package is required to install Bun. Kernel version 5.6 or higher is strongly recommended, but the minimum is 5.1.
{% /callout %}

{% codetabs %}

```bash#NPM
$ npm install -g bun # the last `npm` command you'll ever need
```

```bash#Native
$ curl -fsSL https://bun.sh/install | bash # for macOS, Linux, and WSL
```

```bash#Homebrew
$ brew tap oven-sh/bun # for macOS and Linux
$ brew install bun
```

```bash#Docker
$ docker pull oven/bun
$ docker run --rm --init --ulimit memlock=-1:-1 oven/bun
```

```bash#Proto
$ proto install bun
```

{% /codetabs %}

## Upgrading

Once installed, the binary can upgrade itself.

```sh
$ bun upgrade
```

{% callout %}
**Homebrew users** — To avoid conflicts with Homebrew, use `brew upgrade bun` instead.

**proto users** - Use `proto install bun --pin` instead.
{% /callout %}

Bun automatically releases an (untested) canary build on every commit to `main`. To upgrade to the latest canary build:

```sh
$ bun upgrade --canary
```

[View canary build](https://github.com/oven-sh/bun/releases/tag/canary)

<!--
## Native

Works on macOS x64 & Silicon, Linux x64, Windows Subsystem for Linux.

```sh
$ curl -fsSL https://bun.sh/install | bash
```

Once installed, the binary can upgrade itself.

```sh
$ bun upgrade
```

Bun automatically releases an (untested) canary build on every commit to `main`. To upgrade to the latest canary build:

```sh
$ bun upgrade --canary
```

## Homebrew

Works on macOS and Linux

```sh
$ brew tap oven-sh/bun
$ brew install bun
```

Homebrew recommends using `brew upgrade <package>` to install newer versions.

## Docker

Works on Linux x64

```sh
# this is a comment
$ docker pull oven/bun:edge
this is some output
$ docker run --rm --init --ulimit memlock=-1:-1 oven/bun:edge
$ docker run --rm --init --ulimit memlock=-1:-1 oven/bun:edge
this is some output
``` -->

## Uninstalling

Bun's binary and install cache is located in `~/.bun` by default. To uninstall bun, delete this directory and edit your shell config (`.bashrc`, `.zshrc`, or similar) to remove `~/.bun/bin` from the `$PATH` variable.

```sh
$ rm -rf ~/.bun # make sure to remove ~/.bun/bin from $PATH
```

## TypeScript

To install TypeScript definitions for Bun's built-in APIs in your project, install `bun-types`.

```sh
$ bun add -d bun-types # dev dependency
```

Then include `"bun-types"` in the `compilerOptions.types` in your `tsconfig.json`:

```json-diff
  {
    "compilerOptions": {
+     "types": ["bun-types"]
    }
  }
```

Refer to [Ecosystem > TypeScript](/docs/runtime/typescript) for a complete guide to TypeScript support in Bun.

## Completions

Shell auto-completion should be configured automatically when Bun is installed.

If not, run the following command. It uses `$SHELL` to determine which shell you're using and writes a completion file to the appropriate place on disk. It's automatically re-run on every `bun upgrade`.

```bash
$ bun completions
```

To write the completions to a custom location:

```bash
$ bun completions > path-to-file      # write to file
$ bun completions /path/to/directory  # write into directory
```


[Stric](https://github.com/bunsvr) is a minimalist, fast web framework for Bun.

```ts#index.ts
import { Router } from '@stricjs/router';

// Export the fetch handler and serve with Bun
export default new Router()
  // Return 'Hi' on every request
  .get('/', () => new Response('Hi'));
```

Stric provides support for [ArrowJS](https://www.arrow-js.com), a library for building reactive interfaces. 

{% codetabs %}

```ts#src/App.ts
import { html } from '@stricjs/arrow/utils';

// Code inside this function can use web APIs
export function render() {
  // Render a <p> element with text 'Hi'
  html`<p>Hi</p>`;
};

// Set the path to handle
export const path = '/';
```
```ts#index.ts
import { PageRouter } from '@stricjs/arrow';

// Create a page router, build and serve directly
new PageRouter().serve();
```

{% /codetabs %}

For more info, see Stric's [documentation](https://stricjs.gitbook.io/docs).


Projects that use Express and other major Node.js HTTP libraries should work out of the box.

{% callout %}
If you run into bugs, [please file an issue](https://bun.sh/issues) _in Bun's repo_, not the library. It is Bun's responsibility to address Node.js compatibility issues.
{% /callout %}

```ts
import express from "express";

const app = express();
const port = 8080;

app.get("/", (req, res) => {
  res.send("Hello World!");
});

app.listen(port, () => {
  console.log(`Listening on port ${port}...`);
});
```

Bun implements the [`node:http`](https://nodejs.org/api/http.html) and [`node:https`](https://nodejs.org/api/https.html) modules that these libraries rely on. These modules can also be used directly, though [`Bun.serve`](/docs/api/http) is recommended for most use cases.

{% callout %}
**Note** — Refer to the [Runtime > Node.js APIs](/docs/runtime/nodejs-apis#node_http) page for more detailed compatibility information.
{% /callout %}

```ts
import * as http from "node:http";

http
  .createServer(function (req, res) {
    res.write("Hello World!");
    res.end();
  })
  .listen(8080);
```


[Elysia](https://elysiajs.com) is a Bun-first performance focused web framework that takes full advantage of Bun's HTTP, file system, and hot reloading APIs.
Designed with TypeScript in mind, you don't need to understand TypeScript to gain the benefit of TypeScript with Elysia. The library understands what you want and automatically infers the type from your code.

:zap: Elysia is [one of the fastest Bun web frameworks](https://github.com/SaltyAom/bun-http-framework-benchmark)

```ts#server.ts
import { Elysia } from 'elysia'

const app = new Elysia()
	.get('/', () => 'Hello Elysia')
	.listen(8080)
	 
console.log(`🦊 Elysia is running at on port ${app.server.port}...`)
```

Get started with `bun create`.

```bash
$ bun create elysia ./myapp
$ cd myapp
$ bun run dev
```

Refer to the Elysia [documentation](https://elysiajs.com/quick-start.html) for more information.


Bun supports `.jsx` and `.tsx` files out of the box. Bun's internal transpiler converts JSX syntax into vanilla JavaScript before execution.

```tsx#react.tsx
function Component(props: {message: string}) {
  return (
    <body>
      <h1 style={{color: 'red'}}>{props.message}</h1>
    </body>
  );
}

console.log(<Component message="Hello world!" />);
```

Bun implements special logging for JSX to make debugging easier.

```bash
$ bun run react.tsx
<Component message="Hello world!" />
```

### Prop punning

The Bun runtime also supports "prop punning" for JSX. This is a shorthand syntax useful for assigning a variable to a prop with the same name.

```tsx
function Div(props: {className: string;}) {
  const {className} = props;

  // without punning
  return <div className={className} />;
  // with punning
  return <div {className} />;
}
```

### Server-side rendering

To server-side render (SSR) React in an [HTTP server](/docs/api/http):

```tsx#ssr.tsx
import {renderToReadableStream} from 'react-dom/server';

function Component(props: {message: string}) {
  return (
    <body>
      <h1 style={{color: 'red'}}>{props.message}</h1>
    </body>
  );
}

Bun.serve({
  port: 4000,
  async fetch() {
    const stream = await renderToReadableStream(
      <Component message="Hello from server!" />
    );
    return new Response(stream, {
      headers: {'Content-Type': 'text/html'},
    });
  },
});
```

React `18.3` and later includes an [SSR optimization](https://github.com/facebook/react/pull/25597) that takes advantage of Bun's "direct" `ReadableStream` implementation.


[Hono](https://github.com/honojs/hono) is a lightweight ultrafast web framework designed for the edge.

```ts
import { Hono } from 'hono'
const app = new Hono()

app.get('/', (c) => c.text('Hono!'))

export default app
```

Get started with `bun create` or follow Hono's [Bun quickstart](https://hono.dev/getting-started/bun).
```bash
$ bun create hono ./myapp
$ cd myapp
$ bun run start
```


[Buchta](https://buchtajs.com) is a fullstack framework designed to take full advantage of Bun's strengths. It currently supports Preact and Svelte.

To get started:

```bash
$ bunx buchta init myapp
Project templates: 
- svelte
- default
- preact
Name of template: preact  
Do you want TSX? y  
Do you want SSR? y
Enable livereload? y
Buchta Preact project was setup successfully!
$ cd myapp
$ bun install
$ bunx buchta serve
```

To implement a simple HTTP server with Buchta:

```ts#server.ts
import { Buchta, type BuchtaRequest, type BuchtaResponse } from "buchta";

const app = new Buchta();

app.get("/api/hello/", (req: BuchtaRequest, res: BuchtaResponse) => {
  res.send("Hello, World!");
});

app.run();
```


For more information, refer to Buchta's [documentation](https://buchtajs.com/docs/).

Bun supports [`workspaces`](https://docs.npmjs.com/cli/v9/using-npm/workspaces?v=true#description) in `package.json`. Workspaces make it easy to develop complex software as a _monorepo_ consisting of several independent packages.

To try it, specify a list of sub-packages in the `workspaces` field of your `package.json`; it's conventional to place these sub-packages in a directory called `packages`.

```json
{
  "name": "my-project",
  "version": "1.0.0",
  "workspaces": ["packages/*"]
}
```

{% callout %}
**Glob support** — Bun v0.5.8 added support for simple `<directory>/*` globs in `"workspaces"`. Full glob syntax (e.g. `**` and `?`) is not yet supported (soon!).
{% /callout %}

This has a couple major benefits.

- **Code can be split into logical parts.** If one package relies on another, you can simply add it as a dependency with `bun add`. If package `b` depends on `a`, `bun install` will symlink your local `packages/a` directory into the `node_modules` folder of `b`, instead of trying to download it from the npm registry.
- **Dependencies can be de-duplicated.** If `a` and `b` share a common dependency, it will be _hoisted_ to the root `node_modules` directory. This reduces redundant disk usage and minimizes "dependency hell" issues associated with having multiple versions of a package installed simultaneously.

{% callout %}
⚡️ **Speed** — Installs are fast, even for big monorepos. Bun installs the [Remix](https://github.com/remix-run/remix) monorepo in about `500ms` on Linux.

- 28x faster than `npm install`
- 12x faster than `yarn install` (v1)
- 8x faster than `pnpm install`

{% image src="https://user-images.githubusercontent.com/709451/212829600-77df9544-7c9f-4d8d-a984-b2cd0fd2aa52.png" /%}
{% /callout %}


The default registry is `registry.npmjs.org`. This can be globally configured in `bunfig.toml`:

```toml
[install]
# set default registry as a string
registry = "https://registry.npmjs.org"
# set a token
registry = { url = "https://registry.npmjs.org", token = "123456" }
# set a username/password
registry = "https://username:password@registry.npmjs.org"
```

To configure a private registry scoped to a particular organization:

```toml
[install.scopes]
# registry as string
"@myorg1" = "https://username:password@registry.myorg.com/"

# registry with username/password
# you can reference environment variables
"@myorg2" = { username = "myusername", password = "$NPM_PASS", url = "https://registry.myorg.com/" }

# registry with token
"@myorg3" = { token = "$npm_token", url = "https://registry.myorg.com/" }
```

### `.npmrc`

Bun does not currently read `.npmrc` files. For private registries, migrate your registry configuration to `bunfig.toml` as documented above.


The `bun pm` command group provides a set of utilities for working with Bun's package manager.

To print the path to the `bin` directory for the local project:

```bash
$ bun pm bin
/path/to/current/project/node_modules/.bin
```

To print the path to the global `bin` directory:

```bash
$ bun pm bin -g
<$HOME>/.bun/bin
```

To print a list of installed dependencies in the current project and their resolved versions, excluding their dependencies.

```bash
$ bun pm ls
/path/to/project node_modules (135)
├── eslint@8.38.0
├── react@18.2.0
├── react-dom@18.2.0
├── typescript@5.0.4
└── zod@3.21.4
```

To print all installed dependencies, including nth-order dependencies.

```bash
$ bun pm ls --all
/path/to/project node_modules (135)
├── @eslint-community/eslint-utils@4.4.0
├── @eslint-community/regexpp@4.5.0
├── @eslint/eslintrc@2.0.2
├── @eslint/js@8.38.0
├── @nodelib/fs.scandir@2.1.5
├── @nodelib/fs.stat@2.0.5
├── @nodelib/fs.walk@1.2.8
├── acorn@8.8.2
├── acorn-jsx@5.3.2
├── ajv@6.12.6
├── ansi-regex@5.0.1
├── ...
```

To print the path to Bun's global module cache:

```bash
$ bun pm cache
```

To clear Bun's global module cache:

```bash
$ bun pm cache rm
```


Running `bun install` will create a binary lockfile called `bun.lockb`.

#### Why is it binary?

In a word: Performance. Bun’s lockfile saves & loads incredibly quickly, and saves a lot more data than what is typically inside lockfiles.

#### How do I inspect it?

Run `bun install -y` to generate a Yarn-compatible `yarn.lock` (v1) that can be inspected more easily.

#### Platform-specific dependencies?

Bun stores normalized `cpu` and `os` values from npm in the lockfile, along with the resolved packages. It skips downloading, extracting, and installing packages disabled for the current target at runtime. This means the lockfile won’t change between platforms/architectures even if the packages ultimately installed do change.

#### What does the lockfile store?

Packages, metadata for those packages, the hoisted install order, dependencies for each package, what packages those dependencies resolved to, an integrity hash (if available), what each package was resolved to, and which version (or equivalent).

#### Why is it fast?

It uses linear arrays for all data. [Packages](https://github.com/oven-sh/bun/blob/be03fc273a487ac402f19ad897778d74b6d72963/src/install/install.zig#L1825) are referenced by an auto-incrementing integer ID or a hash of the package name. Strings longer than 8 characters are de-duplicated. Prior to saving on disk, the lockfile is garbage-collected & made deterministic by walking the package tree and cloning the packages in dependency order.

#### Can I opt out?

To install without creating a lockfile:

```bash
$ bun install --no-save
```

To install a Yarn lockfile _in addition_ to `bun.lockb`.

{% codetabs %}

```bash#CLI flag
$ bun install --yarn
```

```toml#bunfig.toml
[install.lockfile]
# whether to save a non-Bun lockfile alongside bun.lockb
# only "yarn" is supported
print = "yarn"
```

{% /codetabs %}

{% details summary="Configuring lockfile" %}

```toml
[install.lockfile]

# path to read bun.lockb from
path = "bun.lockb"

# path to save bun.lockb to
savePath = "bun.lockb"

# whether to save the lockfile to disk
save = true

# whether to save a non-Bun lockfile alongside bun.lockb
# only "yarn" is supported
print = "yarn"
```

{% /details %}


The `bun` CLI contains an `npm`-compatible package manager designed to be a faster replacement for existing package management tools like `npm`, `yarn`, and `pnpm`. It's designed for Node.js compatibility; use it in any Bun or Node.js project.

{% callout %}

**⚡️ 80x faster** — Switch from `npm install` to `bun install` in any Node.js project to make your installations up to 80x faster.

{% image src="https://user-images.githubusercontent.com/709451/147004342-571b6123-17a9-49a2-8bfd-dcfc5204047e.png" height="200" /%}

{% /callout %}

{% details summary="For Linux users" %}
The minimum Linux Kernel version is 5.1. If you're on Linux kernel 5.1 - 5.5, `bun install` should still work, but HTTP requests will be slow due to a lack of support for io_uring's `connect()` operation.

If you're using Ubuntu 20.04, here's how to install a [newer kernel](https://wiki.ubuntu.com/Kernel/LTSEnablementStack):

```bash
# If this returns a version >= 5.6, you don't need to do anything
uname -r

# Install the official Ubuntu hardware enablement kernel
sudo apt install --install-recommends linux-generic-hwe-20.04
```

{% /details %}

## Manage dependencies

### `bun install`

To install all dependencies of a project:

```bash
$ bun install
```

On Linux, `bun install` tends to install packages 20-100x faster than `npm install`. On macOS, it's more like 4-80x.

![package install benchmark](https://user-images.githubusercontent.com/709451/147004342-571b6123-17a9-49a2-8bfd-dcfc5204047e.png)

Running `bun install` will:

- **Install** all `dependencies`, `devDependencies`, and `optionalDependencies`. Bun does not install `peerDependencies` by default.
- **Run** your project's `{pre|post}install` scripts at the appropriate time. For security reasons Bun _does not execute_ lifecycle scripts of installed dependencies.
- **Write** a `bun.lockb` lockfile to the project root.

To install in production mode (i.e. without `devDependencies`):

```bash
$ bun install --production
```

To install dependencies without allowing changes to lockfile (useful on CI):

```bash
$ bun install --frozen-lockfile
```

To perform a dry run (i.e. don't actually install anything):

```bash
$ bun install --dry-run
```

To modify logging verbosity:

```bash
$ bun install --verbose # debug logging
$ bun install --silent  # no logging
```

{% details summary="Configuring behavior" %}
The default behavior of `bun install` can be configured in `bun.toml`:

```toml
[install]

# whether to install optionalDependencies
optional = true

# whether to install devDependencies
dev = true

# whether to install peerDependencies
peer = false

# equivalent to `--production` flag
production = false

# equivalent to `--frozen-lockfile` flag
frozenLockfile = false

# equivalent to `--dry-run` flag
dryRun = false
```

{% /details %}

### `bun add`

To add a particular package:

```bash
$ bun add preact
```

To specify a version, version range, or tag:

```bash
$ bun add zod@3.20.0
$ bun add zod@^3.0.0
$ bun add zod@latest
```

To add a package as a dev dependency (`"devDependencies"`):

```bash
$ bun add --development @types/react
$ bun add -d @types/react
```

To add a package as an optional dependency (`"optionalDependencies"`):

```bash
$ bun add --optional lodash
```

To install a package globally:

```bash
$ bun add --global cowsay # or `bun add -g cowsay`
$ cowsay "Bun!"
 ______
< Bun! >
 ------
        \   ^__^
         \  (oo)\_______
            (__)\       )\/\
                ||----w |
                ||     ||
```

{% details summary="Configuring global installation behavior" %}

```toml
[install]
# where `bun install --global` installs packages
globalDir = "~/.bun/install/global"

# where globally-installed package bins are linked
globalBinDir = "~/.bun/bin"
```

{% /details %}
To view a complete list of options for a given command:

```bash
$ bun add --help
```

### `bun remove`

To remove a dependency:

```bash
$ bun remove preact
```

## Git dependencies

To add a dependency from a git repository:

```bash
$ bun install git@github.com:moment/moment.git
```

Bun supports a variety of protocols, including [`github`](https://docs.npmjs.com/cli/v9/configuring-npm/package-json#github-urls), [`git`](https://docs.npmjs.com/cli/v9/configuring-npm/package-json#git-urls-as-dependencies), `git+ssh`, `git+https`, and many more.

```json
{
  "dependencies": {
    "dayjs": "git+https://github.com/iamkun/dayjs.git",
    "lodash": "git+ssh://github.com/lodash/lodash.git#4.17.21",
    "moment": "git@github.com:moment/moment.git",
    "zod": "github:colinhacks/zod"
  }
}
```

## Tarball dependencies

A package name can correspond to a publically hosted `.tgz` file. During `bun install`, Bun will download and install the package from the specified tarball URL, rather than from the package registry.

```json#package.json
{
  "dependencies": {
    "zod": "https://registry.npmjs.org/zod/-/zod-3.21.4.tgz"
  }
}
```


All packages downloaded from the registry are stored in a global cache at `~/.bun/install/cache`. They are stored in subdirectories named like `${name}@${version}`, so multiple versions of a package can be cached.

{% details summary="Configuring cache behavior" %}

```toml
[install.cache]
# the directory to use for the cache
dir = "~/.bun/install/cache"

# when true, don't load from the global cache.
# Bun may still write to node_modules/.cache
disable = false

# when true, always resolve the latest versions from the registry
disableManifest = false
```

{% /details %}

## Minimizing re-downloads

Bun strives to avoid re-downloading packages mutiple times. When installing a package, if the cache already contains a version in the range specified by `package.json`, Bun will use the cached package instead of downloading it again.

{% details summary="Installation details" %}
If the semver version has pre-release suffix (`1.0.0-beta.0`) or a build suffix (`1.0.0+20220101`), it is replaced with a hash of that value instead, to reduce the chances of errors associated with long file paths.

When the `node_modules` folder exists, before installing, Bun checks that `node_modules` contains all expected packages with appropriate versions. If so `bun install` completes. Bun uses a custom JSON parser which stops parsing as soon as it finds `"name"` and `"version"`.

If a package is missing or has a version incompatible with the `package.json`, Bun checks for a compatible module in the cache. If found, it is installed into `node_modules`. Otherwise, the package will be downloaded from the registry then installed.
{% /details %}

## Fast copying

Once a package is downloaded into the cache, Bun still needs to copy those files into `node_modules`. Bun uses the fastest syscalls available to perform this task. On Linux, it uses hardlinks; on macOS, it uses `clonefile`.

## Saving disk space

Since Bun uses hardlinks to "copy" a module into a project's `node_modules` directory on Linux, the contents of the package only exist in a single location on disk, greatly reducing the amount of disk space dedicated to `node_modules`.

This benefit does not extend to macOS, which uses `clonefile` for performance reasons.

{% details summary="Installation strategies" %}
This behavior is configurable with the `--backend` flag, which is respected by all of Bun's package management commands.

- **`hardlink`**: Default on Linux.
- **`clonefile`** Default on macOS.
- **`clonefile_each_dir`**: Similar to `clonefile`, except it clones each file individually per directory. It is only available on macOS and tends to perform slower than `clonefile`.
- **`copyfile`**: The fallback used when any of the above fail. It is the slowest option. On macOS, it uses `fcopyfile()`; on Linux it uses `copy_file_range()`.
  **`symlink`**: Currently used only `file:` (and eventually `link:`) dependencies. To prevent infinite loops, it skips symlinking the `node_modules` folder.

If you install with `--backend=symlink`, Node.js won't resolve node_modules of dependencies unless each dependency has its own `node_modules` folder or you pass `--preserve-symlinks` to `node`. See [Node.js documentation on `--preserve-symlinks`](https://nodejs.org/api/cli.html#--preserve-symlinks).

```bash
$ bun install --backend symlink
$ node --preserve-symlinks ./foo.js
```

Bun's runtime does not currently expose an equivalent of `--preserve-symlinks`.
{% /details %}


The test runner supports the following lifecycle hooks. This is useful for loading test fixtures, mocking data, and configuring the test environment.

| Hook         | Description                 |
| ------------ | --------------------------- |
| `beforeAll`  | Runs once before all tests. |
| `beforeEach` | Runs before each test.      |
| `afterEach`  | Runs after each test.       |
| `afterAll`   | Runs once after all tests.  |

Perform per-test setup and teardown logic with `beforeEach` and `afterEach`.

```ts
import { beforeEach, afterEach } from "bun:test";

beforeEach(() => {
  console.log("running test.");
});

afterEach(() => {
  console.log("done with test.");
});

// tests...
```

Perform per-scope setup and teardown logic with `beforeAll` and `afterAll`. The _scope_ is determined by where the hook is defined.

To scope the hooks to a particular `describe` block:

```ts
import { describe, beforeAll } from "bun:test";

describe("test group", () => {
  beforeAll(() => {
    // setup
  });

  // tests...
});
```

To scope the hooks to a test file:

```ts
import { describe, beforeAll } from "bun:test";

describe("test group", () => {
  beforeAll(() => {
    // setup
  });

  // tests...
});
```

To scope the hooks to an entire multi-file test run, define the hooks in a separate file.

```ts#setup.ts
import { beforeAll, afterAll } from "bun:test";

beforeAll(() => {
  // global setup
});

afterAll(() => {
  // global teardown
});
```

Then use `--preload` to run the setup script before any test files.

```ts
$ bun test --preload ./setup.ts
```

To avoid typing `--preload` every time you run tests, it can be added to your `bunfig.toml`:

```toml
[test]
preload = ["./setup.ts"]
```


Create mocks with the `mock` function.

```ts
import { test, expect, mock } from "bun:test";
const random = mock(() => Math.random());

test("random", async () => {
  const val = random();
  expect(val).toBeGreaterThan(0);
  expect(random).toHaveBeenCalled();
  expect(random).toHaveBeenCalledTimes(1);
});
```

The result of `mock()` is a new function that's been decorated with some additional properties.

```ts
import { mock } from "bun:test";
const random = mock((multiplier: number) => multiplier * Math.random());

random(2);
random(10);

random.mock.calls;
// [[ 2 ], [ 10 ]]

random.mock.results;
//  [
//    { type: "return", value: 0.6533907460954099 },
//    { type: "return", value: 0.6452713933037312 }
//  ]
```

## `.spyOn()`

It's possible to track calls to a function without replacing it with a mock. Use `spyOn()` to create a spy; these spies can be passed to `.toHaveBeenCalled()` and `.toHaveBeenCalledTimes()`.

```ts
import { test, expect, spyOn } from "bun:test";

const ringo = {
  name: "Ringo",
  sayHi() {
    console.log(`Hello I'm ${this.name}`);
  },
};

const spy = spyOn(ringo, "sayHi");

test("spyon", () => {
  expect(spy).toHaveBeenCalledTimes(0);
  ringo.sayHi();
  expect(spy).toHaveBeenCalledTimes(1);
});
```


Bun's test runner plays well with existing component and DOM testing libraries, including React Testing Library and [`happy-dom`](https://github.com/capricorn86/happy-dom).

## `happy-dom`

For writing headless tests for your frontend code and components, we recommend [`happy-dom`](https://github.com/capricorn86/happy-dom). Happy DOM implements a complete set of HTML and DOM APIs in plain JavaScript, making it possible to simulate a browser environment with high fidelity.

To get started install the `@happy-dom/global-registrator` package as a dev dependency.

```bash
$ bun add -d @happy-dom/global-registrator
```

We'll be using Bun's _preload_ functionality to register the `happy-dom` globals before running our tests. This step will make browser APIs like `document` available in the global scope. Create a file called `happydom.ts` in the root of your project and add the following code:

```ts
import { GlobalRegistrator } from "@happy-dom/global-registrator";

GlobalRegistrator.register();
```

To preload this file before `bun test`, open or create a `bunfig.toml` file and add the following lines.

```toml
[test]
preload = "./happydom.ts"
```

This will execute `happydom.ts` when you run `bun test`. Now you can write tests that use browser APIs like `document` and `window`.

```ts#dom.test.ts
import {test, expect} from 'bun:test';

test('dom test', () => {
  document.body.innerHTML = `<button>My button</button>`;
  const button = document.querySelector('button');
  expect(button?.innerText).toEqual('My button');
});
```

Depending on your `tsconfig.json` setup, you may see a `"Cannot find name 'document'"` type error in the code above. To "inject" the types for `document` and other browser APIs, add the following [triple-slash directive](https://www.typescriptlang.org/docs/handbook/triple-slash-directives.html) to the top of any test file.

```ts-diff#dom.test.ts
+ /// <reference lib="dom" />

  import {test, expect} from 'bun:test';

  test('dom test', () => {
    document.body.innerHTML = `<button>My button</button>`;
    const button = document.querySelector('button');
    expect(button?.innerText).toEqual('My button');
  });
```

Let's run this test with `bun test`:

```bash
$ bun test
bun test v0.x.y

dom.test.ts:
✓ dom test [0.82ms]

 1 pass
 0 fail
 1 expect() calls
Ran 1 tests across 1 files. 1 total [125.00ms]
```

<!-- ## React Testing Library

Once you've set up `happy-dom` as described above, you can use it with React Testing Library. To get started, install the `@testing-library/react` package as a dev dependency.

```bash
$ bun add -d @testing-library/react
``` -->


To automatically re-run tests when files change, use the `--watch` flag:

```sh
$ bun test --watch
```

Bun will watch for changes to any files imported in a test file, and re-run tests when a change is detected.

It's fast.

{% raw %}

<blockquote class="twitter-tweet"><p lang="en" dir="ltr">&quot;bun test --watch url&quot; in a large folder with multiple files that start with &quot;url&quot; <a href="https://t.co/aZV9BP4eFu">pic.twitter.com/aZV9BP4eFu</a></p>&mdash; Jarred Sumner (@jarredsumner) <a href="https://twitter.com/jarredsumner/status/1640890850535436288?ref_src=twsrc%5Etfw">March 29, 2023</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

{% /raw %}


Define tests with a Jest-like API imported from the built-in `bun:test` module. Long term, Bun aims for complete Jest compatibility; at the moment, a [limited set](#matchers) of `expect` matchers are supported.

## Basic usage

To define a simple test:

```ts#math.test.ts
import { expect, test } from "bun:test";

test("2 + 2", () => {
  expect(2 + 2).toBe(4);
});
```

{% details summary="Jest-style globals" %}
As in Jest, you can use `describe`, `test`, `expect`, and other functions without importing them. Unlike Jest, they are not injected into the global scope. Instead, the Bun transpiler will automatically inject an import from `bun:test` internally.

```ts
typeof globalThis.describe; // "undefined"
typeof describe; // "function"
```

This transpiler integration only occurs during `bun test`, and only for test files & preloaded scripts. In practice there's no significant difference to the end user.
{% /details %}

Tests can be grouped into suites with `describe`.

```ts#math.test.ts
import { expect, test, describe } from "bun:test";

describe("arithmetic", () => {
  test("2 + 2", () => {
    expect(2 + 2).toBe(4);
  });

  test("2 * 2", () => {
    expect(2 * 2).toBe(4);
  });
});
```

Tests can be `async`.

```ts
import { expect, test } from "bun:test";

test("2 * 2", async () => {
  const result = await Promise.resolve(2 * 2);
  expect(result).toEqual(4);
});
```

Alternatively, use the `done` callback to signal completion. If you include the `done` callback as a parameter in your test definition, you _must_ call it or the test will hang.

```ts
import { expect, test } from "bun:test";

test("2 * 2", done => {
  Promise.resolve(2 * 2).then(result => {
    expect(result).toEqual(4);
    done();
  });
});
```

## Timeouts

Optionally specify a per-test timeout in milliseconds by passing a number as the third argument to `test`.

```ts
import { test } from "bun:test";

test.skip("wat", async () => {
  const data = await slowOperation();
  expect(data).toBe(42);
}, 500); // test must run in <500ms
```

## `test.skip`

Skip individual tests with `test.skip`. These tests will not be run.

```ts
import { expect, test } from "bun:test";

test.skip("wat", () => {
  // TODO: fix this
  expect(0.1 + 0.2).toEqual(0.3);
});
```

## `test.todo`

Mark a test as a todo with `test.todo`. These tests _will_ be run, and the test runner will expect them to fail. If they pass, you will be prompted to mark it as a regular test.

```ts
import { expect, test } from "bun:test";

test.todo("fix this", () => {
  myTestFunction();
});
```

To exlusively run tests marked as _todo_, use `bun test --todo`.

```sh
$ bun test --todo
```

## `test.only`

To run a particular test or suite of tests use `test.only()` or `describe.only()`. Once declared, running `bun test --skip` will only execute tests/suites that have been marked with `.only()`.

```ts
import { test, describe } from "bun:test";

test("test #1", () => {
  // does not run
});

test.only("test #2", () => {
  // runs
});

describe.only("only", () => {
  test("test #3", () => {
    // runs
  });
});
```

The following command will only execute tests #2 and #3.

```sh
$ bun test --only
```

## `test.if`

To run a test conditionally, use `test.if()`. The test will run if the condition is truthy. This is particularly useful for tests that should only run on specific architectures or operating systems.

```ts
test.if(Math.random() > 0.5)("runs half the time", () => {
  // ...
});
```

```ts
test.if(Math.random() > 0.5)("runs half the time", () => {
  // ...
});

const macOS = process.arch === "darwin";
test.if(macOS)("runs on macOS", () => {
  // runs if macOS
});
```

To instead skip a test based on some condition, use `test.skipIf()` or `describe.skipIf()`.

```ts
const macOS = process.arch === "darwin";

test.skipIf(macOS)("runs on non-macOS", () => {
  // runs if *not* macOS
});
```

## Matchers

Bun implements the following matchers. Full Jest compatibility is on the roadmap; track progress [here](https://github.com/oven-sh/bun/issues/1825).

{% table %}

---

- 🟢
- [`.not`](https://jestjs.io/docs/expect#not)

---

- 🟢
- [`.toBe()`](https://jestjs.io/docs/expect#tobevalue)

---

- 🟢
- [`.toEqual()`](https://jestjs.io/docs/expect#toequalvalue)

---

- 🟢
- [`.toBeNull()`](https://jestjs.io/docs/expect#tobenull)

---

- 🟢
- [`.toBeUndefined()`](https://jestjs.io/docs/expect#tobeundefined)

---

- 🟢
- [`.toBeNaN()`](https://jestjs.io/docs/expect#tobenan)

---

- 🟢
- [`.toBeDefined()`](https://jestjs.io/docs/expect#tobedefined)

---

- 🟢
- [`.toBeFalsy()`](https://jestjs.io/docs/expect#tobefalsy)

---

- 🟢
- [`.toBeTruthy()`](https://jestjs.io/docs/expect#tobetruthy)

---

- 🟢
- [`.toContain()`](https://jestjs.io/docs/expect#tocontainitem)

---

- 🟢
- [`.toStrictEqual()`](https://jestjs.io/docs/expect#tostrictequalvalue)

---

- 🟢
- [`.toThrow()`](https://jestjs.io/docs/expect#tothrowerror)

---

- 🟢
- [`.toHaveLength()`](https://jestjs.io/docs/expect#tohavelengthnumber)

---

- 🟢
- [`.toHaveProperty()`](https://jestjs.io/docs/expect#tohavepropertykeypath-value)

---

- 🔴
- [`.extend`](https://jestjs.io/docs/expect#expectextendmatchers)

---

- 🟢
- [`.anything()`](https://jestjs.io/docs/expect#expectanything)

---

- 🟢
- [`.any()`](https://jestjs.io/docs/expect#expectanyconstructor)

---

- 🔴
- [`.arrayContaining()`](https://jestjs.io/docs/expect#expectarraycontainingarray)

---

- 🔴
- [`.assertions()`](https://jestjs.io/docs/expect#expectassertionsnumber)

---

- 🔴
- [`.closeTo()`](https://jestjs.io/docs/expect#expectclosetonumber-numdigits)

---

- 🔴
- [`.hasAssertions()`](https://jestjs.io/docs/expect#expecthasassertions)

---

- 🔴
- [`.objectContaining()`](https://jestjs.io/docs/expect#expectobjectcontainingobject)

---

- 🟢
- [`.stringContaining()`](https://jestjs.io/docs/expect#expectstringcontainingstring)

---

- 🟢
- [`.stringMatching()`](https://jestjs.io/docs/expect#expectstringmatchingstring--regexp)

---

- 🔴
- [`.addSnapshotSerializer()`](https://jestjs.io/docs/expect#expectaddsnapshotserializerserializer)

---

- 🟢
- [`.resolves()`](https://jestjs.io/docs/expect#resolves) (since Bun v0.6.12+)

---

- 🟢
- [`.rejects()`](https://jestjs.io/docs/expect#rejects) (since Bun v0.6.12+)

---

- 🟢
- [`.toHaveBeenCalled()`](https://jestjs.io/docs/expect#tohavebeencalled)

---

- 🟢
- [`.toHaveBeenCalledTimes()`](https://jestjs.io/docs/expect#tohavebeencalledtimesnumber)

---

- 🔴
- [`.toHaveBeenCalledWith()`](https://jestjs.io/docs/expect#tohavebeencalledwitharg1-arg2-)

---

- 🔴
- [`.toHaveBeenLastCalledWith()`](https://jestjs.io/docs/expect#tohavebeenlastcalledwitharg1-arg2-)

---

- 🔴
- [`.toHaveBeenNthCalledWith()`](https://jestjs.io/docs/expect#tohavebeennthcalledwithnthcall-arg1-arg2-)

---

- 🔴
- [`.toHaveReturned()`](https://jestjs.io/docs/expect#tohavereturned)

---

- 🔴
- [`.toHaveReturnedTimes()`](https://jestjs.io/docs/expect#tohavereturnedtimesnumber)

---

- 🔴
- [`.toHaveReturnedWith()`](https://jestjs.io/docs/expect#tohavereturnedwithvalue)

---

- 🔴
- [`.toHaveLastReturnedWith()`](https://jestjs.io/docs/expect#tohavelastreturnedwithvalue)

---

- 🔴
- [`.toHaveNthReturnedWith()`](https://jestjs.io/docs/expect#tohaventhreturnedwithnthcall-value)

---

- 🟢
- [`.toBeCloseTo()`](https://jestjs.io/docs/expect#tobeclosetonumber-numdigits)

---

- 🟢
- [`.toBeGreaterThan()`](https://jestjs.io/docs/expect#tobegreaterthannumber--bigint)

---

- 🟢
- [`.toBeGreaterThanOrEqual()`](https://jestjs.io/docs/expect#tobegreaterthanorequalnumber--bigint)

---

- 🟢
- [`.toBeLessThan()`](https://jestjs.io/docs/expect#tobelessthannumber--bigint)

---

- 🟢
- [`.toBeLessThanOrEqual()`](https://jestjs.io/docs/expect#tobelessthanorequalnumber--bigint)

---

- 🟢
- [`.toBeInstanceOf()`](https://jestjs.io/docs/expect#tobeinstanceofclass) (Bun v0.5.8+)

---

- 🔴
- [`.toContainEqual()`](https://jestjs.io/docs/expect#tocontainequalitem)

---

- 🟢
- [`.toMatch()`](https://jestjs.io/docs/expect#tomatchregexp--string)

---

- 🟢
- [`.toMatchObject()`](https://jestjs.io/docs/expect#tomatchobjectobject)

---

- 🟢
- [`.toMatchSnapshot()`](https://jestjs.io/docs/expect#tomatchsnapshotpropertymatchers-hint) (Bun v0.5.8+)

---

- 🔴
- [`.toMatchInlineSnapshot()`](https://jestjs.io/docs/expect#tomatchinlinesnapshotpropertymatchers-inlinesnapshot)

---

- 🔴
- [`.toThrowErrorMatchingSnapshot()`](https://jestjs.io/docs/expect#tothrowerrormatchingsnapshothint)

---

- 🔴
- [`.toThrowErrorMatchingInlineSnapshot()`](https://jestjs.io/docs/expect#tothrowerrormatchinginlinesnapshotinlinesnapshot)

{% /table %}


`bun:test` lets you change what time it is in your tests. This was introduced in Bun v0.6.13.

This works with any of the following:

- `Date.now`
- `new Date()`
- `new Intl.DateTimeFormat().format()`

Timers are not impacted yet, but may be in a future release of Bun.

## `setSystemTime`

To change the system time, use `setSystemTime`:

```ts
import { setSystemTime, beforeAll, test, expect } from "bun:test";

beforeAll(() => {
  setSystemTime(new Date("2020-01-01T00:00:00.000Z"));
});

test("it is 2020", () => {
  expect(new Date().getFullYear()).toBe(2020);
});
```

To support existing tests that use Jest's `useFakeTimers` and `useRealTimers`, you can use `useFakeTimers` and `useRealTimers`:

```ts
test("just like in jest", () => {
  jest.useFakeTimers();
  jest.setSystemTime(new Date("2020-01-01T00:00:00.000Z"));
  expect(new Date().getFullYear()).toBe(2020);
  jest.useRealTimers();
  expect(new Date().getFullYear()).toBeGreaterThan(2020);
});

test("unlike in jest", () => {
  const OriginalDate = Date;
  jest.useFakeTimers();
  if (typeof Bun === "undefined") {
    // In Jest, the Date constructor changes
    // That can cause all sorts of bugs because suddenly Date !== Date before the test.
    expect(Date).not.toBe(OriginalDate);
    expect(Date.now).not.toBe(OriginalDate.now);
  } else {
    // In bun:test, Date constructor does not change when you useFakeTimers
    expect(Date).toBe(OriginalDate);
    expect(Date.now).toBe(OriginalDate.now);
  }
});
```

{% callout %}
**Timers** — Note that we have not implemented builtin support for mocking timers yet, but this is on the roadmap.
{% /callout %}

### Reset the system time

To reset the system time, pass no arguments to `setSystemTime`:

```ts
import { setSystemTime, beforeAll } from "bun:test";

test("it was 2020, for a moment.", () => {
  // Set it to something!
  setSystemTime(new Date("2020-01-01T00:00:00.000Z"));
  expect(new Date().getFullYear()).toBe(2020);

  // reset it!
  setSystemTime();

  expect(new Date().getFullYear()).toBeGreaterThan(2020);
});
```

## Set the time zone

To change the time zone, either pass the `$TZ` environment variable to `bun test`.

```sh
TZ=America/Los_Angeles bun test
```

Or set `process.env.TZ` at runtime:

```ts
import { test, expect } from "bun:test";

test("Welcome to California!", () => {
  process.env.TZ = "America/Los_Angeles";
  expect(new Date().getTimezoneOffset()).toBe(420);
  expect(new Intl.DateTimeFormat().resolvedOptions().timeZone).toBe(
    "America/Los_Angeles",
  );
});

test("Welcome to New York!", () => {
  // Unlike in Jest, you can set the timezone multiple times at runtime and it will work.
  process.env.TZ = "America/New_York";
  expect(new Date().getTimezoneOffset()).toBe(240);
  expect(new Intl.DateTimeFormat().resolvedOptions().timeZone).toBe(
    "America/New_York",
  );
});
```


Snapshot tests are written using the `.toMatchSnapshot()` matcher:

```ts
import { test, expect } from "bun:test";

test("snap", () => {
  expect("foo").toMatchSnapshot();
});
```

The first time this test is run, the argument to `expect` will be serialized and written to a special snapshot file in a `__snapshots__` directory alongside the test file. On future runs, the argument is compared against the snapshot on disk. Snapshots can be re-generated with the following command:

```bash
$ bun test --update-snapshots
```


{% callout %}
**Note** — Added in Bun v0.3.0
{% /callout %}

If no `node_modules` directory is found in the working directory or higher, Bun will abandon Node.js-style module resolution in favor of the **Bun module resolution algorithm**.

Under Bun-style module resolution, all imported packages are auto-installed on the fly into a [global module cache](/docs/cli/install#global-cache) during execution (the same cache used by [`bun install`](/docs/cli/install)).

```ts
import { foo } from "foo"; // install `latest` version

foo();
```

The first time you run this script, Bun will auto-install `"foo"` and cache it. The next time you run the script, it will use the cached version.

## Version resolution

To determine which version to install, Bun follows the following algorithm:

1. Check for a `bun.lockb` file in the project root. If it exists, use the version specified in the lockfile.
2. Otherwise, scan up the tree for a `package.json` that includes `"foo"` as a dependency. If found, use the specified semver version or version range.
3. Otherwise, use `latest`.

## Cache behavior

Once a version or version range has been determined, Bun will:

1. Check the module cache for a compatible version. If one exists, use it.
2. When resolving `latest`, Bun will check if `package@latest` has been downloaded and cached in the last _24 hours_. If so, use it.
3. Otherwise, download and install the appropriate version from the `npm` registry.

## Installation

Packages are installed and cached into `<cache>/<pkg>@<version>`, so multiple versions of the same package can be cached at once. Additionally, a symlink is created under `<cache>/<pkg>/<version>` to make it faster to look up all versions of a package that exist in the cache.

## Version specifiers

This entire resolution algorithm can be short-circuited by specifying a version or version range directly in your import statement.

```ts
import { z } from "zod@3.0.0"; // specific version
import { z } from "zod@next"; // npm tag
import { z } from "zod@^3.20.0"; // semver range
```

## Benefits

This auto-installation approach is useful for a few reasons:

- **Space efficiency** — Each version of a dependency only exists in one place on disk. This is a huge space and time savings compared to redundant per-project installations.
- **Portability** — To share simple scripts and gists, your source file is _self-contained_. No need to `zip` together a directory containing your code and config files. With version specifiers in `import` statements, even a `package.json` isn't necessary.
- **Convenience** — There's no need to run `npm install` or `bun install` before running a file or script. Just `bun run` it.
- **Backwards compatibility** — Because Bun still respects the versions specified in `package.json` if one exists, you can switch to Bun-style resolution with a single command: `rm -rf node_modules`.

## Limitations

- No Intellisense. TypeScript auto-completion in IDEs relies on the existence of type declaration files inside `node_modules`. We are investigating various solutions to this.
- No [patch-package](https://github.com/ds300/patch-package) support

<!-- - The implementation details of Bun's install cache will change between versions. Don't think of it as an API. To reliably resolve packages, use Bun's builtin APIs (such as `Bun.resolveSync` or `import.meta.resolve`) instead of relying on the filesystem directly. Bun will likely move to a binary archive format where packages may not correspond to files/folders on disk at all - so if you depend on the filesystem structure instead of the JavaScript API, your code will eventually break. -->

<!-- ## Customizing behavior

To prefer locally-installed versions of packages. Instead of checking npm for latest versions, you can pass the `--prefer-offline` flag to prefer locally-installed versions of packages.

```bash
$ bun run --prefer-offline my-script.ts
```

This will check the install cache for installed versions of packages before checking the npm registry. If no matching version of a package is installed, only then will it check npm for the latest version.

#### Prefer latest

To always use the latest version of a package, you can pass the `--prefer-latest` flag.

```bash
$ bun run --prefer-latest my-script.ts
``` -->

## FAQ

{% details summary="How is this different from what pnpm does?" %}

With pnpm, you have to run `pnpm install`, which creates a `node_modules` folder of symlinks for the runtime to resolve. By contrast, Bun resolves dependencies on the fly when you run a file; there's no need to run any `install` command ahead of time. Bun also doesn't create a `node_modules` folder.

{% /details %}

{% details summary="How is this different from Yarn Plug'N'Play does?" %}
With Yarn, you must run `yarn install` before you run a script. By contrast, Bun resolves dependencies on the fly when you run a file; there's no need to run any `install` command ahead of time.

Yarn Plug'N'Play also uses zip files to store dependencies. This makes dependency loading [slower at runtime](https://twitter.com/jarredsumner/status/1458207919636287490), as random access reads on zip files tend to be slower than the equivalent disk lookup.
{% /details %}

{% details summary="How is this different from what Deno does?" %}

Deno requires an `npm:` specifier before each npm `import`, lacks support for import maps via `compilerOptions.paths` in `tsconfig.json`, and has incomplete support for `package.json` settings. Unlike Deno, Bun does not currently support URL imports.
{% /details %}


Module resolution in JavaScript is a complex topic.

The ecosystem is currently in the midst of a years-long transition from CommonJS modules to native ES modules. TypeScript enforces its own set of rules around import extensions that aren't compatible with ESM. Different build tools support path re-mapping via disparate non-compatible mechanisms.

Bun aims to provide a consistent and predictable module resolution system that just works. Unfortunately it's still quite complex.

## Syntax

Consider the following files.

{% codetabs %}

```ts#index.ts
import { hello } from "./hello";

hello();
```

```ts#hello.ts
export function hello() {
  console.log("Hello world!");
}
```

{% /codetabs %}

When we run `index.ts`, it prints "Hello world".

```bash
$ bun index.ts
Hello world!
```

In this case, we are importing from `./hello`, a relative path with no extension. To resolve this import, Bun will check for the following files in order:

- `./hello.ts`
- `./hello.tsx`
- `./hello.js`
- `./hello.mjs`
- `./hello.cjs`
- `./hello/index.ts`
- `./hello/index.js`
- `./hello/index.json`
- `./hello/index.mjs`

Import paths are case-insensitive.

```ts#index.ts
import { hello } from "./hello";
import { hello } from "./HELLO";
import { hello } from "./hElLo";
```

Import paths can optionally include extensions. If an extension is present, Bun will only check for a file with that exact extension.

```ts#index.ts
import { hello } from "./hello";
import { hello } from "./hello.ts"; // this works
```

There is one exception: if you import `from "*.js{x}"`, Bun will additionally check for a matching `*.ts{x}` file, to be compatible with TypeScript's [ES module support](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-4-7.html#new-file-extensions).

```ts#index.ts
import { hello } from "./hello";
import { hello } from "./hello.ts"; // this works
import { hello } from "./hello.js"; // this also works
```

Bun supports both ES modules (`import`/`export` syntax) and CommonJS modules (`require()`/`module.exports`). The following CommonJS version would also work in Bun.

{% codetabs %}

```ts#index.js
const { hello } = require("./hello");

hello();
```

```ts#hello.js
function hello() {
  console.log("Hello world!");
}

exports.hello = hello;
```

{% /codetabs %}

That said, using CommonJS is discouraged in new projects.

## Resolution

Bun implements the Node.js module resolution algorithm, so you can import packages from `node_modules` with a bare specifier.

```ts
import { stuff } from "foo";
```

The full specification of this algorithm are officially documented in the [Node.js documentation](https://nodejs.org/api/modules.html); we won't rehash it here. Briefly: if you import `from "foo"`, Bun scans up the file system for a `node_modules` directory containing the package `foo`.

Once it finds the `foo` package, Bun reads the `package.json` to determine how the package should be imported. Unless `"type": "module"` is specified, Bun assumes the package is using CommonJS and transpiles into a synchronous ES module internally. To determine the package's entrypoint, Bun first reads the `exports` field in and checks the following conditions in order:

```jsonc#package.json
{
  "name": "foo",
  "exports": {
    "bun": "./index.js",        // highest priority
    "worker": "./index.js",
    "module": "./index.js",
    "node": "./index.js",
    "default": "./index.js",
    "browser": "./index.js"     // lowest priority
  }
}
```

Bun respects subpath [`"exports"`](https://nodejs.org/api/packages.html#subpath-exports) and [`"imports"`](https://nodejs.org/api/packages.html#imports). Specifying any subpath in the `"exports"` map will prevent other subpaths from being importable.

```jsonc#package.json
{
  "name": "foo",
  "exports": {
    ".": "./index.js",
    "./package.json": "./package.json" // subpath
  }
}
```

{% callout %}
**Shipping TypeScript** — Note that Bun supports the special `"bun"` export condition. If your library is written in TypeScript, you can publish your (un-transpiled!) TypeScript files to `npm` directly. If you specify your package's `*.ts` entrypoint in the `"bun"` condition, Bun will directly import and execute your TypeScript source files.
{% /callout %}

If `exports` is not defined, Bun falls back to `"module"` (ESM imports only) then [`"main"`](https://nodejs.org/api/packages.html#main).

```json#package.json
{
  "name": "foo",
  "module": "./index.js",
  "main": "./index.js"
}
```

## Path re-mapping

In the spirit of treating TypeScript as a first-class citizen, the Bun runtime will re-map import paths according to the [`compilerOptions.paths`](https://www.typescriptlang.org/tsconfig#paths) field in `tsconfig.json`. This is a major divergence from Node.js, which doesn't support any form of import path re-mapping.

```jsonc#tsconfig.json
{
  "compilerOptions": {
    "paths": {
      "config": ["./config.ts"],         // map specifier to file
      "components/*": ["components/*"],  // wildcard matching
    }
  }
}
```

If you aren't a TypeScript user, you can create a [`jsconfig.json`](https://code.visualstudio.com/docs/languages/jsconfig) in your project root to achieve the same behavior.

## CommonJS

Bun has native support for CommonJS modules (added in Bun v0.6.5). ES Modules are the recommended module format, but CommonJS modules are still widely used in the Node.js ecosystem. Bun supports both module formats, so that existing CommonJS packages can be used.

In Bun's JavaScript runtime, `require` can be used by both ES Modules and CommonJS modules.

In Bun, you can `require()` ESM modules from CommonJS modules.

| Module Type | `require()`      | `import * as`                                                           |
| ----------- | ---------------- | ----------------------------------------------------------------------- |
| ES Module   | Module Namespace | Module Namespace                                                        |
| CommonJS    | module.exports   | `default` is `module.exports`, keys of module.exports are named exports |

If the target module is an ES Module, `require` returns the module namespace object (equivalent to `import * as`).
If the target module is a CommonJS module, `require` returns the `module.exports` object.

### What is a CommonJS module?

In 2016, ECMAScript added support for [ES Modules](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Modules). ES Modules are the standard for JavaScript modules. However, millions of npm packages still use CommonJS modules.

CommonJS modules are modules that use `module.exports` to export values. Typically, `require` is used to import CommonJS modules.

```ts
// my-commonjs.cjs
const stuff = require("./stuff");
module.exports = { stuff };
```

The biggest difference between CommonJS and ES Modules is that CommonJS modules are synchronous, while ES Modules are asynchronous. There are other differences too, like ES Modules support top-level `await` and CommonJS modules don't. ES Modules are always in [strict mode](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Strict_mode), while CommonJS modules are not. Browsers do not have native support for CommonJS modules, but they do have native support for ES Modules (`<script type="module">`). CommonJS modules are not statically analyzable, while ES Modules only allow static imports and exports.

### Importing CommonJS from ESM

You can `import` or `require` CommonJS modules from ESM modules.

```ts
import { stuff } from "./my-commonjs.cjs";
import Stuff from "./my-commonjs.cjs";
const myStuff = require("./my-commonjs.cjs");
```

### Importing ESM from CommonJS

```ts
// this works in Bun v0.6.5+
// It does not work in Node.js
const { stuff } = require("./my-esm.mjs");
```

### Importing CommonJS from CommonJS

You can `require()` CommonJS modules from CommonJS modules.

```ts
const { stuff } = require("./my-commonjs.cjs");
```

#### Top-level await

If you are using top-level await, you must use `import()` to import ESM modules from CommonJS modules.

```ts
import("./my-esm.js").then(({ stuff }) => {
  // ...
});

// this will throw an error if "my-esm.js" uses top-level await
const { stuff } = require("./my-esm.js");
```

#### Low-level details of CommonJS interop in Bun

Bun's JavaScript runtime has native support for CommonJS as of Bun v0.6.5.

When Bun's JavaScript transpiler detects usages of `module.exports`, it treats the file as CommonJS. The module loader will then wrap the transpiled module in a function shaped like this:

```js
(function (module, exports, require) {
  // transpiled module
})(module, exports, require);
```

`module`, `exports`, and `require` are very much like the `module`, `exports`, and `require` in Node.js. These are assigned via a [`with scope`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/with) in C++. An internal `Map` stores the `exports` object to handle cyclical `require` calls before the module is fully loaded.

Once the CommonJS module is successfully evaluated, a Synthetic Module Record is created with the `default` ES Module [export set to `module.exports`](https://github.com/oven-sh/bun/blob/9b6913e1a674ceb7f670f917fc355bb8758c6c72/src/bun.js/bindings/CommonJSModuleRecord.cpp#L212-L213) and keys of the `module.exports` object are re-exported as named exports (if the `module.exports` object is an object).

When using Bun's bundler, this works differently. The bundler will wrap the CommonJS module in a `require_${moduleName}` function which returns the `module.exports` object.


Some Web APIs aren't relevant in the context of a server-first runtime like Bun, such as the [DOM API](https://developer.mozilla.org/en-US/docs/Web/API/HTML_DOM_API#html_dom_api_interfaces) or [History API](https://developer.mozilla.org/en-US/docs/Web/API/History_API). Many others, though, are broadly useful outside of the browser context; when possible, Bun implements these Web-standard APIs instead of introducing new APIs.

The following Web APIs are partially or completely supported.

{% table %}

---

- HTTP
- [`fetch`](https://developer.mozilla.org/en-US/docs/Web/API/fetch) [`Response`](https://developer.mozilla.org/en-US/docs/Web/API/Response) [`Request`](https://developer.mozilla.org/en-US/docs/Web/API/Request) [`Headers`](https://developer.mozilla.org/en-US/docs/Web/API/Headers) [`AbortController`](https://developer.mozilla.org/en-US/docs/Web/API/AbortController) [`AbortSignal`](https://developer.mozilla.org/en-US/docs/Web/API/AbortSignal)

---

- URLs
- [`URL`](https://developer.mozilla.org/en-US/docs/Web/API/URL) [`URLSearchParams`](https://developer.mozilla.org/en-US/docs/Web/API/URLSearchParams)

---

- Streams
- [`ReadableStream`](https://developer.mozilla.org/en-US/docs/Web/API/ReadableStream) [`WritableStream`](https://developer.mozilla.org/en-US/docs/Web/API/WritableStream) [`TransformStream`](https://developer.mozilla.org/en-US/docs/Web/API/TransformStream) [`ByteLengthQueuingStrategy`](https://developer.mozilla.org/en-US/docs/Web/API/ByteLengthQueuingStrategy) [`CountQueuingStrategy`](https://developer.mozilla.org/en-US/docs/Web/API/CountQueuingStrategy) and associated classes

---

- Blob
- [`Blob`](https://developer.mozilla.org/en-US/docs/Web/API/Blob)

---

- WebSockets
- [`WebSocket`](https://developer.mozilla.org/en-US/docs/Web/API/WebSocket)

---

- Encoding and decoding
- [`atob`](https://developer.mozilla.org/en-US/docs/Web/API/atob) [`btoa`](https://developer.mozilla.org/en-US/docs/Web/API/btoa) [`TextEncoder`](https://developer.mozilla.org/en-US/docs/Web/API/TextEncoder) [`TextDecoder`](https://developer.mozilla.org/en-US/docs/Web/API/TextDecoder)

---

- JSON
- [`JSON`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/JSON)

---

- Timeouts
- [`setTimeout`](https://developer.mozilla.org/en-US/docs/Web/API/setTimeout) [`clearTimeout`](https://developer.mozilla.org/en-US/docs/Web/API/clearTimeout)

---

- Intervals
- [`setInterval`](https://developer.mozilla.org/en-US/docs/Web/API/setInterval)[`clearInterval`](https://developer.mozilla.org/en-US/docs/Web/API/clearInterval)

---

- Crypto
- [`crypto`](https://developer.mozilla.org/en-US/docs/Web/API/Crypto) [`SubtleCrypto`](https://developer.mozilla.org/en-US/docs/Web/API/SubtleCrypto)
  [`CryptoKey`](https://developer.mozilla.org/en-US/docs/Web/API/CryptoKey)

---

- Debugging

- [`console`](https://developer.mozilla.org/en-US/docs/Web/API/console) [`performance`](https://developer.mozilla.org/en-US/docs/Web/API/Performance)

---

- Microtasks
- [`queueMicrotask`](https://developer.mozilla.org/en-US/docs/Web/API/queueMicrotask)

---

- Errors
- [`reportError`](https://developer.mozilla.org/en-US/docs/Web/API/reportError)

---

- User interaction
- [`alert`](https://developer.mozilla.org/en-US/docs/Web/API/Window/alert) [`confirm`](https://developer.mozilla.org/en-US/docs/Web/API/Window/confirm) [`prompt`](https://developer.mozilla.org/en-US/docs/Web/API/Window/prompt) (intended for interactive CLIs)

<!-- - Blocking. Prints the alert message to terminal and awaits `[ENTER]` before proceeding. -->
<!-- - Blocking. Prints confirmation message and awaits `[y/N]` input from user. Returns `true` if user entered `y` or `Y`, `false` otherwise.
- Blocking. Prints prompt message and awaits user input. Returns the user input as a string. -->

---

- Realms
- [`ShadowRealm`](https://github.com/tc39/proposal-shadowrealm)

---

- Events
- [`EventTarget`](https://developer.mozilla.org/en-US/docs/Web/API/EventTarget)
  [`Event`](https://developer.mozilla.org/en-US/docs/Web/API/Event) [`ErrorEvent`](https://developer.mozilla.org/en-US/docs/Web/API/ErrorEvent) [`CloseEvent`](https://developer.mozilla.org/en-US/docs/Web/API/CloseEvent) [`MessageEvent`](https://developer.mozilla.org/en-US/docs/Web/API/MessageEvent)

---

{% /table %}


Bun implements a set of native APIs on the `Bun` global object and through a number of built-in modules. These APIs are heavily optimized and represent the canonical "Bun-native" way to implement some common functionality.

Bun strives to implement standard Web APIs wherever possible. Bun introduces new APIs primarily for server-side tasks where no standard exists, such as file I/O and starting an HTTP server. In these cases, Bun's approach still builds atop standard APIs like `Blob`, `URL`, and `Request`.

```ts
Bun.serve({
  fetch(req: Request) {
    return new Response("Success!");
  },
});
```

Click the link in the right column to jump to the associated documentation.

{% table %}

- Topic
- APIs

---

- HTTP server
- [`Bun.serve`](/docs/api/http#bun-serve)

---

- Bundler
- [`Bun.build`](/docs/bundler)

---

- File I/O
- [`Bun.file`](/docs/api/file-io#reading-files-bun-file) [`Bun.write`](/docs/api/file-io#writing-files-bun-write)

---

- Child processes
- [`Bun.spawn`](/docs/api/spawn#spawn-a-process-bun-spawn) [`Bun.spawnSync`](/docs/api/spawn#blocking-api-bun-spawnsync)

---

- TCP
- [`Bun.listen`](/docs/api/tcp#start-a-server-bun-listen) [`Bun.connect`](/docs/api/tcp#start-a-server-bun-listen)

---

- Transpiler
- [`Bun.Transpiler`](/docs/api/transpiler)

---

- Routing
- [`Bun.FileSystemRouter`](/docs/api/file-system-router)

---

- HTML Rewriting
- [`HTMLRewriter`](/docs/api/html-rewriter)

---

- Hashing
- [`Bun.hash`](/docs/api/hashing#bun-hash) [`Bun.CryptoHasher`](/docs/api/hashing#bun-cryptohasher)

---

- import.meta
- [`import.meta`](/docs/api/import-meta)

---

<!-- - [DNS](/docs/api/dns)
- `Bun.dns`

--- -->

- SQLite
- [`bun:sqlite`](/docs/api/sqlite)

---

- FFI
- [`bun:ffi`](/docs/api/ffi)

---

- Testing
- [`bun:test`](/docs/cli/test)

---

- Node-API
- [`Node-API`](/docs/api/node-api)

---

- Utilities
- [`Bun.version`](/docs/api/utils#bun-version) [`Bun.revision`](/docs/api/utils#bun-revision) [`Bun.env`](/docs/api/utils#bun-env) [`Bun.main`](/docs/api/utils#bun-main) [`Bun.sleep()`](/docs/api/utils#bun-sleep) [`Bun.sleepSync()`](/docs/api/utils#bun-sleepsync) [`Bun.which()`](/docs/api/utils#bun-which) [`Bun.peek()`](/docs/api/utils#bun-peek) [`Bun.openInEditor()`](/docs/api/utils#bun-openineditor) [`Bun.deepEquals()`](/docs/api/utils#bun-deepequals) [`Bun.escapeHTML()`](/docs/api/utils#bun-escapehtlm) [`Bun.enableANSIColors()`](/docs/api/utils#bun-enableansicolors) [`Bun.fileURLToPath()`](/docs/api/utils#bun-fileurltopath) [`Bun.pathToFileURL()`](/docs/api/utils#bun-pathtofileurl) [`Bun.gzipSync()`](/docs/api/utils#bun-gzipsync) [`Bun.gunzipSync()`](/docs/api/utils#bun-gunzipsync) [`Bun.deflateSync()`](/docs/api/utils#bun-deflatesync) [`Bun.inflateSync()`](/docs/api/utils#bun-inflatesync) [`Bun.inspect()`](/docs/api/utils#bun-inspect) [`Bun.nanoseconds()`](/docs/api/utils#bun-nanoseconds) [`Bun.readableStreamTo*()`](/docs/api/utils#bun-readablestreamto) [`Bun.resolveSync()`](/docs/api/utils#bun-resolvesync)

{% /table %}


Bun supports two kinds of automatic reloading via CLI flags:

- `--watch` mode, which hard restarts Bun's process when imported files change (introduced in Bun v0.5.9)
- `--hot` mode, which soft reloads the code (without restarting the process) when imported files change (introduced in Bun v0.2.0)

## `--watch` mode

Watch mode can be used with `bun test` or when running TypeScript, JSX, and JavaScript files.


To run a file in `--watch` mode:

```bash
$ bun --watch index.tsx
```

To run your tests in `--watch` mode:

```bash
$ bun --watch test 
```

In `--watch` mode, Bun keeps track of all imported files and watches them for changes. When a change is detected, Bun restarts the process, preserving the same set of CLI arguments and environment variables used in the initial run. If Bun crashes, `--watch` will attempt to automatically restart the process.

{% callout %}

**⚡️ Reloads are fast.** The filesystem watchers you're probably used to have several layers of libraries wrapping the native APIs or worse, rely on polling.

Instead, Bun uses operating system native filesystem watcher APIs like kqueue or inotify to detect changes to files. Bun also does a number of optimizations to enable it scale to larger projects (such as setting a high rlimit for file descriptors, statically allocated file path buffers, reuse file descriptors when possible, etc).

{% /callout %}

The following examples show Bun live-reloading a file as it is edited, with VSCode configured to save the file [on each keystroke](https://code.visualstudio.com/docs/editor/codebasics#_save-auto-save).

{% codetabs %}

```bash
$ bun run --watch watchy.tsx
```

```tsx#watchy.tsx
import { serve } from "bun";
console.log("I restarted at:", Date.now());

serve({
  port: 4003,

  fetch(request) {
    return new Response("Sup");
  },
});
```

{% /codetabs %}

![bun watch gif](https://user-images.githubusercontent.com/709451/228439002-7b9fad11-0db2-4e48-b82d-2b88c8625625.gif)

Running `bun test` in watch mode and `save-on-keypress` enabled:

```bash
$ bun --watch test 
```

![bun test gif](https://user-images.githubusercontent.com/709451/228396976-38a23864-4a1d-4c96-87cc-04e5181bf459.gif)

## `--hot` mode

Use `bun --hot` to enable hot reloading when executing code with Bun.

```bash
$ bun --hot server.ts
```

Starting from the entrypoint (`server.ts` in the example above), Bun builds a registry of all imported source files (excluding those in `node_modules`) and watches them for changes. When a change is detected, Bun performs a "soft reload". All files are re-evaluated, but all global state (notably, the `globalThis` object) is persisted.

```ts#server.ts
// make TypeScript happy
declare global {
  var count: number;
}

globalThis.count ??= 0;
console.log(`Reloaded ${globalThis.count} times`);
globalThis.count++;

// prevent `bun run` from exiting
setInterval(function () {}, 1000000);
```

If you run this file with `bun --hot server.ts`, you'll see the reload count increment every time you save the file.

```bash
$ bun --hot index.ts
Reloaded 1 times
Reloaded 2 times
Reloaded 3 times
```

Traditional file watchers like `nodemon` restart the entire process, so HTTP servers and other stateful objects are lost. By contrast, `bun --hot` is able to reflect the updated code without restarting the process.

### HTTP servers

Bun provides the following simplified API for implementing HTTP servers. Refer to [API > HTTP](/docs/api/http) for full details.

```ts#server.ts
import {type Serve} from "bun";

globalThis.count ??= 0;
globalThis.count++;

export default {
  fetch(req: Request) {
    return new Response(`Reloaded ${globalThis.count} times`);
  },
  port: 3000,
} satisfies Serve;
```

The file above is simply exporting an object with a `fetch` handler defined. When this file is executed, Bun interprets this as an HTTP server and passes the exported object into `Bun.serve`.

Unlike an explicit call to `Bun.serve`, the object-based syntax works out of the box with `bun --hot`. When you save the file, your HTTP server be reloaded with the updated code without the process being restarted. This results in seriously fast refresh speeds.

{% image src="https://user-images.githubusercontent.com/709451/195477632-5fd8a73e-014d-4589-9ba2-e075ad9eb040.gif" alt="Bun vs Nodemon refresh speeds" caption="Bun on the left, Nodemon on the right." /%}

For more fine-grained control, you can use the `Bun.serve` API directly and handle the server reloading manually.

```ts#server.ts
import type {Serve, Server} from "bun";

// make TypeScript happy
declare global {
  var count: number;
  var server: Server;
}

globalThis.count ??= 0;
globalThis.count++;

// define server parameters
const serverOptions: Serve = {
  port: 3000,
  fetch(req) {
    return new Response(`Reloaded ${globalThis.count} times`);
  }
};

if (!globalThis.server) {
  globalThis.server = Bun.serve(serverOptions);
} else {
  // reload server
  globalThis.server.reload(serverOptions);
}
```

{% callout %}
**Note** — In a future version of Bun, support for Vite's `import.meta.hot` is planned to enable better lifecycle management for hot reloading and to align with the ecosystem.

{% /callout %}

{% details summary="Implementation `details`" %}

On hot reload, Bun:

- Resets the internal `require` cache and ES module registry (`Loader.registry`)
- Runs the garbage collector synchronously (to minimize memory leaks, at the cost of runtime performance)
- Re-transpiles all of your code from scratch (including sourcemaps)
- Re-evaluates the code with JavaScriptCore

This implementation isn't particularly optimized. It re-transpiles files that haven't changed. It makes no attempt at incremental compilation. It's a starting point.

{% /details %}


Bun aims for complete Node.js API compatibility. Most `npm` packages intended for `Node.js` environments will work with Bun out of the box; the best way to know for certain is to try it.

This page is updated regularly to reflect compatibility status of the latest version of Bun. If you run into any bugs with a particular package, please [open an issue](https://bun.sh/issues). Opening issues for compatibility bugs helps us prioritize what to work on next.

## Built-in modules

{% block className="ScrollFrame" %}
{% table %}

- Module
- Status
- Notes

---

- {% anchor id="node_assert" %} [`node:assert`](https://nodejs.org/api/assert.html) {% /anchor %}
- 🟢
- Fully implemented.

---

- {% anchor id="node_async_hooks" %} [`node:async_hooks`](https://nodejs.org/api/async_hooks.html) {% /anchor %}
- 🔴
- Not implemented.

---

- {% anchor id="node_buffer" %} [`node:buffer`](https://nodejs.org/api/buffer.html) {% /anchor %}
- 🟡
- Incomplete implementation of `base64` and `base64url` encodings.

---

- {% anchor id="node_child_process" %} [`node:child_process`](https://nodejs.org/api/child_process.html) {% /anchor %}
- 🟡
- Missing IPC, `Stream` stdio, `proc.gid`, `proc.uid`, advanced serialization.

---

- {% anchor id="node_cluster" %} [`node:cluster`](https://nodejs.org/api/cluster.html) {% /anchor %}
- 🔴
- Not implemented.

---

- {% anchor id="node_console" %} [`node:console`](https://nodejs.org/api/console.html) {% /anchor %}
- 🟢
- Recommended to use `console` global instead

---

- {% anchor id="node_crypto" %} [`node:crypto`](https://nodejs.org/api/crypto.html) {% /anchor %}
- 🟡
- Missing `crypto.Certificate` `crypto.ECDH` `crypto.KeyObject` `crypto.X509Certificate` `crypto.checkPrime{Sync}` `crypto.createPrivateKey` `crypto.createPublicKey` `crypto.createSecretKey` `crypto.diffieHellman` `crypto.generateKey{Sync}` `crypto.generateKeyPair{Sync}` `crypto.generatePrime{Sync}` `crypto.getCipherInfo` `crypto.{get|set}Fips` `crypto.hkdf` `crypto.hkdfSync` `crypto.secureHeapUsed` `crypto.setEngine` `crypto.sign` `crypto.verify`

---

- {% anchor id="node_dgram" %} [`node:dgram`](https://nodejs.org/api/dgram.html) {% /anchor %}
- 🔴
- Not implemented.

---

- {% anchor id="node_diagnostics_channel" %} [`node:diagnostics_channel`](https://nodejs.org/api/diagnostics_channel.html) {% /anchor %}
- 🔴
- Not implemented.

---

- {% anchor id="node_dns" %} [`node:dns`](https://nodejs.org/api/dns.html) {% /anchor %}
- 🟢
- Fully implemented.

---

- {% anchor id="node_domain" %} [`node:domain`](https://nodejs.org/api/domain.html) {% /anchor %}
- 🟢
- Fully implemented.

---

- {% anchor id="node_events" %} [`node:events`](https://nodejs.org/api/events.html) {% /anchor %}
- 🟡
- Missing `EventEmitterAsyncResource` `events.on`.

---

- {% anchor id="node_fs" %} [`node:fs`](https://nodejs.org/api/fs.html) {% /anchor %}
- 🟡
- Missing `fs.fdatasync{Sync}` `fs.opendir{Sync}` `fs.{watchFile|unwatchFile}` `fs.{cp|cpSync}`.

---

- {% anchor id="node_http" %} [`node:http`](https://nodejs.org/api/http.html) {% /anchor %}
- 🟢
- Fully implemented.

---

- {% anchor id="node_http2" %} [`node:http2`](https://nodejs.org/api/http2.html) {% /anchor %}
- 🔴
- Not implemented.

---

- {% anchor id="node_https" %} [`node:https`](https://nodejs.org/api/https.html) {% /anchor %}
- 🟢
- Fully implemented.

---

- {% anchor id="node_inspector" %} [`node:inspector`](https://nodejs.org/api/inspector.html) {% /anchor %}
- 🔴
- Not implemented.

---

- {% anchor id="node_module" %} [`node:module`](https://nodejs.org/api/module.html) {% /anchor %}
- 🟢
- Fully implemented.

---

- {% anchor id="node_net" %} [`node:net`](https://nodejs.org/api/net.html) {% /anchor %}
- 🟡
- Missing `net.{get|set}DefaultAutoSelectFamily` `net.SocketAddress` `net.BlockList`.

---

- {% anchor id="node_os" %} [`node:os`](https://nodejs.org/api/os.html) {% /anchor %}
- 🟢
- Fully implemented.

---

- {% anchor id="node_path" %} [`node:path`](https://nodejs.org/api/path.html) {% /anchor %}
- 🟢
- Fully implemented.

---

- {% anchor id="node_perf_hooks" %} [`node:perf_hooks`](https://nodejs.org/api/perf_hooks.html) {% /anchor %}
- 🟡
- Only `perf_hooks.performance.now()` and `perf_hooks.performance.timeOrigin` are implemented. Recommended to use `performance` global instead of `perf_hooks.performance`.

---

- {% anchor id="node_process" %} [`node:process`](https://nodejs.org/api/process.html) {% /anchor %}
- 🟡
- See `Globals > process`.

---

- {% anchor id="node_punycode" %} [`node:punycode`](https://nodejs.org/api/punycode.html) {% /anchor %}
- 🟢
- Fully implemented. _Deprecated by Node.js._

---

- {% anchor id="node_querystring" %} [`node:querystring`](https://nodejs.org/api/querystring.html) {% /anchor %}
- 🟢
- Fully implemented.

---

- {% anchor id="node_readline" %} [`node:readline`](https://nodejs.org/api/readline.html) {% /anchor %}
- 🟢
- Fully implemented.

---

- {% anchor id="node_repl" %} [`node:repl`](https://nodejs.org/api/repl.html) {% /anchor %}
- 🔴
- Not implemented.

---

- {% anchor id="node_stream" %} [`node:stream`](https://nodejs.org/api/stream.html) {% /anchor %}
- 🟢
- Fully implemented.

---

- {% anchor id="node_string_decoder" %} [`node:string_decoder`](https://nodejs.org/api/string_decoder.html) {% /anchor %}
- 🟢
- Fully implemented.

---

- {% anchor id="node_sys" %} [`node:sys`](https://nodejs.org/api/util.html) {% /anchor %}
- 🟡
- See `node:util`.

---

- {% anchor id="node_timers" %} [`node:timers`](https://nodejs.org/api/timers.html) {% /anchor %}
- 🟢
- Recommended to use global `setTimeout`, et. al. instead.

---

- {% anchor id="node_tls" %} [`node:tls`](https://nodejs.org/api/tls.html) {% /anchor %}
- 🟡
- Missing `tls.createSecurePair` `tls.checkServerIdentity` `tls.rootCertificates`

---

- {% anchor id="node_trace_events" %} [`node:trace_events`](https://nodejs.org/api/tracing.html) {% /anchor %}
- 🔴
- Not implemented.

---

- {% anchor id="node_tty" %} [`node:tty`](https://nodejs.org/api/tty.html) {% /anchor %}
- 🟡
- Missing `tty.ReadStream` and `tty.WriteStream`.

---

- {% anchor id="node_url" %} [`node:url`](https://nodejs.org/api/url.html) {% /anchor %}
- 🟡
- Missing `url.domainTo{ASCII|Unicode}`. Recommended to use `URL` and `URLSearchParams` globals instead.

---

- {% anchor id="node_util" %} [`node:util`](https://nodejs.org/api/util.html) {% /anchor %}
- 🟡
- Missing `util.MIMEParams` `util.MIMEType` `util.formatWithOptions()` `util.getSystemErrorMap()` `util.getSystemErrorName()` `util.parseArgs()` `util.stripVTControlCharacters()` `util.transferableAbortController()` `util.transferableAbortSignal()`.

---

- {% anchor id="node_v8" %} [`node:v8`](https://nodejs.org/api/v8.html) {% /anchor %}
- 🔴
- Not implemented or planned. For profiling, use [`bun:jsc`](/docs/project/benchmarking#bunjsc) instead.

---

- {% anchor id="node_vm" %} [`node:vm`](https://nodejs.org/api/vm.html) {% /anchor %}
- 🟡
- Partially implemented.

---

- {% anchor id="node_wasi" %} [`node:wasi`](https://nodejs.org/api/wasi.html) {% /anchor %}
- 🟡
- Partially implemented.

---

- {% anchor id="node_worker_threads" %} [`node:worker_threads`](https://nodejs.org/api/worker_threads.html) {% /anchor %}
- 🔴
- Not implemented.

---

- {% anchor id="node_zlib" %} [`node:zlib`](https://nodejs.org/api/zlib.html) {% /anchor %}
- 🟡
- Missing `zlib.brotli*`

{% /table %}
{% /block %}

## Globals

The table below lists all globals implemented by Node.js and Bun's current compatibility status.

{% table %}

---

- {% anchor id="node_abortcontroller" %} [`AbortController`](https://developer.mozilla.org/en-US/docs/Web/API/AbortController) {% /anchor %}
- 🟢
- Fully implemented.

---

- {% anchor id="node_abortsignal" %} [`AbortSignal`](https://developer.mozilla.org/en-US/docs/Web/API/AbortSignal) {% /anchor %}
- 🟢
- Fully implemented.

---

- {% anchor id="node_blob" %} [`Blob`](https://developer.mozilla.org/en-US/docs/Web/API/Blob) {% /anchor %}
- 🟢
- Fully implemented.

---

- {% anchor id="node_buffer" %} [`Buffer`](https://nodejs.org/api/buffer.html#class-buffer) {% /anchor %}
- 🟡
- Incomplete implementation of `base64` and `base64url` encodings.

---

- {% anchor id="node_bytelengthqueuingstrategy" %} [`ByteLengthQueuingStrategy`](https://developer.mozilla.org/en-US/docs/Web/API/ByteLengthQueuingStrategy) {% /anchor %}
- 🟢
- Fully implemented.

---

- {% anchor id="node_dirname" %} [`__dirname`](https://nodejs.org/api/globals.html#__dirname) {% /anchor %}
- 🟢
- Fully implemented.

---

- {% anchor id="node_filename" %} [`__filename`](https://nodejs.org/api/globals.html#__filename) {% /anchor %}
- 🟢
- Fully implemented.

---

- {% anchor id="node_atob" %} [`atob()`](https://developer.mozilla.org/en-US/docs/Web/API/atob) {% /anchor %}
- 🟢
- Fully implemented.

---

- {% anchor id="node_broadcastchannel" %} [`BroadcastChannel`](https://developer.mozilla.org/en-US/docs/Web/API/BroadcastChannel) {% /anchor %}
- 🔴
- Not implemented.

---

- {% anchor id="node_btoa" %} [`btoa()`](https://developer.mozilla.org/en-US/docs/Web/API/btoa) {% /anchor %}
- 🟢
- Fully implemented.

---

- {% anchor id="node_clearimmediate" %} [`clearImmediate()`](https://developer.mozilla.org/en-US/docs/Web/API/Window/clearImmediate) {% /anchor %}
- 🟢
- Fully implemented.

---

- {% anchor id="node_clearinterval" %} [`clearInterval()`](https://developer.mozilla.org/en-US/docs/Web/API/Window/clearInterval) {% /anchor %}
- 🟢
- Fully implemented.

---

- {% anchor id="node_cleartimeout" %} [`clearTimeout()`](https://developer.mozilla.org/en-US/docs/Web/API/Window/clearTimeout) {% /anchor %}
- 🟢
- Fully implemented.

---

- {% anchor id="node_compressionstream" %} [`CompressionStream`](https://developer.mozilla.org/en-US/docs/Web/API/CompressionStream) {% /anchor %}
- 🔴
- Not implemented.

---

- {% anchor id="node_console" %} [`console`](https://developer.mozilla.org/en-US/docs/Web/API/console) {% /anchor %}
- 🟢
- Fully implemented.

---

- {% anchor id="node_countqueuingstrategy" %} [`CountQueuingStrategy`](https://developer.mozilla.org/en-US/docs/Web/API/CountQueuingStrategy) {% /anchor %}
- 🟢
- Fully implemented.

---

- {% anchor id="node_crypto" %} [`Crypto`](https://developer.mozilla.org/en-US/docs/Web/API/Crypto) {% /anchor %}
- 🟢
- Fully implemented.

---

- {% anchor id="node_crypto" %} [`crypto`](https://developer.mozilla.org/en-US/docs/Web/API/crypto) {% /anchor %}
- 🟢
- Fully implemented.

---

- {% anchor id="node_cryptokey" %} [`CryptoKey`](https://developer.mozilla.org/en-US/docs/Web/API/CryptoKey) {% /anchor %}
- 🟢
- Fully implemented.

---

- {% anchor id="node_customevent" %} [`CustomEvent`](https://developer.mozilla.org/en-US/docs/Web/API/CustomEvent) {% /anchor %}
- 🟢
- Fully implemented.

---

- {% anchor id="node_decompressionstream" %} [`DecompressionStream`](https://developer.mozilla.org/en-US/docs/Web/API/DecompressionStream) {% /anchor %}
- 🔴
- Not implemented.

---

- {% anchor id="node_event" %} [`Event`](https://developer.mozilla.org/en-US/docs/Web/API/Event) {% /anchor %}
- 🟢
- Fully implemented.

---

- {% anchor id="node_eventtarget" %} [`EventTarget`](https://developer.mozilla.org/en-US/docs/Web/API/EventTarget) {% /anchor %}
- 🟢
- Fully implemented.

---

- {% anchor id="node_exports" %} [`exports`](https://nodejs.org/api/globals.html#exports) {% /anchor %}
- 🟢
- Fully implemented.

---

- {% anchor id="node_fetch" %} [`fetch`](https://developer.mozilla.org/en-US/docs/Web/API/fetch) {% /anchor %}
- 🟢
- Fully implemented.

---

- {% anchor id="node_formdata" %} [`FormData`](https://developer.mozilla.org/en-US/docs/Web/API/FormData) {% /anchor %}
- 🟢
- Fully implemented. Added in Bun 0.5.7.

---

- {% anchor id="node_global" %} [`global`](https://nodejs.org/api/globals.html#global) {% /anchor %}
- 🟢
- Implemented. This is an object containing all objects in the global namespace. It's rarely referenced directly, as its contents are available without an additional prefix, e.g. `__dirname` instead of `global.__dirname`.

---

- {% anchor id="node_globalthis" %} [`globalThis`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/globalThis) {% /anchor %}
- 🟢
- Aliases to `global`.

---

- {% anchor id="node_headers" %} [`Headers`](https://developer.mozilla.org/en-US/docs/Web/API/Headers) {% /anchor %}
- 🟢
- Fully implemented.

---

- {% anchor id="node_messagechannel" %} [`MessageChannel`](https://developer.mozilla.org/en-US/docs/Web/API/MessageChannel) {% /anchor %}
- 🔴
- Not implemented.

---

- {% anchor id="node_messageevent" %} [`MessageEvent`](https://developer.mozilla.org/en-US/docs/Web/API/MessageEvent) {% /anchor %}
- 🟢
- Fully implemented.

---

- {% anchor id="node_messageport" %} [`MessagePort`](https://developer.mozilla.org/en-US/docs/Web/API/MessagePort) {% /anchor %}
- 🔴
- Not implemented.

---

- {% anchor id="node_module" %} [`module`](https://nodejs.org/api/globals.html#module) {% /anchor %}
- 🟢
- Fully implemented.

---

- {% anchor id="node_performanceentry" %} [`PerformanceEntry`](https://developer.mozilla.org/en-US/docs/Web/API/PerformanceEntry) {% /anchor %}
- 🔴
- Not implemented.

---

- {% anchor id="node_performancemark" %} [`PerformanceMark`](https://developer.mozilla.org/en-US/docs/Web/API/PerformanceMark) {% /anchor %}
- 🔴
- Not implemented.

---

- {% anchor id="node_performancemeasure" %} [`PerformanceMeasure`](https://developer.mozilla.org/en-US/docs/Web/API/PerformanceMeasure) {% /anchor %}
- 🔴
- Not implemented.

---

- {% anchor id="node_performanceobserver" %} [`PerformanceObserver`](https://developer.mozilla.org/en-US/docs/Web/API/PerformanceObserver) {% /anchor %}
- 🔴
- Not implemented.

---

- {% anchor id="node_performanceobserverentrylist" %} [`PerformanceObserverEntryList`](https://developer.mozilla.org/en-US/docs/Web/API/PerformanceObserverEntryList) {% /anchor %}
- 🔴
- Not implemented.

---

- {% anchor id="node_performanceresourcetiming" %} [`PerformanceResourceTiming`](https://developer.mozilla.org/en-US/docs/Web/API/PerformanceResourceTiming) {% /anchor %}
- 🔴
- Not implemented.

---

- {% anchor id="node_performance" %} [`performance`](https://developer.mozilla.org/en-US/docs/Web/API/performance) {% /anchor %}
- 🟢
- Fully implemented.

---

- {% anchor id="node_process" %} [`process`](https://nodejs.org/api/process.html) {% /anchor %}
- 🟡
- Missing `process.allowedNodeEnvironmentFlags` `process.channel()` `process.connected` `process.constrainedMemory()` `process.disconnect()` `process.getActiveResourcesInfo/setActiveResourcesInfo()` `process.setuid/setgid/setegid/seteuid/setgroups()` `process.hasUncaughtExceptionCaptureCallback` `process.initGroups()` `process.report` `process.resourceUsage()` `process.send()`.

---

- {% anchor id="node_queuemicrotask" %} [`queueMicrotask()`](https://developer.mozilla.org/en-US/docs/Web/API/queueMicrotask) {% /anchor %}
- 🟢
- Fully implemented.

---

- {% anchor id="node_readablebytestreamcontroller" %} [`ReadableByteStreamController`](https://developer.mozilla.org/en-US/docs/Web/API/ReadableByteStreamController) {% /anchor %}
- 🟢
- Fully implemented.

---

- {% anchor id="node_readablestream" %} [`ReadableStream`](https://developer.mozilla.org/en-US/docs/Web/API/ReadableStream) {% /anchor %}
- 🟢
- Fully implemented.

---

- {% anchor id="node_readablestreambyobreader" %} [`ReadableStreamBYOBReader`](https://developer.mozilla.org/en-US/docs/Web/API/ReadableStreamBYOBReader) {% /anchor %}
- 🔴
- Not implemented.

---

- {% anchor id="node_readablestreambyobrequest" %} [`ReadableStreamBYOBRequest`](https://developer.mozilla.org/en-US/docs/Web/API/ReadableStreamBYOBRequest) {% /anchor %}
- 🔴
- Not implemented.

---

- {% anchor id="node_readablestreamdefaultcontroller" %} [`ReadableStreamDefaultController`](https://developer.mozilla.org/en-US/docs/Web/API/ReadableStreamDefaultController) {% /anchor %}
- 🟢
- Fully implemented.

---

- {% anchor id="node_readablestreamdefaultreader" %} [`ReadableStreamDefaultReader`](https://developer.mozilla.org/en-US/docs/Web/API/ReadableStreamDefaultReader) {% /anchor %}
- 🟢
- Fully implemented.

---

- {% anchor id="node_require" %} [`require()`](https://nodejs.org/api/globals.html#require) {% /anchor %}
- 🟢
- Fully implemented, as well as [`require.main`](https://nodejs.org/api/modules.html#requiremain), [`require.cache`](https://nodejs.org/api/modules.html#requirecache), and [`require.resolve`](https://nodejs.org/api/modules.html#requireresolverequest-options)

---

- {% anchor id="node_response" %} [`Response`](https://developer.mozilla.org/en-US/docs/Web/API/Response) {% /anchor %}
- 🟢
- Fully implemented.

---

- {% anchor id="node_request" %} [`Request`](https://developer.mozilla.org/en-US/docs/Web/API/Request) {% /anchor %}
- 🟢
- Fully implemented.

---

- {% anchor id="node_setimmediate" %} [`setImmediate()`](https://developer.mozilla.org/en-US/docs/Web/API/Window/setImmediate) {% /anchor %}
- 🟢
- Fully implemented.

---

- {% anchor id="node_setinterval" %} [`setInterval()`](https://developer.mozilla.org/en-US/docs/Web/API/Window/setInterval) {% /anchor %}
- 🟢
- Fully implemented.

---

- {% anchor id="node_settimeout" %} [`setTimeout()`](https://developer.mozilla.org/en-US/docs/Web/API/Window/setTimeout) {% /anchor %}
- 🟢
- Fully implemented.

---

- {% anchor id="node_structuredclone" %} [`structuredClone()`](https://developer.mozilla.org/en-US/docs/Web/API/structuredClone) {% /anchor %}
- 🔴
- Not implemented.

---

- {% anchor id="node_subtlecrypto" %} [`SubtleCrypto`](https://developer.mozilla.org/en-US/docs/Web/API/SubtleCrypto) {% /anchor %}
- 🟢
- Fully implemented.

---

- {% anchor id="node_domexception" %} [`DOMException`](https://developer.mozilla.org/en-US/docs/Web/API/DOMException) {% /anchor %}
- 🟢
- Fully implemented.

---

- {% anchor id="node_textdecoder" %} [`TextDecoder`](https://developer.mozilla.org/en-US/docs/Web/API/TextDecoder) {% /anchor %}
- 🟢
- Fully implemented.

---

- {% anchor id="node_textdecoderstream" %} [`TextDecoderStream`](https://developer.mozilla.org/en-US/docs/Web/API/TextDecoderStream) {% /anchor %}
- 🔴
- Not implemented.

---

- {% anchor id="node_textencoder" %} [`TextEncoder`](https://developer.mozilla.org/en-US/docs/Web/API/TextEncoder) {% /anchor %}
- 🟢
- Fully implemented.

---

- {% anchor id="node_textencoderstream" %} [`TextEncoderStream`](https://developer.mozilla.org/en-US/docs/Web/API/TextEncoderStream) {% /anchor %}
- 🔴
- Not implemented.

---

- {% anchor id="node_transformstream" %} [`TransformStream`](https://developer.mozilla.org/en-US/docs/Web/API/TransformStream) {% /anchor %}
- 🟢
- Fully implemented.

---

- {% anchor id="node_transformstreamdefaultcontroller" %} [`TransformStreamDefaultController`](https://developer.mozilla.org/en-US/docs/Web/API/TransformStreamDefaultController) {% /anchor %}
- 🟢
- Fully implemented.

---

- {% anchor id="node_url" %} [`URL`](https://developer.mozilla.org/en-US/docs/Web/API/URL) {% /anchor %}
- 🟢
- Fully implemented.

---

- {% anchor id="node_urlsearchparams" %} [`URLSearchParams`](https://developer.mozilla.org/en-US/docs/Web/API/URLSearchParams) {% /anchor %}
- 🟢
- Fully implemented.

---

- {% anchor id="node_webassembly" %} [`WebAssembly`](https://nodejs.org/api/globals.html#webassembly) {% /anchor %}
- 🟢
- Fully implemented.

---

- {% anchor id="node_writablestream" %} [`WritableStream`](https://developer.mozilla.org/en-US/docs/Web/API/WritableStream) {% /anchor %}
- 🟢
- Fully implemented.

---

- {% anchor id="node_writablestreamdefaultcontroller" %} [`WritableStreamDefaultController`](https://developer.mozilla.org/en-US/docs/Web/API/WritableStreamDefaultController) {% /anchor %}
- 🟢
- Fully implemented.

---

- {% anchor id="node_writablestreamdefaultwriter" %} [`WritableStreamDefaultWriter`](https://developer.mozilla.org/en-US/docs/Web/API/WritableStreamDefaultWriter) {% /anchor %}
- 🟢
- Fully implemented.

{% /table %}


Bun is a new JavaScript & TypeScript runtime designed to be a faster, leaner, and more modern drop-in replacement for Node.js.

## Speed

Bun is designed to start fast and run fast. It's transpiler and runtime are written in Zig, a modern, high-performance language. On Linux, this translates into startup times [4x faster](https://twitter.com/jarredsumner/status/1499225725492076544) than Node.js.

{% image src="/images/bun-run-speed.jpeg" caption="Bun vs Node.js vs Deno running Hello World" /%}

<!-- If no `node_modules` directory is found in the working directory or above, Bun will abandon Node.js-style module resolution in favor of the `Bun module resolution algorithm`. Under Bun-style module resolution, all packages are _auto-installed_ on the fly into a [global module cache](/docs/cli/install#global-cache). For full details on this algorithm, refer to [Runtime > Modules](/docs/runtime/modules). -->

Performance sensitive APIs like `Buffer`, `fetch`, and `Response` are heavily profiled and optimized. Under the hood Bun uses the [JavaScriptCore engine](https://developer.apple.com/documentation/javascriptcore), which is developed by Apple for Safari. It starts and runs faster than V8, the engine used by Node.js and Chromium-based browsers.

## TypeScript

Bun natively supports TypeScript out of the box. All files are transpiled on the fly by Bun's fast native transpiler before being executed. Similar to other build tools, Bun does not perform typechecking; it simply removes type annotations from the file.

```bash
$ bun index.js
$ bun index.jsx
$ bun index.ts
$ bun index.tsx
```

Some aspects of Bun's runtime behavior are affected by the contents of your `tsconfig.json` file. Refer to [Runtime > TypeScript](/docs/runtime/typescript) page for details.

<!-- Before execution, Bun internally transforms all source files to vanilla JavaScript using its fast native transpiler. The transpiler looks at the files extension to determine how to handle it. -->

<!--

every file before execution. It's transpiler  can directly run TypeScript and JSX `{.js|.jsx|.ts|.tsx}` files directly. During execution, Bun internally transpiles all files (including `.js` files) to vanilla JavaScript with it's fast native transpiler. -->

<!-- A loader determines how to map imports &amp; file extensions to transforms and output. -->

<!-- Currently, Bun implements the following loaders: -->

<!-- {% table %}

- Extension
- Transforms
- Output (internal)

---

- `.js`
- JSX + JavaScript
- `.js`

---

- `.jsx`
- JSX + JavaScript
- `.js`

---

- `.ts`
- TypeScript + JavaScript
- `.js`

---

- `.tsx`
- TypeScript + JSX + JavaScript
- `.js`

---

- `.mjs`
- JavaScript
- `.js`

---

- `.cjs`
- JavaScript
- `.js`

---

- `.mts`
- TypeScript
- `.js`

---

- `.cts`
- TypeScript
- `.js`


{% /table %} -->

## JSX

## JSON and TOML

Source files can import a `*.json` or `*.toml` file to load its contents as a plain old JavaScript object.

```ts
import pkg from "./package.json";
import bunfig from "./bunfig.toml";
```

## WASM

As of v0.5.2, experimental support exists for WASI, the [WebAssembly System Interface](https://github.com/WebAssembly/WASI). To run a `.wasm` binary with Bun:

```bash
$ bun ./my-wasm-app.wasm
# if the filename doesn't end with ".wasm"
$ bun run ./my-wasm-app.whatever
```

{% callout %}

**Note** — WASI support is based on [wasi-js](https://github.com/sagemathinc/cowasm/tree/main/packages/wasi-js). Currently, it only supports WASI binaries that use the `wasi_snapshot_preview1` or `wasi_unstable` APIs. Bun's implementation is not fully optimized for performance; this will become more of a priority as WASM grows in popularity.
{% /callout %}

## Node.js compatibility

Long-term, Bun aims for complete Node.js compatibility. Most Node.js packages already work with Bun out of the box, but certain low-level APIs like `dgram` are still unimplemented. Track the current compatibility status at [Ecosystem > Node.js](/docs/runtime/nodejs-apis).

Bun implements the Node.js module resolution algorithm, so dependencies can still be managed with `package.json`, `node_modules`, and CommonJS-style imports.

{% callout %}
**Note** — We recommend using Bun's [built-in package manager](/docs/cli/install) for a performance boost over other npm clients.
{% /callout %}

## Web APIs

<!-- When prudent, Bun attempts to implement Web-standard APIs instead of introducing new APIs. Refer to [Runtime > Web APIs](/docs/web-apis) for a list of Web APIs that are available in Bun. -->

Some Web APIs aren't relevant in the context of a server-first runtime like Bun, such as the [DOM API](https://developer.mozilla.org/en-US/docs/Web/API/HTML_DOM_API#html_dom_api_interfaces) or [History API](https://developer.mozilla.org/en-US/docs/Web/API/History_API). Many others, though, are broadly useful outside of the browser context; when possible, Bun implements these Web-standard APIs instead of introducing new APIs.

The following Web APIs are partially or completely supported.

{% table %}

---

- HTTP
- [`fetch`](https://developer.mozilla.org/en-US/docs/Web/API/fetch) [`Response`](https://developer.mozilla.org/en-US/docs/Web/API/Response) [`Request`](https://developer.mozilla.org/en-US/docs/Web/API/Request) [`Headers`](https://developer.mozilla.org/en-US/docs/Web/API/Headers) [`AbortController`](https://developer.mozilla.org/en-US/docs/Web/API/AbortController) [`AbortSignal`](https://developer.mozilla.org/en-US/docs/Web/API/AbortSignal)

---

- URLs
- [`URL`](https://developer.mozilla.org/en-US/docs/Web/API/URL) [`URLSearchParams`](https://developer.mozilla.org/en-US/docs/Web/API/URLSearchParams)

---

- Streams
- [`ReadableStream`](https://developer.mozilla.org/en-US/docs/Web/API/ReadableStream) [`WritableStream`](https://developer.mozilla.org/en-US/docs/Web/API/WritableStream) [`TransformStream`](https://developer.mozilla.org/en-US/docs/Web/API/TransformStream) [`ByteLengthQueuingStrategy`](https://developer.mozilla.org/en-US/docs/Web/API/ByteLengthQueuingStrategy) [`CountQueuingStrategy`](https://developer.mozilla.org/en-US/docs/Web/API/CountQueuingStrategy) and associated classes

---

- Blob
- [`Blob`](https://developer.mozilla.org/en-US/docs/Web/API/Blob)

---

- WebSockets
- [`WebSocket`](https://developer.mozilla.org/en-US/docs/Web/API/WebSocket)

---

- Encoding and decoding
- [`atob`](https://developer.mozilla.org/en-US/docs/Web/API/atob) [`btoa`](https://developer.mozilla.org/en-US/docs/Web/API/btoa) [`TextEncoder`](https://developer.mozilla.org/en-US/docs/Web/API/TextEncoder) [`TextDecoder`](https://developer.mozilla.org/en-US/docs/Web/API/TextDecoder)

---

- Timeouts
- [`setTimeout`](https://developer.mozilla.org/en-US/docs/Web/API/setTimeout) [`clearTimeout`](https://developer.mozilla.org/en-US/docs/Web/API/clearTimeout)

---

- Intervals
- [`setInterval`](https://developer.mozilla.org/en-US/docs/Web/API/setInterval)[`clearInterval`](https://developer.mozilla.org/en-US/docs/Web/API/clearInterval)

---

- Crypto
- [`crypto`](https://developer.mozilla.org/en-US/docs/Web/API/Crypto) [`SubtleCrypto`](https://developer.mozilla.org/en-US/docs/Web/API/SubtleCrypto)
  [`CryptoKey`](https://developer.mozilla.org/en-US/docs/Web/API/CryptoKey)

---

- Debugging

- [`console`](https://developer.mozilla.org/en-US/docs/Web/API/console) [`performance`](https://developer.mozilla.org/en-US/docs/Web/API/Performance)

---

- Microtasks
- [`queueMicrotask`](https://developer.mozilla.org/en-US/docs/Web/API/queueMicrotask)

---

- Errors
- [`reportError`](https://developer.mozilla.org/en-US/docs/Web/API/reportError)

---

- User interaction
- [`alert`](https://developer.mozilla.org/en-US/docs/Web/API/Window/alert) [`confirm`](https://developer.mozilla.org/en-US/docs/Web/API/Window/confirm) [`prompt`](https://developer.mozilla.org/en-US/docs/Web/API/Window/prompt) (intended for interactive CLIs)

<!-- - Blocking. Prints the alert message to terminal and awaits `[ENTER]` before proceeding. -->
<!-- - Blocking. Prints confirmation message and awaits `[y/N]` input from user. Returns `true` if user entered `y` or `Y`, `false` otherwise.
- Blocking. Prints prompt message and awaits user input. Returns the user input as a string. -->

---

- Realms
- [`ShadowRealm`](https://github.com/tc39/proposal-shadowrealm)

---

- Events
- [`EventTarget`](https://developer.mozilla.org/en-US/docs/Web/API/EventTarget)
  [`Event`](https://developer.mozilla.org/en-US/docs/Web/API/Event) [`ErrorEvent`](https://developer.mozilla.org/en-US/docs/Web/API/ErrorEvent) [`CloseEvent`](https://developer.mozilla.org/en-US/docs/Web/API/CloseEvent) [`MessageEvent`](https://developer.mozilla.org/en-US/docs/Web/API/MessageEvent)

---

{% /table %}

## Bun APIs

Bun exposes a set of Bun-specific APIs on the `Bun` global object and through a number of built-in modules. These APIs represent the canonical "Bun-native" way to perform some common development tasks. They are all heavily optimized for performance. Click the link in the left column to view the associated documentation.

{% table %}

- Topic
- APIs

---

- [HTTP](/docs/api/http)
- `Bun.serve`

---

- [File I/O](/docs/api/file-io)
- `Bun.file` `Bun.write`

---

- [Processes](/docs/api/spawn)
- `Bun.spawn` `Bun.spawnSync`

---

- [TCP](/docs/api/tcp)
- `Bun.listen` `Bun.connect`

---

- [Transpiler](/docs/api/transpiler)
- `Bun.Transpiler`

---

- [Routing](/docs/api/file-system-router)
- `Bun.FileSystemRouter`

---

- [HTMLRewriter](/docs/api/html-rewriter)
- `HTMLRewriter`

---

- [Utils](/docs/api/utils)
- `Bun.peek` `Bun.which`

---

- [SQLite](/docs/api/sqlite)
- `bun:sqlite`

---

- [FFI](/docs/api/ffi)
- `bun:ffi`

---

- [DNS](/docs/api/dns)
- `bun:dns`

---

- [Testing](/docs/api/test)
- `bun:test`

---

- [Node-API](/docs/api/node-api)
- `Node-API`

---

{% /table %}

## Plugins

Support for additional file types can be implemented with plugins. Refer to [Runtime > Plugins](/docs/bundler/plugins) for full documentation.


There are two primary mechanisms for configuring the behavior of Bun.

- environment variables
- `bunfig.toml`: Bun's configuration file

Configuring with `bunfig.toml` is optional. Bun aims to be zero-configuration out of the box, but is also highly configurable for advanced use cases. Your `bunfig.toml` should live in your project root alongside `package.json`.

You can also create a global configuration file at the following paths:

- `$HOME/.bunfig.toml`
- `$XDG_CONFIG_HOME/.bunfig.toml`

If both a global and local `bunfig` are detected, the results are shallow-merged, with local overridding global. CLI flags will override `bunfig` setting where applicable.

## Environment variables

These environment variables are checked by Bun to detect functionality and toggle features.

{% table %}

- Name
- Description

---

- `TMPDIR`
- Bun occasionally requires a directory to store intermediate assets during bundling or other operations. If unset, defaults to the platform-specific temporary directory: `/tmp` on Linux, `/private/tmp` on macOS.

---

- `NO_COLOR`
- If `NO_COLOR=1`, then ANSI color output is [disabled](https://no-color.org/).

---

- `FORCE_COLOR`
- If `FORCE_COLOR=1`, then ANSI color output is force enabled, even if `NO_COLOR` is set.

---

- `DO_NOT_TRACK`
- If `DO_NOT_TRACK=1`, then analytics are [disabled](https://do-not-track.dev/). Bun records bundle timings (so we can answer with data, "is Bun getting faster?") and feature usage (e.g., "are people actually using macros?"). The request body size is about 60 bytes, so it's not a lot of data.

{% /table %}

## Runtime

```toml
# scripts to run before `bun run`ning a file or script
# useful for registering plugins
preload = ["./preload.ts"]

# equivalent to corresponding tsconfig compilerOptions
jsx = "react"
jsxFactory = "h"
jsxFragment = "Fragment"
jsxImportSource = "react"

# Set a default framework to use
# By default, Bun will look for an npm package like `bun-framework-${framework}`, followed by `${framework}`
logLevel = "debug"

# publicDir = "public"
# external = ["jquery"]

[define]
# Replace any usage of "process.env.bagel" with the string `lox`.
# The values are parsed as JSON, except single-quoted strings are supported and `'undefined'` becomes `undefined` in JS.
# This will probably change in a future release to be just regular TOML instead. It is a holdover from the CLI argument parsing.
"process.env.bagel" = "'lox'"

[loaders]
# When loading a .bagel file, run the JS parser
".bagel" = "js"
# - "atom"
# If you pass it a file path, it will open with the file path instead
# It will recognize non-GUI editors, but I don't think it will work yet
```

### Debugging

```toml
[debug]
# When navigating to a blob: or src: link, open the file in your editor
# If not, it tries $EDITOR or $VISUAL
# If that still fails, it will try Visual Studio Code, then Sublime Text, then a few others
# This is used by Bun.openInEditor()
editor = "code"

# List of editors:
# - "subl", "sublime"
# - "vscode", "code"
# - "textmate", "mate"
# - "idea"
# - "webstorm"
# - "nvim", "neovim"
# - "vim","vi"
# - "emacs"
```

## Test runner

```toml
[test]
# setup scripts to run before all test files
preload = ["./setup.ts"]
```

## Package manager

Package management is a complex issue; to support a range of use cases, the behavior of `bun install` can be configured in [`bunfig.toml`](/docs/runtime/configuration).

### Default flags

The following settings modify the core behavior of Bun's package management commands. **The default values are shown below.**

```toml
[install]

# whether to install optionalDependencies
optional = true

# whether to install devDependencies
dev = true

# whether to install peerDependencies
peer = false

# equivalent to `--production` flag
production = false

# equivalent to `--frozen-lockfile` flag
frozenLockfile = false

# equivalent to `--dry-run` flag
dryRun = false
```

### Private scopes and registries

The default registry is `https://registry.npmjs.org/`. This can be globally configured in `bunfig.toml`:

```toml
[install]
# set default registry as a string
registry = "https://registry.npmjs.org"
# set a token
registry = { url = "https://registry.npmjs.org", token = "123456" }
# set a username/password
registry = "https://username:password@registry.npmjs.org"
```

To configure scoped registries:

```toml
[install.scopes]
# registry as string
myorg1 = "https://username:password@registry.myorg.com/"

# registry with username/password
# you can reference environment variables
myorg12 = { username = "myusername", password = "$NPM_PASS", url = "https://registry.myorg.com/" }

# registry with token
myorg3 = { token = "$npm_token", url = "https://registry.myorg.com/" }
```

### Cache

To configure caching behavior:

```toml
[install]
# where `bun install --global` installs packages
globalDir = "~/.bun/install/global"

# where globally-installed package bins are linked
globalBinDir = "~/.bun/bin"

[install.cache]
# the directory to use for the cache
dir = "~/.bun/install/cache"

# when true, don't load from the global cache.
# Bun may still write to node_modules/.cache
disable = false

# when true, always resolve the latest versions from the registry
disableManifest = false
```

### Lockfile

To configure lockfile behavior:

```toml
[install.lockfile]

# path to read bun.lockb from
path = "bun.lockb"

# path to save bun.lockb to
savePath = "bun.lockb"

# whether to save the lockfile to disk
save = true

# whether to save a non-Bun lockfile alongside bun.lockb
# only "yarn" is supported
print = "yarn"
```

## Dev server (`bun dev`)

{% The `bun dev` command is likely to change soon and will likely be deprecated in an upcoming release. We recommend %}

Here is an example:

```toml
# Set a default framework to use
# By default, Bun will look for an npm package like `bun-framework-${framework}`, followed by `${framework}`
framework = "next"

[bundle]
saveTo = "node_modules.bun"
# Don't need this if `framework` is set, but showing it here as an example anyway
entryPoints = ["./app/index.ts"]

[bundle.packages]
# If you're bundling packages that do not actually live in a `node_modules` folder or do not have the full package name in the file path, you can pass this to bundle them anyway
"@bigapp/design-system" = true

[dev]
# Change the default port from 3000 to 5000
# Also inherited by Bun.serve
port = 5000

```


Bun supports `.jsx` and `.tsx` files out of the box. Bun's internal transpiler converts JSX syntax into vanilla JavaScript before execution.

```tsx#react.tsx
function Component(props: {message: string}) {
  return (
    <body>
      <h1 style={{color: 'red'}}>{props.message}</h1>
    </body>
  );
}

console.log(<Component message="Hello world!" />);
```

## Configuration

Bun reads your `tsconfig.json` or `jsconfig.json` configuration files to determines how to perform the JSX transform internally. To avoid using either of these, the following options can also be defined in [`bunfig.json`](/docs/runtime/configuration).

The following compiler options are respected.

### [`jsx`](https://www.typescriptlang.org/tsconfig#jsx)

How JSX constructs are transformed into vanilla JavaScript internally. The table below lists the possible values of `jsx`, along with their transpilation of the following simple JSX component:

```tsx
<Box width={5}>Hello</Box>
```

{% table %}

- Compiler options
- Transpiled output

---

- ```json
  {
    "jsx": "react"
  }
  ```

- ```tsx
  import { createElement } from "react";
  createElement("Box", { width: 5 }, "Hello");
  ```

---

- ```json
  {
    "jsx": "react-jsx"
  }
  ```

- ```tsx
  import { jsx } from "react/jsx-runtime";
  jsx("Box", { width: 5 }, "Hello");
  ```

---

- ```json
  {
    "jsx": "react-jsxdev"
  }
  ```

- ```tsx
  import { jsxDEV } from "react/jsx-dev-runtime";
  jsxDEV("Box", { width: 5, children: "Hello" }, undefined, false, undefined, this);
  ```

  The `jsxDEV` variable name is a convention used by React. The `DEV` suffix is a visible way to indicate that the code is intended for use in development. The development version of React is slowers and includes additional validity checks & debugging tools.

---

- ```json
  {
    "jsx": "preserve"
  }
  ```

- ```tsx
  // JSX is not transpiled
  // "preserve" is not supported by Bun currently
  <Box width={5}>Hello</Box>
  ```

{% /table %}

<!-- {% table %}

- `react`
- `React.createElement("Box", {width: 5}, "Hello")`

---

- `react-jsx`
- `jsx("Box", {width: 5}, "Hello")`

---

- `react-jsxdev`
- `jsxDEV("Box", {width: 5}, "Hello", void 0, false)`

---

- `preserve`
- `<Box width={5}>Hello</Box>` Left as-is; not yet supported by Bun.

{% /table %} -->

### [`jsxFactory`](https://www.typescriptlang.org/tsconfig#jsxFactory)

{% callout %}
**Note** — Only applicable when `jsx` is `react`.
{% /callout %}

The function name used to represent JSX constructs. Default value is `"createElement"`. This is useful for libraries like [Preact](https://preactjs.com/) that use a different function name (`"h"`).

{% table %}

- Compiler options
- Transpiled output

---

- ```json
  {
    "jsx": "react",
    "jsxFactory": "h"
  }
  ```

- ```tsx
  import { createhElement } from "react";
  h("Box", { width: 5 }, "Hello");
  ```

{% /table %}

### [`jsxFragmentFactory`](https://www.typescriptlang.org/tsconfig#jsxFragmentFactory)

{% callout %}
**Note** — Only applicable when `jsx` is `react`.
{% /callout %}

The function name used to represent [JSX fragments](https://react.dev/reference/react/Fragment) such as `<>Hello</>`; only applicable when `jsx` is `react`. Default value is `"Fragment"`.

{% table %}

- Compiler options
- Transpiled output

---

- ```json
  {
    "jsx": "react",
    "jsxFactory": "myjsx",
    "jsxFragmentFactory": "MyFragment"
  }
  ```

- ```tsx
  // input
  <>Hello</>;

  // output
  import { myjsx, MyFragment } from "react";
  createElement("Box", { width: 5 }, "Hello");
  ```

{% /table %}

### [`jsxImportSource`](https://www.typescriptlang.org/tsconfig#jsxImportSource)

{% callout %}
**Note** — Only applicable when `jsx` is `react-jsx` or `react-jsxdev`.
{% /callout %}

The module from which the component factory function (`createElement`, `jsx`, `jsxDEV`, etc) will be imported. Default value is `"react"`. This will typically be necessary when using a component library like Preact.

{% table %}

- Compiler options
- Transpiled output

---

- ```jsonc
  {
    "jsx": "react"
    // jsxImportSource is not defined
    // default to "react"
  }
  ```

- ```tsx
  import { jsx } from "react/jsx-runtime";
  jsx("Box", { width: 5, children: "Hello" });
  ```

---

- ```jsonc
  {
    "jsx": "react-jsx",
    "jsxImportSource": "preact"
  }
  ```

- ```tsx
  import { jsx } from "preact/jsx-runtime";
  jsx("Box", { width: 5, children: "Hello" });
  ```

---

- ```jsonc
  {
    "jsx": "react-jsxdev",
    "jsxImportSource": "preact"
  }
  ```

- ```tsx
  // /jsx-runtime is automatically appended
  import { jsxDEV } from "preact/jsx-dev-runtime";
  jsxDEV("Box", { width: 5, children: "Hello" }, undefined, false, undefined, this);
  ```

{% /table %}

### JSX pragma

All of these values can be set on a per-file basis using _pragmas_. A pragma is a special comment that sets a compiler option in a particular file.

{% table %}

- Pragma
- Equivalent config

---

- ```ts
  // @jsx h
  ```

- ```jsonc
  {
    "jsxFactory": "h"
  }
  ```

---

- ```ts
  // @jsxFrag MyFragment
  ```
- ```jsonc
  {
    "jsxFragmentFactory": "MyFragment"
  }
  ```

---

- ```ts
  // @jsxImportSource preact
  ```
- ```jsonc
  {
    "jsxImportSource": "preact"
  }
  ```

{% /table %}

## Logging

Bun implements special logging for JSX to make debugging easier. Given the following file:

```tsx#index.tsx
import { Stack, UserCard } from "./components";

console.log(
  <Stack>
    <UserCard name="Dom" bio="Street racer and Corona lover" />
    <UserCard name="Jakob" bio="Super spy and Dom's secret brother" />
  </Stack>
);
```

Bun will pretty-print the component tree when logged:

{% image src="https://github.com/oven-sh/bun/assets/3084745/d29db51d-6837-44e2-b8be-84fc1b9e9d97" / %}

## Prop punning

The Bun runtime also supports "prop punning" for JSX. This is a shorthand syntax useful for assigning a variable to a prop with the same name.

```tsx
function Div(props: {className: string;}) {
  const {className} = props;

  // without punning
  return <div className={className} />;
  // with punning
  return <div {className} />;
}
```


Bun treats TypeScript as a first-class citizen.

## Running `.ts` files

Bun can directly execute `.ts` and `.tsx` files just like vanilla JavaScript, with no extra configuration. If you import a `.ts` or `.tsx` file (or an `npm` module that exports these files), Bun internally transpiles it into JavaScript then executes the file.

**Note** — Similar to other build tools, Bun does not typecheck the files. Use [`tsc`](https://www.typescriptlang.org/docs/handbook/compiler-options.html) (the official TypeScript CLI) if you're looking to catch static type errors.

{% callout %}

**Is transpiling still necessary?** — Because Bun can directly execute TypeScript, you may not need to transpile your TypeScript to run in production. Bun internally transpiles every file it executes (both `.js` and `.ts`), so the additional overhead of directly executing your `.ts/.tsx` source files is negligible.

That said, if you are using Bun as a development tool but still targeting Node.js or browsers in production, you'll still need to transpile.

{% /callout %}

## Configuring `tsconfig.json`

Bun supports a number of features that TypeScript doesn't support by default, such as extensioned imports, top-level await, and `exports` conditions. It also implements global APIs like the `Bun`. To enable these features, your `tsconfig.json` must be configured properly.

{% callout %}
If you initialized your project with `bun init`, everything is already configured properly.
{% /callout %}

To get started, install the `bun-types` package.

```sh
$ bun add -d bun-types # dev dependency
```

If you're using a canary build of Bun, use the `canary` tag. The canary package is updated on every commit to the `main` branch.

```sh
$ bun add -d bun-types@canary
```

<!-- ### Quick setup

{% callout %}

**Note** — This approach requires TypeScript 5.0 or later!

{% /callout %}

Add the following to your `tsconfig.json`.

```json-diff
  {
+   "extends": ["bun-types"]
    // other options...
  }
```

{% callout %}
**Note** — The `"extends"` field in your `tsconfig.json` can accept an array of values. If you're already using `"extends"`, just add `"bun-types"` to the array.
{% /callout %}

That's it! You should be able to use Bun's full feature set without seeing any TypeScript compiler errors.

### Manual setup -->

### Recommended `compilerOptions`

These are the recommended `compilerOptions` for a Bun project.

```jsonc
{
  "compilerOptions": {
    // add Bun type definitions
    "types": ["bun-types"],

    // enable latest features
    "lib": ["esnext"],
    "module": "esnext",
    "target": "esnext",

    // if TS 5.x+
    "moduleResolution": "bundler",
    "allowImportingTsExtensions": true,
    "moduleDetection": "force",
    // if TS 4.x or earlier
    "moduleResolution": "nodenext",

    "jsx": "react-jsx", // support JSX
    "allowJs": true, // allow importing `.js` from `.ts`
    "esModuleInterop": true, // allow default imports for CommonJS modules

    // best practices
    "strict": true,
    "forceConsistentCasingInFileNames": true,
    "skipLibCheck": true
  }
}
```

### Add DOM types

Settings `"types": ["bun-types"]` means TypeScript will ignore other global type definitions, including `lib: ["dom"]`. To add DOM types into your project, add the following [triple-slash directives](https://www.typescriptlang.org/docs/handbook/triple-slash-directives.html) at the top of any TypeScript file in your project.

```ts
/// <reference lib="dom" />
/// <reference lib="dom.iterable" />
```

The same applies to other global type definition _libs_ like `webworker`.

## Path mapping

When resolving modules, Bun's runtime respects path mappings defined in [`compilerOptions.paths`](https://www.typescriptlang.org/tsconfig#paths) in your `tsconfig.json`. No other runtime does this.

Given the following `tsconfig.json`...

```json
{
  "compilerOptions": {
    "paths": {
      "data": ["./data.ts"]
    }
  }
}
```

...the import from `"data"` will work as expected.

{% codetabs %}

```ts#index.ts
import { foo } from "data";
console.log(foo); // => "Hello world!"
```

```ts#data.ts
export const foo = "Hello world!"
```

{% /codetabs %}


## TypeScript

Bun natively supports TypeScript out of the box. All files are transpiled on the fly by Bun's fast native transpiler before being executed. Similar to other build tools, Bun does not perform typechecking; it simply removes type annotations from the file.

```bash
$ bun index.js
$ bun index.jsx
$ bun index.ts
$ bun index.tsx
```

Some aspects of Bun's runtime behavior are affected by the contents of your `tsconfig.json` file. Refer to [Runtime > TypeScript](/docs/runtime/typescript) page for details.

## JSX

Bun supports `.jsx` and `.tsx` files out of the box. Bun's internal transpiler converts JSX syntax into vanilla JavaScript before execution.

```tsx#react.tsx
function Component(props: {message: string}) {
  return (
    <body>
      <h1 style={{color: 'red'}}>{props.message}</h1>
    </body>
  );
}

console.log(<Component message="Hello world!" />);
```

Bun implements special logging for JSX to make debugging easier.

```bash
$ bun run react.tsx
<Component message="Hello world!" />
```

## Text files

{% callout %}
Supported in Bun v0.6.0 canary.
{% /callout %}

Text files can be imported as strings.

{% codetabs %}

```ts#index.ts
import text from "./text.txt";
console.log(text);
// => "Hello world!"
```

```txt#text.txt
Hello world!
```

{% /codetabs %}

## JSON and TOML

JSON and TOML files can be directly imported from a source file. The contents will be loaded and returned as a JavaScript object.

```ts
import pkg from "./package.json";
import data from "./data.toml";
```

## WASM

As of v0.5.2, experimental support exists for WASI, the [WebAssembly System Interface](https://github.com/WebAssembly/WASI). To run a `.wasm` binary with Bun:

```bash
$ bun ./my-wasm-app.wasm
# if the filename doesn't end with ".wasm"
$ bun run ./my-wasm-app.whatever
```

{% callout %}

**Note** — WASI support is based on [wasi-js](https://github.com/sagemathinc/cowasm/tree/main/packages/wasi-js). Currently, it only supports WASI binaries that use the `wasi_snapshot_preview1` or `wasi_unstable` APIs. Bun's implementation is not fully optimized for performance; this will become more of a priority as WASM grows in popularity.
{% /callout %}

## Custom loaders

Support for additional file types can be implemented with plugins. Refer to [Runtime > Plugins](/docs/bundler/plugins) for full documentation.

<!--

A loader determines how to map imports &amp; file extensions to transforms and output.

Currently, Bun implements the following loaders:

| Input | Loader                        | Output |
| ----- | ----------------------------- | ------ |
| .js   | JSX + JavaScript              | .js    |
| .jsx  | JSX + JavaScript              | .js    |
| .ts   | TypeScript + JavaScript       | .js    |
| .tsx  | TypeScript + JSX + JavaScript | .js    |
| .mjs  | JavaScript                    | .js    |
| .cjs  | JavaScript                    | .js    |
| .mts  | TypeScript                    | .js    |
| .cts  | TypeScript                    | .js    |
| .toml | TOML                          | .js    |
| .css  | CSS                           | .css   |
| .env  | Env                           | N/A    |
| .\*   | file                          | string |

Everything else is treated as `file`. `file` replaces the import with a URL (or a path).

You can configure which loaders map to which extensions by passing `--loaders` to `bun`. For example:

```sh
$ bun --loader=.js:js
```

This will disable JSX transforms for `.js` files. -->


# RFCs

| Number | Name | Issue |
| ------ | ---- | ----- |


`bun init` is a quick way to start a blank project with Bun. It guesses with sane defaults and is non-destructive when run multiple times.

![Demo](https://user-images.githubusercontent.com/709451/183006613-271960a3-ff22-4f7c-83f5-5e18f684c836.gif)

It creates:

- a `package.json` file with a name that defaults to the current directory name
- a `tsconfig.json` file or a `jsconfig.json` file, depending if the entry point is a TypeScript file or not
- an entry point which defaults to `index.ts` unless any of `index.{tsx, jsx, js, mts, mjs}` exist or the `package.json` specifies a `module` or `main` field
- a `README.md` file

If you pass `-y` or `--yes`, it will assume you want to continue without asking questions.

At the end, it runs `bun install` to install `bun-types`.

Added in Bun v0.1.7.

#### How is `bun init` different than `bun create`?

`bun init` is for blank projects. `bun create` applies templates.


To upgrade Bun, run `bun upgrade`.

It automatically downloads the latest version of Bun and overwrites the currently-running version.

This works by checking the latest version of Bun in [bun-releases-for-updater](https://github.com/Jarred-Sumner/bun-releases-for-updater/releases) and unzipping it using the system-provided `unzip` library (so that Gatekeeper works on macOS)

If for any reason you run into issues, you can also use the curl install script:

```bash
$ curl https://bun.sh/install | bash
```

It will still work when Bun is already installed.

Bun is distributed as a single binary file, so you can also do this manually:

- Download the latest version of Bun for your platform in [bun-releases-for-updater](https://github.com/Jarred-Sumner/bun-releases-for-updater/releases/latest) (`darwin` == macOS)
- Unzip the folder
- Move the `bun` binary to `~/.bun/bin` (or anywhere)

## `--canary`

[Canary](https://github.com/oven-sh/bun/releases/tag/canary) builds are generated on every commit.

To install a [canary](https://github.com/oven-sh/bun/releases/tag/canary) build of Bun, run:

```bash
$ bun upgrade --canary
```

This flag is not persistent (though that might change in the future). If you want to always run the canary build of Bun, set the `BUN_CANARY` environment variable to `1` in your shell's startup script.

This will download the release zip from https://github.com/oven-sh/bun/releases/tag/canary.

To revert to the latest published version of Bun, run:

```bash
$ bun upgrade
```


The `bun` CLI contains a Node.js-compatible package manager designed to be a dramatically faster replacement for `npm`, `yarn`, and `pnpm`. It's a standalone tool that will work in pre-existing Node.js projects; if your project has a `package.json`, `bun install` can help you speed up your workflow.

{% callout %}

**⚡️ 25x faster** — Switch from `npm install` to `bun install` in any Node.js project to make your installations up to 25x faster.

{% image src="https://user-images.githubusercontent.com/709451/147004342-571b6123-17a9-49a2-8bfd-dcfc5204047e.png" height="200" /%}

{% /callout %}

{% details summary="For Linux users" %}
The minimum Linux Kernel version is 5.1. If you're on Linux kernel 5.1 - 5.5, `bun install` should still work, but HTTP requests will be slow due to a lack of support for io_uring's `connect()` operation.

If you're using Ubuntu 20.04, here's how to install a [newer kernel](https://wiki.ubuntu.com/Kernel/LTSEnablementStack):

```bash
# If this returns a version >= 5.6, you don't need to do anything
uname -r

# Install the official Ubuntu hardware enablement kernel
sudo apt install --install-recommends linux-generic-hwe-20.04
```

{% /details %}

## Manage dependencies

### `bun install`

To install all dependencies of a project:

```bash
$ bun install
```

On Linux, `bun install` tends to install packages 20-100x faster than `npm install`. On macOS, it's more like 4-80x.

![package install benchmark](https://user-images.githubusercontent.com/709451/147004342-571b6123-17a9-49a2-8bfd-dcfc5204047e.png)

Running `bun install` will:

- **Install** all `dependencies`, `devDependencies`, and `optionalDependencies`. Bun does not install `peerDependencies` by default.
- **Run** your project's `{pre|post}install` scripts at the appropriate time. For security reasons Bun _does not execute_ lifecycle scripts of installed dependencies.
- **Write** a `bun.lockb` lockfile to the project root.

To install in production mode (i.e. without `devDependencies`):

```bash
$ bun install --production
```

To install with reproducible dependencies, use `--frozen-lockfile`. If your `package.json` disagrees with `bun.lockb`, Bun will exit with an error. This is useful for production builds and CI environments.

```bash
$ bun install --frozen-lockfile
```

To perform a dry run (i.e. don't actually install anything):

```bash
$ bun install --dry-run
```

To modify logging verbosity:

```bash
$ bun install --verbose # debug logging
$ bun install --silent  # no logging
```

{% details summary="Configuring behavior" %}
The default behavior of `bun install` can be configured in `bun.toml`:

```toml
[install]

# whether to install optionalDependencies
optional = true

# whether to install devDependencies
dev = true

# whether to install peerDependencies
peer = false

# equivalent to `--production` flag
production = false

# equivalent to `--frozen-lockfile` flag
frozenLockfile = false

# equivalent to `--dry-run` flag
dryRun = false
```

{% /details %}

### `bun add`

To add a particular package:

```bash
$ bun add preact
```

To specify a version, version range, or tag:

```bash
$ bun add zod@3.20.0
$ bun add zod@^3.0.0
$ bun add zod@latest
```

To add a package as a dev dependency (`"devDependencies"`):

```bash
$ bun add --development @types/react
$ bun add -d @types/react
```

To add a package as an optional dependency (`"optionalDependencies"`):

```bash
$ bun add --optional lodash
```

To add a package and pin to the resolved version, use `--exact`. This will resolve the version of the package and add it to your `package.json` with an exact version number instead of a version range.

```bash
$ bun add react --exact
```

This will add the following to your `package.json`:

```jsonc
{
  "dependencies": {
    // without --exact
    "react": "^18.2.0", // this matches >= 18.2.0 < 19.0.0

    // with --exact
    "react": "18.2.0" // this matches only 18.2.0 exactly
  }
}
```

To install a package globally:

```bash
$ bun add --global cowsay # or `bun add -g cowsay`
$ cowsay "Bun!"
 ______
< Bun! >
 ------
        \   ^__^
         \  (oo)\_______
            (__)\       )\/\
                ||----w |
                ||     ||
```

{% details summary="Configuring global installation behavior" %}

```toml
[install]
# where `bun install --global` installs packages
globalDir = "~/.bun/install/global"

# where globally-installed package bins are linked
globalBinDir = "~/.bun/bin"
```

{% /details %}
To view a complete list of options for a given command:

```bash
$ bun add --help
```

### `bun remove`

To remove a dependency:

```bash
$ bun remove preact
```

## Local packages (`bun link`)

Use `bun link` in a local directory to register the current package as a "linkable" package.

```bash
$ cd /path/to/cool-pkg
$ cat package.json
{
  "name": "cool-pkg",
  "version": "1.0.0"
}
$ bun link
bun link v0.5.7 (7416672e)
Success! Registered "cool-pkg"

To use cool-pkg in a project, run:
  bun link cool-pkg

Or add it in dependencies in your package.json file:
  "cool-pkg": "link:cool-pkg"
```

This package can now be "linked" into other projects using `bun link cool-pkg`. This will create a symlink in the `node_modules` directory of the target project, pointing to the local directory.

```bash
$ cd /path/to/my-app
$ bun link cool-pkg
```

In addition, the `--save` flag can be used to add `cool-pkg` to the `dependencies` field of your app's package.json with a special version specifier that tells Bun to load from the registered local directory instead of installing from `npm`:

```json-diff
  {
    "name": "my-app",
    "version": "1.0.0",
    "dependencies": {
+     "cool-pkg": "link:cool-pkg"
    }
  }
```

## Trusted dependencies

Unlike other npm clients, Bun does not execute arbitrary lifecycle scripts for installed dependencies, such as `postinstall`. These scripts represent a potential security risk, as they can execute arbitrary code on your machine.

<!-- Bun maintains an allow-list of popular packages containing `postinstall` scripts that are known to be safe. To run lifecycle scripts for packages that aren't on this list, add the package to `trustedDependencies` in your package.json. -->

To tell Bun to allow lifecycle scripts for a particular package, add the package to `trustedDependencies` in your package.json.

<!-- ```json-diff
  {
    "name": "my-app",
    "version": "1.0.0",
+   "trustedDependencies": {
+     "my-trusted-package": "*"
+   }
  }
``` -->

```json-diff
  {
    "name": "my-app",
    "version": "1.0.0",
+   "trustedDependencies": ["my-trusted-package"]
  }
```

Bun reads this field and will run lifecycle scripts for `my-trusted-package`.

<!-- If you specify a version range, Bun will only execute lifecycle scripts if the resolved package version matches the range. -->
<!--
```json
{
  "name": "my-app",
  "version": "1.0.0",
  "trustedDependencies": {
    "my-trusted-package": "^1.0.0"
  }
}
``` -->

## Git dependencies

To add a dependency from a git repository:

```bash
$ bun install git@github.com:moment/moment.git
```

Bun supports a variety of protocols, including [`github`](https://docs.npmjs.com/cli/v9/configuring-npm/package-json#github-urls), [`git`](https://docs.npmjs.com/cli/v9/configuring-npm/package-json#git-urls-as-dependencies), `git+ssh`, `git+https`, and many more.

```json
{
  "dependencies": {
    "dayjs": "git+https://github.com/iamkun/dayjs.git",
    "lodash": "git+ssh://github.com/lodash/lodash.git#4.17.21",
    "moment": "git@github.com:moment/moment.git",
    "zod": "github:colinhacks/zod"
  }
}
```

## Tarball dependencies

A package name can correspond to a publically hosted `.tgz` file. During `bun install`, Bun will download and install the package from the specified tarball URL, rather than from the package registry.

```json#package.json
{
  "dependencies": {
    "zod": "https://registry.npmjs.org/zod/-/zod-3.21.4.tgz"
  }
}
```

## CI/CD

Looking to speed up your CI? Use the official `oven-sh/setup-bun` action to install `bun` in a GitHub Actions pipeline.

```yaml#.github/workflows/release.yml
name: bun-types
jobs:
  build:
    name: build-app
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repo
        uses: actions/checkout@v3
      - name: Install bun
        uses: oven-sh/setup-bun@v1
      - name: Install dependencies
        run: bun install
      - name: Build app
        run: bun run build
```


{% callout %}
**Note** — `bunx` is an alias for `bun x`. The `bunx` CLI will be auto-installed when you install `bun`.
{% /callout %}

Use `bunx` to auto-install and run packages from `npm`. It's Bun's equivalent of `npx` or `yarn dlx`.

```bash
$ bunx cowsay "Hello world!"
```

{% callout %}
⚡️ **Speed** — With Bun's fast startup times, `bunx` is [roughly 100x faster](https://twitter.com/jarredsumner/status/1606163655527059458) than `npx` for locally installed packages.
{% /callout %}

Packages can declare executables in the `"bin"` field of their `package.json`. These are known as _package executables_ or _package binaries_.

```jsonc#package.json
{
  // ... other fields
  "name": "my-cli",
  "bin": {
    "my-cli": "dist/index.js"
  }
}
```

These executables are commonly plain JavaScript files marked with a [shebang line](<https://en.wikipedia.org/wiki/Shebang_(Unix)>) to indicate which program should be used to execute them. The following file indicates that it should be executed with `node`.

```js#dist/index.js
#!/usr/bin/env node

console.log("Hello world!");
```

These executables can be run with `bunx`,

```bash
$ bunx my-cli
```

As with `npx`, `bunx` will check for a locally installed package first, then fall back to auto-installing the package from `npm`. Installed packages will be stored in Bun's global cache for future use.

## Arguments and flags

To pass additional command-line flags and arguments through to the executable, place them after the executable name.

```bash
$ bunx my-cli --foo bar
```

## Shebangs

By default, Bun respects shebangs. If an executable is marked with `#!/usr/bin/env node`, Bun will spin up a `node` process to execute the file. However, in some cases it may be desirable to run executables using Bun's runtime, even if the executable indicates otherwise. To do so, include the `--bun` flag.

```bash
$ bunx --bun my-cli
```

The `--bun` flag must occur _before_ the executable name. Flags that appear _after_ the name are passed through to the executable.

```bash
$ bunx --bun my-cli # good
$ bunx my-cli --bun # bad
```

<!-- ## Environment variables

Bun automatically loads environment variables from `.env` files before running a file, script, or executable. The following files are checked, in order:

1. `.env.local` (first)
2. `NODE_ENV` === `"production"` ? `.env.production` : `.env.development`
3. `.env`

To debug environment variables, run `bun run env` to view a list of resolved environment variables. -->


In your project folder root (where `package.json` is):

```bash
$ bun bun ./entry-point-1.js ./entry-point-2.jsx
$ bun dev
```

By default, `bun dev` will look for any HTML files in the `public` directory and serve that. For browsers navigating to the page, the `.html` file extension is optional in the URL, and `index.html` will automatically rewrite for the directory.

Here are examples of routing from `public/` and how they’re matched:
| Dev Server URL | File Path |
|----------------|-----------|
| /dir | public/dir/index.html |
| / | public/index.html |
| /index | public/index.html |
| /hi | public/hi.html |
| /file | public/file.html |
| /font/Inter.woff2 | public/font/Inter.woff2 |
| /hello | public/index.html |

If `public/index.html` exists, it becomes the default page instead of a 404 page, unless that pathname has a file extension.


## `bun init`

Scaffold an empty project with `bun init`. It's an interactive tool.

```bash
$ bun init
bun init helps you get started with a minimal project and tries to
guess sensible defaults. Press ^C anytime to quit.

package name (quickstart):
entry point (index.ts):

Done! A package.json file was saved in the current directory.
 + index.ts
 + .gitignore
 + tsconfig.json (for editor auto-complete)
 + README.md

To get started, run:
  bun run index.ts
```

Press `enter` to accept the default answer for each prompt, or pass the `-y` flag to auto-accept the defaults.

## `bun create`

Template a new Bun project with `bun create`.

```bash
$ bun create <template> <destination>
```

{% callout %}
**Note** — You don’t need `bun create` to use Bun. You don’t need any configuration at all. This command exists to make getting started a bit quicker and easier.
{% /callout %}

A template can take a number of forms:

```bash
$ bun create <template>         # an official template (remote)
$ bun create <username>/<repo>  # a GitHub repo (remote)
$ bun create <local-template>   # a custom template (local)
```

Running `bun create` performs the following steps:

- Download the template (remote templates only)
- Copy all template files into the destination folder. By default Bun will _not overwrite_ any existing files. Use the `--force` flag to overwrite existing files.
- Install dependencies with `bun install`.
- Initialize a fresh Git repo. Opt out with the `--no-git` flag.
- Run the template's configured `start` script, if defined.

## Official templates

The following official templates are available.

```bash
bun create next ./myapp
bun create react ./myapp
bun create svelte-kit ./myapp
bun create elysia ./myapp
bun create hono ./myapp
bun create kingworld ./myapp
```

Each of these corresponds to a directory in the [bun-community/create-templates](https://github.com/bun-community/create-templates) repo. If you think a major framework is missing, please open a PR there. This list will change over time as additional examples are added. To see an up-to-date list, run `bun create` with no arguments.

```bash
$ bun create
Welcome to bun! Create a new project by pasting any of the following:
  <list of templates>
```

{% callout %}
⚡️ **Speed** — At the time of writing, `bun create react app` runs ~11x faster on a M1 Macbook Pro than `yarn create react-app app`.
{% /callout %}

## GitHub repos

A template of the form `<username>/<repo>` will be downloaded from GitHub.

```bash
$ bun create ahfarmer/calculator ./myapp
```

Complete GitHub URLs will also work:

```bash
$ bun create github.com/ahfarmer/calculator ./myapp
$ bun create https://github.com/ahfarmer/calculator ./myapp
```

Bun installs the files as they currently exist current default branch (usually `main`). Unlike `git clone` it doesn't download the commit history or configure a remote.

## Local templates

{% callout %}
**⚠️ Warning** — Unlike remote templates, running `bun create` with a local template will delete the entire destination folder if it already exists! Be careful.
{% /callout %}
Bun's templater can be extended to support custom templates defined on your local file system. These templates should live in one of the following directories:

- `$HOME/.bun-create/<name>`: global templates
- `<project root>/.bun-create/<name>`: project-specific templates

{% callout %}
**Note** — You can customize the global template path by setting the `BUN_CREATE_DIR` environment variable.
{% /callout %}

To create a local template, navigate to `$HOME/.bun-create` and create a new directory with the desired name of your template.

```bash
$ cd $HOME/.bun-create
$ mkdir foo
$ cd foo
```

Then, create a `package.json` file in that directory with the following contents:

```json
{
  "name": "foo"
}
```

You can run `bun create foo` elsewhere on your file system to verify that Bun is correctly finding your local template.

{% table %}

---

- `postinstall`
- runs after installing dependencies

---

- `preinstall`
- runs before installing dependencies

<!-- ---

- `start`
- a command to auto-start the application -->

{% /table %}

Each of these can correspond to a string or array of strings. An array of commands will be executed in order. Here is an example:

```json
{
  "name": "@bun-examples/simplereact",
  "version": "0.0.1",
  "main": "index.js",
  "dependencies": {
    "react": "^17.0.2",
    "react-dom": "^17.0.2"
  },
  "bun-create": {
    "preinstall": "echo 'Installing...'", // a single command
    "postinstall": ["echo 'Done!'"], // an array of commands
    "start": "bun run echo 'Hello world!'"
  }
}
```

When cloning a template, `bun create` will automatically remove the `"bun-create"` section from `package.json` before writing it to the destination folder.

## Reference

### CLI flags

{% table %}

- Flag
- Description

---

- `--force`
- Overwrite existing files

---

- `--no-install`
- Skip installing `node_modules` & tasks

---

- `--no-git`
- Don’t initialize a git repository

---

- `--open`
- Start & open in-browser after finish

{% /table %}

### Environment variables

{% table %}

- Name
- Description

---

- `GITHUB_API_DOMAIN`
- If you’re using a GitHub enterprise or a proxy, you can customize the GitHub domain Bun pings for downloads

---

- `GITHUB_API_TOKEN`
- This lets `bun create` work with private repositories or if you get rate-limited

{% /table %}

{% details summary="How `bun create` works" %}

When you run `bun create ${template} ${destination}`, here’s what happens:

IF remote template

1. GET `registry.npmjs.org/@bun-examples/${template}/latest` and parse it
2. GET `registry.npmjs.org/@bun-examples/${template}/-/${template}-${latestVersion}.tgz`
3. Decompress & extract `${template}-${latestVersion}.tgz` into `${destination}`

   - If there are files that would overwrite, warn and exit unless `--force` is passed

IF GitHub repo

1. Download the tarball from GitHub’s API
2. Decompress & extract into `${destination}`

   - If there are files that would overwrite, warn and exit unless `--force` is passed

ELSE IF local template

1. Open local template folder
2. Delete destination directory recursively
3. Copy files recursively using the fastest system calls available (on macOS `fcopyfile` and Linux, `copy_file_range`). Do not copy or traverse into `node_modules` folder if exists (this alone makes it faster than `cp`)

4. Parse the `package.json` (again!), update `name` to be `${basename(destination)}`, remove the `bun-create` section from the `package.json` and save the updated `package.json` to disk.
   - IF Next.js is detected, add `bun-framework-next` to the list of dependencies
   - IF Create React App is detected, add the entry point in /src/index.{js,jsx,ts,tsx} to `public/index.html`
   - IF Relay is detected, add `bun-macro-relay` so that Relay works
5. Auto-detect the npm client, preferring `pnpm`, `yarn` (v1), and lastly `npm`
6. Run any tasks defined in `"bun-create": { "preinstall" }` with the npm client
7. Run `${npmClient} install` unless `--no-install` is passed OR no dependencies are in package.json
8. Run any tasks defined in `"bun-create": { "preinstall" }` with the npm client
9. Run `git init; git add -A .; git commit -am "Initial Commit";`

   - Rename `gitignore` to `.gitignore`. NPM automatically removes `.gitignore` files from appearing in packages.
   - If there are dependencies, this runs in a separate thread concurrently while node_modules are being installed
   - Using libgit2 if available was tested and performed 3x slower in microbenchmarks

{% /details %}


Bundling is currently an important mechanism for building complex web apps.

Modern apps typically consist of a large number of files and package dependencies. Despite the fact that modern browsers support [ES Module](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Modules) imports, it's still too slow to fetch each file via inidividual HTTP requests. _Bundling_ is the process of concatenating several source files into a single large file that can be loaded in a single request.

{% callout %}
**On bundling** — Despite recent advances like [`modulepreload`](https://developer.mozilla.org/en-US/docs/Web/HTML/Attributes/rel/modulepreload) and [HTTP/3](https://en.wikipedia.org/wiki/HTTP/3), bundling is still the most performant approach.
{% /callout %}

## Bundling your app

Bun's approach to bundling is a little different from other bundlers. Start by passing your app's entrypoint to `bun bun`.

```bash
$ bun bun ./app.js
```

Your entrypoint can be any `js|jsx|ts|tsx|html` file. With this file as a starting point, Bun will construct a graph of imported files and packages, transpile everything, and generate a file called `node_modules.bun`.

## What is `.bun`?

{% callout %}
**Note** — [This format may change soon](https://github.com/oven-sh/bun/issues/121)
{% /callout %}

A `.bun` file contains the pre-transpiled source code of your application, plus a bunch of binary-encoded metadata about your application's structure. as a contains:

- all the bundled source code
- all the bundled source code metadata
- project metadata & configuration

Here are some of the questions `.bun` files answer:

- when I import `react/index.js`, where in the `.bun` is the code for that? (not resolving, just the code)
- what modules of a package are used?
- what framework is used? (e.g., Next.js)
- where is the routes directory?
- how big is each imported dependency?
- what is the hash of the bundle’s contents? (for etags)
- what is the name & version of every npm package exported in this bundle?
- what modules from which packages are used in this project? ("project" is defined as all the entry points used to generate the .bun)

All in one file.

It’s a little like a build cache, but designed for reuse across builds.

{% details summary="Position-independent code" %}

From a design perspective, the most important part of the `.bun` format is how code is organized. Each module is exported by a hash like this:

```js
// preact/dist/preact.module.js
export var $eb6819b = $$m({
  "preact/dist/preact.module.js": (module, exports) => {
    let n, l, u, i, t, o, r, f, e = {}, c = [], s = /acit|ex(?:s|g|n|p|$)|rph|grid|ows|mnc|ntw|ine[ch]|zoo|^ord|itera/i;
    // ... rest of code
```

This makes bundled modules [position-independent](https://en.wikipedia.org/wiki/Position-independent_code). In theory, one could import only the exact modules in-use without reparsing code and without generating a new bundle. One bundle can dynamically become many bundles comprising only the modules in use on the webpage. Thanks to the metadata with the byte offsets, a web server can send each module to browsers [zero-copy](https://en.wikipedia.org/wiki/Zero-copy) using [sendfile](https://man7.org/linux/man-pages/man2/sendfile.2.html). Bun itself is not quite this smart yet, but these optimizations would be useful in production and potentially very useful for React Server Components.

To see the schema inside, have a look at [`JavascriptBundleContainer`](./src/api/schema.d.ts#:~:text=export%20interface-,JavascriptBundleContainer,-%7B). You can find JavaScript bindings to read the metadata in [src/api/schema.js](./src/api/schema.js). This is not really an API yet. It’s missing the part where it gets the binary data from the bottom of the file. Someday, I want this to be usable by other tools too.
{% /details %}

## Where is the code?

`.bun` files are marked as executable.

To print out the code, run `./node_modules.bun` in your terminal or run `bun ./path-to-node_modules.bun`.

Here is a copy-pastable example:

```bash
$ ./node_modules.bun > node_modules.js
```

This works because every `.bun` file starts with this:

```
#!/usr/bin/env bun
```

To deploy to production with Bun, you’ll want to get the code from the `.bun` file and stick that somewhere your web server can find it (or if you’re using Vercel or a Rails app, in a `public` folder).

Note that `.bun` is a binary file format, so just opening it in VSCode or vim might render strangely.

## Advanced

By default, `bun bun` only bundles external dependencies that are `import`ed or `require`d in either app code or another external dependency. An "external dependency" is defined as, "A JavaScript-like file that has `/node_modules/` in the resolved file path and a corresponding `package.json`".

To force Bun to bundle packages which are not located in a `node_modules` folder (i.e., the final, resolved path following all symlinks), add a `bun` section to the root project’s `package.json` with `alwaysBundle` set to an array of package names to always bundle. Here’s an example:

```json
{
  "name": "my-package-name-in-here",
  "bun": {
    "alwaysBundle": ["@mybigcompany/my-workspace-package"]
  }
}
```

Bundled dependencies are not eligible for Hot Module Reloading. The code is served to browsers & Bun.js verbatim. But, in the future, it may be sectioned off into only parts of the bundle being used. That’s possible in the current version of the `.bun` file (so long as you know which files are necessary), but it’s not implemented yet. Longer-term, it will include all `import` and `export` of each module inside.

## What is the module ID hash?

The `$eb6819b` hash used here:

```js
export var $eb6819b = $$m({
```

Is generated like this:

1. Murmur3 32-bit hash of `package.name@package.version`. This is the hash uniquely identifying the npm package.
2. Wyhash 64 of the `package.hash` + `package_path`. `package_path` means "relative to the root of the npm package, where is the module imported?". For example, if you imported `react/jsx-dev-runtime.js`, the `package_path` is `jsx-dev-runtime.js`. `react-dom/cjs/react-dom.development.js` would be `cjs/react-dom.development.js`
3. Truncate the hash generated above to a `u32`

The implementation details of this module ID hash will vary between versions of Bun. The important part is the metadata contains the module IDs, the package paths, and the package hashes, so it shouldn’t really matter in practice if other tooling wants to make use of any of this.


The `bun` CLI can be used to execute JavaScript/TypeScript files, `package.json` scripts, and [executable packages](https://docs.npmjs.com/cli/v9/configuring-npm/package-json#bin).

<!-- ## Speed -->

<!--
Performance sensitive APIs like `Buffer`, `fetch`, and `Response` are heavily profiled and optimized. Under the hood Bun uses the [JavaScriptCore engine](https://developer.apple.com/documentation/javascriptcore), which is developed by Apple for Safari. It starts and runs faster than V8, the engine used by Node.js and Chromium-based browsers. -->

## Run a file

{% callout %}
Compare to `node <file>`
{% /callout %}

Use `bun run` to execute a source file.

```bash
$ bun run index.js
```

Bun supports TypeScript and JSX out of the box. Every file is transpiled on the fly by Bun's fast native transpiler before being executed.

```bash
$ bun run index.js
$ bun run index.jsx
$ bun run index.ts
$ bun run index.tsx
```

The "naked" `bun` command is equivalent to `bun run`.

```bash
$ bun index.tsx
```

## Run a `package.json` script

{% note %}
Compare to `npm run <script>` or `yarn <script>`
{% /note %}

Your `package.json` can define a number of named `"scripts"` that correspond to shell commands.

```jsonc
{
  // ... other fields
  "scripts": {
    "clean": "rm -rf dist && echo 'Done.'",
    "dev": "bun server.ts"
  }
}
```

Use `bun <script>` to execute these scripts.

```bash
$ bun clean
 $ rm -rf dist && echo 'Done.'
 Cleaning...
 Done.
```

Bun executes the script command in a subshell. It checks for the following shells in order, using the first one it finds: `bash`, `sh`, `zsh`.

{% callout %}
⚡️ The startup time for `npm run` on Linux is roughly 170ms; with Bun it is `6ms`.
{% /callout %}

If there is a name conflict between a `package.json` script and a built-in `bun` command (`install`, `dev`, `upgrade`, etc.) Bun's built-in command takes precedence. In this case, use the more explicit `bun run` command to execute your package script.

```bash
$ bun run dev
```

To see a list of available scripts, run `bun run` without any arguments.

```bash
$ bun run
quickstart scripts:

 bun run clean
   rm -rf dist && echo 'Done.'

 bun run dev
   bun server.ts

2 scripts
```

Bun respects lifecycle hooks. For instance, `bun run clean` will execute `preclean` and `postclean`, if defined. If the `pre<script>` fails, Bun will not execute the script itself.

## Environment variables

Bun automatically loads environment variables from `.env` files before running a file, script, or executable. The following files are checked, in order:

1. `.env.local` (first)
2. `NODE_ENV` === `"production"` ? `.env.production` : `.env.development`
3. `.env`

To debug environment variables, run `bun run env` to view a list of resolved environment variables.

## Performance

Bun is designed to start fast and run fast.

Under the hood Bun uses the [JavaScriptCore engine](https://developer.apple.com/documentation/javascriptcore), which is developed by Apple for Safari. In most cases, the startup and running performance is faster than V8, the engine used by Node.js and Chromium-based browsers. Its transpiler and runtime are written in Zig, a modern, high-performance language. On Linux, this translates into startup times [4x faster](https://twitter.com/jarredsumner/status/1499225725492076544) than Node.js.

{% image src="/images/bun-run-speed.jpeg" caption="Bun vs Node.js vs Deno running Hello World" /%}

<!-- If no `node_modules` directory is found in the working directory or above, Bun will abandon Node.js-style module resolution in favor of the `Bun module resolution algorithm`. Under Bun-style module resolution, all packages are _auto-installed_ on the fly into a [global module cache](/docs/cli/install#global-cache). For full details on this algorithm, refer to [Runtime > Modules](/docs/runtime/modules). -->


This command installs completions for `zsh` and/or `fish`. It runs automatically on every `bun upgrade` and on install. It reads from `$SHELL` to determine which shell to install for. It tries several common shell completion directories for your shell and OS.

If you want to copy the completions manually, run `bun completions > path-to-file`. If you know the completions directory to install them to, run `bun completions /path/to/directory`.


Bun ships with a fast built-in test runner. Tests are executed with the Bun runtime, and support the following features.

- TypeScript and JSX
- Lifecycle hooks
- Snapshot testing
- UI & DOM testing
- Watch mode with `--watch`
- Script pre-loading with `--preload`

## Run tests

```bash
$ bun test
```

Tests are written in JavaScript or TypeScript with a Jest-like API. Refer to [Writing tests](/docs/test/writing) for full documentation.

```ts#math.test.ts
import { expect, test } from "bun:test";

test("2 + 2", () => {
  expect(2 + 2).toBe(4);
});
```

The runner recursively searches the working directory for files that match the following patterns:

- `*.test.{js|jsx|ts|tsx}`
- `*_test.{js|jsx|ts|tsx}`
- `*.spec.{js|jsx|ts|tsx}`
- `*_spec.{js|jsx|ts|tsx}`

You can filter the set of tests to run by passing additional positional arguments to `bun test`. Any file in the directory with an _absolute path_ that contains one of the filters will run. Commonly, these filters will be file or directory names; glob patterns are not yet supported.

```bash
$ bun test <filter> <filter> ...
```

The test runner runs all tests in a single process. It loads all `--preload` scripts (see [Lifecycle](/docs/test/lifecycle) for details), then runs all tests. If a test fails, the test runner will exit with a non-zero exit code.

## Watch mode

Similar to `bun run`, you can pass the `--watch` flag to `bun test` to watch for changes and re-run tests.

```bash
$ bun test --watch
```

## Lifecycle hooks

Bun supports the following lifecycle hooks:

| Hook         | Description                 |
| ------------ | --------------------------- |
| `beforeAll`  | Runs once before all tests. |
| `beforeEach` | Runs before each test.      |
| `afterEach`  | Runs after each test.       |
| `afterAll`   | Runs once after all tests.  |

These hooks can be define inside test files, or in a separate file that is preloaded with the `--preload` flag.

```ts
$ bun test --preload ./setup.ts
```

See [Test > Lifecycle](/docs/test/lifecycle) for complete documentation.

## Mocks

Create mocks with the `mock` function. Mocks are automatically reset between tests.

```ts
import { test, expect, mock } from "bun:test";
const random = mock(() => Math.random());

test("random", async () => {
  const val = random();
  expect(val).toBeGreaterThan(0);
  expect(random).toHaveBeenCalled();
  expect(random).toHaveBeenCalledTimes(1);
});
```

See [Test > Mocks](/docs/test/mocks) for complete documentation.

## Snapshot testing

Snapshots are supported by `bun test`. See [Test > Snapshots](/docs/test/snapshots) for complete documentation.

## UI & DOM testing

Bun is compatible with popular UI testing libraries:

- [HappyDOM](https://github.com/capricorn86/happy-dom)
- [DOM Testing Library](https://testing-library.com/docs/dom-testing-library/intro/)
- [React Testing Library](https://testing-library.com/docs/react-testing-library/intro)

See [Test > DOM Testing](/docs/test/dom) for complete documentation.

## Performance

Bun's test runner is fast.

{% image src="/images/buntest.jpeg" caption="Running 266 React SSR tests faster than Jest can print its version number." /%}

<!--
Consider the following directory structure:

```
.
├── a.test.ts
├── b.test.ts
├── c.test.ts
└── foo
    ├── a.test.ts
    └── b.test.ts
```

To run both `a.test.ts` files:

```
$ bun test a
```

To run all tests in the `foo` directory:

```
$ bun test foo
```

Any test file in the directory with an _absolute path_ that contains one of the targets will run. Glob patterns are not yet supported. -->


### `bun install`

bun install is a fast package manager & npm client.

bun install can be configured via `bunfig.toml`, environment variables, and CLI flags.

#### Configuring `bun install` with `bunfig.toml`

`bunfig.toml` is searched for in the following paths on `bun install`, `bun remove`, and `bun add`:

1. `$XDG_CONFIG_HOME/.bunfig.toml` or `$HOME/.bunfig.toml`
2. `./bunfig.toml`

If both are found, the results are merged together.

Configuring with `bunfig.toml` is optional. Bun tries to be zero configuration in general, but that's not always possible.

```toml
# Using scoped packages with bun install
[install.scopes]

# Scope name      The value can be a URL string or an object
"@mybigcompany" = { token = "123456", url = "https://registry.mybigcompany.com" }
# URL is optional and fallsback to the default registry

# The "@" in the scope is optional
mybigcompany2 = { token = "123456" }

# Environment variables can be referenced as a string that starts with $ and it will be replaced
mybigcompany3 = { token = "$npm_config_token" }

# Setting username and password turns it into a Basic Auth header by taking base64("username:password")
mybigcompany4 = { username = "myusername", password = "$npm_config_password", url = "https://registry.yarnpkg.com/" }
# You can set username and password in the registry URL. This is the same as above.
mybigcompany5 = "https://username:password@registry.yarnpkg.com/"

# You can set a token for a registry URL:
mybigcompany6 = "https://:$NPM_CONFIG_TOKEN@registry.yarnpkg.com/"

[install]
# Default registry
# can be a URL string or an object
registry = "https://registry.yarnpkg.com/"
# as an object
#registry = { url = "https://registry.yarnpkg.com/", token = "123456" }

# Install for production? This is the equivalent to the "--production" CLI argument
production = false

# Disallow changes to lockfile? This is the equivalent to the "--fozen-lockfile" CLI argument
frozenLockfile = false

# Don't actually install
dryRun = true

# Install optionalDependencies (default: true)
optional = true

# Install local devDependencies (default: true)
dev = true

# Install peerDependencies (default: false)
peer = false

# When using `bun install -g`, install packages here
globalDir = "~/.bun/install/global"

# When using `bun install -g`, link package bins here
globalBinDir = "~/.bun/bin"

# cache-related configuration
[install.cache]
# The directory to use for the cache
dir = "~/.bun/install/cache"

# Don't load from the global cache.
# Note: Bun may still write to node_modules/.cache
disable = false


# Always resolve the latest versions from the registry
disableManifest = false


# Lockfile-related configuration
[install.lockfile]

# Print a yarn v1 lockfile
# Note: it does not load the lockfile, it just converts bun.lockb into a yarn.lock
print = "yarn"

# Path to read bun.lockb from
path = "bun.lockb"

# Path to save bun.lockb to
savePath = "bun.lockb"

# Save the lockfile to disk
save = true

```

If it's easier to read as TypeScript types:

```ts
export interface Root {
  install: Install;
}

export interface Install {
  scopes: Scopes;
  registry: Registry;
  production: boolean;
  frozenLockfile: boolean;
  dryRun: boolean;
  optional: boolean;
  dev: boolean;
  peer: boolean;
  globalDir: string;
  globalBinDir: string;
  cache: Cache;
  lockfile: Lockfile;
  logLevel: "debug" | "error" | "warn";
}

type Registry =
  | string
  | {
      url?: string;
      token?: string;
      username?: string;
      password?: string;
    };

type Scopes = Record<string, Registry>;

export interface Cache {
  dir: string;
  disable: boolean;
  disableManifest: boolean;
}

export interface Lockfile {
  print?: "yarn";
  path: string;
  savePath: string;
  save: boolean;
}
```

## Configuring with environment variables

Environment variables have a higher priority than `bunfig.toml`.

| Name                             | Description                                                   |
| -------------------------------- | ------------------------------------------------------------- |
| BUN_CONFIG_REGISTRY              | Set an npm registry (default: <https://registry.npmjs.org>)   |
| BUN_CONFIG_TOKEN                 | Set an auth token (currently does nothing)                    |
| BUN_CONFIG_LOCKFILE_SAVE_PATH    | File path to save the lockfile to (default: bun.lockb)        |
| BUN_CONFIG_YARN_LOCKFILE         | Save a Yarn v1-style yarn.lock                                |
| BUN_CONFIG_LINK_NATIVE_BINS      | Point `bin` in package.json to a platform-specific dependency |
| BUN_CONFIG_SKIP_SAVE_LOCKFILE    | Don’t save a lockfile                                         |
| BUN_CONFIG_SKIP_LOAD_LOCKFILE    | Don’t load a lockfile                                         |
| BUN_CONFIG_SKIP_INSTALL_PACKAGES | Don’t install any packages                                    |

Bun always tries to use the fastest available installation method for the target platform. On macOS, that’s `clonefile` and on Linux, that’s `hardlink`. You can change which installation method is used with the `--backend` flag. When unavailable or on error, `clonefile` and `hardlink` fallsback to a platform-specific implementation of copying files.

Bun stores installed packages from npm in `~/.bun/install/cache/${name}@${version}`. Note that if the semver version has a `build` or a `pre` tag, it is replaced with a hash of that value instead. This is to reduce the chances of errors from long file paths, but unfortunately complicates figuring out where a package was installed on disk.

When the `node_modules` folder exists, before installing, Bun checks if the `"name"` and `"version"` in `package/package.json` in the expected node_modules folder matches the expected `name` and `version`. This is how it determines whether it should install. It uses a custom JSON parser which stops parsing as soon as it finds `"name"` and `"version"`.

When a `bun.lockb` doesn’t exist or `package.json` has changed dependencies, tarballs are downloaded & extracted eagerly while resolving.

When a `bun.lockb` exists and `package.json` hasn’t changed, Bun downloads missing dependencies lazily. If the package with a matching `name` & `version` already exists in the expected location within `node_modules`, Bun won’t attempt to download the tarball.

## Platform-specific dependencies?

bun stores normalized `cpu` and `os` values from npm in the lockfile, along with the resolved packages. It skips downloading, extracting, and installing packages disabled for the current target at runtime. This means the lockfile won’t change between platforms/architectures even if the packages ultimately installed do change.

## Peer dependencies?

Peer dependencies are handled similarly to yarn. `bun install` does not automatically install peer dependencies and will try to choose an existing dependency.

## Lockfile

`bun.lockb` is Bun’s binary lockfile format.

## Why is it binary?

In a word: Performance. Bun’s lockfile saves & loads incredibly quickly, and saves a lot more data than what is typically inside lockfiles.

## How do I inspect it?

For now, the easiest thing is to run `bun install -y`. That prints a Yarn v1-style yarn.lock file.

## What does the lockfile store?

Packages, metadata for those packages, the hoisted install order, dependencies for each package, what packages those dependencies resolved to, an integrity hash (if available), what each package was resolved to and which version (or equivalent).

## Why is it fast?

It uses linear arrays for all data. [Packages](https://github.com/oven-sh/bun/blob/be03fc273a487ac402f19ad897778d74b6d72963/src/install/install.zig#L1825) are referenced by an auto-incrementing integer ID or a hash of the package name. Strings longer than 8 characters are de-duplicated. Prior to saving on disk, the lockfile is garbage-collected & made deterministic by walking the package tree and cloning the packages in dependency order.

## Cache

To delete the cache:

```bash
$ rm -rf ~/.bun/install/cache
```

## Platform-specific backends

`bun install` uses different system calls to install dependencies depending on the platform. This is a performance optimization. You can force a specific backend with the `--backend` flag.

**`hardlink`** is the default backend on Linux. Benchmarking showed it to be the fastest on Linux.

```bash
$ rm -rf node_modules
$ bun install --backend hardlink
```

**`clonefile`** is the default backend on macOS. Benchmarking showed it to be the fastest on macOS. It is only available on macOS.

```bash
$ rm -rf node_modules
$ bun install --backend clonefile
```

**`clonefile_each_dir`** is similar to `clonefile`, except it clones each file individually per directory. It is only available on macOS and tends to perform slower than `clonefile`. Unlike `clonefile`, this does not recursively clone subdirectories in one system call.

```bash
$ rm -rf node_modules
$ bun install --backend clonefile_each_dir
```

**`copyfile`** is the fallback used when any of the above fail, and is the slowest. on macOS, it uses `fcopyfile()` and on linux it uses `copy_file_range()`.

```bash
$ rm -rf node_modules
$ bun install --backend copyfile
```

**`symlink`** is typically only used for `file:` dependencies (and eventually `link:`) internally. To prevent infinite loops, it skips symlinking the `node_modules` folder.

If you install with `--backend=symlink`, Node.js won't resolve node_modules of dependencies unless each dependency has its own node_modules folder or you pass `--preserve-symlinks` to `node`. See [Node.js documentation on `--preserve-symlinks`](https://nodejs.org/api/cli.html#--preserve-symlinks).

```bash
$ rm -rf node_modules
$ bun install --backend symlink
$ node --preserve-symlinks ./my-file.js # https://nodejs.org/api/cli.html#--preserve-symlinks
```

Bun's runtime does not currently expose an equivalent of `--preserve-symlinks`, though the code for it does exist.

## npm registry metadata

bun uses a binary format for caching NPM registry responses. This loads much faster than JSON and tends to be smaller on disk.
You will see these files in `~/.bun/install/cache/*.npm`. The filename pattern is `${hash(packageName)}.npm`. It’s a hash so that extra directories don’t need to be created for scoped packages.

Bun's usage of `Cache-Control` ignores `Age`. This improves performance, but means bun may be about 5 minutes out of date to receive the latest package version metadata from npm.


Bun itself is MIT-licensed.

## JavaScriptCore

Bun statically links JavaScriptCore (and WebKit) which is LGPL-2 licensed. WebCore files from WebKit are also licensed under LGPL2. Per LGPL2:

> (1) If you statically link against an LGPL’d library, you must also provide your application in an object (not necessarily source) format, so that a user has the opportunity to modify the library and relink the application.

You can find the patched version of WebKit used by Bun here: <https://github.com/oven-sh/webkit>. If you would like to relink Bun with changes:

- `git submodule update --init --recursive`
- `make jsc`
- `zig build`

This compiles JavaScriptCore, compiles Bun’s `.cpp` bindings for JavaScriptCore (which are the object files using JavaScriptCore) and outputs a new `bun` binary with your changes.

## Linked libraries

Bun statically links these libraries:

{% table %}

- Library
- License

---

- [`boringssl`](https://boringssl.googlesource.com/boringssl/)
- [several licenses](https://boringssl.googlesource.com/boringssl/+/refs/heads/master/LICENSE)

---

- [`libarchive`](https://github.com/libarchive/libarchive)
- [several licenses](https://github.com/libarchive/libarchive/blob/master/COPYING)

---

- [`lol-html`](https://github.com/cloudflare/lol-html/tree/master/c-api)
- BSD 3-Clause

---

- [`mimalloc`](https://github.com/microsoft/mimalloc)
- MIT

---

- [`picohttp`](https://github.com/h2o/picohttpparser)
- dual-licensed under the Perl License or the MIT License

---

- [`zstd`](https://github.com/facebook/zstd)
- dual-licensed under the BSD License or GPLv2 license

---

- [`simdutf`](https://github.com/simdutf/simdutf)
- Apache 2.0

---

- [`tinycc`](https://github.com/tinycc/tinycc)
- LGPL v2.1

---

- [`uSockets`](https://github.com/uNetworking/uSockets)
- Apache 2.0

---

- [`zlib-cloudflare`](https://github.com/cloudflare/zlib)
- zlib

---

- [`c-ares`](https://github.com/c-ares/c-ares)
- MIT licensed

---

- [`libicu`](https://github.com/unicode-org/icu) 72
- [license here](https://github.com/unicode-org/icu/blob/main/icu4c/LICENSE)

---

- [`libbase64`](https://github.com/aklomp/base64/blob/master/LICENSE)
- BSD 2-Clause

---

- A fork of [`uWebsockets`](https://github.com/jarred-sumner/uwebsockets)
- Apache 2.0 licensed

{% /table %}

## Polyfills

For compatibility reasons, the following packages are embedded into Bun's binary and injected if imported.

{% table %}

- Package
- License

---

- [`assert`](https://npmjs.com/package/assert)
- MIT

---

- [`browserify-zlib`](https://npmjs.com/package/browserify-zlib)
- MIT

---

- [`buffer`](https://npmjs.com/package/buffer)
- MIT

---

- [`constants-browserify`](https://npmjs.com/package/constants-browserify)
- MIT

---

- [`crypto-browserify`](https://npmjs.com/package/crypto-browserify)
- MIT

---

- [`domain-browser`](https://npmjs.com/package/domain-browser)
- MIT

---

- [`events`](https://npmjs.com/package/events)
- MIT

---

- [`https-browserify`](https://npmjs.com/package/https-browserify)
- MIT

---

- [`os-browserify`](https://npmjs.com/package/os-browserify)
- MIT

---

- [`path-browserify`](https://npmjs.com/package/path-browserify)
- MIT

---

- [`process`](https://npmjs.com/package/process)
- MIT

---

- [`punycode`](https://npmjs.com/package/punycode)
- MIT

---

- [`querystring-es3`](https://npmjs.com/package/querystring-es3)
- MIT

---

- [`stream-browserify`](https://npmjs.com/package/stream-browserify)
- MIT

---

- [`stream-http`](https://npmjs.com/package/stream-http)
- MIT

---

- [`string_decoder`](https://npmjs.com/package/string_decoder)
- MIT

---

- [`timers-browserify`](https://npmjs.com/package/timers-browserify)
- MIT

---

- [`tty-browserify`](https://npmjs.com/package/tty-browserify)
- MIT

---

- [`url`](https://npmjs.com/package/url)
- MIT

---

- [`util`](https://npmjs.com/package/util)
- MIT

---

- [`vm-browserify`](https://npmjs.com/package/vm-browserify)
- MIT

{% /table %}

## Additional credits

- Bun's JS transpiler, CSS lexer, and Node.js module resolver source code is a Zig port of [@evanw](https://github.com/evanw)’s [esbuild](https://github.com/evanw/esbuild) project.
- Credit to [@kipply](https://github.com/kipply) for the name "Bun"!


Bun is designed for speed. Hot paths are extensively profiled and benchmarked. The source code for all of Bun's public benchmarks can be found in the [`/bench`](https://github.com/oven-sh/bun/tree/main/bench) directory of the Bun repo.

## Measuring time

To precisely measure time, Bun offers two runtime APIs functions:

1. The Web-standard [`performance.now()`](https://developer.mozilla.org/en-US/docs/Web/API/Performance/now) function
2. `Bun.nanoseconds()` which is similar to `performance.now()` except it returns the current time since the application started in nanoseconds. You can use `performance.timeOrigin` to convert this to a Unix timestamp.

## Benchmarking tools

When writing your own benchmarks, it's important to choose the right tool.

- For microbenchmarks, a great general-purpose tool is [`mitata`](https://github.com/evanwashere/mitata).
- For load testing, you _must use_ an HTTP benchmarking tool that is at least as fast as `Bun.serve()`, or your results will be skewed. Some popular Node.js-based benchmarking tools like [`autocannon`](https://github.com/mcollina/autocannon) are not fast enough. We recommend one of the following:
  - [`bombardier`](https://github.com/codesenberg/bombardier)
  - [`oha`](https://github.com/hatoo/oha)
  - [`http_load_test`](https://github.com/uNetworking/uSockets/blob/master/examples/http_load_test.c)
- For benchmarking scripts or CLI commands, we recommend [`hyperfine`](https://github.com/sharkdp/hyperfine).

## Measuring memory usage

Bun has two heaps. One heap is for the JavaScript runtime and the other heap is for everything else.

{% anchor id="bunjsc" /%}

### JavaScript heap stats

The `bun:jsc` module exposes a few functions for measuring memory usage:

```ts
import { heapStats } from "bun:jsc";
console.log(heapStats());
```

{% details summary="View example statistics"  %}

```ts
{
  heapSize: 1657575,
  heapCapacity: 2872775,
  extraMemorySize: 598199,
  objectCount: 13790,
  protectedObjectCount: 62,
  globalObjectCount: 1,
  protectedGlobalObjectCount: 1,
  // A count of every object type in the heap
  objectTypeCounts: {
    CallbackObject: 25,
    FunctionExecutable: 2078,
    AsyncGeneratorFunction: 2,
    'RegExp String Iterator': 1,
    FunctionCodeBlock: 188,
    ModuleProgramExecutable: 13,
    String: 1,
    UnlinkedModuleProgramCodeBlock: 13,
    JSON: 1,
    AsyncGenerator: 1,
    Symbol: 1,
    GetterSetter: 68,
    ImportMeta: 10,
    DOMAttributeGetterSetter: 1,
    UnlinkedFunctionCodeBlock: 174,
    RegExp: 52,
    ModuleLoader: 1,
    Intl: 1,
    WeakMap: 4,
    Generator: 2,
    PropertyTable: 95,
    'Array Iterator': 1,
    JSLexicalEnvironment: 75,
    UnlinkedFunctionExecutable: 2067,
    WeakSet: 1,
    console: 1,
    Map: 23,
    SparseArrayValueMap: 14,
    StructureChain: 19,
    Set: 18,
    'String Iterator': 1,
    FunctionRareData: 3,
    JSGlobalLexicalEnvironment: 1,
    Object: 481,
    BigInt: 2,
    StructureRareData: 55,
    Array: 179,
    AbortController: 2,
    ModuleNamespaceObject: 11,
    ShadowRealm: 1,
    'Immutable Butterfly': 103,
    Primordials: 1,
    'Set Iterator': 1,
    JSGlobalProxy: 1,
    AsyncFromSyncIterator: 1,
    ModuleRecord: 13,
    FinalizationRegistry: 1,
    AsyncIterator: 1,
    InternalPromise: 22,
    Iterator: 1,
    CustomGetterSetter: 65,
    Promise: 19,
    WeakRef: 1,
    InternalPromisePrototype: 1,
    Function: 2381,
    AsyncFunction: 2,
    GlobalObject: 1,
    ArrayBuffer: 2,
    Boolean: 1,
    Math: 1,
    CallbackConstructor: 1,
    Error: 2,
    JSModuleEnvironment: 13,
    WebAssembly: 1,
    HashMapBucket: 300,
    Callee: 3,
    symbol: 37,
    string: 2484,
    Performance: 1,
    ModuleProgramCodeBlock: 12,
    JSSourceCode: 13,
    JSPropertyNameEnumerator: 3,
    NativeExecutable: 290,
    Number: 1,
    Structure: 1550,
    SymbolTable: 108,
    GeneratorFunction: 2,
    'Map Iterator': 1
  },
  protectedObjectTypeCounts: {
    CallbackConstructor: 1,
    BigInt: 1,
    RegExp: 2,
    GlobalObject: 1,
    UnlinkedModuleProgramCodeBlock: 13,
    HashMapBucket: 2,
    Structure: 41,
    JSPropertyNameEnumerator: 1
  }
}
```

{% /details %}

JavaScript is a garbage-collected language, not reference counted. It's normal and correct for objects to not be freed immediately in all cases, though it's not normal for objects to never be freed.

To force garbage collection to run manually:

```js
Bun.gc(true); // synchronous
Bun.gc(false); // asynchronous
```

Heap snapshots let you inspect what objects are not being freed. You can use the `bun:jsc` module to take a heap snapshot and then view it with Safari or WebKit GTK developer tools. To generate a heap snapshot:

```ts
import { generateHeapSnapshot } from "bun";

const snapshot = generateHeapSnapshot();
await Bun.write("heap.json", JSON.stringify(snapshot, null, 2));
```

To view the snapshot, open the `heap.json` file in Safari's Developer Tools (or WebKit GTK)

1. Open the Developer Tools
2. Click "Timeline"
3. Click "JavaScript Allocations" in the menu on the left. It might not be visible until you click the pencil icon to show all the timelines
4. Click "Import" and select your heap snapshot JSON

{% image alt="Import heap json" src="https://user-images.githubusercontent.com/709451/204428943-ba999e8f-8984-4f23-97cb-b4e3e280363e.png" caption="Importing a heap snapshot" /%}

Once imported, you should see something like this:

{% image alt="Viewing heap snapshot in Safari" src="https://user-images.githubusercontent.com/709451/204429337-b0d8935f-3509-4071-b991-217794d1fb27.png" caption="Viewing heap snapshot in Safari Dev Tools" /%}

### Native heap stats

Bun uses mimalloc for the other heap. To report a summary of non-JavaScript memory usage, set the `MIMALLOC_SHOW_STATS=1` environment variable. and stats will print on exit.

```js
MIMALLOC_SHOW_STATS=1 bun script.js

# will show something like this:
heap stats:    peak      total      freed    current       unit      count
  reserved:   64.0 MiB   64.0 MiB      0       64.0 MiB                        not all freed!
 committed:   64.0 MiB   64.0 MiB      0       64.0 MiB                        not all freed!
     reset:      0          0          0          0                            ok
   touched:  128.5 KiB  128.5 KiB    5.4 MiB   -5.3 MiB                        ok
  segments:      1          1          0          1                            not all freed!
-abandoned:      0          0          0          0                            ok
   -cached:      0          0          0          0                            ok
     pages:      0          0         53        -53                            ok
-abandoned:      0          0          0          0                            ok
 -extended:      0
 -noretire:      0
     mmaps:      0
   commits:      0
   threads:      0          0          0          0                            ok
  searches:     0.0 avg
numa nodes:       1
   elapsed:       0.068 s
   process: user: 0.061 s, system: 0.014 s, faults: 0, rss: 57.4 MiB, commit: 64.0 MiB
```


Bun is a project with an incredibly large scope and is still in its early days. Long-term, Bun aims to provide an all-in-one tookit to replace the complex, fragmented toolchains common today: Node.js, Jest, Webpack, esbuild, Babel, yarn, PostCSS, etc.

Refer to [Bun's Roadmap](https://github.com/oven-sh/bun/issues/159) on GitHub to learn more about the project's long-term plans and priorities.

<!--
{% table %}

- Feature
- Implemented in

---

- Web Streams with HTMLRewriter
- Bun.js

---

- Source Maps (unbundled is supported)
- JS Bundler

---

- Source Maps
- CSS

---

- JavaScript Minifier
- JS Transpiler

---

- CSS Minifier
- CSS

---

- CSS Parser (it only bundles)
- CSS

---

- Tree-shaking
- JavaScript

---

- Tree-shaking
- CSS

---

- [TypeScript Decorators](https://www.typescriptlang.org/docs/handbook/decorators.html)
- TS Transpiler

---

- `@jsxPragma` comments
- JS Transpiler

---

- Sharing `.bun` files
- Bun

---

- Dates & timestamps
- TOML parser

---

- [Hash components for Fast Refresh](https://github.com/oven-sh/bun/issues/18)
- JSX Transpiler

{% /table %} -->

<!-- ## Limitations & intended usage

Today, Bun is mostly focused on Bun.js: the JavaScript runtime.

While you could use Bun's bundler & transpiler separately to build for browsers or node, Bun doesn't have a minifier or support tree-shaking yet. For production browser builds, you probably should use a tool like esbuild or swc.

## Upcoming breaking changes

- Bun's CLI flags will change to better support Bun as a JavaScript runtime. They were chosen when Bun was just a frontend development tool.
- Bun's bundling format will change to accommodate production browser bundles and on-demand production bundling -->


Configuring a development environment for Bun can take 10-30 minutes depending on your internet connection and computer speed. You will need ~10GB of free disk space for the repository and build artifacts.

If you are using Windows, you must use a WSL environment as Bun does not yet compile on Windows natively.

Before starting, you will need to already have a release build of Bun installed, as we use our bundler to transpile and minify our code.

{% codetabs %}

```bash#Native
$ curl -fsSL https://bun.sh/install | bash # for macOS, Linux, and WSL
```

```bash#npm
$ npm install -g bun # the last `npm` command you'll ever need
```

```bash#Homebrew
$ brew tap oven-sh/bun # for macOS and Linux
$ brew install bun
```

```bash#Docker
$ docker pull oven/bun
$ docker run --rm --init --ulimit memlock=-1:-1 oven/bun
```

```bash#proto
$ proto install bun
```

{% /codetabs %}

## Install LLVM

Bun requires LLVM 15 and Clang 15 (`clang` is part of LLVM). This version requirement is to match WebKit (precompiled), as mismatching versions will cause memory allocation failures at runtime. In most cases, you can install LLVM through your system package manager:

{% codetabs %}

```bash#macOS (Homebrew)
$ brew install llvm@15
```

```bash#Ubuntu/Debian
$ # LLVM has an automatic installation script that is compatible with all versions of Ubuntu
$ wget https://apt.llvm.org/llvm.sh -O - | sudo bash -s -- 15 all
```

```bash#Arch
$ sudo pacman -S llvm clang lld
```

{% /codetabs %}

If none of the above solutions apply, you will have to install it [manually](https://github.com/llvm/llvm-project/releases/tag/llvmorg-15.0.7).

Make sure LLVM 15 is in your path:

```bash
$ which clang-15
```

If not, run this to manually link it:

{% codetabs %}

```bash#macOS (Homebrew)
# use fish_add_path if you're using fish
$ export PATH="$PATH:$(brew --prefix llvm@15)/bin"
$ export LDFLAGS="$LDFLAGS -L$(brew --prefix llvm@15)/lib"
$ export CPPFLAGS="$CPPFLAGS -I$(brew --prefix llvm@15)/include"
```

{% /codetabs %}

## Install Dependencies

Using your system's package manager, install the rest of Bun's dependencies:

{% codetabs %}

```bash#macOS (Homebrew)
$ brew install automake ccache cmake coreutils esbuild gnu-sed go libiconv libtool ninja pkg-config rust
```

```bash#Ubuntu/Debian
$ sudo apt install cargo ccache cmake git golang libtool ninja-build pkg-config rustc esbuild
```

```bash#Arch
$ pacman -S base-devel ccache cmake esbuild git go libiconv libtool make ninja pkg-config python rust sed unzip
```

{% /codetabs %}

{% details summary="Ubuntu — Unable to locate package esbuild" %}

The `apt install esbuild` command may fail with an `Unable to locate package` error if you are using a Ubuntu mirror that does not contain an exact copy of the original Ubuntu server. Note that the same error may occur if you are not using any mirror but have the Ubuntu Universe enabled in the `sources.list`. In this case, you can install esbuild manually:

```bash
$ curl -fsSL https://esbuild.github.io/dl/latest | sh
$ chmod +x ./esbuild
$ sudo mv ./esbuild /usr/local/bin
```

{% /details %}

In addition to this, you will need an npm package manager (`bun`, `npm`, etc) to install the `package.json` dependencies.

## Install Zig

Zig can be installed either with our npm package [`@oven/zig`](https://www.npmjs.com/package/@oven/zig), or by using [zigup](https://github.com/marler8997/zigup).

```bash
$ bun install -g @oven/zig
$ zigup 0.11.0-dev.3737+9eb008717
```

## Building

After cloning the repository, run the following command. The runs

```bash
$ make setup
```

Then to build Bun:

```bash
$ make dev
```

The binary will be located at `packages/debug-bun-{platform}-{arch}/bun-debug`. It is recommended to add this to your `$PATH`. To verify the build worked, lets print the version number on the development build of Bun.

```bash
$ packages/debug-bun-*/bun-debug --version
bun 0.x.y__dev
```

## VSCode

VSCode is the recommended IDE for working on Bun, as it has been configured. Once opening, you can run `Extensions: Show Recommended Extensions` to install the recommended extensions for Zig and C++. ZLS is automatically configured.

## JavaScript builtins

When you change anything in `src/js/builtins/*` or switch branches, run this:

```bash
$ make regenerate-bindings
```

That inlines the TypeScript code into C++ headers.

{% callout %}
Make sure you have `ccache` installed, otherwise regeneration will take much longer than it should.
{% /callout %}

## Code generation scripts

Bun leverages a lot of code generation scripts.

The [./src/bun.js/bindings/headers.h](https://github.com/oven-sh/bun/blob/main/src/bun.js/bindings/headers.h) file has bindings to & from Zig <> C++ code. This file is generated by running the following:

```bash
$ make headers
```

This ensures that the types for Zig and the types for C++ match up correctly, by using comptime reflection over functions exported/imported.

TypeScript files that end with `*.classes.ts` are another code generation script. They generate C++ boilerplate for classes implemented in Zig. The generated code lives in:

- [src/bun.js/bindings/ZigGeneratedClasses.cpp](https://github.com/oven-sh/bun/tree/main/src/bun.js/bindings/ZigGeneratedClasses.cpp)
- [src/bun.js/bindings/ZigGeneratedClasses.h](https://github.com/oven-sh/bun/tree/main/src/bun.js/bindings/ZigGeneratedClasses.h)
- [src/bun.js/bindings/generated_classes.zig](https://github.com/oven-sh/bun/tree/main/src/bun.js/bindings/generated_classes.zig)
  To generate the code, run:

```bash
$ make codegen
```

Lastly, we also have a [code generation script](src/bun.js/scripts/generate-jssink.js) for our native stream implementations.
To run that, run:

```bash
$ make generate-sink
```

You probably won't need to run that one much.

## Modifying ESM modules

Certain modules like `node:fs`, `node:stream`, `bun:sqlite`, and `ws` are implemented in JavaScript. These live in `src/js/{node,bun,thirdparty}` files and are pre-bundled using Bun. The bundled code is committed so CI builds can run without needing a copy of Bun.

When these are changed, run:

```
$ make esm
```

In debug builds, Bun automatically loads these from the filesystem, wherever it was compiled, so no need to re-run `make dev`. In release builds, this same behavior can be done via the environment variable `BUN_OVERRIDE_MODULE_PATH`. When set to the repository root, Bun will read from the bundled modules in the repository instead of the ones baked into the binary.

## Release build

To build a release build of Bun, run:

```bash
$ make release-bindings -j12
$ make release
```

The binary will be located at `packages/bun-{platform}-{arch}/bun`.

## Valgrind

On Linux, valgrind can help find memory issues.

Keep in mind:

- JavaScriptCore doesn't support valgrind. It will report spurious errors.
- Valgrind is slow
- Mimalloc will sometimes cause spurious errors when debug build is enabled

You'll need a very recent version of Valgrind due to DWARF 5 debug symbols. You may need to manually compile Valgrind instead of using it from your Linux package manager.

`--fair-sched=try` is necessary if running multithreaded code in Bun (such as the bundler). Otherwise it will hang.

```bash
$ valgrind --fair-sched=try --track-origins=yes bun-debug <args>
```

## Updating `WebKit`

The Bun team will occasionally bump the version of WebKit used in Bun. When this happens, you may see something like this with you run `git status`.

```bash
$ git status
On branch my-branch
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git restore <file>..." to discard changes in working directory)
	modified:   src/bun.js/WebKit (new commits)
```

For performance reasons, `make submodule` does not automatically update the WebKit submodule. To update, run the following commands from the root of the Bun repo:

```bash
$ bun install
$ make regenerate-bindings
```

<!-- Check the [Bun repo](https://github.com/oven-sh/bun/tree/main/src/bun.js) to get the hash of the commit of WebKit is currently being used.

{% image width="270" src="https://github.com/oven-sh/bun/assets/3084745/51730b73-89ef-4358-9a41-9563a60a54be" /%} -->

<!--
```bash
$ cd src/bun.js/WebKit
$ git fetch
$ git checkout <hash>
``` -->

## Troubleshooting

### libarchive

If you see an error when compiling `libarchive`, run this:

```bash
$ brew install pkg-config
```

### missing files on `zig build obj`

If you see an error about missing files on `zig build obj`, make sure you built the headers.

```bash
$ make headers
```

### cmakeconfig.h not found

If you see an error about `cmakeconfig.h` not being found, this is because the precompiled WebKit did not install properly.

```bash
$ bun install
```

Check to see the command installed webkit, and you can manully look for `node_modules/bun-webkit-{platform}-{arch}`:

```bash
# this should reveal two directories. if not, something went wrong
$ echo node_modules/bun-webkit*
```

### macOS `library not found for -lSystem`

If you see this error when compiling, run:

```bash
$ xcode-select --install
```

## Arch Linux / Cannot find `libatomic.a`

Bun requires `libatomic` to be statically linked. On Arch Linux, it is only given as a shared library, but as a workaround you can symlink it to get the build working locally.

```bash
$ sudo ln -s /lib/libatomic.so /lib/libatomic.a
```

The built version of bun may not work on other systems if compiled this way.


Spawn child processes with `Bun.spawn` or `Bun.spawnSync`.

## Spawn a process (`Bun.spawn()`)

Provide a command as an array of strings. The result of `Bun.spawn()` is a `Bun.Subprocess` object.

```ts
Bun.spawn(["echo", "hello"]);
```

The second argument to `Bun.spawn` is a parameters object that can be used to configure the subprocess.

```ts
const proc = Bun.spawn(["echo", "hello"], {
  cwd: "./path/to/subdir", // specify a working direcory
  env: { ...process.env, FOO: "bar" }, // specify environment variables
  onExit(proc, exitCode, signalCode, error) {
    // exit handler
  },
});

proc.pid; // process ID of subprocess
```

## Input stream

By default, the input stream of the subprocess is undefined; it can be configured with the `stdin` parameter.

```ts
const proc = Bun.spawn(["cat"], {
  stdin: await fetch("https://raw.githubusercontent.com/oven-sh/bun/main/examples/hashing.js"),
});

const text = await new Response(proc.stdout).text();
console.log(text); // "const input = "hello world".repeat(400); ..."
```

{% table %}

---

- `null`
- **Default.** Provide no input to the subprocess

---

- `"pipe"`
- Return a `FileSink` for fast incremental writing

---

- `"inherit"`
- Inherit the `stdin` of the parent process

---

- `Bun.file()`
- Read from the specified file.

---

- `TypedArray | DataView`
- Use a binary buffer as input.

---

- `Response`
- Use the response `body` as input.

---

- `Request`
- Use the request `body` as input.

---

- `number`
- Read from the file with a given file descriptor.

{% /table %}

The `"pipe"` option lets incrementally write to the subprocess's input stream from the parent process.

```ts
const proc = Bun.spawn(["cat"], {
  stdin: "pipe", // return a FileSink for writing
});

// enqueue string data
proc.stdin.write("hello");

// enqueue binary data
const enc = new TextEncoder();
proc.stdin.write(enc.encode(" world!"));

// send buffered data
proc.stdin.flush();

// close the input stream
proc.stdin.end();
```

## Output streams

You can read results from the subprocess via the `stdout` and `stderr` properties. By default these are instances of `ReadableStream`.

```ts
const proc = Bun.spawn(["echo", "hello"]);
const text = await new Response(proc.stdout).text();
console.log(text); // => "hello"
```

Configure the output stream by passing one of the following values to `stdout/stderr`:

{% table %}

---

- `"pipe"`
- **Default for `stdout`.** Pipe the output to a `ReadableStream` on the returned `Subprocess` object.

---

- `"inherit"`
- **Default for `stderr`.** Inherit from the parent process.

---

- `Bun.file()`
- Write to the specified file.

---

- `null`
- Write to `/dev/null`.

---

- `number`
- Write to the file with the given file descriptor.

{% /table %}

## Exit handling

Use the `onExit` callback to listen for the process exiting or being killed.

```ts
const proc = Bun.spawn(["echo", "hello"], {
  onExit(proc, exitCode, signalCode, error) {
    // exit handler
  },
});
```

For convenience, the `exited` property is a `Promise` that resolves when the process exits.

```ts
const proc = Bun.spawn(["echo", "hello"]);

await proc.exited; // resolves when process exit
proc.killed; // boolean — was the process killed?
proc.exitCode; // null | number
proc.signalCode; // null | "SIGABRT" | "SIGALRM" | ...
```

To kill a process:

```ts
const proc = Bun.spawn(["echo", "hello"]);
proc.kill();
proc.killed; // true

proc.kill(); // specify an exit code
```

The parent `bun` process will not terminate until all child processes have exited. Use `proc.unref()` to detach the child process from the parent.

```
const proc = Bun.spawn(["echo", "hello"]);
proc.unref();
```

## Blocking API (`Bun.spawnSync()`)

Bun provides a synchronous equivalent of `Bun.spawn` called `Bun.spawnSync`. This is a blocking API that supports the same inputs and parameters as `Bun.spawn`. It returns a `SyncSubprocess` object, which differs from `Subprocess` in a few ways.

1. It contains a `success` property that indicates whether the process exited with a zero exit code.
2. The `stdout` and `stderr` properties are instances of `Buffer` instead of `ReadableStream`.
3. There is no `stdin` property. Use `Bun.spawn` to incrementally write to the subprocess's input stream.

```ts
const proc = Bun.spawnSync(["echo", "hello"]);

console.log(proc.stdout.toString());
// => "hello\n"
```

As a rule of thumb, the asynchronous `Bun.spawn` API is better for HTTP servers and apps, and `Bun.spawnSync` is better for building command-line tools.

## Benchmarks

{%callout%}
⚡️ Under the hood, `Bun.spawn` and `Bun.spawnSync` use [`posix_spawn(3)`](https://man7.org/linux/man-pages/man3/posix_spawn.3.html).
{%/callout%}

Bun's `spawnSync` spawns processes 60% faster than the Node.js `child_process` module.

```bash
$ bun spawn.mjs
cpu: Apple M1 Max
runtime: bun 0.2.0 (arm64-darwin)

benchmark              time (avg)             (min … max)       p75       p99      p995
--------------------------------------------------------- -----------------------------
spawnSync echo hi  888.14 µs/iter    (821.83 µs … 1.2 ms) 905.92 µs      1 ms   1.03 ms
$ node spawn.node.mjs
cpu: Apple M1 Max
runtime: node v18.9.1 (arm64-darwin)

benchmark              time (avg)             (min … max)       p75       p99      p995
--------------------------------------------------------- -----------------------------
spawnSync echo hi    1.47 ms/iter     (1.14 ms … 2.64 ms)   1.57 ms   2.37 ms   2.52 ms
```

## Reference

A simple reference of the Spawn API and types are shown below. The real types have complex generics to strongly type the `Subprocess` streams with the options passed to `Bun.spawn` and `Bun.spawnSync`. For full details, find these types as defined [bun.d.ts](https://github.com/oven-sh/bun/blob/main/packages/bun-types/bun.d.ts).

```ts
interface Bun {
  spawn(command: string[], options?: SpawnOptions.OptionsObject): Subprocess;
  spawnSync(command: string[], options?: SpawnOptions.OptionsObject): SyncSubprocess;

  spawn(options: { cmd: string[] } & SpawnOptions.OptionsObject): Subprocess;
  spawnSync(options: { cmd: string[] } & SpawnOptions.OptionsObject): SyncSubprocess;
}

namespace SpawnOptions {
  interface OptionsObject {
    cwd?: string;
    env?: Record<string, string>;
    stdin?: SpawnOptions.Readable;
    stdout?: SpawnOptions.Writable;
    stderr?: SpawnOptions.Writable;
    onExit?: (proc: Subprocess, exitCode: number | null, signalCode: string | null, error: Error | null) => void;
  }

  type Readable =
    | "pipe"
    | "inherit"
    | "ignore"
    | null // equivalent to "ignore"
    | undefined // to use default
    | BunFile
    | ArrayBufferView
    | number;

  type Writable =
    | "pipe"
    | "inherit"
    | "ignore"
    | null // equivalent to "ignore"
    | undefined // to use default
    | BunFile
    | ArrayBufferView
    | number
    | ReadableStream
    | Blob
    | Response
    | Request;
}

interface Subprocess<Stdin, Stdout, Stderr> {
  readonly pid: number;
  // the exact stream types here are derived from the generic parameters
  readonly stdin: number | ReadableStream | FileSink | undefined;
  readonly stdout: number | ReadableStream | undefined;
  readonly stderr: number | ReadableStream | undefined;

  readonly exited: Promise<number>;

  readonly exitCode: number | undefined;
  readonly signalCode: Signal | null;
  readonly killed: boolean;

  ref(): void;
  unref(): void;
  kill(code?: number): void;
}

interface SyncSubprocess<Stdout, Stderr> {
  readonly pid: number;
  readonly success: boolean;
  // the exact buffer types here are derived from the generic parameters
  readonly stdout: Buffer | undefined;
  readonly stderr: Buffer | undefined;
}

type ReadableSubprocess = Subprocess<any, "pipe", "pipe">;
type WritableSubprocess = Subprocess<"pipe", any, any>;
type PipedSubprocess = Subprocess<"pipe", "pipe", "pipe">;
type NullSubprocess = Subprocess<null, null, null>;

type ReadableSyncSubprocess = SyncSubprocess<"pipe", "pipe">;
type NullSyncSubprocess = SyncSubprocess<null, null>;

type Signal =
  | "SIGABRT"
  | "SIGALRM"
  | "SIGBUS"
  | "SIGCHLD"
  | "SIGCONT"
  | "SIGFPE"
  | "SIGHUP"
  | "SIGILL"
  | "SIGINT"
  | "SIGIO"
  | "SIGIOT"
  | "SIGKILL"
  | "SIGPIPE"
  | "SIGPOLL"
  | "SIGPROF"
  | "SIGPWR"
  | "SIGQUIT"
  | "SIGSEGV"
  | "SIGSTKFLT"
  | "SIGSTOP"
  | "SIGSYS"
  | "SIGTERM"
  | "SIGTRAP"
  | "SIGTSTP"
  | "SIGTTIN"
  | "SIGTTOU"
  | "SIGUNUSED"
  | "SIGURG"
  | "SIGUSR1"
  | "SIGUSR2"
  | "SIGVTALRM"
  | "SIGWINCH"
  | "SIGXCPU"
  | "SIGXFSZ"
  | "SIGBREAK"
  | "SIGLOST"
  | "SIGINFO";
```


The page primarily documents the Bun-native `Bun.serve` API. Bun also implements [`fetch`](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API) and the Node.js [`http`](https://nodejs.org/api/http.html) and [`https`](https://nodejs.org/api/https.html) modules.

{% callout %}
These modules have been re-implemented to use Bun's fast internal HTTP infrastructure. Feel free to use these modules directly; frameworks like [Express](https://expressjs.com/) that depend on these modules should work out of the box. For granular compatibility information, see [Runtime > Node.js APIs](/docs/runtime/nodejs-apis).
{% /callout %}

To start a high-performance HTTP server with a clean API, the recommended approach is [`Bun.serve`](#start-a-server-bun-serve).

## `Bun.serve()`

Start an HTTP server in Bun with `Bun.serve`.

```ts
Bun.serve({
  fetch(req) {
    return new Response(`Bun!`);
  },
});
```

The `fetch` handler handles incoming requests. It receives a [`Request`](https://developer.mozilla.org/en-US/docs/Web/API/Request) object and returns a [`Response`](https://developer.mozilla.org/en-US/docs/Web/API/Response) or `Promise<Response>`.

```ts
Bun.serve({
  fetch(req) {
    const url = new URL(req.url);
    if (url.pathname === "/") return new Response(`Home page!`);
    if (url.pathname === "/blog") return new Response("Blog!");
    return new Response(`404!`);
  },
});
```

To configure which port and hostname the server will listen on:

```ts
Bun.serve({
  port: 8080, // defaults to $PORT, then 3000
  hostname: "mydomain.com", // defaults to "0.0.0.0"
  fetch(req) {
    return new Response(`404!`);
  },
});
```

## Error handling

To activate development mode, set `development: true`. By default, development mode is _enabled_ unless `NODE_ENV` is `production`.

```ts
Bun.serve({
  development: true,
  fetch(req) {
    throw new Error("woops!");
  },
});
```

In development mode, Bun will surface errors in-browser with a built-in error page.

{% image src="/images/exception_page.png" caption="Bun's built-in 500 page" /%}

To handle server-side errors, implement an `error` handler. This function should return a `Response` to served to the client when an error occurs. This response will supercede Bun's default error page in `development` mode.

```ts
Bun.serve({
  fetch(req) {
    throw new Error("woops!");
  },
  error(error) {
    return new Response(`<pre>${error}\n${error.stack}</pre>`, {
      headers: {
        "Content-Type": "text/html",
      },
    });
  },
});
```

{% callout %}
**Note** — Full debugger support is planned.
{% /callout %}

The call to `Bun.serve` returns a `Server` object. To stop the server, call the `.stop()` method.

```ts
const server = Bun.serve({
  fetch() {
    return new Response("Bun!");
  },
});

server.stop();
```

## TLS

Bun supports TLS out of the box, powered by [BoringSSL](https://boringssl.googlesource.com/boringssl). Enable TLS by passing in a value for `key` and `cert`; both are required to enable TLS.

```ts-diff
  Bun.serve({
    fetch(req) {
      return new Response("Hello!!!");
    },

+   tls: {
+     key: Bun.file("./key.pem"),
+     cert: Bun.file("./cert.pem"),
+   }
  });
```

The `key` and `cert` fields expect the _contents_ of your TLS key and certificate, _not a path to it_. This can be a string, `BunFile`, `TypedArray`, or `Buffer`.

```ts
Bun.serve({
  fetch() {},

  tls: {
    // BunFile
    key: Bun.file("./key.pem"),
    // Buffer
    key: fs.readFileSync("./key.pem"),
    // string
    key: fs.readFileSync("./key.pem", "utf8"),
    // array of above
    key: [Bun.file("./key1.pem"), Bun.file("./key2.pem")],
  },
});
```

{% callout %}

**Note** — Earlier versions of Bun supported passing a file path as `keyFile` and `certFile`; this has been deprecated as of `v0.6.3`.

{% /callout %}

If your private key is encrypted with a passphrase, provide a value for `passphrase` to decrypt it.

```ts-diff
  Bun.serve({
    fetch(req) {
      return new Response("Hello!!!");
    },

    tls: {
      key: Bun.file("./key.pem"),
      cert: Bun.file("./cert.pem"),
+     passphrase: "my-secret-passphrase",
    }
  });
```

Optionally, you can override the trusted CA certificates by passing a value for `ca`. By default, the server will trust the list of well-known CAs curated by Mozilla. When `ca` is specified, the Mozilla list is overwritten.

```ts-diff
  Bun.serve({
    fetch(req) {
      return new Response("Hello!!!");
    },
    tls: {
      key: Bun.file("./key.pem"), // path to TLS key
      cert: Bun.file("./cert.pem"), // path to TLS cert
+     ca: Bun.file("./ca.pem"), // path to root CA certificate
    }
  });
```

To override Diffie-Helman parameters:

```ts
Bun.serve({
  // ...
  tls: {
    // other config
    dhParamsFile: "/path/to/dhparams.pem", // path to Diffie Helman parameters
  },
});
```

## Hot reloading

Thus far, the examples on this page have used the explicit `Bun.serve` API. Bun also supports an alternate syntax.

```ts#server.ts
import {type Serve} from "bun";

export default {
  fetch(req) {
    return new Response(`Bun!`);
  },
} satisfies Serve;
```

Instead of passing the server options into `Bun.serve`, export it. This file can be executed as-is; when Bun runs a file with a `default` export containing a `fetch` handler, it passes it into `Bun.serve` under the hood.

This syntax has one major advantage: it is hot-reloadable out of the box. When any source file is changed, Bun will reload the server with the updated code _without restarting the process_. This makes hot reloads nearly instantaneous. Use the `--hot` flag when starting the server to enable hot reloading.

```bash
$ bun --hot server.ts
```

It's possible to configure hot reloading while using the explicit `Bun.serve` API; for details refer to [Runtime > Hot reloading](/docs/runtime/hot).

## Streaming files

To stream a file, return a `Response` object with a `BunFile` object as the body.

```ts
import { serve, file } from "bun";

serve({
  fetch(req) {
    return new Response(Bun.file("./hello.txt"));
  },
});
```

{% callout %}
⚡️ **Speed** — Bun automatically uses the [`sendfile(2)`](https://man7.org/linux/man-pages/man2/sendfile.2.html) system call when possible, enabling zero-copy file transfers in the kernel—the fastest way to send files.
{% /callout %}

**[v0.3.0+]** You can send part of a file using the [`slice(start, end)`](https://developer.mozilla.org/en-US/docs/Web/API/Blob/slice) method on the `Bun.file` object. This automatically sets the `Content-Range` and `Content-Length` headers on the `Response` object.

```ts
Bun.serve({
  fetch(req) {
    // parse `Range` header
    const [start = 0, end = Infinity] = req.headers
      .get("Range") // Range: bytes=0-100
      .split("=") // ["Range: bytes", "0-100"]
      .at(-1) // "0-100"
      .split("-") // ["0", "100"]
      .map(Number); // [0, 100]

    // return a slice of the file
    const bigFile = Bun.file("./big-video.mp4");
    return new Response(bigFile.slice(start, end));
  },
});
```

## Benchmarks

Below are Bun and Node.js implementations of a simple HTTP server that responds `Bun!` to each incoming `Request`.

{% codetabs %}

```ts#Bun
Bun.serve({
  fetch(req: Request) {
    return new Response(`Bun!`);
  },
  port: 3000,
});
```

```ts#Node
require("http")
  .createServer((req, res) => res.end("Bun!"))
  .listen(8080);
```

{% /codetabs %}
The `Bun.serve` server can handle roughly 2.5x more requests per second than Node.js on Linux.

{% table %}

- Runtime
- Requests per second

---

- Node 16
- ~64,000

---

- Bun
- ~160,000

{% /table %}

{% image width="499" alt="image" src="https://user-images.githubusercontent.com/709451/162389032-fc302444-9d03-46be-ba87-c12bd8ce89a0.png" /%}

## Reference

{% details summary="See TypeScript definitions" %}

```ts
interface Bun {
  serve(options: {
    fetch: (req: Request, server: Server) => Response | Promise<Response>;
    hostname?: string;
    port?: number;
    development?: boolean;
    error?: (error: Error) => Response | Promise<Response>;
    tls?: {
      key?:
        | string
        | TypedArray
        | BunFile
        | Array<string | TypedArray | BunFile>;
      cert?:
        | string
        | TypedArray
        | BunFile
        | Array<string | TypedArray | BunFile>;
      ca?: string | TypedArray | BunFile | Array<string | TypedArray | BunFile>;
      passphrase?: string;
      dhParamsFile?: string;
    };
    maxRequestBodySize?: number;
    lowMemoryMode?: boolean;
  }): Server;
}

interface Server {
  development: boolean;
  hostname: string;
  port: number;
  pendingRequests: number;
  stop(): void;
}
```

{% /details %}


Bun provides a fast API for resolving routes against file-system paths. This API is primarily intended for library authors. At the moment only Next.js-style file-system routing is supported, but other styles may be added in the future.

## Next.js-style

The `FileSystemRouter` class can resolve routes against a `pages` directory. (The Next.js 13 `app` directory is not yet supported.) Consider the following `pages` directory:

```txt
pages
├── index.tsx
├── settings.tsx
├── blog
│   ├── [slug].tsx
│   └── index.tsx
└── [[...catchall]].tsx
```

The `FileSystemRouter` can be used to resolve routes against this directory:

```ts
const router = new Bun.FileSystemRouter({
  style: "nextjs",
  dir: "./pages",
  origin: "https://mydomain.com",
  assetPrefix: "_next/static/"
});
router.match("/");

// =>
{
  filePath: "/path/to/pages/index.tsx",
  kind: "exact",
  name: "/",
  pathname: "/",
  src: "https://mydomain.com/_next/static/pages/index.tsx"
}
```

Query parameters will be parsed and returned in the `query` property.

```ts
router.match("/settings?foo=bar");

// =>
{
  filePath: "/Users/colinmcd94/Documents/bun/fun/pages/settings.tsx",
  kind: "dynamic",
  name: "/settings",
  pathname: "/settings?foo=bar",
  src: "https://mydomain.com/_next/static/pages/settings.tsx"
  query: {
    foo: "bar"
  }
}
```

The router will automatically parse URL parameters and return them in the `params` property:

```ts
router.match("/blog/my-cool-post");

// =>
{
  filePath: "/Users/colinmcd94/Documents/bun/fun/pages/blog/[slug].tsx",
  kind: "dynamic",
  name: "/blog/[slug]",
  pathname: "/blog/my-cool-post",
  src: "https://mydomain.com/_next/static/pages/blog/[slug].tsx"
  params: {
    slug: "my-cool-post"
  }
}
```

The `.match()` method also accepts `Request` and `Response` objects. The `url` property will be used to resolve the route.

```ts
router.match(new Request("https://example.com/blog/my-cool-post"));
```

The router will read the directory contents on initialization. To re-scan the files, use the `.reload()` method.

```ts
router.reload();
```

## Reference

```ts
interface Bun {
  class FileSystemRouter {
    constructor(params: {
      dir: string;
      style: "nextjs";
      origin?: string;
      assetPrefix?: string;
    });

    reload(): void;

    match(path: string | Request | Response): {
      filePath: string;
      kind: "exact" | "catch-all" | "optional-catch-all" | "dynamic";
      name: string;
      pathname: string;
      src: string;
      params?: Record<string, string>;
      query?: Record<string, string>;
    } | null
  }
}
```


Bun provides a fast native implementation of the `HTMLRewriter` pattern developed by Cloudflare. It provides a convenient, `EventListener`-like API for traversing and transforming HTML documents.

```ts
const rewriter = new HTMLRewriter();

rewriter.on("*", {
  element(el) {
    console.log(el.tagName); // "body" | "div" | ...
  },
});
```

To parse and/or transform the HTML:

```ts#rewriter.ts
rewriter.transform(
  new Response(`
<!DOCTYPE html>
<html>
<!-- comment -->
<head>
  <title>My First HTML Page</title>
</head>
<body>
  <h1>My First Heading</h1>
  <p>My first paragraph.</p>
</body>
`));
```

View the full documentation on the [Cloudflare website](https://developers.cloudflare.com/workers/runtime-apis/html-rewriter/).


Use Bun's native TCP API to implement performance sensitive systems like database clients, game servers, or anything that needs to communicate over TCP (instead of HTTP). This is a low-level API intended for library authors and for advanced use cases.

## Start a server (`Bun.listen()`)

To start a TCP server with `Bun.listen`:

```ts
Bun.listen({
  hostname: "localhost",
  port: 8080,
  socket: {
    data(socket, data) {}, // message received from client
    open(socket) {}, // socket opened
    close(socket) {}, // socket closed
    drain(socket) {}, // socket ready for more data
    error(socket, error) {}, // error handler
  },
});
```

{% details summary="An API designed for speed" %}

In Bun, a set of handlers are declared once per server instead of assigning callbacks to each socket, as with Node.js `EventEmitters` or the web-standard `WebSocket` API.

```ts
Bun.listen({
  hostname: "localhost",
  port: 8080,
  socket: {
    open(socket) {},
    data(socket, data) {},
    drain(socket) {},
    close(socket) {},
    error(socket, error) {},
  },
});
```

For performance-sensitive servers, assigning listeners to each socket can cause significant garbage collector pressure and increase memory usage. By contrast, Bun only allocates one handler function for each event and shares it among all sockets. This is a small optimization, but it adds up.

{% /details %}

Contextual data can be attached to a socket in the `open` handler.

```ts
type SocketData = { sessionId: string };

Bun.listen<SocketData>({
  hostname: "localhost",
  port: 8080,
  socket: {
    data(socket, data) {
      socket.write(`${socket.data.sessionId}: ack`);
    },
    open(socket) {
      socket.data = { sessionId: "abcd" };
    },
  },
});
```

To enable TLS, pass a `tls` object containing `key` and `cert` fields.

```ts
Bun.listen({
  hostname: "localhost",
  port: 8080,
  socket: {
    data(socket, data) {},
  },
  tls: {
    // can be string, BunFile, TypedArray, Buffer, or array thereof
    key: Bun.file("./key.pem"),
    cert: Bun.file("./cert.pem"),
  },
});
```

{% callout %}

**Note** — Earlier versions of Bun supported passing a file path as `keyFile` and `certFile`; this has been deprecated as of `v0.6.3`.

{% /callout %}

The `key` and `cert` fields expect the _contents_ of your TLS key and certificate. This can be a string, `BunFile`, `TypedArray`, or `Buffer`.

```ts
Bun.listen({
  // ...
  tls: {
    // BunFile
    key: Bun.file("./key.pem"),
    // Buffer
    key: fs.readFileSync("./key.pem"),
    // string
    key: fs.readFileSync("./key.pem", "utf8"),
    // array of above
    key: [Bun.file('./key1.pem'), Bun.file('./key2.pem']
  },
});
```

The result of `Bun.listen` is a server that conforms to the `TCPSocket` interface.

```ts
const server = Bun.listen({
  /* config*/
});

// stop listening
// parameter determines whether active connections are closed
server.stop(true);

// let Bun process exit even if server is still listening
server.unref();
```

## Create a connection (`Bun.connect()`)

Use `Bun.connect` to connect to a TCP server. Specify the server to connect to with `hostname` and `port`. TCP clients can define the same set of handlers as `Bun.listen`, plus a couple client-specific handlers.

```ts
// The client
const socket = Bun.connect({
  hostname: "localhost",
  port: 8080,

  socket: {
    data(socket, data) {},
    open(socket) {},
    close(socket) {},
    drain(socket) {},
    error(socket, error) {},

    // client-specific handlers
    connectError(socket, error) {}, // connection failed
    end(socket) {}, // connection closed by server
    timeout(socket) {}, // connection timed out
  },
});
```

To require TLS, specify `tls: true`.

```ts
// The client
const socket = Bun.connect({
  // ... config
  tls: true,
});
```

## Hot reloading

Both TCP servers and sockets can be hot reloaded with new handlers.

{% codetabs %}

```ts#Server
const server = Bun.listen({ /* config */ })

// reloads handlers for all active server-side sockets
server.reload({
  socket: {
    data(){
      // new 'data' handler
    }
  }
})
```

```ts#Client
const socket = Bun.connect({ /* config */ })
socket.reload({
  data(){
    // new 'data' handler
  }
})
```

{% /codetabs %}

## Buffering

Currently, TCP sockets in Bun do not buffer data. For performance-sensitive code, it's important to consider buffering carefully. For example, this:

```ts
socket.write("h");
socket.write("e");
socket.write("l");
socket.write("l");
socket.write("o");
```

...performs significantly worse than this:

```ts
socket.write("hello");
```

To simplify this for now, consider using Bun's `ArrayBufferSink` with the `{stream: true}` option:

```ts
const sink = new ArrayBufferSink({ stream: true, highWaterMark: 1024 });

sink.write("h");
sink.write("e");
sink.write("l");
sink.write("l");
sink.write("o");

queueMicrotask(() => {
  var data = sink.flush();
  if (!socket.write(data)) {
    // put it back in the sink if the socket is full
    sink.write(data);
  }
});
```

{% callout %}
**Corking** — Support for corking is planned, but in the meantime backpressure must be managed manually with the `drain` handler.
{% /callout %}


Bun exposes its internal transpiler via the `Bun.Transpiler` class. To create an instance of Bun's transpiler:

```ts
const transpiler = new Bun.Transpiler({
  loader: "tsx", // "js | "jsx" | "ts" | "tsx"
});
```

## `.transformSync()`

Transpile code synchronously with the `.transformSync()` method. Modules are not resolved and the code is not executed. The result is a string of vanilla JavaScript code.

<!-- It is synchronous and runs in the same thread as other JavaScript code. -->

{% codetabs %}

```js#Example
const transpiler = new Bun.Transpiler({
  loader: 'tsx',
});

const code = `
import * as whatever from "./whatever.ts"
export function Home(props: {title: string}){
  return <p>{props.title}</p>;
}`;

const result = transpiler.transformSync(code);
```

```js#Result
import { __require as require } from "bun:wrap";
import * as JSX from "react/jsx-dev-runtime";
var jsx = require(JSX).jsxDEV;

export default jsx(
  "div",
  {
    children: "hi!",
  },
  undefined,
  false,
  undefined,
  this,
);
```

{% /codetabs %}

To override the default loader specified in the `new Bun.Transpiler()` constructor, pass a second argument to `.transformSync()`.

```ts
await transpiler.transform("<div>hi!</div>", "tsx");
```

{% details summary="Nitty gritty" %}
When `.transformSync` is called, the transpiler is run in the same thread as the currently executed code.

If a macro is used, it will be run in the same thread as the transpiler, but in a separate event loop from the rest of your application. Currently, globals between macros and regular code are shared, which means it is possible (but not recommended) to share states between macros and regular code. Attempting to use AST nodes outside of a macro is undefined behavior.
{% /details %}

## `.transform()`

The `transform()` method is an async version of `.transformSync()` that returns a `Promise<string>`.

```js
const transpiler = new Bun.Transpiler({ loader: "jsx" });
const result = await transpiler.transform("<div>hi!</div>");
console.log(result);
```

Unless you're transpiling _many_ large files, you should probably use `Bun.Transpiler.transformSync`. The cost of the threadpool will often take longer than actually transpiling code.

```ts
await transpiler.transform("<div>hi!</div>", "tsx");
```

{% details summary="Nitty gritty" %}
The `.tranform()` method runs the transpiler in Bun's worker threadpool, so if you run it 100 times, it will run it across `Math.floor($cpu_count * 0.8)` threads, without blocking the main JavaScript thread.

If your code uses a macro, it will potentially spawn a new copy of Bun's JavaScript runtime environment in that new thread.
{% /details %}

## `.scan()`

The `Transpiler` instance can also scan some source code and return a list of its imports and exports, plus additional metadata about each one. [Type-only](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-3-8.html#type-only-imports-and-export) imports and exports are ignored.

{% codetabs %}

```ts#Example
const transpiler = new Bun.Transpiler({
  loader: 'tsx',
});

const code = `
import React from 'react';
import type {ReactNode} from 'react';
const val = require('./cjs.js')
import('./loader');

export const name = "hello";
`;

const result = transpiler.scan(code);
```

```json#Output
{
  "exports": [
    "name"
  ],
  "imports": [
    {
      "kind": "import-statement",
      "path": "react"
    },
    {
      "kind": "import-statement",
      "path": "remix"
    },
    {
      "kind": "dynamic-import",
      "path": "./loader"
    }
  ]
}
```

{% /codetabs %}

Each import in the `imports` array has a `path` and `kind`. Bun categories imports into the following kinds:

- `import-statement`: `import React from 'react'`
- `require-call`: `const val = require('./cjs.js')`
- `require-resolve`: `require.resolve('./cjs.js')`
- `dynamic-import`: `import('./loader')`
- `import-rule`: `@import 'foo.css'`
- `url-token`: `url('./foo.png')`
<!-- - `internal`: `import {foo} from 'bun:internal'`
- `entry-point`: `import {foo} from 'bun:entry'` -->

## `.scanImports()`

For performance-sensitive code, you can use the `.scanImports()` method to get a list of imports. It's faster than `.scan()` (especially for large files) but marginally less accurate due to some performance optimizations.

{% codetabs %}

```ts#Example
const transpiler = new Bun.Transpiler({
  loader: 'tsx',
});

const code = `
import React from 'react';
import type {ReactNode} from 'react';
const val = require('./cjs.js')
import('./loader');

export const name = "hello";
`;

const result = transpiler.scanImports(code);
`);
```

```json#Results
[
  {
    kind: "import-statement",
    path: "react"
  }, {
    kind: "require-call",
    path: "./cjs.js"
  }, {
    kind: "dynamic-import",
    path: "./loader"
  }
]
```

{% /codetabs %}

## Reference

```ts
type Loader = "jsx" | "js" | "ts" | "tsx";

interface TranspilerOptions {
  // Replace key with value. Value must be a JSON string.
  // { "process.env.NODE_ENV": "\"production\"" }
  define?: Record<string, string>,

  // Default loader for this transpiler
  loader?: Loader,

  // Default platform to target
  // This affects how import and/or require is used
  target?: "browser" | "bun" | "node",

  // Specify a tsconfig.json file as stringified JSON or an object
  // Use this to set a custom JSX factory, fragment, or import source
  // For example, if you want to use Preact instead of React. Or if you want to use Emotion.
  tsconfig?: string | TSConfig,

  // Replace imports with macros
  macro?: MacroMap,

  // Specify a set of exports to eliminate
  // Or rename certain exports
  exports?: {
      eliminate?: string[];
      replace?: Record<string, string>;
  },

  // Whether to remove unused imports from transpiled file
  // Default: false
  trimUnusedImports?: boolean,

  // Whether to enable a set of JSX optimizations
  // jsxOptimizationInline ...,

  // Experimental whitespace minification
  minifyWhitespace?: boolean,

  // Whether to inline constant values
  // Typically improves performance and decreases bundle size
  // Default: true
  inline?: boolean,
}

// Map import paths to macros
interface MacroMap {
  // {
  //   "react-relay": {
  //     "graphql": "bun-macro-relay/bun-macro-relay.tsx"
  //   }
  // }
  [packagePath: string]: {
    [importItemName: string]: string,
  },
}

class Bun.Transpiler {
  constructor(options: TranspilerOptions)

  transform(code: string, loader?: Loader): Promise<string>
  transformSync(code: string, loader?: Loader): string

  scan(code: string): {exports: string[], imports: Import}
  scanImports(code: string): Import[]
}

type Import = {
  path: string,
  kind:
  // import foo from 'bar'; in JavaScript
  | "import-statement"
  // require("foo") in JavaScript
  | "require-call"
  // require.resolve("foo") in JavaScript
  | "require-resolve"
  // Dynamic import() in JavaScript
  | "dynamic-import"
  // @import() in CSS
  | "import-rule"
  // url() in CSS
  | "url-token"
  // The import was injected by Bun
  | "internal" 
  // Entry point (not common)
  | "entry-point"
}

const transpiler = new Bun.Transpiler({ loader: "jsx" });
```


## `Bun.version`

A `string` containing the version of the `bun` CLI that is currently running.

```ts
Bun.version;
// => "0.6.4"
```

## `Bun.revision`

The git commit of [Bun](https://github.com/oven-sh/bun) that was compiled to create the current `bun` CLI.

```ts
Bun.revision;
// => "f02561530fda1ee9396f51c8bc99b38716e38296"
```

## `Bun.env`

An alias for `process.env`.

## `Bun.main`

An absolute path to the entrypoint of the current program (the file that was executed with `bun run`).

```ts#script.ts
Bun.main;
// /path/to/script.ts
```

This is particular useful for determining whether a script is being directly executed, as opposed to being imported by another script.

```ts
if (import.meta.path === Bun.main) {
  // this script is being directly executed
} else {
  // this file is being imported from another script
}
```

This is analogous to the [`require.main = module` trick](https://stackoverflow.com/questions/6398196/detect-if-called-through-require-or-directly-by-command-line) in Node.js.

## `Bun.sleep()`

`Bun.sleep(ms: number)` (added in Bun v0.5.6)

Returns a `Promise` that resolves after the given number of milliseconds.

```ts
console.log("hello");
await Bun.sleep(1000);
console.log("hello one second later!");
```

Alternatively, pass a `Date` object to receive a `Promise` that resolves at that point in time.

```ts
const oneSecondInFuture = new Date(Date.now() + 1000);

console.log("hello");
await Bun.sleep(oneSecondInFuture);
console.log("hello one second later!");
```

## `Bun.sleepSync()`

`Bun.sleepSync(ms: number)` (added in Bun v0.5.6)

A blocking synchronous version of `Bun.sleep`.

```ts
console.log("hello");
Bun.sleepSync(1000); // blocks thread for one second
console.log("hello one second later!");
```

## `Bun.which()`

`Bun.which(bin: string)`

Returns the path to an executable, similar to typing `which` in your terminal.

```ts
const ls = Bun.which("ls");
console.log(ls); // "/usr/bin/ls"
```

By default Bun looks at the current `PATH` environment variable to determine the path. To configure `PATH`:

```ts
const ls = Bun.which("ls", {
  PATH: "/usr/local/bin:/usr/bin:/bin",
});
console.log(ls); // "/usr/bin/ls"
```

Pass a `cwd` option to resolve for executable from within a specific directory.

```ts
const ls = Bun.which("ls", {
  cwd: "/tmp",
  PATH: "",
});

console.log(ls); // null
```

## `Bun.peek()`

`Bun.peek(prom: Promise)` (added in Bun v0.2.2)

Reads a promise's result without `await` or `.then`, but only if the promise has already fulfilled or rejected.

```ts
import { peek } from "bun";

const promise = Promise.resolve("hi");

// no await!
const result = peek(promise);
console.log(result); // "hi"
```

This is important when attempting to reduce number of extraneous microticks in performance-sensitive code. It's an advanced API and you probably shouldn't use it unless you know what you're doing.

```ts
import { peek } from "bun";
import { expect, test } from "bun:test";

test("peek", () => {
  const promise = Promise.resolve(true);

  // no await necessary!
  expect(peek(promise)).toBe(true);

  // if we peek again, it returns the same value
  const again = peek(promise);
  expect(again).toBe(true);

  // if we peek a non-promise, it returns the value
  const value = peek(42);
  expect(value).toBe(42);

  // if we peek a pending promise, it returns the promise again
  const pending = new Promise(() => {});
  expect(peek(pending)).toBe(pending);

  // If we peek a rejected promise, it:
  // - returns the error
  // - does not mark the promise as handled
  const rejected = Promise.reject(
    new Error("Successfully tested promise rejection"),
  );
  expect(peek(rejected).message).toBe("Successfully tested promise rejection");
});
```

The `peek.status` function lets you read the status of a promise without resolving it.

```ts
import { peek } from "bun";
import { expect, test } from "bun:test";

test("peek.status", () => {
  const promise = Promise.resolve(true);
  expect(peek.status(promise)).toBe("fulfilled");

  const pending = new Promise(() => {});
  expect(peek.status(pending)).toBe("pending");

  const rejected = Promise.reject(new Error("oh nooo"));
  expect(peek.status(rejected)).toBe("rejected");
});
```

## `Bun.openInEditor()`

Opens a file in your default editor. Bun auto-detects your editor via the `$VISUAL` or `$EDITOR` environment variables.

```ts
const currentFile = import.meta.url;
Bun.openInEditor(currentFile);
```

You can override this via the `debug.editor` setting in your [`bunfig.toml`](/docs/runtime/configuration)

```toml-diff#bunfig.toml
+ [debug]
+ editor = "code"
```

Or specify an editor with the `editor` param. You can also specify a line and column number.

```ts
Bun.openInEditor(import.meta.url, {
  editor: "vscode", // or "subl"
  line: 10,
  column: 5,
});
```

Bun.ArrayBufferSink;

## `Bun.deepEquals()`

Nestedly checks if two objects are equivalent. This is used internally by `expect().toEqual()` in `bun:test`.

```ts
const foo = { a: 1, b: 2, c: { d: 3 } };

// true
Bun.deepEquals(foo, { a: 1, b: 2, c: { d: 3 } });

// false
Bun.deepEquals(foo, { a: 1, b: 2, c: { d: 4 } });
```

A third boolean parameter can be used to enable "strict" mode. This is used by `expect().toStrictEqual()` in the test runner.

```ts
const a = { entries: [1, 2] };
const b = { entries: [1, 2], extra: undefined };

Bun.deepEquals(a, b); // => true
Bun.deepEquals(a, b, true); // => false
```

In strict mode, the following are considered unequal:

```ts
// undefined values
Bun.deepEquals({}, { a: undefined }, true); // false

// undefined in arrays
Bun.deepEquals(["asdf"], ["asdf", undefined], true); // false

// sparse arrays
Bun.deepEquals([, 1], [undefined, 1], true); // false

// object literals vs instances w/ same properties
class Foo {
  a = 1;
}
Bun.deepEquals(new Foo(), { a: 1 }, true); // false
```

## `Bun.escapeHTML()`

`Bun.escapeHTML(value: string | object | number | boolean): boolean`

Escapes the following characters from an input string:

- `"` becomes `"&quot;"`
- `&` becomes `"&amp;"`
- `'` becomes `"&#x27;"`
- `<` becomes `"&lt;"`
- `>` becomes `"&gt;"`

This function is optimized for large input. On an M1X, it processes 480 MB/s -
20 GB/s, depending on how much data is being escaped and whether there is non-ascii
text. Non-string types will be converted to a string before escaping.

<!-- ## `Bun.enableANSIColors()` -->

## `Bun.fileURLToPath()`

Converts a `file://` URL to an absolute path.

```ts
const path = Bun.fileURLToPath(new URL("file:///foo/bar.txt"));
console.log(path); // "/foo/bar.txt"
```

## `Bun.pathToFileURL()`

Converts an absolute path to a `file://` URL.

```ts
const url = Bun.pathToFileURL("/foo/bar.txt");
console.log(url); // "file:///foo/bar.txt"
```

<!-- Bun.hash; -->

## `Bun.gzipSync()`

Compresses a `Uint8Array` using zlib's GZIP algorithm.

```ts
const buf = Buffer.from("hello".repeat(100)); // Buffer extends Uint8Array
const compressed = Bun.gzipSync(buf);

buf; // => Uint8Array(500)
compressed; // => Uint8Array(30)
```

Optionally, pass a parameters object as the second argument:

{% details summary="zlib compression options"%}

```ts
export type ZlibCompressionOptions = {
  /**
   * The compression level to use. Must be between `-1` and `9`.
   * - A value of `-1` uses the default compression level (Currently `6`)
   * - A value of `0` gives no compression
   * - A value of `1` gives least compression, fastest speed
   * - A value of `9` gives best compression, slowest speed
   */
  level?: -1 | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9;
  /**
   * How much memory should be allocated for the internal compression state.
   *
   * A value of `1` uses minimum memory but is slow and reduces compression ratio.
   *
   * A value of `9` uses maximum memory for optimal speed. The default is `8`.
   */
  memLevel?: 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9;
  /**
   * The base 2 logarithm of the window size (the size of the history buffer).
   *
   * Larger values of this parameter result in better compression at the expense of memory usage.
   *
   * The following value ranges are supported:
   * - `9..15`: The output will have a zlib header and footer (Deflate)
   * - `-9..-15`: The output will **not** have a zlib header or footer (Raw Deflate)
   * - `25..31` (16+`9..15`): The output will have a gzip header and footer (gzip)
   *
   * The gzip header will have no file name, no extra data, no comment, no modification time (set to zero) and no header CRC.
   */
  windowBits?:
    | -9
    | -10
    | -11
    | -12
    | -13
    | -14
    | -15
    | 9
    | 10
    | 11
    | 12
    | 13
    | 14
    | 15
    | 25
    | 26
    | 27
    | 28
    | 29
    | 30
    | 31;
  /**
   * Tunes the compression algorithm.
   *
   * - `Z_DEFAULT_STRATEGY`: For normal data **(Default)**
   * - `Z_FILTERED`: For data produced by a filter or predictor
   * - `Z_HUFFMAN_ONLY`: Force Huffman encoding only (no string match)
   * - `Z_RLE`: Limit match distances to one (run-length encoding)
   * - `Z_FIXED` prevents the use of dynamic Huffman codes
   *
   * `Z_RLE` is designed to be almost as fast as `Z_HUFFMAN_ONLY`, but give better compression for PNG image data.
   *
   * `Z_FILTERED` forces more Huffman coding and less string matching, it is
   * somewhat intermediate between `Z_DEFAULT_STRATEGY` and `Z_HUFFMAN_ONLY`.
   * Filtered data consists mostly of small values with a somewhat random distribution.
   */
  strategy?: number;
};
```

{% /details %}

## `Bun.gunzipSync()`

Decompresses a `Uint8Array` using zlib's GUNZIP algorithm.

```ts
const buf = Buffer.from("hello".repeat(100)); // Buffer extends Uint8Array
const compressed = Bun.gunzipSync(buf);

const dec = new TextDecoder();
const uncompressed = Bun.inflateSync(compressed);
dec.decode(uncompressed);
// => "hellohellohello..."
```

## `Bun.deflateSync()`

Compresses a `Uint8Array` using zlib's DEFLATE algorithm.

```ts
const buf = Buffer.from("hello".repeat(100));
const compressed = Bun.deflateSync(buf);

buf; // => Uint8Array(25)
compressed; // => Uint8Array(10)
```

The second argument supports the same set of configuration options as [`Bun.gzipSync`](#bun.gzipSync).

## `Bun.inflateSync()`

DEcompresses a `Uint8Array` using zlib's INFLATE algorithm.

```ts
const buf = Buffer.from("hello".repeat(100));
const compressed = Bun.deflateSync(buf);

const dec = new TextDecoder();
const decompressed = Bun.inflateSync(compressed);
dec.decode(decompressed);
// => "hellohellohello..."
```

## `Bun.inspect()`

Serializes an object to a `string` exactly as it would be printed by `console.log`.

```ts
const obj = { foo: "bar" };
const str = Bun.inspect(obj);
// => '{\nfoo: "bar" \n}'

const arr = new Uint8Array([1, 2, 3]);
const str = Bun.inspect(arr);
// => "Uint8Array(3) [ 1, 2, 3 ]"
```

## `Bun.nanoseconds()`

Returns the number of nanoseconds since the current `bun` process started, as a `number`. Useful for high-precision timing and benchmarking.

```ts
Bun.nanoseconds();
// => 7288958
```

## `Bun.readableStreamTo*()`

Bun implements a set of convenience functions for asynchronously consuming the body of a `ReadableStream` and converting it to various binary formats.

```ts
const stream = (await fetch("https://bun.sh")).body;
stream; // => ReadableStream

await Bun.readableStreamToArrayBuffer(stream);
// => ArrayBuffer

await Bun.readableStreamToBlob(stream);
// => Blob

await Bun.readableStreamToJSON(stream);
// => object

await Bun.readableStreamToText(stream);
// => string

// returns all chunks as an array
await Bun.readableStreamToArray(stream);
// => unknown[]
```

## `Bun.resolveSync()`

Resolves a file path or module specifier using Bun's internal module resolution algorithm. The first argument is the path to resolve, and the second argument is the "root". If no match is found, an `Error` is thrown.

```ts
Bun.resolveSync("./foo.ts", "/path/to/project");
// => "/path/to/project/foo.ts"

Bun.resolveSync("zod", "/path/to/project");
// => "/path/to/project/node_modules/zod/index.ts"
```

To resolve relative to the current working directory, pass `process.cwd` or `"."` as the root.

```ts
Bun.resolveSync("./foo.ts", process.cwd());
Bun.resolveSync("./foo.ts", "/path/to/project");
```

To resolve relative to the directory containing the current file, pass `import.meta.dir`.

```ts
Bun.resolveSync("./foo.ts", import.meta.dir);
```


This page is intended as an introduction to working with binary data in JavaScript. Bun implements a number of data types and utilities for working with binary data, most of which are Web-standard. Any Bun-specific APIs will be noted as such.

Below is a quick "cheat sheet" that doubles as a table of contents. Click an item in the left column to jump to that section.

{% table %}

---

- [`TypedArray`](#typedarray)
- A family of classes that provide an `Array`-like interface for interacting with binary data. Includes `Uint8Array`, `Uint16Array`, `Int8Array`, and more.

---

- [`Buffer`](#buffer)
- A subclass of `Uint8Array` that implements a wide range of convenience methods. Unlike the other elements in this table, this is a Node.js API (which Bun implements). It can't be used in the browser.

---

- [`DataView`](#dataview)
- A class that provides a `get/set` API for writing some number of bytes to an `ArrayBuffer` at a particular byte offset. Often used reading or writing binary protocols.

---

- [`Blob`](#blob)
- A readonly blob of binary data usually representing a file. Has a MIME `type`, a `size`, and methods for converting to `ArrayBuffer`, `ReadableStream`, and string.

---

<!-- - [`File`](#file)
- _Browser only_. A subclass of `Blob` that represents a file. Has a `name` and `lastModified` timestamp. There is experimental support in Node.js v20; Bun does not support `File` yet; most of its functionality is provided by `BunFile`.

--- -->

- [`BunFile`](#bunfile)
- _Bun only_. A subclass of `Blob` that represents a lazily-loaded file on disk. Created with `Bun.file(path)`.

{% /table %}

## `ArrayBuffer` and views

Until 2009, there was no language-native way to store and manipulate binary data in JavaScript. ECMAScript v5 introduced a range of new mechanisms for this. The most fundamental building block is `ArrayBuffer`, a simple data structure that represents a sequence of bytes in memory.

```ts
// this buffer can store 8 bytes
const buf = new ArrayBuffer(8);
```

Despite the name, it isn't an array and supports none of the array methods and operators one might expect. In fact, there is no way to directly read or write values from an `ArrayBuffer`. There's very little you can do with one except check its size and create "slices" from it.

```ts
const buf = new ArrayBuffer(8);

buf.byteLength; // => 8

const slice = buf.slice(0, 4); // returns new ArrayBuffer
slice.byteLength; // => 4
```

To do anything interesting we need a construct known as a "view". A view is a class that _wraps_ an `ArrayBuffer` instance and lets you read and manipulate the underlying data. There are two types of views: _typed arrays_ and `DataView`.

### `DataView`

The `DataView` class is a lower-level interface for reading and manipulating the data in an `ArrayBuffer`.

Below we create a new `DataView` and set the first byte to 5.

```ts
const buf = new ArrayBuffer(4);
// [0x0, 0x0, 0x0, 0x0]

const dv = new DataView(buf);
dv.setUint8(0, 3); // write value 3 at byte offset 0
dv.getUint8(0); // => 3
// [0x11, 0x0, 0x0, 0x0]
```

Now lets write a `Uint16` at byte offset `1`. This requires two bytes. We're using the value `513`, which is `2 * 256 + 1`; in bytes, that's `00000010 00000001`.

```ts
dv.setUint16(1, 513);
// [0x11, 0x10, 0x1, 0x0]

console.log(dv.getUint16(1)); // => 513
```

We've now assigned a value to the first three bytes in our underlying `ArrayBuffer`. Even though the second and third bytes were created using `setUint16()`, we can still read each of its component bytes using `getUint8()`.

```ts
console.log(dv.getUint8(1)); // => 2
console.log(dv.getUint8(2)); // => 1
```

Attempting to write a value that requires more space than is available in the underlying `ArrayBuffer` will cuase an error. Below we attempt to write a `Float64` (which requires 8 bytes) at byte offset `0`, but there are only four total bytes in the buffer.

```ts
dv.setFloat64(0, 3.1415);
// ^ RangeError: Out of bounds access
```

The following methods are available on `DataView`:

{% table %}

- Getters
- Setters

---

- [`getBigInt64()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/DataView/getBigInt64)
- [`setBigInt64()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/DataView/setBigInt64)

---

- [`getBigUint64()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/DataView/getBigUint64)
- [`setBigUint64()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/DataView/setBigUint64)

---

- [`getFloat32()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/DataView/getFloat32)
- [`setFloat32()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/DataView/setFloat32)

---

- [`getFloat64()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/DataView/getFloat64)
- [`setFloat64()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/DataView/setFloat64)

---

- [`getInt16()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/DataView/getInt16)
- [`setInt16()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/DataView/setInt16)

---

- [`getInt32()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/DataView/getInt32)
- [`setInt32()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/DataView/setInt32)

---

- [`getInt8()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/DataView/getInt8)
- [`setInt8()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/DataView/setInt8)

---

- [`getUint16()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/DataView/getUint16)
- [`setUint16()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/DataView/setUint16)

---

- [`getUint32()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/DataView/getUint32)
- [`setUint32()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/DataView/setUint32)

---

- [`getUint8()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/DataView/getUint8)
- [`setUint8()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/DataView/setUint8)

{% /table %}

### `TypedArray`

Typed arrays are a family of classes that provide an `Array`-like interface for interacting with data in an `ArrayBuffer`. Whereas a `DataView` lets you write numbers of varying size at a particular offset, a `TypedArray` interprets the underlying bytes as an array of numbers, each of a fixed size.

{% callout %}
**Note** — It's common to refer to this family of classes collectively by their shared superclass `TypedArray`. This class as _internal_ to JavaScript; you can't directly create instances of it, and `TypedArray` is not defined in the global scope. Think of it as an `interface` or an abstract class.
{% /callout %}

```ts
const buffer = new ArrayBuffer(3);
const arr = new Uint8Array(buffer);

// contents are initialized to zero
console.log(arr); // Uint8Array(3) [0, 0, 0]

// assign values like an array
arr[0] = 0;
arr[1] = 10;
arr[2] = 255;
arr[3] = 255; // no-op, out of bounds
```

While an `ArrayBuffer` is a generic sequence of bytes, these typed array classes interpret the bytes as an array of numbers of a given byte size.
The top row contains the raw bytes, and the later rows contain how these bytes will be interpreted when _viewed_ using different typed array classes.

The following classes are typed arrays, along with a description of how they interpret the bytes in an `ArrayBuffer`:

{% table %}

- Class
- Description

---

- [`Uint8Array`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Uint8Array)
- Every one (1) byte is interpreted as an unsigned 8-bit integer. Range 0 to 255.

---

- [`Uint16Array`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Uint16Array)
- Every two (2) bytes are interpreted as an unsigned 16-bit integer. Range 0 to 65535.

---

- [`Uint32Array`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Uint32Array)
- Every four (4) bytes are interpreted as an unsigned 32-bit integer. Range 0 to 4294967295.

---

- [`Int8Array`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Int8Array)
- Every one (1) byte is interpreted as a signed 8-bit integer. Range -128 to 127.

---

- [`Int16Array`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Int16Array)
- Every two (2) bytes are interpreted as a signed 16-bit integer. Range -32768 to 32767.

---

- [`Int32Array`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Int32Array)
- Every four (4) bytes are interpreted as a signed 32-bit integer. Range -2147483648 to 2147483647.

---

- [`Float32Array`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Float32Array)
- Every four (4) bytes are interpreted as a 32-bit floating point number. Range -3.4e38 to 3.4e38.

---

- [`Float64Array`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Float64Array)
- Every eight (8) bytes are interpreted as a 64-bit floating point number. Range -1.7e308 to 1.7e308.

---

- [`BigInt64Array`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/BigInt64Array)
- Every eight (8) bytes are interpreted as an unsigned `BigInt`. Range -9223372036854775808 to 9223372036854775807 (though `BigInt` is capable of representing larger numbers).

---

- [`BigUint64Array`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/BigUint64Array)
- Every eight (8) bytes are interpreted as an unsigned `BigInt`. Range 0 to 18446744073709551615 (though `BigInt` is capable of representing larger numbers).

---

- [`Uint8ClampedArray`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Uint8ClampedArray)
- Same as `Uint8Array`, but automatically "clamps" to the range 0-255 when assigning a value to an element.

{% /table %}

The table below demonstrates how the bytes in an `ArrayBuffer` are interpreted when viewed using different typed array classes.

{% table %}

---

- `ArrayBuffer`
- `00000000`
- `00000001`
- `00000010`
- `00000011`
- `00000100`
- `00000101`
- `00000110`
- `00000111`

---

- `Uint8Array`
- 0
- 1
- 2
- 3
- 4
- 5
- 6
- 7

---

- `Uint16Array`
- 256 (`1 * 256 + 0`) {% colspan=2 %}
- 770 (`3 * 256 + 2`) {% colspan=2 %}
- 1284 (`5 * 256 + 4`) {% colspan=2 %}
- 1798 (`7 * 256 + 6`) {% colspan=2 %}

---

- `Uint32Array`
- 50462976 {% colspan=4 %}
- 117835012 {% colspan=4 %}

---

- `BigUint64Array`
- 506097522914230528n {% colspan=8 %}

{% /table %}

To create a typed array from a pre-defined `ArrayBuffer`:

```ts
// create typed array from ArrayBuffer
const buf = new ArrayBuffer(10);
const arr = new Uint8Array(buf);

arr[0] = 30;
arr[1] = 60;

// all elements are initialized to zero
console.log(arr); // => Uint8Array(10) [ 30, 60, 0, 0, 0, 0, 0, 0, 0, 0 ];
```

If we tried to instantiate a `Uint32Array` from this same `ArrayBuffer`, we'd get an error.

```ts
const buf = new ArrayBuffer(10);
const arr = new Uint32Array(buf);
//          ^  RangeError: ArrayBuffer length minus the byteOffset
//             is not a multiple of the element size
```

A `Uint32` value requires four bytes (16 bits). Because the `ArrayBuffer` is 10 bytes long, there's no way to cleanly divide its contents into 4-byte chunks.

To fix this, we can create a typed array over a particular "slice" of an `ArrayBuffer`. The `Uint16Array` below only "views" the _first_ 8 bytes of the underlying `ArrayBuffer`. To achieve these, we specify a `byteOffset` of `0` and a `length` of `2`, which indicates the number of `Uint32` numbers we want our array to hold.

```ts
// create typed array from ArrayBuffer slice
const buf = new ArrayBuffer(10);
const arr = new Uint32Array(buf, 0, 2);

/*
  buf    _ _ _ _ _ _ _ _ _ _    10 bytes
  arr   [_______,_______]       2 4-byte elements
*/

arr.byteOffset; // 0
arr.length; // 2
```

You don't need to explicitly create an `ArrayBuffer` instance; you can instead directly specify a length in the typed array constructor:

```ts
const arr2 = new Uint8Array(5);

// all elements are initialized to zero
// => Uint8Array(5) [0, 0, 0, 0, 0]
```

Typed arrays can also be instantiated directly from an array of numbers, or another typed array:

```ts
// from an array of numbers
const arr1 = new Uint8Array([0, 1, 2, 3, 4, 5, 6, 7]);
arr1[0]; // => 0;
arr1[7]; // => 7;

// from another typed array
const arr2 = new Uint8Array(arr);
```

Broadly speaking, typed arrays provide the same methods as regular arrays, with a few exceptions. For example, `push` and `pop` are not available on typed arrays, because they would require resizing the underlying `ArrayBuffer`.

```ts
const arr = new Uint8Array([0, 1, 2, 3, 4, 5, 6, 7]);

// supports common array methods
arr.filter(n => n > 128); // Uint8Array(1) [255]
arr.map(n => n * 2); // Uint8Array(8) [0, 2, 4, 6, 8, 10, 12, 14]
arr.reduce((acc, n) => acc + n, 0); // 28
arr.forEach(n => console.log(n)); // 0 1 2 3 4 5 6 7
arr.every(n => n < 10); // true
arr.find(n => n > 5); // 6
arr.includes(5); // true
arr.indexOf(5); // 5
```

Refer to the [MDN documentation](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/TypedArray) for more information on the properties and methods of typed arrays.

### `Uint8Array`

It's worth specifically highlighting `Uint8Array`, as it represents a classic "byte array"—a sequence of 8-bit unsigned integers between 0 and 255. This is the most common typed array you'll encounter in JavaScript.

It is the return value of [`TextEncoder#encode`](https://developer.mozilla.org/en-US/docs/Web/API/TextEncoder), and the input type of [`TextDecoder#decode`](https://developer.mozilla.org/en-US/docs/Web/API/TextDecoder), two utility classes designed to translate strings and various binary encodings, most notably `"utf-8"`.

```ts
const encoder = new TextEncoder();
const bytes = encoder.encode("hello world");
// => Uint8Array(11) [ 104, 101, 108, 108, 111, 32, 119, 111, 114, 108, 100 ]

const decoder = new TextDecoder();
const text = decoder.decode(bytes);
// => hello world
```

### `Buffer`

Bun implements `Buffer`, a Node.js API for working with binary data that pre-dates the introduction of typed arrays in the JavaScript spec. It has since been re-implemented as a subclass of `Uint8Array`. It provides a wide range of methods, including several Array-like and `DataView`-like methods.

```ts
const buf = Buffer.from("hello world");
// => Buffer(16) [ 116, 104, 105, 115, 32, 105, 115, 32, 97, 32, 115, 116, 114, 105, 110, 103 ]

buf.length; // => 11
buf[0]; // => 104, ascii for 'h'
buf.writeUInt8(72, 0); // => ascii for 'H'

console.log(buf.toString());
// => Hello world
```

For complete documentation, refer to the [Node.js documentation](https://nodejs.org/api/buffer.html).

## `Blob`

`Blob` is a Web API commonly used for representing files. `Blob` was initially implemented in browsers (unlike `ArrayBuffer` which is part of JavaScript itself), but it is now supported in Node and Bun.

It isn't common to directly create `Blob` instances. More often, you'll recieve instances of `Blob` from an external source (like an `<input type="file">` element in the browser) or library. That said, it is possible to create a `Blob` from one or more string or binary "blob parts".

```ts
const blob = new Blob(["<html>Hello</html>"], {
  type: "text/html",
});

blob.type; // => text/html
blob.size; // => 19
```

These parts can be `string`, `ArrayBuffer`, `TypedArray`, `DataView`, or other `Blob` instances. The blob parts are concatenated together in the order they are provided.

```ts
const blob = new Blob([
  "<html>",
  new Blob(["<body>"]),
  new Uint8Array([104, 101, 108, 108, 111]), // "hello" in binary
  "</body></html>",
]);
```

The contents of a `Blob` can be asynchronously read in various formats.

```ts
await blob.text(); // => <html><body>hello</body></html>
await blob.arrayBuffer(); // => ArrayBuffer (copies contents)
await blob.stream(); // => ReadableStream
```

### `BunFile`

`BunFile` is a subclass of `Blob` used to represent a lazily-loaded file on disk. Like `File`, it adds a `name` and `lastModified` property. Unlike `File`, it does not require the file to be loaded into memory.

```ts
const file = Bun.file("index.txt");
// => BunFile
```

### `File`

{% callout %}
Browser only. Experimental support in Node.js 20.
{% /callout %}

[`File`](https://developer.mozilla.org/en-US/docs/Web/API/File) is a subclass of `Blob` that adds a `name` and `lastModified` property. It's commonly used in the browser to represent files uploaded via a `<input type="file">` element. Node.js and Bun implement `File`.

```ts
// on browser!
// <input type="file" id="file" />

const files = document.getElementById("file").files;
// => File[]
```

```ts
const file = new File(["<html>Hello</html>"], "index.html", {
  type: "text/html",
});
```

Refer to the [MDN documentation](https://developer.mozilla.org/en-US/docs/Web/API/Blob) for complete docs information.

## Streams

Streams are an important abstraction for working with binary data without loading it all into memory at once. They are commonly used for reading and writing files, sending and receiving network requests, and processing large amounts of data.

Bun implements the Web APIs [`ReadableStream`](https://developer.mozilla.org/en-US/docs/Web/API/ReadableStream) and [`WritableStream`](https://developer.mozilla.org/en-US/docs/Web/API/ReadableStream).

{% callout %}
Bun also implements the `node:stream` module, including [`Readable`](https://nodejs.org/api/stream.html#stream_readable_streams), [`Writable`](https://nodejs.org/api/stream.html#stream_writable_streams), and [`Duplex`](https://nodejs.org/api/stream.html#stream_duplex_and_transform_streams). For complete documentation, refer to the Node.js docs.
{% /callout %}

To create a simple readable stream:

```ts
const stream = new ReadableStream({
  start(controller) {
    controller.enqueue("hello");
    controller.enqueue("world");
    controller.close();
  },
});
```

The contents of this stream can be read chunk-by-chunk with `for await` syntax.

```ts
for await (const chunk of stream) {
  console.log(chunk);
  // => "hello"
  // => "world"
}
```

For a more complete discusson of streams in Bun, see [API > Streams](/docs/api/streams).

## Conversion

Converting from one binary format to another is a common task. This section is intended as a reference.

### From `ArrayBuffer`

Since `ArrayBuffer` stores the data that underlies other binary structures like `TypedArray`, the snippets below are not _converting_ from `ArrayBuffer` to another format. Instead, they are _creating_ a new instance using the data stored underlying data.

#### To `TypedArray`

```ts
new Uint8Array(buf);
```

#### To `DataView`

```ts
new DataView(buf);
```

#### To `Buffer`

```ts
// create Buffer over entire ArrayBuffer
Buffer.from(buf);

// create Buffer over a slice of the ArrayBuffer
Buffer.from(buf, 0, 10);
```

#### To `string`

```ts
new TextDecoder().decode(buf);
```

#### To `number[]`

```ts
Array.from(new Uint8Array(buf));
```

#### To `Blob`

```ts
new Blob([buf], { type: "text/plain" });
```

<!-- #### To `File`

```ts
new File([buf], "filename.txt", { type: "text/plain", lastModified: Date.now() });
``` -->

#### To `ReadableStream`

The following snippet creates a `ReadableStream` and enqueues the entire `ArrayBuffer` as a single chunk.

```ts
new ReadableStream({
  start(controller) {
    controller.enqueue(buf);
    controller.close();
  },
});
```

{% details summary="With chunking" %}
To stream the `ArrayBuffer` in chunks, use a `Uint8Array` view and enqueue each chunk.

```ts
const view = new Uint8Array(buf);
const chunkSize = 1024;

new ReadableStream({
  start(controller) {
    for (let i = 0; i < view.length; i += chunkSize) {
      controller.enqueue(view.slice(i, i + chunkSize));
    }
    controller.close();
  },
});
```

{% /details %}

### From `TypedArray`

#### To `ArrayBuffer`

This retrieves the underlying `ArrayBuffer`. Note that a `TypedArray` can be a view of a _slice_ of the underlying buffer, so the sizes may differ.

```ts
arr.buffer;
```

#### To `DataView`

To creates a `DataView` over the same byte range as the TypedArray.

```ts
new DataView(arr.buffer, arr.byteOffset, arr.byteLength);
```

#### To `Buffer`

```ts
Buffer.from(arr);
```

#### To `string`

```ts
new TextDecoder().decode(arr);
```

#### To `number[]`

```ts
Array.from(arr);
```

#### To `Blob`

```ts
new Blob([arr.buffer], { type: "text/plain" });
```

<!-- #### To `File`

```ts
new File([arr.buffer], "filename.txt", { type: "text/plain", lastModified: Date.now() });
``` -->

#### To `ReadableStream`

```ts
new ReadableStream({
  start(controller) {
    controller.enqueue(arr);
    controller.close();
  },
});
```

{% details summary="With chunking" %}
To stream the `ArrayBuffer` in chunks, split the `TypedArray` into chunks and enqueue each one individually.

```ts
new ReadableStream({
  start(controller) {
    for (let i = 0; i < arr.length; i += chunkSize) {
      controller.enqueue(arr.slice(i, i + chunkSize));
    }
    controller.close();
  },
});
```

{% /details %}

### From `DataView`

#### To `ArrayBuffer`

```ts
view.buffer;
```

#### To `TypedArray`

Only works if the `byteLength` of the `DataView` is a multiple of the `BYTES_PER_ELEMENT` of the `TypedArray` subclass.

```ts
new Uint8Array(view.buffer, view.byteOffset, view.byteLength);
new Uint16Array(view.buffer, view.byteOffset, view.byteLength / 2);
new Uint32Array(view.buffer, view.byteOffset, view.byteLength / 4);
// etc...
```

#### To `Buffer`

```ts
Buffer.from(view.buffer, view.byteOffset, view.byteLength);
```

#### To `string`

```ts
new TextDecoder().decode(view);
```

#### To `number[]`

```ts
Array.from(view);
```

#### To `Blob`

```ts
new Blob([view.buffer], { type: "text/plain" });
```

<!-- #### To `File`

```ts
new File([view.buffer], "filename.txt", { type: "text/plain", lastModified: Date.now() });
``` -->

#### To `ReadableStream`

```ts
new ReadableStream({
  start(controller) {
    controller.enqueue(view.buffer);
    controller.close();
  },
});
```

{% details summary="With chunking" %}
To stream the `ArrayBuffer` in chunks, split the `DataView` into chunks and enqueue each one individually.

```ts
new ReadableStream({
  start(controller) {
    for (let i = 0; i < view.byteLength; i += chunkSize) {
      controller.enqueue(view.buffer.slice(i, i + chunkSize));
    }
    controller.close();
  },
});
```

{% /details %}

### From `Buffer`

#### To `ArrayBuffer`

```ts
buf.buffer;
```

#### To `TypedArray`

```ts
new Uint8Array(buf);
```

#### To `DataView`

```ts
new DataView(buf.buffer, buf.byteOffset, buf.byteLength);
```

#### To `string`

```ts
buf.toString();
```

#### To `number[]`

```ts
Array.from(buf);
```

#### To `Blob`

```ts
new Blob([buf], { type: "text/plain" });
```

<!-- #### To `File`

```ts
new File([buf], "filename.txt", { type: "text/plain", lastModified: Date.now() });
``` -->

#### To `ReadableStream`

```ts
new ReadableStream({
  start(controller) {
    controller.enqueue(buf);
    controller.close();
  },
});
```

{% details summary="With chunking" %}
To stream the `ArrayBuffer` in chunks, split the `Buffer` into chunks and enqueue each one individually.

```ts
new ReadableStream({
  start(controller) {
    for (let i = 0; i < buf.length; i += chunkSize) {
      controller.enqueue(buf.slice(i, i + chunkSize));
    }
    controller.close();
  },
});
```

{% /details %}

### From `Blob`

#### To `ArrayBuffer`

The `Blob` class provides a convenience method for this purpose.

```ts
await blob.arrayBuffer();
```

#### To `TypedArray`

```ts
new Uint8Array(await blob.arrayBuffer());
```

#### To `DataView`

```ts
new DataView(await blob.arrayBuffer());
```

#### To `Buffer`

```ts
Buffer.from(await blob.arrayBuffer());
```

#### To `string`

```ts
await blob.text();
```

#### To `number[]`

```ts
Array.from(new Uint8Array(await blob.arrayBuffer()));
```

#### To `ReadableStream`

```ts
blob.stream();
```

<!-- ### From `File` -->

### From `ReadableStream`

It's common to use [`Response`](https://developer.mozilla.org/en-US/docs/Web/API/Response) as a convenient intermediate representation to make it easier to convert `ReadableStream` to other formats.

```ts
stream; // ReadableStream

const buffer = new Response(stream).arrayBuffer();
```

However this approach is verbose and adds overhead that slows down overall performance unnecessarily. Bun implements a set of optimized convenience functions for converting `ReadableStream` various binary formats.

#### To `ArrayBuffer`

```ts
// with Response
new Response(stream).arrayBuffer();

// with Bun function
Bun.readableStreamToArrayBuffer(stream);
```

#### To `TypedArray`

```ts
// with Response
const buf = await new Response(stream).arrayBuffer();
new Uint8Array(buf);

// with Bun function
new Uint8Array(Bun.readableStreamToArrayBuffer(stream));
```

#### To `DataView`

```ts
// with Response
const buf = await new Response(stream).arrayBuffer();
new DataView(buf);

// with Bun function
new DataView(Bun.readableStreamToArrayBuffer(stream));
```

#### To `Buffer`

```ts
// with Response
const buf = await new Response(stream).arrayBuffer();
Buffer.from(buf);

// with Bun function
Buffer.from(Bun.readableStreamToArrayBuffer(stream));
```

#### To `string`

```ts
// with Response
new Response(stream).text();

// with Bun function
await Bun.readableStreamToString(stream);
```

#### To `number[]`

```ts
// with Response
const buf = await new Response(stream).arrayBuffer();
Array.from(new Uint8Array(buf));

// with Bun function
Array.from(new Uint8Array(Bun.readableStreamToArrayBuffer(stream)));
```

Bun provides a utility for resolving a `ReadableStream` to an array of its chunks. Each chunk may be a string, typed array, or `ArrayBuffer`.

```ts
// with Bun function
Bun.readableStreamToArray(stream);
```

#### To `Blob`

```ts
new Response(stream).blob();
```

<!-- #### To `File`

```ts
new Response(stream)
  .blob()
  .then(blob => new File([blob], "filename.txt", { type: "text/plain", lastModified: Date.now() }));
``` -->

#### To `ReadableStream`

To split a `ReadableStream` into two streams that can be consumed independently:

```ts
const [a, b] = stream.tee();
```

<!-- - Use Buffer
- TextEncoder
- `Bun.ArrayBufferSink`
- ReadableStream
- AsyncIterator
- TypedArray vs ArrayBuffer vs DataView
- Bun.indexOfLine
- “direct” readablestream
  - readable stream has assumptions about
  - its very generic
  - all data is copies and queued
  - direct : no queueing
  - just a write function
  - you can write strings
  - more synchronous
  - corking works better -->


{% callout %}

<!-- **Note** — The `Bun.file` and `Bun.write` APIs documented on this page are heavily optimized and represent the recommended way to perform file-system tasks using Bun. Existing Node.js projects may use Bun's [nearly complete](/docs/runtime/nodejs-apis#node_fs) implementation of the [`node:fs`](https://nodejs.org/api/fs.html) module. -->

**Note** — The `Bun.file` and `Bun.write` APIs documented on this page are heavily optimized and represent the recommended way to perform file-system tasks using Bun. For operations that are not yet available with `Bun.file`, such as `mkdir`, you can use Bun's [nearly complete](/docs/runtime/nodejs-apis#node_fs) implementation of the [`node:fs`](https://nodejs.org/api/fs.html) module.

{% /callout %}

Bun provides a set of optimized APIs for reading and writing files.

## Reading files (`Bun.file()`)

`Bun.file(path): BunFile`

Create a `BunFile` instance with the `Bun.file(path)` function. A `BunFile` represents a lazily-loaded file; initializing it does not actually read the file from disk.

```ts
const foo = Bun.file("foo.txt"); // relative to cwd
foo.size; // number of bytes
foo.type; // MIME type
```

The reference conforms to the [`Blob`](https://developer.mozilla.org/en-US/docs/Web/API/Blob) interface, so the contents can be read in various formats.

```ts
const foo = Bun.file("foo.txt");

await foo.text(); // contents as a string
await foo.stream(); // contents as ReadableStream
await foo.arrayBuffer(); // contents as ArrayBuffer
```

File references can also be created using numerical [file descriptors](https://en.wikipedia.org/wiki/File_descriptor) or `file://` URLs.

```ts
Bun.file(1234);
Bun.file(new URL(import.meta.url)); // reference to the current file
```

A `BunFile` can point to a location on disk where a file does not exist.

```ts
const notreal = Bun.file("notreal.txt");
notreal.size; // 0
notreal.type; // "text/plain;charset=utf-8"
```

The default MIME type is `text/plain;charset=utf-8`, but it can be overridden by passing a second argument to `Bun.file`.

```ts
const notreal = Bun.file("notreal.json", { type: "application/json" });
notreal.type; // => "application/json;charset=utf-8"
```

For convenience, Bun exposes `stdin`, `stdout` and `stderr` as instances of `BunFile`.

```ts
Bun.stdin; // readonly
Bun.stdout;
Bun.stderr;
```

## Writing files (`Bun.write()`)

`Bun.write(destination, data): Promise<number>`

The `Bun.write` function is a multi-tool for writing payloads of all kinds to disk.

The first argument is the `destination` which can have any of the following types:

- `string`: A path to a location on the file system. Use the `"path"` module to manipulate paths.
- `URL`: A `file://` descriptor.
- `BunFile`: A file reference.

The second argument is the data to be written. It can be any of the following:

- `string`
- `Blob` (including `BunFile`)
- `ArrayBuffer` or `SharedArrayBuffer`
- `TypedArray` (`Uint8Array`, et. al.)
- `Response`

All possible permutations are handled using the fastest available system calls on the current platform.

{% details summary="See syscalls" %}

{% table %}

- Output
- Input
- System call
- Platform

---

- file
- file
- copy_file_range
- Linux

---

- file
- pipe
- sendfile
- Linux

---

- pipe
- pipe
- splice
- Linux

---

- terminal
- file
- sendfile
- Linux

---

- terminal
- terminal
- sendfile
- Linux

---

- socket
- file or pipe
- sendfile (if http, not https)
- Linux

---

- file (doesn't exist)
- file (path)
- clonefile
- macOS

---

- file (exists)
- file
- fcopyfile
- macOS

---

- file
- Blob or string
- write
- macOS

---

- file
- Blob or string
- write
- Linux

{% /table %}

{% /details %}

To write a string to disk:

```ts
const data = `It was the best of times, it was the worst of times.`;
await Bun.write("output.txt", data);
```

To copy a file to another location on disk:

```js
const input = Bun.file("input.txt");
const output = Bun.file("output.txt"); // doesn't exist yet!
await Bun.write(output, input);
```

To write a byte array to disk:

```ts
const encoder = new TextEncoder();
const data = encoder.encode("datadatadata"); // Uint8Array
await Bun.write("output.txt", data);
```

To write a file to `stdout`:

```ts
const input = Bun.file("input.txt");
await Bun.write(Bun.stdout, input);
```

To write an HTTP response to disk:

```ts
const response = await fetch("https://bun.sh");
await Bun.write("index.html", response);
```

## Incremental writing with `FileSink`

Bun provides a native incremental file writing API called `FileSink`. To retrieve a `FileSink` instance from a `BunFile`:

```ts
const file = Bun.file("output.txt");
const writer = file.writer();
```

To incrementally write to the file, call `.write()`.

```ts
const file = Bun.file("output.txt");
const writer = file.writer();

writer.write("it was the best of times\n");
writer.write("it was the worst of times\n");
```

These chunks will be buffered internally. To flush the buffer to disk, use `.flush()`. This returns the number of flushed bytes.

```ts
writer.flush(); // write buffer to disk
```

The buffer will also auto-flush when the `FileSink`'s _high water mark_ is reached; that is, when its internal buffer is full. This value can be configured.

```ts
const file = Bun.file("output.txt");
const writer = file.writer({ highWaterMark: 1024 * 1024 }); // 1MB
```

To flush the buffer and close the file:

```ts
writer.end();
```

Note that, by default, the `bun` process will stay alive until this `FileSink` is explicitly closed with `.end()`. To opt out of this behavior, you can "unref" the instance.

```ts
writer.unref();

// to "re-ref" it later
writer.ref();
```

## Benchmarks

The following is a 3-line implementation of the Linux `cat` command.

```ts#cat.ts
// Usage
// $ bun ./cat.ts ./path-to-file

import { resolve } from "path";

const path = resolve(process.argv.at(-1));
await Bun.write(Bun.stdout, Bun.file(path));
```

To run the file:

```bash
$ bun ./cat.ts ./path-to-file
```

It runs 2x faster than GNU `cat` for large files on Linux.

{% image src="/images/cat.jpg" /%}

## Reference

```ts
interface Bun {
  stdin: BunFile;
  stdout: BunFile;
  stderr: BunFile;

  file(path: string | number | URL, options?: { type?: string }): BunFile;

  write(
    destination: string | number | BunFile | URL,
    input:
      | string
      | Blob
      | ArrayBuffer
      | SharedArrayBuffer
      | TypedArray
      | Response,
  ): Promise<number>;
}

interface BunFile {
  readonly size: number;
  readonly type: string;

  text(): Promise<string>;
  stream(): Promise<ReadableStream>;
  arrayBuffer(): Promise<ArrayBuffer>;
  json(): Promise<any>;
  writer(params: { highWaterMark?: number }): FileSink;
}

export interface FileSink {
  write(
    chunk: string | ArrayBufferView | ArrayBuffer | SharedArrayBuffer,
  ): number;
  flush(): number | Promise<number>;
  end(error?: Error): number | Promise<number>;
  start(options?: { highWaterMark?: number }): void;
  ref(): void;
  unref(): void;
}
```


Bun.js has fast paths for common use cases that make Web APIs live up to the performance demands of servers and CLIs.

`Bun.file(path)` returns a [`Blob`](https://developer.mozilla.org/en-US/docs/Web/API/Blob) that represents a lazily-loaded file.

When you pass a file blob to `Bun.write`, Bun automatically uses a faster system call:

```js
const blob = Bun.file("input.txt");
await Bun.write("output.txt", blob);
```

On Linux, this uses the [`copy_file_range`](https://man7.org/linux/man-pages/man2/copy_file_range.2.html) syscall and on macOS, this becomes `clonefile` (or [`fcopyfile`](https://developer.apple.com/library/archive/documentation/System/Conceptual/ManPages_iPhoneOS/man3/copyfile.3.html)).

`Bun.write` also supports [`Response`](https://developer.mozilla.org/en-US/docs/Web/API/Response) objects. It automatically converts to a [`Blob`](https://developer.mozilla.org/en-US/docs/Web/API/Blob).

```js
// Eventually, this will stream the response to disk but today it buffers
await Bun.write("index.html", await fetch("https://example.com"));
```


{% callout %}

Bun implements the `createHash` and `createHmac` functions from [`node:crypto`](https://nodejs.org/api/crypto.html) in addition to the Bun-native APIs documented below.

{% /callout %}

## `Bun.password`

{% callout %}
**Note** — Added in Bun 0.6.8.
{% /callout %}

`Bun.password` is a collection of utility functions for hashing and verifying passwords with various cryptographically secure algorithms.

```ts
const password = "super-secure-pa$$word";

const hash = await Bun.password.hash(password);
// => $argon2id$v=19$m=65536,t=2,p=1$tFq+9AVr1bfPxQdh6E8DQRhEXg/M/SqYCNu6gVdRRNs$GzJ8PuBi+K+BVojzPfS5mjnC8OpLGtv8KJqF99eP6a4

const isMatch = await Bun.password.verify(password, hash);
// => true
```

The second argument to `Bun.password.hash` accepts a params object that lets you pick and configure the hashing algorithm.

```ts
const password = "super-secure-pa$$word";

// use argon2 (default)
const argonHash = await Bun.password.hash(password, {
  algorithm: "argon2id", // "argon2id" | "argon2i" | "argon2d"
  memoryCost: 4, // memory usage in kibibytes
  timeCost: 3, // the number of iterations
});

// use bcrypt
const bcryptHash = await Bun.password.hash(password, {
  algorithm: "bcrypt",
  cost: 4, // number between 4-31
});
```

The algorithm used to create the hash is stored in the hash itself. When using `bcrypt`, the returned hash is encoded in [Modular Crypt Format](https://passlib.readthedocs.io/en/stable/modular_crypt_format.html) for compatibility with most existing `bcrypt` implementations; with `argon2` the result is encoded in the newer [PHC format](https://github.com/P-H-C/phc-string-format/blob/master/phc-sf-spec.md).

The `verify` function automatically detects the algorithm based on the input hash and use the correct verification method. It can correctly infer the algorithm from both PHC- or MCF-encoded hashes.

```ts
const password = "super-secure-pa$$word";

const hash = await Bun.password.hash(password, {
  /* config */
});

const isMatch = await Bun.password.verify(password, hash);
// => true
```

Synchronous versions of all functions are also available. Keep in mind that these functions are computationally expensive, so using a blocking API may degrade application performance.

```ts
const password = "super-secure-pa$$word";

const hash = Bun.password.hashSync(password, {
  /* config */
});

const isMatch = Bun.password.verifySync(password, hash);
// => true
```

## `Bun.hash`

`Bun.hash` is a collection of utilities for _non-cryptographic_ hashing. Non-cryptographic hashing algorithms are optimized for speed of computation over collision-resistance or security.

The standard `Bun.hash` functions uses [Wyhash](https://github.com/wangyi-fudan/wyhash) to generate a 64-bit hash from an input of arbitrary size.

```ts
Bun.hash("some data here");
// 976213160445840
```

The input can be a string, `TypedArray`, `DataView`, `ArrayBuffer`, or `SharedArrayBuffer`.

```ts
const arr = new Uint8Array([1, 2, 3, 4]);

Bun.hash("some data here");
Bun.hash(arr);
Bun.hash(arr.buffer);
Bun.hash(new DataView(arr.buffer));
```

Optionally, an integer seed can be specified as the second parameter.

```ts
Bun.hash("some data here", 1234);
// 1173484059023252
```

Additional hashing algorithms are available as properties on `Bun.hash`. The API is the same for each.

```ts
Bun.hash.wyhash("data", 1234); // equivalent to Bun.hash()
Bun.hash.crc32("data", 1234);
Bun.hash.adler32("data", 1234);
Bun.hash.cityHash32("data", 1234);
Bun.hash.cityHash64("data", 1234);
Bun.hash.murmur32v3("data", 1234);
Bun.hash.murmur64v2("data", 1234);
```

## `Bun.CryptoHasher`

`Bun.CryptoHasher` is a general-purpose utility class that lets you incrementally compute a hash of string or binary data using a range of cryptographic hash algorithms. The following algorithms are supported:

- `"blake2b256"`
- `"md4"`
- `"md5"`
- `"ripemd160"`
- `"sha1"`
- `"sha224"`
- `"sha256"`
- `"sha384"`
- `"sha512"`
- `"sha512-256"`

```ts
const hasher = new Bun.CryptoHasher("sha256");
hasher.update("hello world");
hasher.digest();
// Uint8Array(32) [ <byte>, <byte>, ... ]
```

Once initialized, data can be incrementally fed to to the hasher using `.update()`. This method accepts `string`, `TypedArray`, and `ArrayBuffer`.

```ts
const hasher = new Bun.CryptoHasher();

hasher.update("hello world");
hasher.update(new Uint8Array([1, 2, 3]));
hasher.update(new ArrayBuffer(10));
```

If a `string` is passed, an optional second parameter can be used to specify the encoding (default `'utf-8'`). The following encodings are supported:

{% table %}

---

- Binary encodings
- `"base64"` `"base64url"` `"hex"` `"binary"`

---

- Character encodings
- `"utf8"` `"utf-8"` `"utf16le"` `"latin1"`

---

- Legacy character encodings
- `"ascii"` `"binary"` `"ucs2"` `"ucs-2"`

{% /table %}

```ts
hasher.update("hello world"); // defaults to utf8
hasher.update("hello world", "hex");
hasher.update("hello world", "base64");
hasher.update("hello world", "latin1");
```

After the data has been feed into the hasher, a final hash can be computed using `.digest()`. By default, this method returns a `Uint8Array` containing the hash.

```ts
const hasher = new Bun.CryptoHasher();
hasher.update("hello world");

hasher.digest();
// => Uint8Array(32) [ 185, 77, 39, 185, 147, ... ]
```

The `.digest()` method can optionally return the hash as a string. To do so, specify an encoding:

```ts
hasher.digest("base64");
// => "uU0nuZNNPgilLlLX2n2r+sSE7+N6U4DukIj3rOLvzek="

hasher.digest("hex");
// => "b94d27b9934d3e08a52e52d7da7dabfac484efe37a5380ee9088f7ace2efcde9"
```

Alternatively, the method can write the hash into a pre-existing `TypedArray` instance. This may be desirable in some performance-sensitive applications.

```ts
const arr = new Uint8Array(32);

hasher.digest(arr);

console.log(arr);
// => Uint8Array(32) [ 185, 77, 39, 185, 147, ... ]
```

<!-- Bun.sha; -->


Use the built-in `bun:ffi` module to efficiently call native libraries from JavaScript. It works with languages that support the C ABI (Zig, Rust, C/C++, C#, Nim, Kotlin, etc).

## Usage (`bun:ffi`)

To print the version number of `sqlite3`:

```ts
import { dlopen, FFIType, suffix } from "bun:ffi";

// `suffix` is either "dylib", "so", or "dll" depending on the platform
// you don't have to use "suffix", it's just there for convenience
const path = `libsqlite3.${suffix}`;

const {
  symbols: {
    sqlite3_libversion, // the function to call
  },
} = dlopen(
  path, // a library name or file path
  {
    sqlite3_libversion: {
      // no arguments, returns a string
      args: [],
      returns: FFIType.cstring,
    },
  },
);

console.log(`SQLite 3 version: ${sqlite3_libversion()}`);
```

## Performance

According to [our benchmark](https://github.com/oven-sh/bun/tree/main/bench/ffi), `bun:ffi` is roughly 2-6x faster than Node.js FFI via `Node-API`.

{% image src="/images/ffi.png" height="400" /%}

Bun generates & just-in-time compiles C bindings that efficiently convert values between JavaScript types and native types. To compile C, Bun embeds [TinyCC](https://github.com/TinyCC/tinycc), a small and fast C compiler.

## Usage

### Zig

```zig
// add.zig
pub export fn add(a: i32, b: i32) i32 {
  return a + b;
}
```

To compile:

```bash
$ zig build-lib add.zig -dynamic -OReleaseFast
```

Pass a path to the shared library and a map of symbols to import into `dlopen`:

```ts
import { dlopen, FFIType, suffix } from "bun:ffi";

const path = `libadd.${suffix}`;

const lib = dlopen(path, {
  add: {
    args: [FFIType.i32, FFIType.i32],
    returns: FFIType.i32,
  },
});

lib.symbols.add(1, 2);
```

### Rust

```rust
// add.rs
#[no_mangle]
pub extern "C" fn add(a: isize, b: isize) -> isize {
    a + b
}
```

To compile:

```bash
$ rustc --crate-type cdylib add.rs
```

## FFI types

The following `FFIType` values are supported.

| `FFIType` | C Type         | Aliases                     |
| --------- | -------------- | --------------------------- |
| cstring   | `char*`        |                             |
| function  | `(void*)(*)()` | `fn`, `callback`            |
| ptr       | `void*`        | `pointer`, `void*`, `char*` |
| i8        | `int8_t`       | `int8_t`                    |
| i16       | `int16_t`      | `int16_t`                   |
| i32       | `int32_t`      | `int32_t`, `int`            |
| i64       | `int64_t`      | `int64_t`                   |
| i64_fast  | `int64_t`      |                             |
| u8        | `uint8_t`      | `uint8_t`                   |
| u16       | `uint16_t`     | `uint16_t`                  |
| u32       | `uint32_t`     | `uint32_t`                  |
| u64       | `uint64_t`     | `uint64_t`                  |
| u64_fast  | `uint64_t`     |                             |
| f32       | `float`        | `float`                     |
| f64       | `double`       | `double`                    |
| bool      | `bool`         |                             |
| char      | `char`         |                             |

## Strings

JavaScript strings and C-like strings are different, and that complicates using strings with native libraries.

{% details summary="How are JavaScript strings and C strings different?" %}
JavaScript strings:

- UTF16 (2 bytes per letter) or potentially latin1, depending on the JavaScript engine &amp; what characters are used
- `length` stored separately
- Immutable

C strings:

- UTF8 (1 byte per letter), usually
- The length is not stored. Instead, the string is null-terminated which means the length is the index of the first `\0` it finds
- Mutable

{% /details %}

To solve this, `bun:ffi` exports `CString` which extends JavaScript's built-in `String` to support null-terminated strings and add a few extras:

```ts
class CString extends String {
  /**
   * Given a `ptr`, this will automatically search for the closing `\0` character and transcode from UTF-8 to UTF-16 if necessary.
   */
  constructor(ptr: number, byteOffset?: number, byteLength?: number): string;

  /**
   * The ptr to the C string
   *
   * This `CString` instance is a clone of the string, so it
   * is safe to continue using this instance after the `ptr` has been
   * freed.
   */
  ptr: number;
  byteOffset?: number;
  byteLength?: number;
}
```

To convert from a null-terminated string pointer to a JavaScript string:

```ts
const myString = new CString(ptr);
```

To convert from a pointer with a known length to a JavaScript string:

```ts
const myString = new CString(ptr, 0, byteLength);
```

The `new CString()` constructor clones the C string, so it is safe to continue using `myString` after `ptr` has been freed.

```ts
my_library_free(myString.ptr);

// this is safe because myString is a clone
console.log(myString);
```

When used in `returns`, `FFIType.cstring` coerces the pointer to a JavaScript `string`. When used in `args`, `FFIType.cstring` is identical to `ptr`.

## Function pointers

{% callout %}

**Note** — Async functions are not yet supported.

{% /callout %}

To call a function pointer from JavaScript, use `CFunction`. This is useful if using Node-API (napi) with Bun, and you've already loaded some symbols.

```ts
import { CFunction } from "bun:ffi";

let myNativeLibraryGetVersion = /* somehow, you got this pointer */

const getVersion = new CFunction({
  returns: "cstring",
  args: [],
  ptr: myNativeLibraryGetVersion,
});
getVersion();
```

If you have multiple function pointers, you can define them all at once with `linkSymbols`:

```ts
import { linkSymbols } from "bun:ffi";

// getVersionPtrs defined elsewhere
const [majorPtr, minorPtr, patchPtr] = getVersionPtrs();

const lib = linkSymbols({
  // Unlike with dlopen(), the names here can be whatever you want
  getMajor: {
    returns: "cstring",
    args: [],

    // Since this doesn't use dlsym(), you have to provide a valid ptr
    // That ptr could be a number or a bigint
    // An invalid pointer will crash your program.
    ptr: majorPtr,
  },
  getMinor: {
    returns: "cstring",
    args: [],
    ptr: minorPtr,
  },
  getPatch: {
    returns: "cstring",
    args: [],
    ptr: patchPtr,
  },
});

const [major, minor, patch] = [lib.symbols.getMajor(), lib.symbols.getMinor(), lib.symbols.getPatch()];
```

## Callbacks

Use `JSCallback` to create JavaScript callback functions that can be passed to C/FFI functions. The C/FFI function can call into the JavaScript/TypeScript code. This is useful for asynchronous code or whenever you want to call into JavaScript code from C.

```ts
import { dlopen, JSCallback, ptr, CString } from "bun:ffi";

const {
  symbols: { search },
  close,
} = dlopen("libmylib", {
  search: {
    returns: "usize",
    args: ["cstring", "callback"],
  },
});

const searchIterator = new JSCallback((ptr, length) => /hello/.test(new CString(ptr, length)), {
  returns: "bool",
  args: ["ptr", "usize"],
});

const str = Buffer.from("wwutwutwutwutwutwutwutwutwutwutut\0", "utf8");
if (search(ptr(str), searchIterator)) {
  // found a match!
}

// Sometime later:
setTimeout(() => {
  searchIterator.close();
  close();
}, 5000);
```

When you're done with a JSCallback, you should call `close()` to free the memory.

{% callout %}

**⚡️ Performance tip** — For a slight performance boost, directly pass `JSCallback.prototype.ptr` instead of the `JSCallback` object:

```ts
const onResolve = new JSCallback(arg => arg === 42, {
  returns: "bool",
  args: ["i32"],
});
const setOnResolve = new CFunction({
  returns: "bool",
  args: ["function"],
  ptr: myNativeLibrarySetOnResolve,
});

// This code runs slightly faster:
setOnResolve(onResolve.ptr);

// Compared to this:
setOnResolve(onResolve);
```

{% /callout %}

## Pointers

Bun represents [pointers](<https://en.wikipedia.org/wiki/Pointer_(computer_programming)>) as a `number` in JavaScript.

{% details summary="How does a 64 bit pointer fit in a JavaScript number?" %}
64-bit processors support up to [52 bits of addressable space](https://en.wikipedia.org/wiki/64-bit_computing#Limits_of_processors). [JavaScript numbers](https://en.wikipedia.org/wiki/Double-precision_floating-point_format#IEEE_754_double-precision_binary_floating-point_format:_binary64) support 53 bits of usable space, so that leaves us with about 11 bits of extra space.

**Why not `BigInt`?** `BigInt` is slower. JavaScript engines allocate a separate `BigInt` which means they can't fit into a regular JavaScript value. If you pass a `BigInt` to a function, it will be converted to a `number`
{% /details %}

To convert from a `TypedArray` to a pointer:

```ts
import { ptr } from "bun:ffi";
let myTypedArray = new Uint8Array(32);
const myPtr = ptr(myTypedArray);
```

To convert from a pointer to an `ArrayBuffer`:

```ts
import { ptr, toArrayBuffer } from "bun:ffi";
let myTypedArray = new Uint8Array(32);
const myPtr = ptr(myTypedArray);

// toArrayBuffer accepts a `byteOffset` and `byteLength`
// if `byteLength` is not provided, it is assumed to be a null-terminated pointer
myTypedArray = new Uint8Array(toArrayBuffer(myPtr, 0, 32), 0, 32);
```

To read data from a pointer, you have two options. For long-lived pointers, use a `DataView`:

```ts
import { toArrayBuffer } from "bun:ffi";
let myDataView = new DataView(toArrayBuffer(myPtr, 0, 32));

console.log(
  myDataView.getUint8(0, true),
  myDataView.getUint8(1, true),
  myDataView.getUint8(2, true),
  myDataView.getUint8(3, true),
);
```

For short-lived pointers, use `read`:

```ts
import { read } from "bun:ffi";

console.log(
  // ptr, byteOffset
  read.u8(myPtr, 0),
  read.u8(myPtr, 1),
  read.u8(myPtr, 2),
  read.u8(myPtr, 3),
);
```

The `read` function behaves similarly to `DataView`, but it's usually faster because it doesn't need to create a `DataView` or `ArrayBuffer`.

| `FFIType` | `read` function |
| --------- | --------------- |
| ptr       | `read.ptr`      |
| i8        | `read.i8`       |
| i16       | `read.i16`      |
| i32       | `read.i32`      |
| i64       | `read.i64`      |
| u8        | `read.u8`       |
| u16       | `read.u16`      |
| u32       | `read.u32`      |
| u64       | `read.u64`      |
| f32       | `read.f32`      |
| f64       | `read.f64`      |

### Memory management

`bun:ffi` does not manage memory for you. You must free the memory when you're done with it.

#### From JavaScript

If you want to track when a `TypedArray` is no longer in use from JavaScript, you can use a [FinalizationRegistry](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/FinalizationRegistry).

#### From C, Rust, Zig, etc

{% callout %}
**Note** — Available in Bun v0.1.8 and later.
{% /callout %}

If you want to track when a `TypedArray` is no longer in use from C or FFI, you can pass a callback and an optional context pointer to `toArrayBuffer` or `toBuffer`. This function is called at some point later, once the garbage collector frees the underlying `ArrayBuffer` JavaScript object.

The expected signature is the same as in [JavaScriptCore's C API](https://developer.apple.com/documentation/javascriptcore/jstypedarraybytesdeallocator?language=objc):

```c
typedef void (*JSTypedArrayBytesDeallocator)(void *bytes, void *deallocatorContext);
```

```ts
import { toArrayBuffer } from "bun:ffi";

// with a deallocatorContext:
toArrayBuffer(
  bytes,
  byteOffset,

  byteLength,

  // this is an optional pointer to a callback
  deallocatorContext,

  // this is a pointer to a function
  jsTypedArrayBytesDeallocator,
);

// without a deallocatorContext:
toArrayBuffer(
  bytes,
  byteOffset,

  byteLength,

  // this is a pointer to a function
  jsTypedArrayBytesDeallocator,
);
```

### Memory safety

Using raw pointers outside of FFI is extremely not recommended. A future version of Bun may add a CLI flag to disable `bun:ffi`.

### Pointer alignment

If an API expects a pointer sized to something other than `char` or `u8`, make sure the `TypedArray` is also that size. A `u64*` is not exactly the same as `[8]u8*` due to alignment.

### Passing a pointer

Where FFI functions expect a pointer, pass a `TypedArray` of equivalent size:

```ts
import { dlopen, FFIType } from "bun:ffi";

const {
  symbols: { encode_png },
} = dlopen(myLibraryPath, {
  encode_png: {
    // FFIType's can be specified as strings too
    args: ["ptr", "u32", "u32"],
    returns: FFIType.ptr,
  },
});

const pixels = new Uint8ClampedArray(128 * 128 * 4);
pixels.fill(254);
pixels.subarray(0, 32 * 32 * 2).fill(0);

const out = encode_png(
  // pixels will be passed as a pointer
  pixels,

  128,
  128,
);
```

The [auto-generated wrapper](https://github.com/oven-sh/bun/blob/6a65631cbdcae75bfa1e64323a6ad613a922cd1a/src/bun.js/ffi.exports.js#L180-L182) converts the pointer to a `TypedArray`.

{% details summary="Hardmode" %}

If you don't want the automatic conversion or you want a pointer to a specific byte offset within the `TypedArray`, you can also directly get the pointer to the `TypedArray`:

```ts
import { dlopen, FFIType, ptr } from "bun:ffi";

const {
  symbols: { encode_png },
} = dlopen(myLibraryPath, {
  encode_png: {
    // FFIType's can be specified as strings too
    args: ["ptr", "u32", "u32"],
    returns: FFIType.ptr,
  },
});

const pixels = new Uint8ClampedArray(128 * 128 * 4);
pixels.fill(254);

// this returns a number! not a BigInt!
const myPtr = ptr(pixels);

const out = encode_png(
  myPtr,

  // dimensions:
  128,
  128,
);
```

{% /details %}

### Reading pointers

```ts
const out = encode_png(
  // pixels will be passed as a pointer
  pixels,

  // dimensions:
  128,
  128,
);

// assuming it is 0-terminated, it can be read like this:
let png = new Uint8Array(toArrayBuffer(out));

// save it to disk:
await Bun.write("out.png", png);
```


Streams are an important abstraction for working with binary data without loading it all into memory at once. They are commonly used for reading and writing files, sending and receiving network requests, and processing large amounts of data.

Bun implements the Web APIs [`ReadableStream`](https://developer.mozilla.org/en-US/docs/Web/API/ReadableStream) and [`WritableStream`](https://developer.mozilla.org/en-US/docs/Web/API/ReadableStream).

{% callout %}
Bun also implements the `node:stream` module, including [`Readable`](https://nodejs.org/api/stream.html#stream_readable_streams), [`Writable`](https://nodejs.org/api/stream.html#stream_writable_streams), and [`Duplex`](https://nodejs.org/api/stream.html#stream_duplex_and_transform_streams). For complete documentation, refer to the [Node.js docs](https://nodejs.org/api/stream.html).
{% /callout %}

To create a simple `ReadableStream`:

```ts
const stream = new ReadableStream({
  start(controller) {
    controller.enqueue("hello");
    controller.enqueue("world");
    controller.close();
  },
});
```

The contents of a `ReadableStream` can be read chunk-by-chunk with `for await` syntax.

```ts
for await (const chunk of stream) {
  console.log(chunk);
  // => "hello"
  // => "world"
}
```

## Direct `ReadableStream`

Bun implements an optimized version of `ReadableStream` that avoid unnecessary data copying & queue management logic. With a traditional `ReadableStream`, chunks of data are _enqueued_. Each chunk is copied into a queue, where it sits until the stream is ready to send more data.

```ts
const stream = new ReadableStream({
  start(controller) {
    controller.enqueue("hello");
    controller.enqueue("world");
    controller.close();
  },
});
```

With a direct `ReadableStream`, chunks of data are written directly to the stream. No queueing happens, and there's no need to clone the chunk data into memory. The `controller` API is updated to reflect this; instead of `.enqueue()` you call `.write`.

```ts
const stream = new ReadableStream({
  type: "direct",
  pull(controller) {
    controller.write("hello");
    controller.write("world");
  },
});
```

When using a direct `ReadableStream`, all chunk queueing is handled by the destination. The consumer of the stream receives exactly what is passed to `controller.write()`, without any encoding or modification.

## `Bun.ArrayBufferSink`

The `Bun.ArrayBufferSink` class is a fast incremental writer for constructing an `ArrayBuffer` of unknown size.

```ts
const sink = new Bun.ArrayBufferSink();

sink.write("h");
sink.write("e");
sink.write("l");
sink.write("l");
sink.write("o");

sink.end();
// ArrayBuffer(5) [ 104, 101, 108, 108, 111 ]
```

To instead retrieve the data as a `Uint8Array`, pass the `asUint8Array` option to the constructor.

```ts-diff
const sink = new Bun.ArrayBufferSink({
+ asUint8Array: true
});

sink.write("h");
sink.write("e");
sink.write("l");
sink.write("l");
sink.write("o");

sink.end();
// Uint8Array(5) [ 104, 101, 108, 108, 111 ]
```

The `.write()` method supports strings, typed arrays, `ArrayBuffer`, and `SharedArrayBuffer`.

```ts
sink.write("h");
sink.write(new Uint8Array([101, 108]));
sink.write(Buffer.from("lo").buffer);

sink.end();
```

Once `.end()` is called, no more data can be written to the `ArrayBufferSink`. However, in the context of buffering a stream, it's useful to continuously write data and periodically `.flush()` the contents (say, into a `WriteableStream`). To support this, pass `stream: true` to the constructor.

```ts
const sink = new Bun.ArrayBufferSink({
  stream: true,
});

sink.write("h");
sink.write("e");
sink.write("l");
sink.flush();
// ArrayBuffer(5) [ 104, 101, 108 ]

sink.write("l");
sink.write("o");
sink.flush();
// ArrayBuffer(5) [ 108, 111 ]
```

The `.flush()` method returns the buffered data as an `ArrayBuffer` (or `Uint8Array` if `asUint8Array: true`) and clears internal buffer.

To manually set the size of the internal buffer in bytes, pass a value for `highWaterMark`:

```ts
const sink = new Bun.ArrayBufferSink({
  highWaterMark: 1024 * 1024, // 1 MB
});
```

{% details summary="Reference" %}

```ts
/**
 * Fast incremental writer that becomes an `ArrayBuffer` on end().
 */
export class ArrayBufferSink {
  constructor();

  start(options?: {
    asUint8Array?: boolean;
    /**
     * Preallocate an internal buffer of this size
     * This can significantly improve performance when the chunk size is small
     */
    highWaterMark?: number;
    /**
     * On {@link ArrayBufferSink.flush}, return the written data as a `Uint8Array`.
     * Writes will restart from the beginning of the buffer.
     */
    stream?: boolean;
  }): void;

  write(
    chunk: string | ArrayBufferView | ArrayBuffer | SharedArrayBuffer,
  ): number;
  /**
   * Flush the internal buffer
   *
   * If {@link ArrayBufferSink.start} was passed a `stream` option, this will return a `ArrayBuffer`
   * If {@link ArrayBufferSink.start} was passed a `stream` option and `asUint8Array`, this will return a `Uint8Array`
   * Otherwise, this will return the number of bytes written since the last flush
   *
   * This API might change later to separate Uint8ArraySink and ArrayBufferSink
   */
  flush(): number | Uint8Array | ArrayBuffer;
  end(): ArrayBuffer | Uint8Array;
}
```

{% /details %}


Bun implements the `node:dns` module.

```ts
import * as dns from "node:dns";

const addrs = await dns.promises.resolve4("bun.sh", { ttl: true });
console.log(addrs);
// => [{ address: "172.67.161.226", family: 4, ttl: 0 }, ...]
```

<!--
## `Bun.dns` - lookup a domain
`Bun.dns` includes utilities to make DNS requests, similar to `node:dns`. As of Bun v0.5.0, the only implemented function is `dns.lookup`, though more will be implemented soon.
You can lookup the IP addresses of a hostname by using `dns.lookup`.
```ts
import { dns } from "bun";
const [{ address }] = await dns.lookup("example.com");
console.log(address); // "93.184.216.34"
```
If you need to limit IP addresses to either IPv4 or IPv6, you can specify the `family` as an option.
```ts
import { dns } from "bun";
const [{ address }] = await dns.lookup("example.com", { family: 6 });
console.log(address); // "2606:2800:220:1:248:1893:25c8:1946"
```
Bun supports three backends for DNS resolution:
- `c-ares` - This is the default on Linux, and it uses the [c-ares](https://c-ares.org/) library to perform DNS resolution.
- `system` - Uses the system's non-blocking DNS resolver, if available. Otherwise, falls back to `getaddrinfo`. This is the default on macOS, and the same as `getaddrinfo` on Linux.
- `getaddrinfo` - Uses the POSIX standard `getaddrinfo` function, which may cause performance issues under concurrent load.

You can choose a particular backend by specifying `backend` as an option.
```ts
import { dns } from "bun";
const [{ address, ttl }] = await dns.lookup("example.com", {
  backend: "c-ares"
});
console.log(address); // "93.184.216.34"
console.log(ttl); // 21237
```
Note: the `ttl` property is only accurate when the `backend` is c-ares. Otherwise, `ttl` will be `0`.
This was added in Bun v0.5.0. -->


{% callout %}
**Note** — Bun provides a browser- and Node.js-compatible [console](https://developer.mozilla.org/en-US/docs/Web/API/console) global. This page only documents Bun-native APIs.
{% /callout %}

In Bun, the `console` object can be used as an `AsyncIterable` to sequentially read lines from `process.stdin`.

```ts
for await (const line of console) {
  console.log(line);
}
```

This is useful for implementing interactive programs, like the following addition calculator.

```ts#adder.ts
console.log(`Let's add some numbers!`);
console.write(`Count: 0\n> `);

let count = 0;
for await (const line of console) {
  count += Number(line);
  console.write(`Count: ${count}\n> `);
}
```

To run the file:

```bash
$ bun adder.ts
Let's add some numbers!
Count: 0
> 5
Count: 5
> 5
Count: 10
> 5
Count: 15
```


Node-API is an interface for building native add-ons to Node.js. Bun implements 95% of this interface from scratch, so most existing Node-API extensions will work with Bun out of the box. Track the completion status of it in [this issue](https://github.com/oven-sh/bun/issues/158).

As in Node.js, `.node` files (Node-API modules) can be required directly in Bun.

```js
const napi = require("./my-node-module.node");
```

Alternatively, use `process.dlopen`:

```js
let mod = { exports: {} };
process.dlopen(mod, "./my-node-module.node");
```

Bun polyfills the [`detect-libc`](https://npmjs.com/package/detect-libc) package, which is used by many Node-API modules to detect which `.node` binding to `require`.


Bun natively implements a high-performance [SQLite3](https://www.sqlite.org/) driver. To use it import from the built-in `bun:sqlite` module.

```ts
import { Database } from "bun:sqlite";

const db = new Database(":memory:");
const query = db.query("select 'Hello world' as message;");
query.get(); // => { message: "Hello world" }
```

The API is simple, synchronous, and fast. Credit to [better-sqlite3](https://github.com/JoshuaWise/better-sqlite3) and its contributors for inspiring the API of `bun:sqlite`.

Features include:

- Transactions
- Parameters (named & positional)
- Prepared statements
- Datatype conversions (`BLOB` becomes `Uint8Array`)
- The fastest performance of any SQLite driver for JavaScript

The `bun:sqlite` module is roughly 3-6x faster than `better-sqlite3` and 8-9x faster than `deno.land/x/sqlite` for read queries. Each driver was benchmarked against the [Northwind Traders](https://github.com/jpwhite3/northwind-SQLite3/blob/46d5f8a64f396f87cd374d1600dbf521523980e8/Northwind_large.sqlite.zip) dataset. View and run the [benchmark source](https://github.com/oven-sh/bun/tree/main/bench/sqlite).

{% image width="738" alt="SQLite benchmarks for Bun, better-sqlite3, and deno.land/x/sqlite" src="https://user-images.githubusercontent.com/709451/168459263-8cd51ca3-a924-41e9-908d-cf3478a3b7f3.png" caption="Benchmarked on an M1 MacBook Pro (64GB) running macOS 12.3.1" /%}

## Database

To open or create a SQLite3 database:

```ts
import { Database } from "bun:sqlite";

const db = new Database("mydb.sqlite");
```

To open an in-memory database:

```ts
import { Database } from "bun:sqlite";

// all of these do the same thing
const db = new Database(":memory:");
const db = new Database();
const db = new Database("");
```

To open in `readonly` mode:

```ts
import { Database } from "bun:sqlite";
const db = new Database("mydb.sqlite", { readonly: true });
```

To create the database if the file doesn't exist:

```ts
import { Database } from "bun:sqlite";
const db = new Database("mydb.sqlite", { create: true });
```

### `.close()`

To close a database:

```ts
const db = new Database();
db.close();
```

Note: `close()` is called automatically when the database is garbage collected. It is safe to call multiple times but has no effect after the first.

### `.serialize()`

`bun:sqlite` supports SQLite's built-in mechanism for [serializing](https://www.sqlite.org/c3ref/serialize.html) and [deserializing](https://www.sqlite.org/c3ref/deserialize.html) databases to and from memory.

```ts
const olddb = new Database("mydb.sqlite");
const contents = olddb.serialize(); // => Uint8Array
const newdb = new Database(contents);
```

Internally, `.serialize()` calls [`sqlite3_serialize`](https://www.sqlite.org/c3ref/serialize.html).

### `.query()`

Use the `db.query()` method on your `Database` instance to [prepare](https://www.sqlite.org/c3ref/prepare.html) a SQL query. The result is a `Statement` instance that will be cached on the `Database` instance. _The query will not be executed._

```ts
const query = db.query(`select "Hello world" as message`);
```

{% callout %}

**Note** — Use the `.prepare()` method to prepare a query _without_ caching it on the `Database` instance.

```ts
// compile the prepared statement
const query = db.prepare("SELECT * FROM foo WHERE bar = ?");
```

{% /callout %}

## Statements

A `Statement` is a _prepared query_, which means it's been parsed and compiled into an efficient binary form. It can be executed multiple times in a performant way.

Create a statement with the `.query` method on your `Database` instance.

```ts
const query = db.query(`select "Hello world" as message`);
```

Queries can contain parameters. These can be numerical (`?1`) or named (`$param` or `:param` or `@param`).

```ts
const query = db.query(`SELECT ?1, ?2;`);
const query = db.query(`SELECT $param1, $param2;`);
```

Values are bound to these parameters when the query is executed. A `Statement` can be executed with several different methods, each returning the results in a different form.

### `.all()`

Use `.all()` to run a query and get back the results as an array of objects.

```ts
const query = db.query(`select $message;`);
query.all({ $message: "Hello world" });
// => [{ message: "Hello world" }]
```

Internally, this calls [`sqlite3_reset`](https://www.sqlite.org/capi3ref.html#sqlite3_reset) and repeatedly calls [`sqlite3_step`](https://www.sqlite.org/capi3ref.html#sqlite3_step) until it returns `SQLITE_DONE`.

### `.get()`

Use `.get()` to run a query and get back the first result as an object.

```ts
const query = db.query(`select $message;`);
query.get({ $message: "Hello world" });
// => { $message: "Hello world" }
```

Internally, this calls [`sqlite3_reset`](https://www.sqlite.org/capi3ref.html#sqlite3_reset) followed by [`sqlite3_step`](https://www.sqlite.org/capi3ref.html#sqlite3_step) until it no longer returns `SQLITE_ROW`. If the query returns no rows, `undefined` is returned.

### `.run()`

Use `.run()` to run a query and get back `undefined`. This is useful for queries schema-modifying queries (e.g. `CREATE TABLE`) or bulk write operations.

```ts
const query = db.query(`create table foo;`);
query.run();
// => undefined
```

Internally, this calls [`sqlite3_reset`](https://www.sqlite.org/capi3ref.html#sqlite3_reset) and calls [`sqlite3_step`](https://www.sqlite.org/capi3ref.html#sqlite3_step) once. Stepping through all the rows is not necessary when you don't care about the results.

### `.values()`

Use `values()` to run a query and get back all results as an array of arrays.

```ts
const query = db.query(`select $message;`);
query.values({ $message: "Hello world" });

query.values(2);
// [
//   [ "Iron Man", 2008 ],
//   [ "The Avengers", 2012 ],
//   [ "Ant-Man: Quantumania", 2023 ],
// ]
```

Internally, this calls [`sqlite3_reset`](https://www.sqlite.org/capi3ref.html#sqlite3_reset) and repeatedly calls [`sqlite3_step`](https://www.sqlite.org/capi3ref.html#sqlite3_step) until it returns `SQLITE_DONE`.

### `.finalize()`

Use `.finalize()` to destroy a `Statement` and free any resources associated with it. Once finalized, a `Statement` cannot be executed again. Typically, the garbage collector will do this for you, but explicit finalization may be useful in performance-sensitive applications.

```ts
const query = db.query("SELECT title, year FROM movies");
const movies = query.all();
query.finalize();
```

### `.toString()`

Calling `toString()` on a `Statement` instance prints the expanded SQL query. This is useful for debugging.

```ts
import { Database } from "bun:sqlite";

// setup
const query = db.query("SELECT $param;");

console.log(query.toString()); // => "SELECT NULL"

query.run(42);
console.log(query.toString()); // => "SELECT 42"

query.run(365);
console.log(query.toString()); // => "SELECT 365"
```

Internally, this calls [`sqlite3_expanded_sql`](https://www.sqlite.org/capi3ref.html#sqlite3_expanded_sql). The parameters are expanded using the most recently bound values.

## Parameters

Queries can contain parameters. These can be numerical (`?1`) or named (`$param` or `:param` or `@param`). Bind values to these parameters when executing the query:

{% codetabs %}

```ts#Query
const query = db.query("SELECT * FROM foo WHERE bar = $bar");
const results = query.all({
  $bar: "bar",
});
```

```json#Results
[
  { "$bar": "bar" }
]
```

{% /codetabs %}

Numbered (positional) parameters work too:

{% codetabs %}

```ts#Query
const query = db.query("SELECT ?1, ?2");
const results = query.all("hello", "goodbye");
```

```ts#Results
[
  {
    "?1": "hello",
    "?2": "goodbye"
  }
]
```

{% /codetabs %}

## Transactions

Transactions are a mechanism for executing multiple queries in an _atomic_ way; that is, either all of the queries succeed or none of them do. Create a transaction with the `db.transaction()` method:

```ts
const insertCat = db.prepare("INSERT INTO cats (name) VALUES ($name)");
const insertCats = db.transaction((cats) => {
  for (const cat of cats) insertCat.run(cat);
});
```

At this stage, we haven't inserted any cats! The call to `db.transaction()` returns a new function (`insertCats`) that _wraps_ the function that executes the queries.

To execute the transaction, call this function. All arguments will be passed through to the wrapped function; the return value of the wrapped function will be returned by the transaction function. The wrapped function also has access to the `this` context as defined where the transaction is executed.

```ts
const insert = db.prepare("INSERT INTO cats (name) VALUES ($name)");
const insertCats = db.transaction((cats) => {
  for (const cat of cats) insert.run(cat);
  return cats.length;
});

const count = insertCats([
  { $name: "Keanu" },
  { $name: "Salem" },
  { $name: "Crookshanks" },
]);

console.log(`Inserted ${count} cats`);
```

The driver will automatically [`begin`](https://www.sqlite.org/lang_transaction.html) a transaction when `insertCats` is called and `commit` it when the wrapped function returns. If an exception is thrown, the transaction will be rolled back. The exception will propagate as usual; it is not caught.

{% callout %}
**Nested transactions** — Transaction functions can be called from inside other transaction functions. When doing so, the inner transaction becomes a [savepoint](https://www.sqlite.org/lang_savepoint.html).

{% details summary="View nested transaction example" %}

```ts
// setup
import { Database } from "bun:sqlite";
const db = Database.open(":memory:");
db.run(
  "CREATE TABLE expenses (id INTEGER PRIMARY KEY AUTOINCREMENT, note TEXT, dollars INTEGER);",
);
db.run(
  "CREATE TABLE cats (id INTEGER PRIMARY KEY AUTOINCREMENT, name TEXT UNIQUE, age INTEGER)",
);
const insertExpense = db.prepare(
  "INSERT INTO expenses (note, dollars) VALUES (?, ?)",
);
const insert = db.prepare("INSERT INTO cats (name, age) VALUES ($name, $age)");
const insertCats = db.transaction((cats) => {
  for (const cat of cats) insert.run(cat);
});

const adopt = db.transaction((cats) => {
  insertExpense.run("adoption fees", 20);
  insertCats(cats); // nested transaction
});

adopt([
  { $name: "Joey", $age: 2 },
  { $name: "Sally", $age: 4 },
  { $name: "Junior", $age: 1 },
]);
```

{% /details %}
{% /callout %}

Transactions also come with `deferred`, `immediate`, and `exclusive` versions.

```ts
insertCats(cats); // uses "BEGIN"
insertCats.deferred(cats); // uses "BEGIN DEFERRED"
insertCats.immediate(cats); // uses "BEGIN IMMEDIATE"
insertCats.exclusive(cats); // uses "BEGIN EXCLUSIVE"
```

### `.loadExtension()`

To load a [SQLite extension](https://www.sqlite.org/loadext.html), call `.loadExtension(name)` on your `Database` instance

```ts
import { Database } from "bun:sqlite";

const db = new Database();
db.loadExtension("myext");
```

{% details summary="For macOS users" %}
**MacOS users** By default, macOS ships with Apple's proprietary build of SQLite, which doesn't support extensions. To use extensions, you'll need to install a vanilla build of SQLite.

```bash
$ brew install sqlite
$ which sqlite # get path to binary
```

To point `bun:sqlite` to the new build, call `Database.setCustomSQLite(path)` before creating any `Database` instances. (On other operating systems, this is a no-op.) Pass a path to the SQLite `.dylib` file, _not_ the executable. With recent versions of Homebrew this is something like `/opt/homebrew/Cellar/sqlite/<version>/libsqlite3.dylib`.

```ts
import { Database } from "bun:sqlite";

Database.setCustomSQLite("/path/to/libsqlite.dylib");

const db = new Database();
db.loadExtension("myext");
```

{% /details %}

## Reference

```ts
class Database {
  constructor(
    filename: string,
    options?:
      | number
      | {
          readonly?: boolean;
          create?: boolean;
          readwrite?: boolean;
        },
  );

  query<Params, ReturnType>(sql: string): Statement<Params, ReturnType>;
}

class Statement<Params, ReturnType> {
  all(params: Params): ReturnType[];
  get(params: Params): ReturnType | undefined;
  run(params: Params): void;
  values(params: Params): unknown[][];

  finalize(): void; // destroy statement and clean up resources
  toString(): string; // serialize to SQL

  columnNames: string[]; // the column names of the result set
  paramsCount: number; // the number of parameters expected by the statement
  native: any; // the native object representing the statement
}

type SQLQueryBindings =
  | string
  | bigint
  | TypedArray
  | number
  | boolean
  | null
  | Record<string, string | bigint | TypedArray | number | boolean | null>;
```

### Datatypes

| JavaScript type | SQLite type            |
| --------------- | ---------------------- |
| `string`        | `TEXT`                 |
| `number`        | `INTEGER` or `DECIMAL` |
| `boolean`       | `INTEGER` (1 or 0)     |
| `Uint8Array`    | `BLOB`                 |
| `Buffer`        | `BLOB`                 |
| `bigint`        | `INTEGER`              |
| `null`          | `NULL`                 |


See the [`bun test`](/docs/cli/test) documentation.


The `import.meta` object is a way for a module to access information about itself. It's part of the JavaScript language, but its contents are not standardized. Each "host" (browser, runtime, etc) is free to implement any properties it wishes on the `import.meta` object.

Bun implements the following properties.

```ts#/path/to/project/file.ts
import.meta.dir;   // => "/path/to/project"
import.meta.file;  // => "file.ts"
import.meta.path;  // => "/path/to/project/file.ts"

import.meta.main;  // `true` if this file is directly executed by `bun run`
                   // `false` otherwise

import.meta.resolveSync("zod")
// resolve an import specifier relative to the directory
```

{% table %}

---

- `import.meta.dir`
- Absolute path to the directory containing the current fil, e.g. `/path/to/project`. Equivalent to `__dirname` in Node.js.

---

- `import.meta.file`
- The name of the current file, e.g. `index.tsx`. Equivalent to `__filename` in Node.js.

---

- `import.meta.path`
- Absolute path to the current file, e.g. `/path/to/project/index.tx`.

---

- `import.meta.main`
- `boolean` Indicates whether the current file is the entrypoint to the current `bun` process. Is the file being directly executed by `bun run` or is it being imported?

---

- `import.meta.resolve{Sync}`
- Resolve a module specifier (e.g. `"zod"` or `"./file.tsx`) to an absolute path. While file would be imported if the specifier were imported from this file?

  ```ts
  import.meta.resolveSync("zod");
  // => "/path/to/project/node_modules/zod/index.ts"

  import.meta.resolveSync("./file.tsx");
  // => "/path/to/project/file.tsx"
  ```

{% /table %}


Bun implements the following globals.

{% table %}

- Global
- Source
- Notes

---

- [`AbortController`](https://developer.mozilla.org/en-US/docs/Web/API/AbortController)
- Web
- &nbsp;

---

- [`AbortSignal`](https://developer.mozilla.org/en-US/docs/Web/API/AbortSignal)
- Web
- &nbsp;

---

- [`alert`](https://developer.mozilla.org/en-US/docs/Web/API/Window/alert)
- Web
- Intended for command-line tools

---

- [`Blob`](https://developer.mozilla.org/en-US/docs/Web/API/Blob)
- Web
- &nbsp;

---

- [`Buffer`](https://nodejs.org/api/buffer.html#class-buffer)
- Node.js
- See [Node.js > `Buffer`](/docs/runtime/nodejs-apis#node_buffer)

---

- `Bun`
- Bun
- Subject to change as additional APIs are added

---

- [`ByteLengthQueuingStrategy`](https://developer.mozilla.org/en-US/docs/Web/API/ByteLengthQueuingStrategy)
- Web
- &nbsp;

---

- [`confirm`](https://developer.mozilla.org/en-US/docs/Web/API/Window/confirm)
- Web
- Intended for command-line tools

---

- [`__dirname`](https://nodejs.org/api/globals.html#__dirname)
- Node.js
- &nbsp;

---

- [`__filename`](https://nodejs.org/api/globals.html#__filename)
- Node.js
- &nbsp;

---

- [`atob()`](https://developer.mozilla.org/en-US/docs/Web/API/atob)
- Web
- &nbsp;

---

- [`btoa()`](https://developer.mozilla.org/en-US/docs/Web/API/btoa)
- Web
- &nbsp;

---

- `BuildMessage`
- Bun
- &nbsp;

---

- [`clearImmediate()`](https://developer.mozilla.org/en-US/docs/Web/API/Window/clearImmediate)
- Web
- &nbsp;

---

- [`clearInterval()`](https://developer.mozilla.org/en-US/docs/Web/API/Window/clearInterval)
- Web
- &nbsp;

---

- [`clearTimeout()`](https://developer.mozilla.org/en-US/docs/Web/API/Window/clearTimeout)
- Web
- &nbsp;

---

- [`console`](https://developer.mozilla.org/en-US/docs/Web/API/console)
- Web
- &nbsp;

---

- [`CountQueuingStrategy`](https://developer.mozilla.org/en-US/docs/Web/API/CountQueuingStrategy)
- Web
- &nbsp;

---

- [`Crypto`](https://developer.mozilla.org/en-US/docs/Web/API/Crypto)
- Web
- &nbsp;

---

- [`crypto`](https://developer.mozilla.org/en-US/docs/Web/API/crypto)
- Web
- &nbsp;

---

- [`CryptoKey`](https://developer.mozilla.org/en-US/docs/Web/API/CryptoKey)
- Web
- &nbsp;

---

- [`CustomEvent`](https://developer.mozilla.org/en-US/docs/Web/API/CustomEvent)
- Web
- &nbsp;

---

- [`Event`](https://developer.mozilla.org/en-US/docs/Web/API/Event)
- Web
- Also [`ErrorEvent`](https://developer.mozilla.org/en-US/docs/Web/API/ErrorEvent) [`CloseEvent`](https://developer.mozilla.org/en-US/docs/Web/API/CloseEvent) [`MessageEvent`](https://developer.mozilla.org/en-US/docs/Web/API/MessageEvent).

---

- [`EventTarget`](https://developer.mozilla.org/en-US/docs/Web/API/EventTarget)
- Web
- &nbsp;

---

- [`exports`](https://nodejs.org/api/globals.html#exports)
- Node.js
- &nbsp;

---

- [`fetch`](https://developer.mozilla.org/en-US/docs/Web/API/fetch)
- Web
- &nbsp;

---

- [`FormData`](https://developer.mozilla.org/en-US/docs/Web/API/FormData)
- Web
- &nbsp;

---

- [`global`](https://nodejs.org/api/globals.html#global)
- Node.js
- See [Node.js > `global`](/docs/runtime/nodejs-apis#node_global).

---

- [`globalThis`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/globalThis)
- Cross-platform
- Aliases to `global`

---

- [`Headers`](https://developer.mozilla.org/en-US/docs/Web/API/Headers)
- Web
- &nbsp;

---

- [`HTMLRewriter`](/docs/api/html-rewriter)
- Cloudflare
- &nbsp;

---

- [`JSON`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/JSON)
- Web
- &nbsp;

---

- [`MessageEvent`](https://developer.mozilla.org/en-US/docs/Web/API/MessageEvent)
- Web
- &nbsp;

---

- [`module`](https://nodejs.org/api/globals.html#module)
- Node.js
- &nbsp;

---

- [`performance`](https://developer.mozilla.org/en-US/docs/Web/API/performance)
- Web
- &nbsp;

---

- [`process`](https://nodejs.org/api/process.html)
- Node.js
- See [Node.js > `process`](/docs/runtime/nodejs-apis#node_process)

---

- [`prompt`](https://developer.mozilla.org/en-US/docs/Web/API/Window/prompt)
- Web
- Intended for command-line tools

---

- [`queueMicrotask()`](https://developer.mozilla.org/en-US/docs/Web/API/queueMicrotask)
- Web
- &nbsp;

---

- [`ReadableByteStreamController`](https://developer.mozilla.org/en-US/docs/Web/API/ReadableByteStreamController)
- Web
- &nbsp;

---

- [`ReadableStream`](https://developer.mozilla.org/en-US/docs/Web/API/ReadableStream)
- Web
- &nbsp;

---

- [`ReadableStreamDefaultController`](https://developer.mozilla.org/en-US/docs/Web/API/ReadableStreamDefaultController)
- Web
- &nbsp;

---

- [`ReadableStreamDefaultReader`](https://developer.mozilla.org/en-US/docs/Web/API/ReadableStreamDefaultReader)
- Web
- &nbsp;

---

- [`reportError`](https://developer.mozilla.org/en-US/docs/Web/API/reportError)
- Web
- &nbsp;

---

- [`require()`](https://nodejs.org/api/globals.html#require)
- Node.js
- &nbsp;

---

- `ResolveMessage`
- Bun
- &nbsp;

---

- [`Response`](https://developer.mozilla.org/en-US/docs/Web/API/Response)
- Web
- &nbsp;

---

- [`Request`](https://developer.mozilla.org/en-US/docs/Web/API/Request)
- Web
- &nbsp;

---

- [`setImmediate()`](https://developer.mozilla.org/en-US/docs/Web/API/Window/setImmediate)
- Web
- &nbsp;

---

- [`setInterval()`](https://developer.mozilla.org/en-US/docs/Web/API/Window/setInterval)
- Web
- &nbsp;

---

- [`setTimeout()`](https://developer.mozilla.org/en-US/docs/Web/API/Window/setTimeout)
- Web
- &nbsp;

---

- [`ShadowRealm`](https://github.com/tc39/proposal-shadowrealm)
- Web
- Stage 3 proposal

---

- [`SubtleCrypto`](https://developer.mozilla.org/en-US/docs/Web/API/SubtleCrypto)
- Web
- &nbsp;

---

- [`DOMException`](https://developer.mozilla.org/en-US/docs/Web/API/DOMException)
- Web
- &nbsp;

---

- [`TextDecoder`](https://developer.mozilla.org/en-US/docs/Web/API/TextDecoder)
- Web
- &nbsp;

---

- [`TextEncoder`](https://developer.mozilla.org/en-US/docs/Web/API/TextEncoder)
- Web
- &nbsp;

---

- [`TransformStream`](https://developer.mozilla.org/en-US/docs/Web/API/TransformStream)
- Web
- &nbsp;

---

- [`TransformStreamDefaultController`](https://developer.mozilla.org/en-US/docs/Web/API/TransformStreamDefaultController)
- Web
- &nbsp;

---

- [`URL`](https://developer.mozilla.org/en-US/docs/Web/API/URL)
- Web
- &nbsp;

---

- [`URLSearchParams`](https://developer.mozilla.org/en-US/docs/Web/API/URLSearchParams)
- Web
- &nbsp;

---

- [`WebAssembly`](https://nodejs.org/api/globals.html#webassembly)
- Web
- &nbsp;

---

- [`WritableStream`](https://developer.mozilla.org/en-US/docs/Web/API/WritableStream)
- Web
- &nbsp;

---

- [`WritableStreamDefaultController`](https://developer.mozilla.org/en-US/docs/Web/API/WritableStreamDefaultController)
- Web
- &nbsp;

---

- [`WritableStreamDefaultWriter`](https://developer.mozilla.org/en-US/docs/Web/API/WritableStreamDefaultWriter)
- Web
- &nbsp;

{% /table %}


`Bun.serve()` supports server-side WebSockets, with on-the-fly compression, TLS support, and a Bun-native publish-subscribe API.

{% callout %}

**⚡️ 7x more throughput** — Bun's WebSockets are fast. For a [simple chatroom](https://github.com/oven-sh/bun/tree/main/bench/websocket-server/README.md) on Linux x64, Bun can handle 7x more requests per second than Node.js + [`"ws"`](https://github.com/websockets/ws).

| Messages sent per second | Runtime                        | Clients |
| ------------------------ | ------------------------------ | ------- |
| ~700,000                 | (`Bun.serve`) Bun v0.2.1 (x64) | 16      |
| ~100,000                 | (`ws`) Node v18.10.0 (x64)     | 16      |

Internally Bun's WebSocket implementation is built on [uWebSockets](https://github.com/uNetworking/uWebSockets).
{% /callout %}

## Start a WebSocket server

Below is a simple WebSocket server built with `Bun.serve`, in which all incoming requests are [upgraded](https://developer.mozilla.org/en-US/docs/Web/HTTP/Protocol_upgrade_mechanism) to WebSocket connections in the `fetch` handler. The socket handlers are declared in the `websocket` parameter.

```ts
Bun.serve({
  fetch(req, server) {
    // upgrade the request to a WebSocket
    if (server.upgrade(req)) {
      return; // do not return a Response
    }
    return new Response("Upgrade failed :(", { status: 500 });
  },
  websocket: {}, // handlers
});
```

The following WebSocket event handlers are supported:

```ts
Bun.serve({
  fetch(req, server) {}, // upgrade logic
  websocket: {
    message(ws, message) {}, // a message is received
    open(ws) {}, // a socket is opened
    close(ws, code, message) {}, // a socket is closed
    drain(ws) {}, // the socket is ready to receive more data
  },
});
```

{% details summary="An API designed for speed" %}

In Bun, handlers are declared once per server, instead of per socket.

`ServerWebSocket` expects you to pass a `WebSocketHandler` object to the `Bun.serve()` method which has methods for `open`, `message`, `close`, `drain`, and `error`. This is different than the client-side `WebSocket` class which extends `EventTarget` (onmessage, onopen, onclose),

Clients tend to not have many socket connections open so an event-based API makes sense.

But servers tend to have **many** socket connections open, which means:

- Time spent adding/removing event listeners for each connection adds up
- Extra memory spent on storing references to callbacks function for each connection
- Usually, people create new functions for each connection, which also means more memory

So, instead of using an event-based API, `ServerWebSocket` expects you to pass a single object with methods for each event in `Bun.serve()` and it is reused for each connection.

This leads to less memory usage and less time spent adding/removing event listeners.
{% /details %}

The first argument to each handler is the instance of `ServerWebSocket` handling the event. The `ServerWebSocket` class is a fast, Bun-native implementation of [`WebSocket`](https://developer.mozilla.org/en-US/docs/Web/API/WebSocket) with some additional features.

```ts
Bun.serve({
  fetch(req, server) {}, // upgrade logic
  websocket: {
    message(ws, message) {
      ws.send(message); // echo back the message
    },
  },
});
```

### Sending messages

Each `ServerWebSocket` instance has a `.send()` method for sending messages to the client. It supports a range of input types.

```ts
ws.send("Hello world"); // string
ws.send(response.arrayBuffer()); // ArrayBuffer
ws.send(new Uint8Array([1, 2, 3])); // TypedArray | DataView
```

### Headers

Once the upgrade succeeds, Bun will send a `101 Switching Protocols` response per the [spec](https://developer.mozilla.org/en-US/docs/Web/HTTP/Protocol_upgrade_mechanism). Additional `headers` can be attched to this `Response` in the call to `server.upgrade()`.

```ts
Bun.serve({
  fetch(req, server) {
    const sessionId = await generateSessionId();
    server.upgrade(req, {
      headers: {
        "Set-Cookie": `SessionId=${sessionId}`,
      },
    });
  },
  websocket: {}, // handlers
});
```

### Contextual data

Contextual `data` can be attached to a new WebSocket in the `.upgrade()` call. This data is made available on the `ws.data` property inside the WebSocket handlers.

```ts
type WebSocketData = {
  createdAt: number;
  channelId: string;
  authToken: string;
};

// TypeScript: specify the type of `data`
Bun.serve<WebSocketData>({
  fetch(req, server) {
    // use a library to parse cookies
    const cookies = parseCookies(req.headers.get("Cookie"));
    server.upgrade(req, {
      // this object must conform to WebSocketData
      data: {
        createdAt: Date.now(),
        channelId: new URL(req.url).searchParams.get("channelId"),
        authToken: cookies["X-Token"],
      },
    });

    return undefined;
  },
  websocket: {
    // handler called when a message is received
    async message(ws, message) {
      const user = getUserFromToken(ws.data.authToken);

      await saveMessageToDatabase({
        channel: ws.data.channelId,
        message: String(message),
        userId: user.id,
      });
    },
  },
});
```

To connect to this server from the browser, create a new `WebSocket`.

```ts#browser.js
const socket = new WebSocket("ws://localhost:3000/chat");

socket.addEventListener("message", event => {
  console.log(event.data);
})
```

{% callout %}
**Identifying users** — The cookies that are currently set on the page will be sent with the WebSocket upgrade request and available on `req.headers` in the `fetch` handler. Parse these cookies to determine the identity of the connecting user and set the value of `data` accordingly.
{% /callout %}

### Pub/Sub

Bun's `ServerWebSocket` implementation implements a native publish-subscribe API for topic-based broadcasting. Individual sockets can `.subscribe()` to a topic (specified with a string identifier) and `.publish()` messages to all other subscribers to that topic. This topic-based broadcast API is similar to [MQTT](https://en.wikipedia.org/wiki/MQTT) and [Redis Pub/Sub](https://redis.io/topics/pubsub).

```ts
const server = Bun.serve<{ username: string }>({
  fetch(req, server) {
    const url = new URL(req.url);
    if (url.pathname === "/chat") {
      console.log(`upgrade!`);
      const username = getUsernameFromReq(req);
      const success = server.upgrade(req, { data: { username } });
      return success
        ? undefined
        : new Response("WebSocket upgrade error", { status: 400 });
    }

    return new Response("Hello world");
  },
  websocket: {
    open(ws) {
      const msg = `${ws.data.username} has entered the chat`;
      ws.subscribe("the-group-chat");
      ws.publish("the-group-chat", msg);
    },
    message(ws, message) {
      // this is a group chat
      // so the server re-broadcasts incoming message to everyone
      ws.publish("the-group-chat", `${ws.data.username}: ${message}`);
    },
    close(ws) {
      const msg = `${ws.data.username} has left the chat`;
      ws.unsubscribe("the-group-chat");
      ws.publish("the-group-chat", msg);
    },
  },
});

console.log(`Listening on ${server.hostname}:${server.port}`);
```

Calling `.publish(data)` will send the message to all subscribers of a topic _except_ the socket that called `.publish()`.

### Compression

Per-message [compression](https://websockets.readthedocs.io/en/stable/topics/compression.html) can be enabled with the `perMessageDeflate` parameter.

```ts
Bun.serve({
  fetch(req, server) {}, // upgrade logic
  websocket: {
    // enable compression and decompression
    perMessageDeflate: true,
  },
});
```

Compression can be enabled for individual messages by passing a `boolean` as the second argument to `.send()`.

```ts
ws.send("Hello world", true);
```

For fine-grained control over compression characteristics, refer to the [Reference](#reference).

### Backpressure

The `.send(message)` method of `ServerWebSocket` returns a `number` indicating the result of the operation.

- `-1` — The message was enqueued but there is backpressure
- `0` — The message was dropped due to a connection issue
- `1+` — The number of bytes sent

This gives you better control over backpressure in your server.

## Connect to a `Websocket` server

To connect to an external socket server, either from a browser or from Bun, create an instance of `WebSocket` with the constructor.

```ts
const socket = new WebSocket("ws://localhost:3000");
```

In browsers, the cookies that are currently set on the page will be sent with the WebSocket upgrade request. This is a standard feature of the `WebSocket` API.

For convenience, Bun lets you setting custom headers directly in the constructor. This is a Bun-specific extension of the `WebSocket` standard. _This will not work in browsers._

```ts
const socket = new WebSocket("ws://localhost:3000", {
  headers: {
    // custom headers
  },
});
```

To add event listeners to the socket:

```ts
// message is received
socket.addEventListener("message", event => {});

// socket opened
socket.addEventListener("open", event => {});

// socket closed
socket.addEventListener("close", event => {});

// error handler
socket.addEventListener("error", event => {});
```

## Reference

```ts
namespace Bun {
  export function serve(params: {
    fetch: (req: Request, server: Server) => Response | Promise<Response>;
    websocket?: {
      message: (
        ws: ServerWebSocket,
        message: string | ArrayBuffer | Uint8Array,
      ) => void;
      open?: (ws: ServerWebSocket) => void;
      close?: (ws: ServerWebSocket) => void;
      error?: (ws: ServerWebSocket, error: Error) => void;
      drain?: (ws: ServerWebSocket) => void;
      perMessageDeflate?:
        | boolean
        | {
            compress?: boolean | Compressor;
            decompress?: boolean | Compressor;
          };
    };
  }): Server;
}

type Compressor =
  | `"disable"`
  | `"shared"`
  | `"dedicated"`
  | `"3KB"`
  | `"4KB"`
  | `"8KB"`
  | `"16KB"`
  | `"32KB"`
  | `"64KB"`
  | `"128KB"`
  | `"256KB"`;

interface Server {
  pendingWebsockets: number;
  publish(
    topic: string,
    data: string | ArrayBufferView | ArrayBuffer,
    compress?: boolean,
  ): number;
  upgrade(
    req: Request,
    options?: {
      headers?: HeadersInit;
      data?: any;
    },
  ): boolean;
}

interface ServerWebSocket {
  readonly data: any;
  readonly readyState: number;
  readonly remoteAddress: string;
  send(message: string | ArrayBuffer | Uint8Array, compress?: boolean): number;
  close(code?: number, reason?: string): void;
  subscribe(topic: string): void;
  unsubscribe(topic: string): void;
  publish(topic: string, message: string | ArrayBuffer | Uint8Array): void;
  isSubscribed(topic: string): boolean;
  cork(cb: (ws: ServerWebSocket) => void): void;
}
```

<!--
### `Bun.serve(params)`

{% param name="params" %}
Configuration object for WebSocket server
{% /param %}

{% param name=" fetch" %}
`(req: Request, server: Server) => Response | Promise<Response>`

Call `server.upgrade(req)` to upgrade the request to a WebSocket connection. This method returns `true` if the upgrade succeeds, or `false` if the upgrade fails.
{% /param %}

{% param name=" websocket" %}
Configuration object for WebSocket server
{% /param %}

{% param name="  message" %}
`(ws: ServerWebSocket, message: string | ArrayBuffer | Uint8Array) => void`

This handler is called when a `WebSocket` receives a message.
{% /param %}

{% param name="  open" %}
`(ws: ServerWebSocket) => void`

This handler is called when a `WebSocket` is opened.
{% /param %}

{% param name="  close" %}
`(ws: ServerWebSocket, code: number, message: string) => void`

This handler is called when a `WebSocket` is closed.
{% /param %}

{% param name="  drain" %}
`(ws: ServerWebSocket) => void`

This handler is called when a `WebSocket` is ready to receive more data.
{% /param %}

{% param name="  perMessageDeflate" %}
`boolean | {\n  compress?: boolean | Compressor;\n  decompress?: boolean | Compressor \n}`

Enable per-message compression and decompression. This is a boolean value or an object with `compress` and `decompress` properties. Each property can be a boolean value or one of the following `Compressor` types:

- `"disable"`
- `"shared"`
- `"dedicated"`
- `"3KB"`
- `"4KB"`
- `"8KB"`
- `"16KB"`
- `"32KB"`
- `"64KB"`
- `"128KB"`
- `"256KB"`

{% /param %}

### `ServerWebSocket`

{% param name="readyState" %}
`number`

The current state of the `WebSocket` connection. This is one of the following values:

- `0` `CONNECTING`
- `1` `OPEN`
- `2` `CLOSING`
- `3` `CLOSED`

{% /param %}

{% param name="remoteAddress" %}

`string`

The remote address of the `WebSocket` connection
{% /param %}

{% param name="data" %}
The data associated with the `WebSocket` connection. This is set in the `server.upgrade()` call.
{% /param %}

{% param name=".send()" %}
`send(message: string | ArrayBuffer | Uint8Array, compress?: boolean): number`

Send a message to the client. Returns a `number` indicating the result of the operation.

- `-1`: the message was enqueued but there is backpressure
- `0`: the message was dropped due to a connection issue
- `1+`: the number of bytes sent

The `compress` argument will enable compression for this message, even if the `perMessageDeflate` option is disabled.
{% /param %}

{% param name=".subscribe()" %}
`subscribe(topic: string): void`

Subscribe to a topic
{% /param %}

{% param name=".unsubscribe()" %}
`unsubscribe(topic: string): void`

Unsubscribe from a topic
{% /param %}

{% param name=".publish()" %}
`publish(topic: string, data: string | ArrayBufferView | ArrayBuffer, compress?: boolean): number;`

Send a message to all subscribers of a topic
{% /param %}

{% param name=".isSubscribed()" %}
`isSubscribed(topic: string): boolean`

Check if the `WebSocket` is subscribed to a topic
{% /param %}
{% param name=".cork()" %}
`cork(cb: (ws: ServerWebSocket) => void): void;`

Batch a set of operations on a `WebSocket` connection. The `message`, `open`, and `drain` callbacks are automatically corked, so
you only need to call this if you are sending messages outside of those
callbacks or in async functions.

```ts
ws.cork((ws) => {
  ws.send("first");
  ws.send("second");
  ws.send("third");
});
```

{% /param %}

{% param name=".close()" %}
`close(code?: number, message?: string): void`

Close the `WebSocket` connection
{% /param %}

### `Server`

{% param name="pendingWebsockets" %}
Number of in-flight `WebSocket` messages
{% /param %}

{% param name=".publish()" %}
`publish(topic: string, data: string | ArrayBufferView | ArrayBuffer, compress?: boolean): number;`

Send a message to all subscribers of a topic
{% /param %}

{% param name=".upgrade()" %}
`upgrade(req: Request): boolean`

Upgrade a request to a `WebSocket` connection. Returns `true` if the upgrade succeeds, or `false` if the upgrade fails.
{% /param %} -->


{% callout %}
**Warning** — This will soon have breaking changes. It was designed when Bun was mostly a dev server and not a JavaScript runtime.
{% /callout %}

Frameworks preconfigure Bun to enable developers to use Bun with their existing tooling.

Frameworks are configured via the `framework` object in the `package.json` of the framework (not in the application’s `package.json`):

Here is an example:

```json
{
  "name": "bun-framework-next",
  "version": "0.0.0-18",
  "description": "",
  "framework": {
    "displayName": "Next.js",
    "static": "public",
    "assetPrefix": "_next/",
    "router": {
      "dir": ["pages", "src/pages"],
      "extensions": [".js", ".ts", ".tsx", ".jsx"]
    },
    "css": "onimportcss",
    "development": {
      "client": "client.development.tsx",
      "fallback": "fallback.development.tsx",
      "server": "server.development.tsx",
      "css": "onimportcss",
      "define": {
        "client": {
          ".env": "NEXT_PUBLIC_",
          "defaults": {
            "process.env.__NEXT_TRAILING_SLASH": "false",
            "process.env.NODE_ENV": "\"development\"",
            "process.env.__NEXT_ROUTER_BASEPATH": "''",
            "process.env.__NEXT_SCROLL_RESTORATION": "false",
            "process.env.__NEXT_I18N_SUPPORT": "false",
            "process.env.__NEXT_HAS_REWRITES": "false",
            "process.env.__NEXT_ANALYTICS_ID": "null",
            "process.env.__NEXT_OPTIMIZE_CSS": "false",
            "process.env.__NEXT_CROSS_ORIGIN": "''",
            "process.env.__NEXT_STRICT_MODE": "false",
            "process.env.__NEXT_IMAGE_OPTS": "null"
          }
        },
        "server": {
          ".env": "NEXT_",
          "defaults": {
            "process.env.__NEXT_TRAILING_SLASH": "false",
            "process.env.__NEXT_OPTIMIZE_FONTS": "false",
            "process.env.NODE_ENV": "\"development\"",
            "process.env.__NEXT_OPTIMIZE_IMAGES": "false",
            "process.env.__NEXT_OPTIMIZE_CSS": "false",
            "process.env.__NEXT_ROUTER_BASEPATH": "''",
            "process.env.__NEXT_SCROLL_RESTORATION": "false",
            "process.env.__NEXT_I18N_SUPPORT": "false",
            "process.env.__NEXT_HAS_REWRITES": "false",
            "process.env.__NEXT_ANALYTICS_ID": "null",
            "process.env.__NEXT_CROSS_ORIGIN": "''",
            "process.env.__NEXT_STRICT_MODE": "false",
            "process.env.__NEXT_IMAGE_OPTS": "null",
            "global": "globalThis",
            "window": "undefined"
          }
        }
      }
    }
  }
}
```

Here are type definitions:

```ts
type Framework = Environment & {
  // This changes what’s printed in the console on load
  displayName?: string;

  // This allows a prefix to be added (and ignored) to requests.
  // Useful for integrating an existing framework that expects internal routes to have a prefix
  // e.g. "_next"
  assetPrefix?: string;

  development?: Environment;
  production?: Environment;

  // The directory used for serving unmodified assets like fonts and images
  // Defaults to "public" if exists, else "static", else disabled.
  static?: string;

  // "onimportcss" disables the automatic "onimportcss" feature
  // If the framework does routing, you may want to handle CSS manually
  // "facade" removes CSS imports from JavaScript files,
  //    and replaces an imported object with a proxy that mimics CSS module support without doing any class renaming.
  css?: "onimportcss" | "facade";

  // Bun's filesystem router
  router?: Router;
};

type Define = {
  // By passing ".env", Bun will automatically load .env.local, .env.development, and .env if exists in the project root
  //    (in addition to the processes’ environment variables)
  // When "*", all environment variables will be automatically injected into the JavaScript loader
  // When a string like "NEXT_PUBLIC_", only environment variables starting with that prefix will be injected

  ".env": string | "*";

  // These environment variables will be injected into the JavaScript loader
  // These are the equivalent of Webpack’s resolve.alias and esbuild’s --define.
  // Values are parsed as JSON, so they must be valid JSON. The only exception is '' is a valid string, to simplify writing stringified JSON in JSON.
  // If not set, `process.env.NODE_ENV` will be transformed into "development".
  "defaults": Record<string, string>;
};

type Environment = {
  // This is a wrapper for the client-side entry point for a route.
  // This allows frameworks to run initialization code on pages.
  client: string;
  // This is a wrapper for the server-side entry point for a route.
  // This allows frameworks to run initialization code on pages.
  server: string;
  // This runs when "server" code fails to load due to an exception.
  fallback: string;

  // This is how environment variables and .env is configured.
  define?: Define;
};

// Bun's filesystem router
// Currently, Bun supports pages by either an absolute match or a parameter match.
// pages/index.tsx will be executed on navigation to "/" and "/index"
// pages/posts/[id].tsx will be executed on navigation to "/posts/123"
// Routes & parameters are automatically passed to `fallback` and `server`.
type Router = {
  // This determines the folder to look for pages
  dir: string[];

  // These are the allowed file extensions for pages.
  extensions?: string[];
};
```

To use a framework, you pass `bun bun --use package-name`.

Your framework’s `package.json` `name` should start with `bun-framework-`. This is so that people can type something like `bun bun --use next` and it will check `bun-framework-next` first. This is similar to how Babel plugins tend to start with `babel-plugin-`.

For developing frameworks, you can also do `bun bun --use ./relative-path-to-framework`.

If you’re interested in adding a framework integration, please reach out. There’s a lot here, and it’s not entirely documented yet.


- pages
- auto-bundle dependencies
- pages is function that returns a list of pages?
- plugins for svelte and vue
- custom loaders
- HMR
- server endpoints

```ts
Bun.serve({});
```


## With `bun dev`

When importing CSS in JavaScript-like loaders, CSS is treated special.

By default, Bun will transform a statement like this:

```js
import "../styles/global.css";
```

### When `platform` is `browser`

```js
globalThis.document?.dispatchEvent(
  new CustomEvent("onimportcss", {
    detail: "http://localhost:3000/styles/globals.css",
  }),
);
```

An event handler for turning that into a `<link>` is automatically registered when HMR is enabled. That event handler can be turned off either in a framework’s `package.json` or by setting `globalThis["Bun_disableCSSImports"] = true;` in client-side code. Additionally, you can get a list of every .css file imported this way via `globalThis["__BUN"].allImportedStyles`.

### When `platform` is `bun`

```js
//@import url("http://localhost:3000/styles/globals.css");
```

Additionally, Bun exposes an API for SSR/SSG that returns a flat list of URLs to css files imported. That function is `Bun.getImportedStyles()`.

```ts
// This specifically is for "framework" in package.json when loaded via `bun dev`
// This API needs to be changed somewhat to work more generally with Bun.js
// Initially, you could only use Bun.js through `bun dev`
// and this API was created at that time
addEventListener("fetch", async (event: FetchEvent) => {
  let route = Bun.match(event);
  const App = await import("pages/_app");

  // This returns all .css files that were imported in the line above.
  // It’s recursive, so any file that imports a CSS file will be included.
  const appStylesheets = bun.getImportedStyles();

  // ...rest of code
});
```

This is useful for preventing flash of unstyled content.

## With `bun bun`

Bun bundles `.css` files imported via `@import` into a single file. It doesn’t autoprefix or minify CSS today. Multiple `.css` files imported in one JavaScript file will _not_ be bundled into one file. You’ll have to import those from a `.css` file.

This input:

```css
@import url("./hi.css");
@import url("./hello.css");
@import url("./yo.css");
```

Becomes:

```css
/* hi.css */
/* ...contents of hi.css */
/* hello.css */
/* ...contents of hello.css */
/* yo.css */
/* ...contents of yo.css */
```

## CSS runtime

To support hot CSS reloading, Bun inserts `@supports` annotations into CSS that tag which files a stylesheet is composed of. Browsers ignore this, so it doesn’t impact styles.

By default, Bun’s runtime code automatically listens to `onimportcss` and will insert the `event.detail` into a `<link rel="stylesheet" href={${event.detail}}>` if there is no existing `link` tag with that stylesheet. That’s how Bun’s equivalent of `style-loader` works.


To create a new Next.js app with bun:

```bash
$ bun create next ./app
$ cd app
$ bun dev # start dev server
```

To use an existing Next.js app with bun:

```bash
$ bun add bun-framework-next
$ echo "framework = 'next'" > bunfig.toml
$ bun bun # bundle dependencies
$ bun dev # start dev server
```

Many of Next.js’ features are supported, but not all.

Here’s what doesn’t work yet:

- `getStaticPaths`
- same-origin `fetch` inside of `getStaticProps` or `getServerSideProps`
- locales, zones, `assetPrefix` (workaround: change `--origin \"http://localhost:3000/assetPrefixInhere\"`)
- `next/image` is polyfilled to a regular `<img src>` tag.
- `proxy` and anything else in `next.config.js`
- API routes, middleware (middleware is easier to support, though! Similar SSR API)
- styled-jsx (technically not Next.js, but often used with it)
- React Server Components

When using Next.js, Bun automatically reads configuration from `.env.local`, `.env.development` and `.env` (in that order). `process.env.NEXT_PUBLIC_` and `process.env.NEXT_` automatically are replaced via `--define`.

Currently, any time you import new dependencies from `node_modules`, you will need to re-run `bun bun --use next`. This will eventually be automatic.


To create a new React app:

```bash
$ bun create react ./app
$ cd app
$ bun dev # start dev server
```

To use an existing React app:

```bash
$ bun add -d react-refresh # install React Fast Refresh
$ bun bun ./src/index.js # generate a bundle for your entry point(s)
$ bun dev # start the dev server
```

From there, Bun relies on the filesystem for mapping dev server paths to source files. All URL paths are relative to the project root (where `package.json` is located).

Here are examples of routing source code file paths:

| Dev Server URL             | File Path (relative to cwd) |
| -------------------------- | --------------------------- |
| /src/components/Button.tsx | src/components/Button.tsx   |
| /src/index.tsx             | src/index.tsx               |
| /pages/index.js            | pages/index.js              |

You do not need to include file extensions in `import` paths. CommonJS-style import paths without the file extension work.

You can override the public directory by passing `--public-dir="path-to-folder"`.

If no directory is specified and `./public/` doesn’t exist, Bun will try `./static/`. If `./static/` does not exist, but won’t serve from a public directory. If you pass `--public-dir=./` Bun will serve from the current directory, but it will check the current directory last instead of first.


## Creating a Discord bot with Bun

Discord bots perform actions in response to _application commands_. There are 3 types of commands accessible in different interfaces: the chat input, a message's context menu (top-right menu or right-clicking in a message), and a user's context menu (right-clicking on a user).

To get started you can use the interactions template:

```bash
bun create discord-interactions my-interactions-bot
cd my-interactions-bot
```

If you don't have a Discord bot/application yet, you can create one [here (https://discord.com/developers/applications/me)](https://discord.com/developers/applications/me).

Invite bot to your server by visiting `https://discord.com/api/oauth2/authorize?client_id=<your_application_id>&scope=bot%20applications.commands`

Afterwards you will need to get your bot's token, public key, and application id from the application page and put them into `.env.example` file

Then you can run the http server that will handle your interactions:

```bash
$ bun install
$ mv .env.example .env
$ bun run.js # listening on port 1337
```

Discord does not accept an insecure HTTP server, so you will need to provide an SSL certificate or put the interactions server behind a secure reverse proxy. For development, you can use ngrok/cloudflare tunnel to expose local ports as secure URL.


Bun's bundler implements a `--compile` flag for generating a standalone binary from a TypeScript or JavaScript file.

{% codetabs %}

```bash
$ bun build ./cli.ts --compile --outfile mycli
```

```ts#cli.ts
console.log("Hello world!");
```

{% /codetabs %}

This bundles `cli.ts` into an executable that can be executed directly:

```
$ ./mycli
Hello world!
```

All imported files and packages are bundled into the executable, along with a copy of the Bun runtime. All built-in Bun and Node.js APIs are supported.

{% callout %}

**Note** — Currently, the `--compile` flag can only accept a single entrypoint at a time and does not support the following flags:

- `--outdir` — use `outfile` instead.
- `--external`
- `--splitting`
- `--publicPath`

{% /callout %}


Macros are a mechanism for running JavaScript functions _at bundle-time_. The value returned from these functions are directly inlined into your bundle.

<!-- embed the result in your (browser) bundle. This is useful for things like embedding the current Git commit hash in your code, making fetch requests to your API at build-time, dead code elimination, and more. -->

As a toy example, consider this simple function that returns a random number.

```ts
export function random() {
  return Math.random();
}
```

This is just a regular function in a regular file, but we can use it as a macro like so:

```ts#cli.tsx
import { random } from './random.ts' with { type: 'macro' };

console.log(`Your random number is ${random()}`);
```

{% callout %}
**Note** — Macros are indicated using [_import attribute_](https://github.com/tc39/proposal-import-attributes) syntax. If you haven't seen this syntax before, it's a Stage 3 TC39 proposal that lets you attach additional metadata to `import` statements.
{% /callout %}

Now we'll bundle this file with `bun build`. The bundled file will be printed to stdout.

```bash
$ bun build ./cli.tsx
console.log(`Your random number is ${0.6805550949689833}`);
```

As you can see, the source code of the `random` function occurs nowhere in the bundle. Instead, it is executed _during bundling_ and function call (`random()`) is replaced with the result of the function. Since the source code will never be included in the bundle, macros can safely perform privileged operations like reading from a database.

## When to use macros

If you have several build scripts for small things where you would otherwise have a one-off build script, bundle-time code execution can be easier to maintain. It lives with the rest of your code, it runs with the rest of the build, it is automatically paralellized, and if it fails, the build fails too.

If you find yourself running a lot of code at bundle-time though, consider running a server instead.

## Import attributes

Bun Macros are import statements annotated using either:

- `with { type: 'macro' }` — an [import attribute](https://github.com/tc39/proposal-import-attributes), a Stage 3 ECMA Scrd
- `assert { type: 'macro' }` — an import assertion, an earlier incarnation of import attributes that has now been abandoned (but is [already supported](https://caniuse.com/mdn-javascript_statements_import_import_assertions) by a number of browsers and runtimes)

## Security considerations

Macros must explicitly be imported with `{ type: "macro" }` in order to be executed at bundle-time. These imports have no effect if they are not called, unlike regular JavaScript imports which may have side effects.

You can disable macros entirely by passing the `--no-macros` flag to Bun. It produces a build error like this:

```js
error: Macros are disabled

foo();
^
./hello.js:3:1 53
```

To reduce the potential attack surface for malicious packages, macros cannot be _invoked_ from inside `node_modules/**/*`. If a package attempts to invoke a macro, you'll see an error like this:

```js
error: For security reasons, macros cannot be run from node_modules.

beEvil();
^
node_modules/evil/index.js:3:1 50
```

Your application code can still import macros from `node_modules` and invoke them.

```ts
import {macro} from "some-package" with { type: "macro" };

macro();
```

## Export condition `"macro"`

When shipping a library containing a macro to `npm` or another package registry, use the `"macro"` [export condition](https://nodejs.org/api/packages.html#conditional-exports) to provide a special version of your package exclusively for the macro environment.

```jsonc#package.json
{
  "name": "my-package",
  "exports": {
    "import": "./index.js",
    "require": "./index.js",
    "default": "./index.js",
    "macro": "./index.macro.js"
  }
}
```

With this configuration, users can consume your package at runtime or at bundle-time using the same import specifier:

```ts
import pkg from "my-package";                            // runtime import
import {macro} from "my-package" with { type: "macro" }; // macro import
```

The first import will resolve to `./node_modules/my-package/index.js`, while the second will be resolved by Bun's bundler to `./node_modules/my-package/index.macro.js`.

## Execution

When Bun's transpiler sees a macro import, it calls the function inside the transpiler using Bun's JavaScript runtime and converts the return value from JavaScript into an AST node. These JavaScript functions are called at bundle-time, not runtime.

Macros are executed synchronously in the transpiler during the visiting phase—before plugins and before the transpiler generates the AST. They are executed in the order they are imported. The transpiler will wait for the macro to finish executing before continuing. The transpiler will also `await` any `Promise` returned by a macro.

Bun's bundler is multi-threaded. As such, macros execute in parallel inside of multiple spawned JavaScript "workers".

## Dead code elimination

The bundler performs dead code elimination _after_ running and inlining macros. So given the following macro:

```ts#returnFalse.ts
export function returnFalse() {
  return false;
}
```

...then bundling the following file will produce an empty bundle.

```ts
import {returnFalse} from './returnFalse.ts' with { type: 'macro' };

if (returnFalse()) {
  console.log("This code is eliminated");
}
```

## Serializablility

Bun's transpiler needs to be able to serialize the result of the macro so it can be inlined into the AST. All JSON-compatible data structures are supported:

```ts#macro.ts
export function getObject() {
  return {
    foo: "bar",
    baz: 123,
    array: [ 1, 2, { nested: "value" }],
  };
}
```

Macros can be async, or return `Promise` instances. Bun's transpiler will automatically `await` the `Promise` and inline the result.

```ts#macro.ts
export async function getText() {
  return "async value";
}
```

The transpiler implements special logic for serializing common data formats like `Response`, `Blob`, `TypedArray`.

- `TypedArray`: Resolves to a base64-encoded string.
- `Response`: Bun will read the `Content-Type` and serialize accordingly; for instance, a `Response` with type `application/json` will be automatically parsed into an object and `text/plain` will be inlined as a string. Responses with an unrecognized or `undefined` `type` will be base-64 encoded.
- `Blob`: As with `Response`, the serialization depends on the `type` property.

The result of `fetch` is `Promise<Response>`, so it can be directly returned.

```ts#macro.ts
export function getObject() {
  return fetch("https://bun.sh")
}
```

Functions and instances of most classes (except those mentioned above) are not serializable.

```ts
export function getText(url: string) {
  // this doesn't work!
  return () => {};
}
```

## Arguments

Macros can accept inputs, but only in limited cases. The value must be statically known. For example, the following is not allowed:

```ts
import {getText} from './getText.ts' with { type: 'macro' };

export function howLong() {
  // the value of `foo` cannot be statically known
  const foo = Math.random() ? "foo" : "bar";

  const text = getText(`https://example.com/${foo}`);
  console.log("The page is ", text.length, " characters long");
}
```

However, if the value of `foo` is known at bundle-time (say, if it's a constant or the result of another macro) then it's allowed:

```ts
import {getText} from './getText.ts' with { type: 'macro' };
import {getFoo} from './getFoo.ts' with { type: 'macro' };

export function howLong() {
  // this works because getFoo() is statically known
  const foo = getFoo();
  const text = getText(`https://example.com/${foo}`);
  console.log("The page is", text.length, "characters long");
}
```

This outputs:

```ts
function howLong() {
  console.log("The page is", 1322, "characters long");
}
export { howLong };
```

## Examples

### Embed latest git commit hash

{% codetabs %}

```ts#getGitCommitHash.ts
export function getGitCommitHash() {
  const {stdout} = Bun.spawnSync({
    cmd: ["git", "rev-parse", "HEAD"],
    stdout: "pipe",
  });

  return stdout.toString();
}
```

{% /codetabs %}

<!-- --target=browser so they can clearly see it's for browsers -->

When we build it, the `getGitCommitHash` is replaced with the result of calling the function:

{% codetabs %}

```ts#input
import { getGitCommitHash } from './getGitCommitHash.ts' with { type: 'macro' };

console.log(`The current Git commit hash is ${getGitCommitHash()}`);
```

```bash#output
console.log(`The current Git commit hash is 3ee3259104f`);
```

{% /codetabs %}

You're probably thinking "Why not just use `process.env.GIT_COMMIT_HASH`?" Well, you can do that too. But can you do this with an environment variable?

### Make `fetch()` requests at bundle-time

In this example, we make an outgoing HTTP request using `fetch()`, parse the HTML response using `HTMLRewriter`, and return an object containing the title and meta tags–all at bundle-time.

```ts
export async function extractMetaTags(url: string) {
  const response = await fetch(url);
  const meta = {
    title: "",
  };
  new HTMLRewriter()
    .on("title", {
      text(element) {
        meta.title += element.text;
      },
    })
    .on("meta", {
      element(element) {
        const name =
          element.getAttribute("name") || element.getAttribute("property") || element.getAttribute("itemprop");

        if (name) meta[name] = element.getAttribute("content");
      },
    })
    .transform(response);

  return meta;
}
```

<!-- --target=browser so they can clearly see it's for browsers -->

The `extractMetaTags` function is erased at bundle-time and replaced with the result of the function call. This means that the `fetch` request happens at bundle-time, and the result is embedded in the bundle. Also, the branch throwing the error is eliminated since it's unreachable.

{% codetabs %}

```ts#input
import { extractMetaTags } from './meta.ts' with { type: 'macro' };

export const Head = () => {
  const headTags = extractMetaTags("https://example.com");

  if (headTags.title !== "Example Domain") {
    throw new Error("Expected title to be 'Example Domain'");
  }

  return <head>
    <title>{headTags.title}</title>
    <meta name="viewport" content={headTags.viewport} />
  </head>;
};
```

```ts#output
import { jsx, jsxs } from "react/jsx-runtime";
export const Head = () => {
  jsxs("head", {
    children: [
      jsx("title", {
        children: "Example Domain",
      }),
      jsx("meta", {
        name: "viewport",
        content: "width=device-width, initial-scale=1",
      }),
    ],
  });
};

export { Head };
```

{% /codetabs %}


<!-- This document is a work in progress. It's not currently included in the actual docs. -->

The goal of this document is to break down why bundling is necessary, how it works, and how the bundler became such a key part of modern JavaScript development. The content is not specific to Bun's bundler, but is rather aimed at anyone looking for a greater understanding of how bundlers work and, by extension, how most modern frameworks are implemented.

## What is bundling

With the adoption of ECMAScript modules (ESM), browsers can now resolve `import`/`export` statements in JavaScript files loaded via `<script>` tags.

{% codetabs %}

```html#index.html
<html>
  <head>
    <script type="module" src="/index.js" ></script>
  </head>
</html>
```

```js#index.js
import {sayHello} from "./hello.js";

sayHello();
```

```js#hello.js
export function sayHello() {
  console.log("Hello, world!");
}
```

{% /codetabs %}

When a user visits this website, the files are loaded in the following order:

{% image src="/images/module_loading_unbundled.png" /%}

{% callout %}
**Relative imports** — Relative imports are resolved relative to the URL of the importing file. Because we're importing `./hello.js` from `/index.js`, the browser resolves it to `/hello.js`. If instead we'd imported `./hello.js` from `/src/index.js`, the browser would have resolved it to `/src/hello.js`.
{% /callout %}

This approach works, it requires three round-trip HTTP requests before the browser is ready to render the page. On slow internet connections, this may add up to a non-trivial delay.

This example is extremely simplistic. A modern app may be loading dozens of modules from `node_modules`, each consisting of hundrends of files. Loading each of these files with a separate HTTP request becomes untenable very quickly. While most of these requests will be running in parallel, the number of round-trip requests can still be very high; plus, there are limits on how many simultaneous requests a browser can make.

{% callout %}
Some recent advances like modulepreload and HTTP/3 are intended to solve some of these problems, but at the moment bundling is still the most performant approach.
{% /callout %}

The answer: bundling.

## Entrypoints

A bundler accepts an "entrypoint" to your source code (in this case, `/index.js`) and outputs a single file containing all of the code needed to run your app. If does so by parsing your source code, reading the `import`/`export` statements, and building a "module graph" of your app's dependencies.

{% image src="/images/bundling.png" /%}

We can now load `/bundle.js` from our `index.html` file and eliminate a round trip request, decreasing load times for our app.

{% image src="/images/module_loading_bundled.png" /%}

## Loaders

Bundlers typically have some set of built-in "loaders".

## Transpilation

The JavaScript files above are just that: plain JavaScript. They can be directly executed by any modern browser.

But modern tooling goes far beyond HTML, JavaScript, and CSS. JSX, TypeScript, and PostCSS/CSS-in-JS are all popular technologies that involve non-standard syntax that must be converted into vanilla JavaScript and CSS before if can be consumed by a browser.

## Chunking

## Module resolution

## Plugins


Bun's fast native bundler is now in beta. It can be used via the `bun build` CLI command or the `Bun.build()` JavaScript API.

{% codetabs group="a" %}

```ts#JavaScript
await Bun.build({
  entrypoints: ['./index.tsx'],
  outdir: './build',
});
```

```sh#CLI
$ bun build ./index.tsx --outdir ./build
```

{% /codetabs %}

It's fast. The numbers below represent performance on esbuild's [three.js benchmark](https://github.com/oven-sh/bun/tree/main/bench/bundle).

{% image src="/images/bundler-speed.png" caption="Bundling 10 copies of three.js from scratch, with sourcemaps and minification" /%}

## Why bundle?

The bundler is a key piece of infrastructure in the JavaScript ecosystem. As a brief overview of why bundling is so important:

- **Reducing HTTP requests.** A single package in `node_modules` may consist of hundreds of files, and large applications may have dozens of such dependencies. Loading each of these files with a separate HTTP request becomes untenable very quickly, so bundlers are used to convert our application source code into a smaller number of self-contained "bundles" that can be loaded with a single request.
- **Code transforms.** Modern apps are commonly built with languages or tools like TypeScript, JSX, and CSS modules, all of which must be converted into plain JavaScript and CSS before they can be consumed by a browser. The bundler is the natural place to configure these transformations.
- **Framework features.** Frameworks rely on bundler plugins & code transformations to implement common patterns like file-system routing, client-server code co-location (think `getServerSideProps` or Remix loaders), and server components.

Let's jump into the bundler API.

## Basic example

Let's build our first bundle. You have the following two files, which implement a simple client-side rendered React app.

{% codetabs %}

```tsx#./index.tsx
import * as ReactDOM from 'react-dom/client';
import {Component} from "./Component"

const root = ReactDOM.createRoot(document.getElementById('root'));
root.render(<Component message="Sup!" />)
```

```tsx#./Component.tsx
export function Component(props: {message: string}) {
  return <p>{props.message}</p>
}
```

{% /codetabs %}

Here, `index.tsx` is the "entrypoint" to our application. Commonly, this will be a script that performs some _side effect_, like starting a server or—in this case—initializing a React root. Because we're using TypeScript & JSX, we need to bundle our code before it can be sent to the browser.

To create our bundle:

{% codetabs group="a" %}

```ts#JavaScript
await Bun.build({
  entrypoints: ['./index.tsx'],
  outdir: './out',
})
```

```bash#CLI
$ bun build ./index.tsx --outdir ./out
```

{% /codetabs %}

For each file specified in `entrypoints`, Bun will generate a new bundle. This bundle will be written to disk in the `./out` directory (as resolved from the current working directory). After running the build, the file system looks like this:

```ts
.
├── index.tsx
├── Component.tsx
└── out
    └── index.js
```

The contents of `out/index.js` will look something like this:

```js#out/index.js
// ...
// ~20k lines of code
// including the contents of `react-dom/client` and all its dependencies
// this is where the $jsxDEV and $createRoot functions are defined


// Component.tsx
function Component(props) {
  return $jsxDEV("p", {
    children: props.message
  }, undefined, false, undefined, this);
}

// index.tsx
var rootNode = document.getElementById("root");
var root = $createRoot(rootNode);
root.render($jsxDEV(Component, {
  message: "Sup!"
}, undefined, false, undefined, this));
```

{% details summary="Tutorial: Run this file in your browser" %}
We can load this file in the browser to see our app in action. Create an `index.html` file in the `out` directory:

```bash
$ touch out/index.html
```

Then paste the following contents into it:

```html
<html>
  <body>
    <div id="root"></div>
    <script type="module" src="/index.js"></script>
  </body>
</html>
```

Then spin up a static file server serving the `out` directory:

```bash
$ bunx serve out
```

Visit `http://localhost:5000` to see your bundled app in action.

{% /details %}

## Content types

Like the Bun runtime, the bundler supports an array of file types out of the box. The following table breaks down the bundler's set of standard "loaders". Refer to [Bundler > File types](/docs/runtime/loaders) for full documentation.

{% table %}

- Extensions
- Details

---

- `.js` `.cjs` `.mjs` `.mts` `.cts` `.ts` `.tsx`
- Uses Bun's built-in transpiler to parse the file and transpile TypeScript/JSX syntax to vanilla JavaScript. The bundler executes a set of default transforms, including dead code elimination, tree shaking, and environment variable inlining. At the moment Bun does not attempt to down-convert syntax; if you use recently ECMAScript syntax, that will be reflected in the bundled code.

---

- `.json`
- JSON files are parsed and inlined into the bundle as a JavaScript object.

  ```ts
  import pkg from "./package.json";
  pkg.name; // => "my-package"
  ```

---

- `.toml`
- TOML files are parsed and inlined into the bundle as a JavaScript object.

  ```ts
  import config from "./bunfig.toml";
  config.logLevel; // => "debug"
  ```

---

- `.txt`
- The contents of the text file are read and inlined into the bundle as a string.

  ```ts
  import contents from "./file.txt";
  console.log(contents); // => "Hello, world!"
  ```

---

- `.node` `.wasm`
- These files are supported by the Bun runtime, but during bundling they are treated as [assets](#assets).

{% /table %}

### Assets

If the bundler encounters an import with an unrecognized extension, it treats the imported file as an _external file_. The referenced file is copied as-is into `outdir`, and the import is resolved as a _path_ to the file.

{% codetabs %}

```ts#Input
// bundle entrypoint
import logo from "./logo.svg";
console.log(logo);
```

```ts#Output
// bundled output
var logo = "./logo-ab237dfe.svg";
console.log(logo);
```

{% /codetabs %}

{% callout %}
The exact behavior of the file loader is also impacted by [`naming`](#naming) and [`publicPath`](#publicpath).
{% /callout %}

Refer to the [Bundler > Loaders](/docs/bundler/loaders#file) page for more complete documentation on the file loader.

### Plugins

The behavior described in this table can be overridden or extended with [plugins](/docs/bundler/plugins). Refer to the [Bundler > Loaders](/docs/bundler/plugins) page for complete documentation.

## API

### `entrypoints`

**Required.** An array of paths corresponding to the entrypoints of our application. One bundle will be generated for each entrypoint.

{% codetabs group="a" %}

```ts#JavaScript
const result = await Bun.build({
  entrypoints: ["./index.ts"],
});
// => { success: boolean, outputs: BuildArtifact[], logs: BuildMessage[] }
```

```bash#CLI
$ bun build --entrypoints ./index.ts
# the bundle will be printed to stdout
# <bundled code>
```

{% /codetabs %}

### `outdir`

The directory where output files will be written.

{% codetabs group="a" %}

```ts#JavaScript
const result = await Bun.build({
  entrypoints: ['./index.ts'],
  outdir: './out'
});
// => { success: boolean, outputs: BuildArtifact[], logs: BuildMessage[] }
```

```bash#CLI
$ bun build --entrypoints ./index.ts --outdir ./out
# a summary of bundled files will be printed to stdout
```

{% /codetabs %}

If `outdir` is not passed to the JavaScript API, bundled code will not be written to disk. Bundled files are returned in an array of `BuildArtifact` objects. These objects are Blobs with extra properties; see [Outputs](#outputs) for complete documentation.

```ts
const result = await Bun.build({
  entrypoints: ["./index.ts"],
});

for (const result of result.outputs) {
  // Can be consumed as blobs
  await result.text();

  // Bun will set Content-Type and Etag headers
  new Response(result);

  // Can be written manually, but you should use `outdir` in this case.
  Bun.write(path.join("out", result.path), result);
}
```

When `outdir` is set, the `path` property on a `BuildArtifact` will be the absolute path to where it was written to.

### `target`

The intended execution environment for the bundle.

{% codetabs group="a" %}

```ts#JavaScript
await Bun.build({
  entrypoints: ['./index.ts'],
  outdir: './out',
  target: 'browser', // default
})
```

```bash#CLI
$ bun build --entrypoints ./index.ts --outdir ./out --target browser
```

{% /codetabs %}

Depending on the target, Bun will apply different module resolution rules and optimizations.

<!-- - Module resolution. For example, when bundling for the browser, Bun will prioritize the `"browser"` export condition when resolving imports. An error will be thrown if any Node.js or Bun built-ins are imported or used, e.g. `node:fs` or `Bun.serve`. -->

{% table %}

---

- `browser`
- _Default._ For generating bundles that are intended for execution by a browser. Prioritizes the `"browser"` export condition when resolving imports. An error will be thrown if any Node.js or Bun built-ins are imported or used, e.g. `node:fs` or `Bun.serve`.

---

- `bun`
- For generating bundles that are intended to be run by the Bun runtime. In many cases, it isn't necessary to bundle server-side code; you can directly execute the source code without modification. However, bundling your server code can reduce startup times and improve running performance.

  All bundles generated with `target: "bun"` are marked with a special `// @bun` pragma, which indicates to the Bun runtime that there's no need to re-transpile the file before execution.

  If any entrypoints contains a Bun shebang (`#!/usr/bin/env bun`) the bundler will default to `target: "bun"` instead of `"browser`.

---

- `node`
- For generating bundles that are intended to be run by Node.js. Prioritizes the `"node"` export condition when resolving imports, and outputs `.mjs`. In the future, this will automatically polyfill the `Bun` global and other built-in `bun:*` modules, though this is not yet implemented.

{% /table %}

{% callout %}

{% /callout %}

### `format`

Specifies the module format to be used in the generated bundles.

Currently the bundler only supports one module format: `"esm"`. Support for `"cjs"` and `"iife"` are planned.

{% codetabs %}

```ts#JavaScript
await Bun.build({
  entrypoints: ['./index.tsx'],
  outdir: './out',
  format: "esm",
})
```

```bash#CLI
$ bun build ./index.tsx --outdir ./out --format esm
```

{% /codetabs %}

<!-- ### `bundling`

Whether to enable bundling.

{% codetabs group="a" %}

```ts#JavaScript
await Bun.build({
  entrypoints: ['./index.tsx'],
  outdir: './out',
  bundling: true, // default
})
```

```bash#CLI
# bundling is enabled by default
$ bun build ./index.tsx --outdir ./out
```

{% /codetabs %}

Set to `false` to disable bundling. Instead, files will be transpiled and individually written to `outdir`.

{% codetabs group="a" %}

```ts#JavaScript
await Bun.build({
  entrypoints: ['./index.tsx'],
  outdir: './out',
  bundling: false,
})
```

```bash#CLI
$ bun build ./index.tsx --outdir ./out --no-bundling
```

{% /codetabs %} -->

### `splitting`

Whether to enable code splitting.

{% codetabs group="a" %}

```ts#JavaScript
await Bun.build({
  entrypoints: ['./index.tsx'],
  outdir: './out',
  splitting: false, // default
})
```

```bash#CLI
$ bun build ./index.tsx --outdir ./out --splitting
```

{% /codetabs %}

When `true`, the bundler will enable _code splitting_. When multiple entrypoints both import the same file, module, or set of files/modules, it's often useful to split the shared code into a separate bundle. This shared bundle is known as a _chunk_. Consider the following files:

{% codetabs %}

```ts#entry-a.ts
import { shared } from './shared.ts';
```

```ts#entry-b.ts
import { shared } from './shared.ts';
```

```ts#shared.ts
export const shared = 'shared';
```

{% /codetabs %}

To bundle `entry-a.ts` and `entry-b.ts` with code-splitting enabled:

{% codetabs group="a" %}

```ts#JavaScript
await Bun.build({
  entrypoints: ['./entry-a.ts', './entry-b.ts'],
  outdir: './out',
  splitting: true,
})
```

```bash#CLI
$ bun build ./entry-a.ts ./entry-b.ts --outdir ./out --splitting
```

{% /codetabs %}

Running this build will result in the following files:

```txt
.
├── entry-a.tsx
├── entry-b.tsx
├── shared.tsx
└── out
    ├── entry-a.js
    ├── entry-b.js
    └── chunk-2fce6291bf86559d.js

```

The generated `chunk-2fce6291bf86559d.js` file contains the shared code. To avoid collisions, the file name automatically includes a content hash by default. This can be customized with [`naming`](#naming).

### `plugins`

A list of plugins to use during bundling.

{% codetabs group="a" %}

```ts#JavaScript
await Bun.build({
  entrypoints: ['./index.tsx'],
  outdir: './out',
  plugins: [/* ... */],
})
```

```bash#CLI
n/a
```

{% /codetabs %}

Bun implements a univeral plugin system for both Bun's runtime and bundler. Refer to the [plugin documentation](/docs/bundler/plugins) for complete documentation.

<!-- ### `manifest`

Whether to return a build manifest in the result of `Bun.build`.

```ts
const result = await Bun.build({
  entrypoints: ["./index.tsx"],
  outdir: "./out",
  manifest: true, // default is true
});

console.log(result.manifest);
```

{% details summary="Manifest structure" %}

The manifest has the following form:

```ts
export type BuildManifest = {
  inputs: {
    [path: string]: {
      output: {
        path: string;
      };
      imports: {
        path: string;
        kind: ImportKind;
        external?: boolean;
      }[];
    };
  };
  outputs: {
    [path: string]: {
      type: "chunk" | "entry-point" | "asset";
      inputs: { path: string }[];
      imports: {
        path: string;
        kind: ImportKind;
        external?: boolean;
        asset?: boolean;
      }[];
      exports: string[];
    };
  };
};

export type ImportKind =
  | "entry-point"
  | "import-statement"
  | "require-call"
  | "dynamic-import"
  | "require-resolve"
  | "import-rule"
  | "url-token";
```

{% /details %}

By design, the manifest is a simple JSON object that can easily be serialized or written to disk. It is also compatible with esbuild's [`metafile`](https://esbuild.github.io/api/#metafile) format. -->

### `sourcemap`

Specifies the type of sourcemap to generate.

{% codetabs group="a" %}

```ts#JavaScript
await Bun.build({
  entrypoints: ['./index.tsx'],
  outdir: './out',
  sourcemap: "external", // default "none"
})
```

```bash#CLI
$ bun build ./index.tsx --outdir ./out --sourcemap=external
```

{% /codetabs %}

{% table %}

---

- `"none"`
- _Default._ No sourcemap is generated.

---

- `"inline"`
- A sourcemap is generated and appended to the end of the generated bundle as a base64 payload.

  ```ts
  // <bundled code here>

  //# sourceMappingURL=data:application/json;base64,<encoded sourcemap here>
  ```

---

- `"external"`
- A separate `*.js.map` file is created alongside each `*.js` bundle.

{% /table %}

{% callout %}

Generated bundles contain a [debug id](https://sentry.engineering/blog/the-case-for-debug-ids) that can be used to associate a bundle with its corresponding sourcemap. This `debugId` is added as a comment at the bottom of the file.

```ts
// <generated bundle code>

//# debugId=<DEBUG ID>
```

The associated `*.js.map` sourcemap will be a JSON file containing an equivalent `debugId` property.

{% /callout %}

### `minify`

Whether to enable minification. Default `false`.

{% callout %}
When targeting `bun`, identifiers will be minified by default.
{% /callout %}

To enable all minification options:

{% codetabs group="a" %}

```ts#JavaScript
await Bun.build({
  entrypoints: ['./index.tsx'],
  outdir: './out',
  minify: true, // default false
})
```

```bash#CLI
$ bun build ./index.tsx --outdir ./out --minify
```

{% /codetabs %}

To granularly enable certain minifications:

{% codetabs group="a" %}

```ts#JavaScript
await Bun.build({
  entrypoints: ['./index.tsx'],
  outdir: './out',
  minify: {
    whitespace: true,
    identifiers: true,
    syntax: true,
  },
})
```

```bash#CLI
$ bun build ./index.tsx --outdir ./out --minify-whitespace --minify-identifiers --minify-syntax
```

{% /codetabs %}

<!-- ### `treeshaking`

boolean; -->

### `external`

A list of import paths to consider _external_. Defaults to `[]`.

{% codetabs group="a" %}

```ts#JavaScript
await Bun.build({
  entrypoints: ['./index.tsx'],
  outdir: './out',
  external: ["lodash", "react"], // default: []
})
```

```bash#CLI
$ bun build ./index.tsx --outdir ./out --external lodash --external react
```

{% /codetabs %}

An external import is one that will not be included in the final bundle. Instead, the `import` statement will be left as-is, to be resolved at runtime.

For instance, consider the following entrypoint file:

```ts#index.tsx
import _ from "lodash";
import {z} from "zod";

const value = z.string().parse("Hello world!")
console.log(_.upperCase(value));
```

Normally, bundling `index.tsx` would generate a bundle containing the entire source code of the `"zod"` package. If instead, we want to leave the `import` statement as-is, we can mark it as external:

{% codetabs group="a" %}

```ts#JavaScript
await Bun.build({
  entrypoints: ['./index.tsx'],
  outdir: './out',
  external: ['zod'],
})
```

```bash#CLI
$ bun build ./index.tsx --outdir ./out --external zod
```

{% /codetabs %}

The generated bundle will look something like this:

```js#out/index.js
import {z} from "zod";

// ...
// the contents of the "lodash" package
// including the `_.upperCase` function

var value = z.string().parse("Hello world!")
console.log(_.upperCase(value));
```

To mark all imports as external, use the wildcard `*`:

{% codetabs %}

```ts#JavaScript
await Bun.build({
  entrypoints: ['./index.tsx'],
  outdir: './out',
  external: ['*'],
})
```

```bash#CLI
$ bun build ./index.tsx --outdir ./out --external '*'
```

{% /codetabs %}

### `naming`

Customizes the generated file names. Defaults to `./[dir]/[name].[ext]`.

{% codetabs group="a" %}

```ts#JavaScript
await Bun.build({
  entrypoints: ['./index.tsx'],
  outdir: './out',
  naming: "[dir]/[name].[ext]", // default
})
```

```bash#CLI
$ bun build ./index.tsx --outdir ./out --entry-naming [dir]/[name].[ext]
```

{% /codetabs %}

By default, the names of the generated bundles are based on the name of the associated entrypoint.

```txt
.
├── index.tsx
└── out
    └── index.js
```

With multiple entrypoints, the generated file hierarchy will reflect the directory structure of the entrypoints.

```txt
.
├── index.tsx
└── nested
    └── index.tsx
└── out
    ├── index.js
    └── nested
        └── index.js
```

The names and locations of the generated files can be customized with the `naming` field. This field accepts a template string that is used to generate the filenames for all bundles corresponding to entrypoints. where the following tokens are replaced with their corresponding values:

- `[name]` - The name of the entrypoint file, without the extension.
- `[ext]` - The extension of the generated bundle.
- `[hash]` - A hash of the bundle contents.
- `[dir]` - The relative path from the build root to the parent directory of the file.

For example:

{% table %}

- Token
- `[name]`
- `[ext]`
- `[hash]`
- `[dir]`

---

- `./index.tsx`
- `index`
- `js`
- `a1b2c3d4`
- `""` (empty string)

---

- `./nested/entry.ts`
- `entry`
- `js`
- `c3d4e5f6`
- `"nested"`

{% /table %}

We can combine these tokens to create a template string. For instance, to include the hash in the generated bundle names:

{% codetabs group="a" %}

```ts#JavaScript
await Bun.build({
  entrypoints: ['./index.tsx'],
  outdir: './out',
  naming: 'files/[dir]/[name]-[hash].[ext]',
})
```

```bash#CLI
$ bun build ./index.tsx --outdir ./out --entry-naming [name]-[hash].[ext]
```

{% /codetabs %}

This build would result in the following file structure:

```txt
.
├── index.tsx
└── out
    └── files
        └── index-a1b2c3d4.js
```

When a `string` is provided for the `naming` field, it is used only for bundles _that correspond to entrypoints_. The names of [chunks](#splitting) and copied assets are not affected. Using the JavaScript API, separate template strings can be specified for each type of generated file.

{% codetabs group="a" %}

```ts#JavaScript
await Bun.build({
  entrypoints: ['./index.tsx'],
  outdir: './out',
  naming: {
    // default values
    entry: '[dir]/[name].[ext]',
    chunk: '[name]-[hash].[ext]',
    asset: '[name]-[hash].[ext]',
  },
})
```

```bash#CLI
$ bun build ./index.tsx --outdir ./out --entry-naming "[dir]/[name].[ext]" --chunk-naming "[name]-[hash].[ext]" --asset-naming "[name]-[hash].[ext]"
```

{% /codetabs %}

### `root`

The root directory of the project.

{% codetabs group="a" %}

```ts#JavaScript
await Bun.build({
  entrypoints: ['./pages/a.tsx', './pages/b.tsx'],
  outdir: './out',
  root: '.',
})
```

```bash#CLI
n/a
```

{% /codetabs %}

If unspecified, it is computed to be the first common ancestor of all entrypoint files. Consider the following file structure:

```txt
.
└── pages
  └── index.tsx
  └── settings.tsx
```

We can build both entrypoints in the `pages` directory:

{% codetabs group="a" %}

```ts#JavaScript
await Bun.build({
  entrypoints: ['./pages/index.tsx', './pages/settings.tsx'],
  outdir: './out',
})
```

```bash#CLI
$ bun build ./pages/index.tsx ./pages/settings.tsx --outdir ./out
```

{% /codetabs %}

This would result in a file structure like this:

```txt
.
└── pages
  └── index.tsx
  └── settings.tsx
└── out
  └── index.js
  └── settings.js
```

Since the `pages` directory is the first common ancestor of the entrypoint files, it is considered the project root. This means that the generated bundles live at the top level of the `out` directory; there is no `out/pages` directory.

This behavior can be overridden by specifying the `root` option:

{% codetabs group="a" %}

```ts#JavaScript
await Bun.build({
  entrypoints: ['./pages/index.tsx', './pages/settings.tsx'],
  outdir: './out',
  root: '.',
})
```

```bash#CLI
$ bun build ./pages/index.tsx ./pages/settings.tsx --outdir ./out --root .
```

{% /codetabs %}

By specifying `.` as `root`, the generated file structure will look like this:

```txt
.
└── pages
  └── index.tsx
  └── settings.tsx
└── out
  └── pages
    └── index.js
    └── settings.js
```

### `publicPath`

A prefix to be appended to any import paths in bundled code.

<!-- $ bun build ./index.tsx --outdir ./out --publicPath https://cdn.example.com -->

In many cases, generated bundles will contain no `import` statements. After all, the goal of bundling is to combine all of the code into a single file. However there are a number of cases with the generated bundles will contain `import` statements.

- **Asset imports** — When importing an unrecognized file type like `*.svg`, the bundler defers to the [`file` loader](/docs/bundler/loaders#file), which copies the file into `outdir` as is. The import is converted into a variable
- **External modules** — Files and modules can be marked as [`external`](#external), in which case they will not be included in the bundle. Instead, the `import` statement will be left in the final bundle.
- **Chunking**. When [`splitting`](#splitting) is enabled, the bundler may generate separate "chunk" files that represent code that is shared among multiple entrypoints.

In any of these cases, the final bundles may contain paths to other files. By default these imports are _relative_. Here is an example of a simple asset import:

{% codetabs %}

```ts#Input
import logo from './logo.svg';
console.log(logo);
```

```ts#Output
// logo.svg is copied into <outdir>
// and hash is added to the filename to prevent collisions
var logo = './logo-a7305bdef.svg';
console.log(logo);
```

{% /codetabs %}

Setting `publicPath` will prefix all file paths with the specified value.

{% codetabs group="a" %}

```ts#JavaScript
await Bun.build({
  entrypoints: ['./index.tsx'],
  outdir: './out',
  publicPath: 'https://cdn.example.com/', // default is undefined
})
```

```bash#CLI
n/a
```

{% /codetabs %}

The output file would now look something like this.

```ts-diff#Output
- var logo = './logo-a7305bdef.svg';
+ var logo = 'https://cdn.example.com/logo-a7305bdef.svg';
```

### `define`

A map of global identifiers to be replaced at build time. Keys of this object are identifier names, and values are JSON strings that will be inlined.

{% callout }
This is not needed to inline `process.env.NODE_ENV`, as Bun does this automatically.
{% /callout %}

{% codetabs %}

```ts#JavaScript
await Bun.build({
  entrypoints: ['./index.tsx'],
  outdir: './out',
  define: {
    STRING: JSON.stringify("value"),
    "nested.boolean": "true",
  },
})
```

```bash#CLI
$ bun build ./index.tsx --outdir ./out --define 'STRING="value"' --define "nested.boolean=true"
```

{% /codetabs %}

### `loader`

A map of file extensions to [built-in loader names](https://bun.sh/docs/bundler/loaders#built-in-loaders). This can be used to quickly customize how certain file files are loaded.

{% codetabs %}

```ts#JavaScript
await Bun.build({
  entrypoints: ['./index.tsx'],
  outdir: './out',
  loader: {
    ".png": "dataurl",
    ".txt": "file",
  },
})
```

```bash#CLI
$ bun build ./index.tsx --outdir ./out --loader .png:dataurl --loader .txt:file
```

{% /codetabs %}

## Outputs

The `Bun.build` function returns a `Promise<BuildOutput>`, defined as:

```ts
interface BuildOutput {
  outputs: BuildArtifact[];
  success: boolean;
  logs: Array<object>; // see docs for details
}

interface BuildArtifact extends Blob {
  kind: "entry-point" | "chunk" | "asset" | "sourcemap";
  path: string;
  loader: Loader;
  hash: string | null;
  sourcemap: BuildArtifact | null;
}
```

The `outputs` array contains all the files that were generated by the build. Each artifact implements the `Blob` interface.

```ts
const build = Bun.build({
  /* */
});

for (const output of build.outputs) {
  await output.arrayBuffer(); // => ArrayBuffer
  await output.text(); // string
}
```

Each artifact also contains the following properties:

{% table %}

---

- `kind`
- What kind of build output this file is. A build generates bundled entrypoints, code-split "chunks", sourcemaps, and copied assets (like images).

---

- `path`
- Absolute path to the file on disk

---

- `loader`
- The loader was used to interpret the file. See [Bundler > Loaders](/docs/bundler/loaders) to see how Bun maps file extensions to the appropriate built-in loader.

---

- `hash`
- The hash of the file contents. Always defined for assets.

---

- `sourcemap`
- The sourcemap file corresponding to this file, if generated. Only defined for entrypoints and chunks.

{% /table %}

Similar to `BunFile`, `BuildArtifact` objects can be passed directly into `new Response()`.

```ts
const build = Bun.build({
  /* */
});

const artifact = build.outputs[0];

// Content-Type header is automatically set
return new Response(artifact);
```

The Bun runtime implements special pretty-printing of `BuildArtifact` object to make debugging easier.

{% codetabs %}

```ts#Build_script
// build.ts
const build = Bun.build({/* */});

const artifact = build.outputs[0];
console.log(artifact);
```

```sh#Shell_output
$ bun run build.ts
BuildArtifact (entry-point) {
  path: "./index.js",
  loader: "tsx",
  kind: "entry-point",
  hash: "824a039620219640",
  Blob (114 bytes) {
    type: "text/javascript;charset=utf-8"
  },
  sourcemap: null
}
```

{% /codetabs %}

### Executables

Bun supports "compiling" a JavaScript/TypeScript entrypoint into a standalone executable. This executable contains a copy of the Bun binary.

```sh
$ bun build ./cli.tsx --outfile mycli --compile
$ ./mycli
```

Refer to [Bundler > Executables](/docs/bundler/executables) for complete documentation.

## Logs and errors

`Bun.build` only throws if invalid options are provided. Read the `success` property to determine if the build was successful; the `logs` property will contain additional details.

```ts
const result = await Bun.build({
  entrypoints: ["./index.tsx"],
  outdir: "./out",
});

if (!result.success) {
  console.error("Build failed");
  for (const message of result.logs) {
    // Bun will pretty print the message object
    console.error(message);
  }
}
```

Each message is either a `BuildMessage` or `ResolveMessage` object, which can be used to trace what problems happened in the build.

```ts
class BuildMessage {
  name: string;
  position?: Position;
  message: string;
  level: "error" | "warning" | "info" | "debug" | "verbose";
}

class ResolveMessage extends BuildMessage {
  code: string;
  referrer: string;
  specifier: string;
  importKind: ImportKind;
}
```

If you want to throw an error from a failed build, consider passing the logs to an `AggregateError`. If uncaught, Bun will pretty-print the contained messages nicely.

```ts
if (!result.success) {
  throw new AggregateError(result.logs, "Build failed");
}
```

## Reference

```ts
interface Bun {
  build(options: BuildOptions): Promise<BuildOutput>;
}

interface BuildOptions {
  entrypoints: string[]; // required
  outdir?: string; // default: no write (in-memory only)
  format?: "esm"; // later: "cjs" | "iife"
  target?: "browser" | "bun" | "node"; // "browser"
  splitting?: boolean; // true
  plugins?: BunPlugin[]; // [] // See https://bun.sh/docs/bundler/plugins
  loader?: { [k in string]: Loader }; // See https://bun.sh/docs/bundler/loaders
  manifest?: boolean; // false
  external?: string[]; // []
  sourcemap?: "none" | "inline" | "external"; // "none"
  root?: string; // computed from entrypoints
  naming?:
    | string
    | {
        entry?: string; // '[dir]/[name].[ext]'
        chunk?: string; // '[name]-[hash].[ext]'
        asset?: string; // '[name]-[hash].[ext]'
      };
  publicPath?: string; // e.g. http://mydomain.com/
  minify?:
    | boolean // false
    | {
        identifiers?: boolean;
        whitespace?: boolean;
        syntax?: boolean;
      };
}

interface BuildOutput {
  outputs: BuildArtifact[];
  success: boolean;
  logs: Array<BuildMessage | ResolveMessage>;
}

interface BuildArtifact extends Blob {
  path: string;
  loader: Loader;
  hash?: string;
  kind: "entry-point" | "chunk" | "asset" | "sourcemap";
  sourcemap?: BuildArtifact;
}

type Loader = "js" | "jsx" | "ts" | "tsx" | "json" | "toml" | "file" | "napi" | "wasm" | "text";

interface BuildOutput {
  outputs: BuildArtifact[];
  success: boolean;
  logs: Array<BuildMessage | ResolveMessage>;
}

declare class ResolveMessage {
  readonly name: "ResolveMessage";
  readonly position: Position | null;
  readonly code: string;
  readonly message: string;
  readonly referrer: string;
  readonly specifier: string;
  readonly importKind:
    | "entry_point"
    | "stmt"
    | "require"
    | "import"
    | "dynamic"
    | "require_resolve"
    | "at"
    | "at_conditional"
    | "url"
    | "internal";
  readonly level: "error" | "warning" | "info" | "debug" | "verbose";

  toString(): string;
}
```

<!--
interface BuildManifest {
  inputs: {
    [path: string]: {
      output: {
        path: string;
      };
      imports: {
        path: string;
        kind: ImportKind;
        external?: boolean;
        asset?: boolean; // whether the import defaulted to "file" loader
      }[];
    };
  };
  outputs: {
    [path: string]: {
      type: "chunk" | "entrypoint" | "asset";
      inputs: { path: string }[];
      imports: {
        path: string;
        kind: ImportKind;
        external?: boolean;
      }[];
      exports: string[];
    };
  };
} -->


{% callout %}
**Note** — Introduced in Bun v0.1.11.
{% /callout %}

Bun provides a universal plugin API that can be used to extend both the _runtime_ and _bundler_.

Plugins intercept imports and perform custom loading logic: reading files, transpiling code, etc. They can be used to add support for additional file types, like `.scss` or `.yaml`. In the context of Bun's bundler, plugins can be used to implement framework-level features like CSS extraction, macros, and client-server code co-location.

## Usage

A plugin is defined as simple JavaScript object containing a `name` property and a `setup` function. Register a plugin with Bun using the `plugin` function.

```tsx#yamlPlugin.ts
import type { BunPlugin } from "bun";

const myPlugin: BunPlugin = {
  name: "YAML loader",
  setup(build) {
    // implementation
  },
};
```

This plugin can be passed into the `plugins` array when calling `Bun.build`.

```ts
Bun.build({
  entrypoints: ["./app.ts"],
  outdir: "./out",
  plugins: [myPlugin],
});
```

<!-- It can also be "registered" with the Bun runtime using the `Bun.plugin()` function. Once registered, the currently executing `bun` process will incorporate the plugin into its module resolution algorithm.

```ts
import {plugin} from "bun";

plugin(myPlugin);
``` -->

## `--preload`

To consume this plugin, add this file to the `preload` option in your [`bunfig.toml`](/docs/runtime/configuration). Bun automatically loads the files/modules specified in `preload` before running a file.

```toml
preload = ["./yamlPlugin.ts"]
```

To preload files during `bun test`:

```toml
[test]
preload = ["./loader.ts"]
```

{% details summary="Usage without preload" %}

Alternatively, you can import this file manually at the top of your project's entrypoint, before any application code is imported.

```ts#app.ts
import "./yamlPlugin.ts";
import { config } from "./config.yml";

console.log(config);
```

{% /details %}

## Third-party plugins

By convention, third-party plugins intended for consumption should export a factory function that accepts some configuration and returns a plugin object.

```ts
import { plugin } from "bun";
import fooPlugin from "bun-plugin-foo";

plugin(
  fooPlugin({
    // configuration
  }),
);

// application code
```

Bun's plugin API is based on [esbuild](https://esbuild.github.io/plugins). Only [a subset](/docs/bundler/vs-esbuild#plugin-api) of the esbuild API is implemented, but some esbuild plugins "just work" in Bun, like the official [MDX loader](https://mdxjs.com/packages/esbuild/):

```jsx
import { plugin } from "bun";
import mdx from "@mdx-js/esbuild";

plugin(mdx());

import { renderToStaticMarkup } from "react-dom/server";
import Foo from "./bar.mdx";
console.log(renderToStaticMarkup(<Foo />));
```

## Loaders

<!-- The plugin logic is implemented in the `setup` function using the builder provided as the first argument (`build` in the example above). The `build` variable provides two methods: `onResolve` and `onLoad`. -->

<!-- ## `onResolve` -->

<!-- The `onResolve` method lets you intercept imports that match a particular regex and modify the resolution behavior, such as re-mapping the import to another file. In the simplest case, you can simply remap the matched import to a new path.

```ts
import { plugin } from "bun";

plugin({
  name: "YAML loader",
  setup(build) {
    build.onResolve();
    // implementation
  },
});
``` -->

<!--
Internally, Bun's transpiler automatically turns `plugin()` calls into separate files (at most 1 per file). This lets loaders activate before the rest of your application runs with zero configuration. -->

Plugins are primarily used to extend Bun with loaders for additional file types. Let's look at a simple plugin that implements a loader for `.yaml` files.

```ts#yamlPlugin.ts
import { plugin } from "bun";

plugin({
  name: "YAML",
  async setup(build) {
    const { load } = await import("js-yaml");
    const { readFileSync } = await import("fs");

    // when a .yaml file is imported...
    build.onLoad({ filter: /\.(yaml|yml)$/ }, (args) => {

      // read and parse the file
      const text = readFileSync(args.path, "utf8");
      const exports = load(text) as Record<string, any>;

      // and returns it as a module
      return {
        exports,
        loader: "object", // special loader for JS objects
      };
    });
  },
});
```

With this plugin, data can be directly imported from `.yaml` files.

{% codetabs %}

```ts#index.ts
import "./yamlPlugin.ts"
import {name, releaseYear} from "./data.yml"

console.log(name, releaseYear);
```

```yaml#data.yml
name: Fast X
releaseYear: 2023
```

{% /codetabs %}

Note that the returned object has a `loader` property. This tells Bun which of its internal loaders should be used to handle the result. Even though we're implementing a loader for `.yaml`, the result must still be understandable by one of Bun's built-in loaders. It's loaders all the way down.

In this case we're using `"object"`—a built-in loader (intended for use by plugins) that converts a plain JavaScript object to an equivalent ES module. Any of Bun's built-in loaders are supported; these same loaders are used by Bun internally for handling files of various kinds. The table below is a quick reference; refer to [Bundler > Loaders](/docs/bundler/loaders) for complete documentation.

{% table %}

- Loader
- Extensions
- Output

---

- `js`
- `.mjs` `.cjs`
- Transpile to JavaScript files

---

- `jsx`
- `.js` `.jsx`
- Transform JSX then transpile

---

- `ts`
- `.ts` `.mts` `cts`
- Transform TypeScript then transpile

---

- `tsx`
- `.tsx`
- Transform TypeScript, JSX, then transpile

---

- `toml`
- `.toml`
- Parse using Bun's built-in TOML parser

---

- `json`
- `.json`
- Parse using Bun's built-in JSON parser

---

- `napi`
- `.node`
- Import a native Node.js addon

---

- `wasm`
- `.wasm`
- Import a native Node.js addon

---

- `object`
- _none_
- A special loader intended for plugins that converts a plain JavaScript object to an equivalent ES module. Each key in the object corresponds to a named export.

{% /callout %}

Loading a YAML file is useful, but plugins support more than just data loading. Let's look at a plugin that lets Bun import `*.svelte` files.

```ts#sveltePlugin.ts
import { plugin } from "bun";

await plugin({
  name: "svelte loader",
  async setup(build) {
    const { compile } = await import("svelte/compiler");
    const { readFileSync } = await import("fs");

    // when a .svelte file is imported...
    build.onLoad({ filter: /\.svelte$/ }, ({ path }) => {

      // read and compile it with the Svelte compiler
      const file = readFileSync(path, "utf8");
      const contents = compile(file, {
        filename: path,
        generate: "ssr",
      }).js.code;

      // and return the compiled source code as "js"
      return {
        contents,
        loader: "js",
      };
    });
  },
});
```

> Note: in a production implementation, you'd want to cache the compiled output and include additional error handling.

The object returned from `build.onLoad` contains the compiled source code in `contents` and specifies `"js"` as its loader. That tells Bun to consider the returned `contents` to be a JavaScript module and transpile it using Bun's built-in `js` loader.

With this plugin, Svelte components can now be directly imported and consumed.

```js
import "./sveltePlugin.ts";
import MySvelteComponent from "./component.svelte";

console.log(mySvelteComponent.render());
```

## Reading `Bun.build`'s config

Plugins can read and write to the [build config](/docs/bundler#api) with `build.config`.

```ts
Bun.build({
  entrypoints: ["./app.ts"],
  outdir: "./dist",
  sourcemap: "external",
  plugins: [
    {
      name: "demo",
      setup(build) {
        console.log(build.config.sourcemap); // "external"

        build.config.minify = true; // enable minification

        // `plugins` is readonly
        console.log(`Number of plugins: ${build.config.plugins.length}`);
      },
    },
  ],
});
```

## Reference

```ts
namespace Bun {
  function plugin(plugin: { name: string; setup: (build: PluginBuilder) => void }): void;
}

type PluginBuilder = {
  onResolve: (
    args: { filter: RegExp; namespace?: string },
    callback: (args: { path: string; importer: string }) => {
      path: string;
      namespace?: string;
    } | void,
  ) => void;
  onLoad: (
    args: { filter: RegExp; namespace?: string },
    callback: (args: { path: string }) => {
      loader?: Loader;
      contents?: string;
      exports?: Record<string, any>;
    },
  ) => void;
  config: BuildConfig;
};

type Loader = "js" | "jsx" | "ts" | "tsx" | "json" | "toml" | "object";
```

The `onLoad` method optionally accepts a `namespace` in addition to the `filter` regex. This namespace will be be used to prefix the import in transpiled code; for instance, a loader with a `filter: /\.yaml$/` and `namespace: "yaml:"` will transform an import from `./myfile.yaml` into `yaml:./myfile.yaml`.


{% callout %}
**Note** — Available in Bun v0.6.0 and later.
{% /callout %}

Bun's bundler API is inspired heavily by [esbuild](https://esbuild.github.io/). Migrating to Bun's bundler from esbuild should be relatively painless. This guide will briefly explain why you might consider migrating to Bun's bundler and provide a side-by-side API comparison reference for those who are already familiar with esbuild's API.

There are a few behavioral differences to note.

- **Bundling by default**. Unlike esbuild, Bun _always bundles by default_. This is why the `--bundle` flag isn't necessary in the Bun example. To transpile each file individually, use [`Bun.Transpiler`](/docs/api/transpiler).
- **It's just a bundler**. Unlike esbuild, Bun's bundler does not include a built-in development server or file watcher. It's just a bundler. The bundler is intended for use in conjunction with `Bun.serve` and other runtime APIs to achieve the same effect. As such, all options relating to HTTP/file watching are not applicable.

## Performance

With an performance-minded API coupled with the extensively optimized Zig-based JS/TS parser, Bun's bundler is 1.75x faster than esbuild on esbuild's [three.js benchmark](https://github.com/oven-sh/bun/tree/main/bench/bundle).

{% image src="/images/bundler-speed.png" caption="Bundling 10 copies of three.js from scratch, with sourcemaps and minification" /%}

## CLI API

Bun and esbuild both provide a command-line interface.

```bash
$ esbuild <entrypoint> --outdir=out --bundle
$ bun build <entrypoint> --outdir=out
```

In Bun's CLI, simple boolean flags like `--minify` do not accept an argument. Other flags like `--outdir <path>` do accept an argument; these flags can be written as `--outdir out` or `--outdir=out`. Some flags like `--define` can be specified several times: `--define foo=bar --define bar=baz`.

{% table %}

- `esbuild`
- `bun build`

---

- `--bundle`
- n/a
- Bun always bundles, use `--no-bundle` to disable this behavior.

---

- `--define:K=V`
- `--define K=V`
- Small syntax difference; no colon.

  ```bash
  $ esbuild --define:foo=bar
  $ bun build --define foo=bar
  ```

---

- `--external:<pkg>`
- `--external <pkg>`
- Small syntax difference; no colon.

  ```bash
  $ esbuild --external:react
  $ bun build --external react
  ```

---

- `--format`
- `--format`
- Bun only supports `"esm"` currently but other module formats are planned. esbuild defaults to `"iife"`.

---

- `--loader:.ext=loader`
- `--loader .ext:loader`
- Bun supports a different set of built-in loaders than esbuild; see [Bundler > Loaders](/docs/bundler/loaders) for a complete reference. The esbuild loaders `dataurl`, `binary`, `base64`, `copy`, and `empty` are not yet implemented.

  The syntax for `--loader` is slightly different.

  ```bash
  $ esbuild app.ts --bundle --loader:.svg=text
  $ bun build app.ts --loader .svg:text
  ```

---

- `--minify`
- `--minify`
- No differences

---

- `--outdir`
- `--outdir`
- No differences

---

- `--outfile`
- `--outfile`

---

- `--packages`
- n/a
- Not supported

---

- `--platform`
- `--target`
- Renamed to `--target` for consistency with tsconfig. Does not support `neutral`.

---

- `--serve`
- n/a
- Not applicable

---

- `--sourcemap`
- `--sourcemap`
- No differences

---

- `--splitting`
- `--splitting`
- No differences

---

- `--target`
- n/a
- No supported. Bun's bundler performs no syntactic downleveling at this time.

---

- `--watch`
- n/a
- Not applicable

---

- `--allow-overwrite`
- n/a
- Overwriting is never allowed

---

- `--analyze`
- n/a
- Not supported

---

- `--asset-names`
- `--asset-naming`
- Renamed for consistency with `naming` in JS API

---

- `--banner`
- n/a
- Not supported

---

- `--certfile`
- n/a
- Not applicable

---

- `--charset=utf8`
- n/a
- Not supported

---

- `--chunk-names`
- `--chunk-naming`
- Renamed for consistency with `naming` in JS API

---

- `--color`
- n/a
- Always enabled

---

- `--drop`
- n/a
- Not supported

---

- `--entry-names`
- `--entry-naming`
- Renamed for consistency with `naming` in JS API

---

- `--footer`
- n/a
- Not supported

---

- `--global-name`
- n/a
- Not applicable, Bun does not support `iife` output at this time

---

- `--ignore-annotations`
- n/a
- Not supported

---

- `--inject`
- n/a
- Not supported

---

- `--jsx`
- `--jsx-runtime <runtime>`
- Supports `"automatic"` (uses `jsx` transform) and `"classic"` (uses `React.createElement`)

---

- `--jsx-dev`
- n/a
- Bun reads `compilerOptions.jsx` from `tsconfig.json` to determine a default. If `compilerOptions.jsx` is `"react-jsx"`, or if `NODE_ENV=production`, Bun will use the `jsx` transform. Otherwise, it uses `jsxDEV`. For any to Bun uses `jsxDEV`. The bundler does not support `preserve`.

---

- `--jsx-factory`
- `--jsx-factory`

---

- `--jsx-fragment`
- `--jsx-fragment`

---

- `--jsx-import-source`
- `--jsx-import-source`

---

- `--jsx-side-effects`
- n/a
- JSX is always assumed to be side-effect-free

---

- `--keep-names`
- n/a
- Not supported

---

- `--keyfile`
- n/a
- Not applicable

---

- `--legal-comments`
- n/a
- Not supported

---

- `--log-level`
- n/a
- Not supported. This can be set in `bunfig.toml` as `logLevel`.

---

- `--log-limit`
- n/a
- Not supported

---

- `--log-override:X=Y`
- n/a
- Not supported

---

- `--main-fields`
- n/a
- Not supported

---

- `--mangle-cache`
- n/a
- Not supported

---

- `--mangle-props`
- n/a
- Not supported

---

- `--mangle-quoted`
- n/a
- Not supported

---

- `--metafile`
- n/a
- Not supported

---

- `--minify-whitespace`
- `--minify-whitespace`

---

- `--minify-identifiers`
- `--minify-identifiers`

---

- `--minify-syntax`
- `--minify-syntax`

---

- `--out-extension`
- n/a
- Not supported

---

- `--outbase`
- `--root`

---

- `--preserve-symlinks`
- n/a
- Not supported

---

- `--public-path`
- `--public-path`

---

- `--pure`
- n/a
- Not supported

---

- `--reserve-props`
- n/a
- Not supported

---

- `--resolve-extensions`
- n/a
- Not supported

---

- `--servedir`
- n/a
- Not applicable

---

- `--source-root`
- n/a
- Not supported

---

- `--sourcefile`
- n/a
- Not supported. Bun does not support `stdin` input yet.

---

- `--sourcemap`
- `--sourcemap`
- No differences

---

- `--sources-content`
- n/a
- Not supported

---

- `--supported`
- n/a
- Not supported

---

- `--tree-shaking`
- n/a
- Always `true`

---

- `--tsconfig`
- `--tsconfig-override`

---

- `--version`
- n/a
- Run `bun --version` to see the version of Bun.

{% /table %}

## JavaScript API

{% table %}

- `esbuild.build()`
- `Bun.build()`

---

- `absWorkingDir`
- n/a
- Always set to `process.cwd()`

---

- `alias`
- n/a
- Not supported

---

- `allowOverwrite`
- n/a
- Always `false`

---

- `assetNames`
- `naming.asset`
- Uses same templating syntax as esbuild, but `[ext]` must be included explicitly.

  ```ts
  Bun.build({
    entrypoints: ["./index.tsx"],
    naming: {
      asset: "[name].[ext]",
    },
  });
  ```

---

- `banner`
- n/a
- Not supported

---

- `bundle`
- n/a
- Always `true`. Use [`Bun.Transpiler`](/docs/api/transpiler) to transpile without bundling.

---

- `charset`
- n/a
- Not supported

---

- `chunkNames`
- `naming.chunk`
- Uses same templating syntax as esbuild, but `[ext]` must be included explicitly.

  ```ts
  Bun.build({
    entrypoints: ["./index.tsx"],
    naming: {
      chunk: "[name].[ext]",
    },
  });
  ```

---

- `color`
- n/a
- Bun returns logs in the `logs` property of the build result.

---

- `conditions`
- n/a
- Not supported. Export conditions priority is determined by `target`.

---

- `define`
- `define`

---

- `drop`
- n/a
- Not supported

---

- `entryNames`
- `naming` or `naming.entry`
- Bun supports a `naming` key that can either be a string or an object. Uses same templating syntax as esbuild, but `[ext]` must be included explicitly.

  ```ts
  Bun.build({
    entrypoints: ["./index.tsx"],
    // when string, this is equivalent to entryNames
    naming: "[name].[ext]",

    // granular naming options
    naming: {
      entry: "[name].[ext]",
      asset: "[name].[ext]",
      chunk: "[name].[ext]",
    },
  });
  ```

---

- `entryPoints`
- `entrypoints`
- Capitalization difference

---

- `external`
- `external`
- No differences

---

- `footer`
- n/a
- Not supported

---

- `format`
- `format`
- Only supports `"esm"` currently. Support for `"cjs"` and `"iife"` is planned.

---

- `globalName`
- n/a
- Not supported

---

- `ignoreAnnotations`
- n/a
- Not supported

---

- `inject`
- n/a
- Not supported

---

- `jsx`
- `jsx`
- Not supported in JS API, configure in `tsconfig.json`

---

- `jsxDev`
- `jsxDev`
- Not supported in JS API, configure in `tsconfig.json`

---

- `jsxFactory`
- `jsxFactory`
- Not supported in JS API, configure in `tsconfig.json`

---

- `jsxFragment`
- `jsxFragment`
- Not supported in JS API, configure in `tsconfig.json`

---

- `jsxImportSource`
- `jsxImportSource`
- Not supported in JS API, configure in `tsconfig.json`

---

- `jsxSideEffects`
- `jsxSideEffects`
- Not supported in JS API, configure in `tsconfig.json`

---

- `keepNames`
- n/a
- Not supported

---

- `legalComments`
- n/a
- Not supported

---

- `loader`
- `loader`
- Bun supports a different set of built-in loaders than esbuild; see [Bundler > Loaders](/docs/bundler/loaders) for a complete reference. The esbuild loaders `dataurl`, `binary`, `base64`, `copy`, and `empty` are not yet implemented.

---

- `logLevel`
- n/a
- Not supported

---

- `logLimit`
- n/a
- Not supported

---

- `logOverride`
- n/a
- Not supported

---

- `mainFields`
- n/a
- Not supported

---

- `mangleCache`
- n/a
- Not supported

---

- `mangleProps`
- n/a
- Not supported

---

- `mangleQuoted`
- n/a
- Not supported

---

- `metafile`
- n/a
- Not supported

<!-- - `manifest`
- When `manifest` is `true`, the result of `Bun.build()` will contain a `manifest` property. The manifest is compatible with esbuild's metafile format. -->

---

- `minify`
- `minify`
- In Bun, `minify` can be a boolean or an object.

  ```ts
  Bun.build({
    entrypoints: ['./index.tsx'],
    // enable all minification
    minify: true

    // granular options
    minify: {
      identifiers: true,
      syntax: true,
      whitespace: true
    }
  })
  ```

---

- `minifyIdentifiers`
- `minify.identifiers`
- See `minify`

---

- `minifySyntax`
- `minify.syntax`
- See `minify`

---

- `minifyWhitespace`
- `minify.whitespace`
- See `minify`

---

- `nodePaths`
- n/a
- Not supported

---

- `outExtension`
- n/a
- Not supported

---

- `outbase`
- `root`
- Different name

---

- `outdir`
- `outdir`
- No differences

---

- `outfile`
- `outfile`
- No differences

---

- `packages`
- n/a
- Not supported, use `external`

---

- `platform`
- `target`
- Supports `"bun"`, `"node"` and `"browser"` (the default). Does not support `"neutral"`.

---

- `plugins`
- `plugins`
- Bun's plugin API is a subset of esbuild's. Some esbuild plugins will work out of the box with Bun.

---

- `preserveSymlinks`
- n/a
- Not supported

---

- `publicPath`
- `publicPath`
- No differences

---

- `pure`
- n/a
- Not supported

---

- `reserveProps`
- n/a
- Not supported

---

- `resolveExtensions`
- n/a
- Not supported

---

- `sourceRoot`
- n/a
- Not supported

---

- `sourcemap`
- `sourcemap`
- Supports `"inline"`, `"external"`, and `"none"`

---

- `sourcesContent`
- n/a
- Not supported

---

- `splitting`
- `splitting`
- No differences

---

- `stdin`
- n/a
- Not supported

---

- `supported`
- n/a
- Not supported

---

- `target`
- n/a
- No support for syntax downleveling

---

- `treeShaking`
- n/a
- Always `true`

---

- `tsconfig`
- n/a
- Not supported

---

- `write`
- n/a
- Set to `true` if `outdir`/`outfile` is set, otherwise `false`

---

{% /table %}

## Plugin API

Bun's plugin API is designed to be esbuild compatible. Bun doesn't support esbuild's entire plugin API surface, but the core functionality is implemented. Many third-party `esbuild` plugins will work out of the box with Bun.

{% callout %}
Long term, we aim for feature parity with esbuild's API, so if something doesn't work please file an issue to help us prioritize.

{% /callout %}

Plugins in Bun and esbuild are defined with a `builder` object.

```ts
import type { BunPlugin } from "bun";

const myPlugin: BunPlugin = {
  name: "my-plugin",
  setup(builder) {
    // define plugin
  },
};
```

The `builder` object provides some methods for hooking into parts of the bundling process. Bun implements `onResolve` and `onLoad`; it does not yet implement the esbuild hooks `onStart`, `onEnd`, and `onDispose`, and `resolve` utilities. `initialOptions` is partially implemented, being read-only and only having a subset of esbuild's options; use [`config`](/docs/bundler/plugins#reading-bunbuilds-config) (same thing but with Bun's `BuildConfig` format) instead.

```ts
import type { BunPlugin } from "bun";
const myPlugin: BunPlugin = {
  name: "my-plugin",
  setup(builder) {
    builder.onResolve(
      {
        /* onResolve.options */
      },
      args => {
        return {
          /* onResolve.results */
        };
      },
    );
    builder.onLoad(
      {
        /* onLoad.options */
      },
      args => {
        return {
          /* onLoad.results */
        };
      },
    );
  },
};
```

### `onResolve`

#### `options`

{% table %}

- 🟢
- `filter`

---

- 🟢
- `namespace`

{% /table %}

#### `arguments`

{% table %}

- 🟢
- `path`

---

- 🟢
- `importer`

---

- 🔴
- `namespace`

---

- 🔴
- `resolveDir`

---

- 🔴
- `kind`

---

- 🔴
- `pluginData`

{% /table %}

#### `results`

{% table %}

- 🟢
- `namespace`

---

- 🟢
- `path`

---

- 🔴
- `errors`

---

- 🔴
- `external`

---

- 🔴
- `pluginData`

---

- 🔴
- `pluginName`

---

- 🔴
- `sideEffects`

---

- 🔴
- `suffix`

---

- 🔴
- `warnings`

---

- 🔴
- `watchDirs`

---

- 🔴
- `watchFiles`

{% /table %}

### `onLoad`

#### `options`

{% table %}

---

- 🟢
- `filter`

---

- 🟢
- `namespace`

{% /table %}

#### `arguments`

{% table %}

---

- 🟢
- `path`

---

- 🔴
- `namespace`

---

- 🔴
- `suffix`

---

- 🔴
- `pluginData`

{% /table %}

#### `results`

{% table %}

---

- 🟢
- `contents`

---

- 🟢
- `loader`

---

- 🔴
- `errors`

---

- 🔴
- `pluginData`

---

- 🔴
- `pluginName`

---

- 🔴
- `resolveDir`

---

- 🔴
- `warnings`

---

- 🔴
- `watchDirs`

---

- 🔴
- `watchFiles`

{% /table %}


The Bun bundler implements a set of default loaders out of the box. As a rule of thumb, the bundler and the runtime both support the same set of file types out of the box.

`.js` `.cjs` `.mjs` `.mts` `.cts` `.ts` `.tsx` `.jsx` `.toml` `.json` `.txt` `.wasm` `.node`

Bun uses the file extension to determine which built-in _loader_ should be used to parse the file. Every loader has a name, such as `js`, `tsx`, or `json`. These names are used when building [plugins](/docs/bundler/plugins) that extend Bun with custom loaders.

## Built-in loaders

### `js`

**JavaScript**. Default for `.cjs` and `.mjs`.

Parses the code and applies a set if default transforms, like dead-code elimination, tree shaking, and environment variable inlining. Note that Bun does not attempt to down-convert syntax at the moment.

### `jsx`

**JavaScript + JSX.**. Default for `.js` and `.jsx`.

Same as the `js` loader, but JSX syntax is supported. By default, JSX is downconverted to plain JavaScript; the details of how this is done depends on the `jsx*` compiler options in your `tsconfig.json`. Refer to the TypeScript documentation [on JSX](https://www.typescriptlang.org/docs/handbook/jsx.html) for more information.

### `ts`

**TypeScript loader**. Default for `.ts`, `.mts`, and `.cts`.

Strips out all TypeScript syntax, then behaves identically to the `js` loader. Bun does not perform typechecking.

### `tsx`

**TypeScript + JSX loader**. Default for `.tsx`. Transpiles both TypeScript and JSX to vanilla JavaScript.

### `json`

**JSON loader**. Default for `.json`.

JSON files can be directly imported.

```ts
import pkg from "./package.json";
pkg.name; // => "my-package"
```

During bundling, the parsed JSON is inlined into the bundle as a JavaScript object.

```ts
var pkg = {
  name: "my-package",
  // ... other fields
};
pkg.name;
```

If a `.json` file is passed as an entrypoint to the bundler, it will be converted to a `.js` module that `export default`s the parsed object.

{% codetabs %}

```json#Input
{
  "name": "John Doe",
  "age": 35,
  "email": "johndoe@example.com"
}
```

```js#Output
export default {
  name: "John Doe",
  age: 35,
  email: "johndoe@example.com"
}
```

{% /codetabs %}

### `toml`

**TOML loader**. Default for `.toml`.

TOML files can be directly imported. Bun will parse them with its fast native TOML parser.

```ts
import config from "./bunfig.toml";
config.logLevel; // => "debug"
```

During bundling, the parsed TOML is inlined into the bundle as a JavaScript object.

```ts
var config = {
  logLevel: "debug",
  // ...other fields
};
config.logLevel;
```

If a `.toml` file is passed as an entrypoint, it will be converted to a `.js` module that `export default`s the parsed object.

{% codetabs %}

```toml#Input
name = "John Doe"
age = 35
email = "johndoe@example.com"
```

```js#Output
export default {
  name: "John Doe",
  age: 35,
  email: "johndoe@example.com"
}
```

{% /codetabs %}

### `text`

**Text loader**. Default for `.txt`.

The contents of the text file are read and inlined into the bundle as a string.
Text files can be directly imported. The file is read and returned as a string.

```ts
import contents from "./file.txt";
console.log(contents); // => "Hello, world!"
```

When referenced during a build, the contents are into the bundle as a string.

```ts
var contents = `Hello, world!`;
console.log(contents);
```

If a `.txt` file is passed as an entrypoint, it will be converted to a `.js` module that `export default`s the file contents.

{% codetabs %}

```txt#Input
Hello, world!
```

```js#Output
export default "Hello, world!";
```

{% /codetabs %}

### `wasm`

**WebAssembly loader**. Default for `.wasm`.

In the runtime, WebAssembly files can be directly imported. The file is read and returned as a `WebAssembly.Module`.

```ts
import wasm from "./module.wasm";
console.log(wasm); // => WebAssembly.Module
```

In the bundler, `.wasm` files are handled using the [`file`](#file) loader.

### `napi`

**Native addon loader**. Default for `.node`.

In the runtime, native addons can be directly imported.

```ts
import addon from "./addon.node";
console.log(addon);
```

In the bundler, `.node` files are handled using the [`file`](#file) loader.

### `file`

**File loader**. Default for all unrecognized file types.

The file loader resolves the import as a _path/URL_ to the imported file. It's commonly used for referencing media or font assets.

```ts#logo.ts
import logo from "./logo.svg";
console.log(logo);
```

_In the runtime_, Bun checks that the `logo.svg` file exists and converts it to an absolute path to the location of `logo.svg` on disk.

```bash
$ bun run logo.ts
/path/to/project/logo.svg
```

_In the bundler_, things are slightly different. The file is copied into `outdir` as-is, and the import is resolved as a relative path pointing to the copied file.

```ts#Output
var logo = "./logo.svg";
console.log(logo);
```

If a value is specified for `publicPath`, the import will use value as a prefix to construct an absolute path/URL.

{% table %}

- Public path
- Resolved import

---

- `""` (default)
- `/logo.svg`

---

- `"/assets"`
- `/assets/logo.svg`

---

- `"https://cdn.example.com/"`
- `https://cdn.example.com/logo.svg`

{% /table %}

{% callout %}
The location and file name of the copied file is determined by the value of [`naming.asset`](/docs/bundler#naming).
{% /callout %}
This loader is copied into the `outdir` as-is. The name of the copied file is determined using the value of `naming.asset`.

{% details summary="Fixing TypeScript import errors" %}
If you're using TypeScript, you may get an error like this:

```ts
// TypeScript error
// Cannot find module './logo.svg' or its corresponding type declarations.
```

This can be fixed by creating `*.d.ts` file anywhere in your project (any name will work) with the following contents:

```ts
declare module "*.svg" {
  const content: string;
  export default content;
}
```

This tells TypeScript that any default imports from `.svg` should be treated as a string.
{% /details %}


