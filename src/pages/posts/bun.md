---
layout: ../../layouts/Layout.astro
title: 'A Permissions System for the Bun JavaScript Runtime'
pubDate: 2025-07-31
description: 'A project adding file system and process spawning permissions to Bun.'
author: 'me'
image:
  url: 'https://bun.sh/logo.svg'
  alt: 'Bun logo'
tags: ["javascript", "bun", "open source", "security"]
---
# A Permissions System for the Bun JavaScript Runtime

Published on: 2025-05-08

This spring I developed a permissions system for the Bun JavaScript Runtime. It is similar to what is found in Deno and Node, but more limited in scope because of time constraints. To develop the system, I had to familiarize myself with the large Zig/C++/JS codebase of Bun. The project earned me an A grade in my Language-Based Security course at KTH.

See the code [here](https://github.com/haval0/bun/tree/permissions).

## Background

### Security
JavaScript is one of the most widely used programming languages in the world, but struggles with inherent security issues, for example relating to type safety, prototype pollution, or injection attacks. When someone wants to run some JavaScript on their computer, they can probably trust that they'll be safe if they wrote all of that code themselves. But that is rarely the case. There's not only the case of needing to use external libraries that may or may not be secure, but it is also increasingly popular to run dynamic workloads sent to you by your users on your server (e.g. edge functions). This necessitates protective measures.

The safest would be to run each workload in a separate VM, but not all developers will do that. As such, other JavaScript runtimes like Deno and Node provide permissions systems that allow restricting the capabilities of running scripts. This can for example mean only allowing access to certain directories, or only allowing access to certain executables (e.g. allowing only `git`). This project explored adding these capabilities to Bun, a relatively new and fast-growing JavaScript runtime.

### Bun
Bun is a JavaScript runtime written mostly in Zig, and released their v1.0 in 2023. Under the hood they use the JavaScriptCore engine from the WebKit project, which is part of the reason for their improved performance compared to Deno and Node. They also aim to add a complete JavaScript development toolkit on top of this, including a package manager, bundler, and test runner, with ambitious future plans.

### The Plan

1. Implement a prototype permission model covering Bun's native file system API and their reimplementation of `node:fs`. 
2. Extend the permission model to Bun's native process spawning API and their reimplementation of `node:child_process`.

These two steps should cover all of Bun's API surface for file system access and process spawning, thereby enabling a user to protect their system better when running untrusted code in Bun.

### The Work

To add the permission system to Bun, I first had to understand how Bun is configured through command line flags. Each CLI argument has a separate declaration, which also states the help message for the flag. I added the flags `--allow-fs` and `--allow-run`, both taking a list of strings as input. For example, `bun run --allow-fs "/home/folder" script.js` should only allow access to files inside the system path `/home/folder`.

The next step was to read the values passed to the flags during initialization, and store them somewhere during runtime. I decided to store them in a `permissions` struct under Bun's `VirtualMachine` struct, which would let me easily access the configured permissions from the backend implementation of every JavaScript API function (like `Bun.file()`). Importantly, this location is out of reach of the running script itself, which means it cannot modify its own permissions (preventing privilege escalation).

Then came the tedious and error-prone task of identifying every relevant API entrypoint in Bun's Zig backend. I had to read a lot of Bun's code before I felt confident enough that I understood which code locations were appropriate for doing the permissions checks. I had to make sure every single function that had some kind of file system or process spawning access would have to pass my permissions checks. Luckily, the Bun architecture is somewhat unified on these points. It would however be much more laborious if I were to split the file system permissions into separate read and write permissions.

I also had to make sure all paths are resolved to absolute paths before comparing them. E.g. if the user has allowed access to the absolute path `/home/folder`, and the script is attempting to access a relative path (e.g. `../file.txt`), then my implementation uses one of Bun's existing utility functions to check which folder the process is executing from, to be able to tell if `../file.txt` is inside the allowed path or not.

Lastly, since permissions are tied to a `VirtualMachine` instance, I had to make sure the script could not somehow escape its own `VirtualMachine`, thereby bypassing the permissions. The two ways this could have happened is (1) by spawning a new Bun process, which is already preventable with my process spawning permissions system, and (2) by spawning a Web Worker, which spawns a script in a new thread and new `VirtualMachine` context. To prevent the second escape, I therefore had to add a fix to copy permissions over from the parent `VirtualMachine` to the new `VirtualMachine` of child workers every time one is created.

### Results

Example of successful operation:

```js
// spawnTest.js
const proc = Bun.spawn(["echo", "hello"]);
const text = await new Response(proc.stdout).text();
console.log(text);
```
Allowed executable, script runs as expected:

```sh
$ bun run --allow-run "/usr/bin/echo" spawnTest.js
hello
```

Disallowed executable, script errors as expected:

```sh
$ bun run --allow-run " " spawnTest.js
1 | const proc = Bun.spawn(["echo", "hello"]);
                     ^
error: No permission to run command /usr/bin/echo
      at /home/group25/dev/bun/test/permissions/mySpawnPerm.js:1:18
```

And here's a comparison table with Deno and Node:

| **Feature** | **Ours (Bun)** | **Deno** | **Node** |
|-------------|-----------------|----------|----------|
| **Deny by default** | ❌ | ✅ (prompt if used) | ❌ |
| **File System Perms** | ✅ | ✅ | ✅ |
| Separate R/W flags | ❌ | ✅ | ✅ |
| Whitelisting | ✅ | ✅ | ✅ |
| Blacklisting | ❌ | ✅ | ❌ |
| Wildcard patterns | ❌ | ❌ | ✅ |
| **Subprocess Perms** | ✅ | ✅ | ✅ (block subproc.) |
| Whitelisting | ✅ | ✅ | ❌ |
| **Worker Perms** | ✅ (always inherit) | ✅ (inherit/custom) | ✅ (block workers) |
| **Query Perms in JS** | ❌ | ✅ | ✅ |

Note that Deno offers permissions configuration of many other features as well. None of those are listed above since neither Node nor our implementation attempts any of them yet.

### Conclusion

I was quite happy with how far I was able to take this project, coming into it knowing nothing about Bun at all. This project was done for the Language-Based Security course at KTH, where it received an A grade. 
