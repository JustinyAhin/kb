# Tailwind IntelliSense diagnostics

Zed's Tailwind warnings come from `@tailwindcss/language-server`. They are
separate from ESLint, `svelte-check`, and the Tailwind build step.

For example:

```txt
The class `h-[66px]` can be written as `h-16.5`
```

This is the `tailwindCSS.lint.suggestCanonicalClasses` diagnostic.

## Focused project check

Pin `@tailwindcss/language-server` as a dev dependency and expose a focused
script, for example:

```json
{
  "scripts": {
    "lint:tailwind": "node scripts/lint-tailwind.mjs"
  }
}
```

The check should scan project-owned Svelte files and exclude generated
shadcn-svelte primitives such as `src/lib/components/ui/**`.

Run it with:

```sh
bun run lint:tailwind
```

Include it in the normal `bun check` and `bun lint` workflow when the project
needs these diagnostics enforced in CI.

## Running the same check outside Zed

Zed's bundled language server is usually at:

```sh
~/Library/Application\ Support/Zed/languages/tailwindcss-language-server/node_modules/.bin/tailwindcss-language-server
```

If that path does not exist, use:

```sh
bunx @tailwindcss/language-server --stdio
```

`bunx` may print package-resolution noise before the LSP stream, so prefer the
pinned project dependency or Zed's installed binary for automation.

The following probe opens one CSS entrypoint and one Svelte file, then prints
the diagnostics published by the Tailwind language server. Save it as
`scripts/lint-tailwind.mjs` and pass a file path as the first argument when
needed.

```js
import { spawn } from 'node:child_process';
import fs from 'node:fs';
import path from 'node:path';
import { pathToFileURL } from 'node:url';

const root = process.cwd();
const server =
	process.env.TAILWIND_LS ||
	path.join(
		process.env.HOME,
		'Library/Application Support/Zed/languages/tailwindcss-language-server/node_modules/.bin/tailwindcss-language-server'
	);
const target = path.join(root, process.argv[2] || 'src/lib/components/layout/header.svelte');
const css = path.join(root, 'src/routes/layout.css');
const rootUri = pathToFileURL(root).href;
const targetUri = pathToFileURL(target).href;
const cssUri = pathToFileURL(css).href;
const child = spawn(server, ['--stdio'], { cwd: root, stdio: ['pipe', 'pipe', 'inherit'] });

let id = 1;
let buffer = Buffer.alloc(0);
let diagnostics = [];

const send = (message) => {
	const body = JSON.stringify(message);
	child.stdin.write(`Content-Length: ${Buffer.byteLength(body)}\r\n\r\n${body}`);
};

const request = (method, params) => send({ jsonrpc: '2.0', id: id++, method, params });
const notify = (method, params) => send({ jsonrpc: '2.0', method, params });
const respond = (requestId, result) => send({ jsonrpc: '2.0', id: requestId, result });

const settings = {
	tailwindCSS: {
		validate: true,
		includeLanguages: { svelte: 'html' },
		classAttributes: ['class', 'className', 'ngClass', 'class:list'],
		lint: { suggestCanonicalClasses: 'warning' },
		experimental: { configFile: { 'src/routes/layout.css': 'src/**/*' } }
	}
};

const openDocuments = () => {
	notify('initialized', {});
	notify('workspace/didChangeConfiguration', { settings });
	notify('textDocument/didOpen', {
		textDocument: { uri: cssUri, languageId: 'tailwindcss', version: 1, text: fs.readFileSync(css, 'utf8') }
	});
	notify('textDocument/didOpen', {
		textDocument: { uri: targetUri, languageId: 'svelte', version: 1, text: fs.readFileSync(target, 'utf8') }
	});
};

const handle = (message) => {
	if (message.method === 'textDocument/publishDiagnostics' && message.params.uri === targetUri) {
		diagnostics = message.params.diagnostics;
	}
	if (message.method === 'workspace/configuration') {
		respond(
			message.id,
			message.params.items.map((item) =>
				item.section ? item.section.split('.').reduce((value, key) => value?.[key], settings) : settings
			)
		);
	}
	if (message.method === 'workspace/workspaceFolders') {
		respond(message.id, [{ uri: rootUri, name: path.basename(root) }]);
	}
	if (message.id === 1) openDocuments();
};

child.stdout.on('data', (chunk) => {
	buffer = Buffer.concat([buffer, chunk]);
	for (;;) {
		const headerEnd = buffer.indexOf('\r\n\r\n');
		if (headerEnd === -1) return;
		const match = buffer.slice(0, headerEnd).toString().match(/Content-Length: (\d+)/i);
		const length = Number(match?.[1]);
		const start = headerEnd + 4;
		const end = start + length;
		if (!length || buffer.length < end) return;
		handle(JSON.parse(buffer.slice(start, end).toString()));
		buffer = buffer.slice(end);
	}
});

request('initialize', {
	processId: process.pid,
	rootUri,
	workspaceFolders: [{ uri: rootUri, name: path.basename(root) }],
	capabilities: { workspace: { configuration: true, workspaceFolders: true }, textDocument: { publishDiagnostics: {} } }
});

setTimeout(() => {
	for (const diagnostic of diagnostics) console.log(`${diagnostic.code}: ${diagnostic.message}`);
	child.kill();
}, 12000);
```

