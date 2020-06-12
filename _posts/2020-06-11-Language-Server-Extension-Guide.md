---
layout: post
title: VSCode LSP Extension 指南
date: 2020-06-11 18:11:04
author: admin
comments: true
categories: [VSCode]
tags: [VSCode, LSP]
---

<!-- more -->

---

* 目录
{:toc}
---

## 背景

现有的IDE 在编写一些成熟、流行的语言时，是非常顺手和流畅的，因为它们支持很多语言功能，比如语法高亮、自动补全、语法检测、自动跳转等。 但当你自己手写了一种新的编程语言，现有的IDE可能就无法提供那么好的支持了，这个时候你就需要一个自己的语言服务（Language Server，下面简称LS）。

VSCode 提供了灵活的拓展功能，可以针对新的语言开发出新的拓展来支持它们。今天来学习一下。

## LSP

一图胜千言，“有事找中间人”，可屏蔽各种细节，降低重复工作。

[![](/images/posts/lsp-languages-editors.png)](/images/posts/lsp-languages-editors.png)



## 实现LS

### 0. 概述

在VS Code中，语言服务器有两部分:

- 语言客户端:用JavaScript / TypeScript编写的普通VS代码扩展。
- 语言服务器:在独立进程中运行的语言分析工具。

在单独的进程中运行语言服务器有两个好处:

- 分析工具可以用任何语言实现，只要它能够按照语言服务器协议与语言客户机通信。
- 由于语言分析工具通常占用大量CPU和内存，因此在单独的进程中运行它们可以避免性能开销。

### 1. 准备

VSCode 官方准备好了Demo，可以下载下来使用：[lsp-sample](https://github.com/Microsoft/vscode-extension-samples/tree/master/lsp-sample)。

```shell
> git clone https://github.com/Microsoft/vscode-extension-samples.git
> cd vscode-extension-samples/lsp-sample
> npm install
> npm run compile
```

用VSCode打开这个目录，可以看到整体文件结构如下：

```
.
├── client // 语言客户端
│   ├── src
│   │   ├── test // 语言客户端到语言服务端的 end-to-end 测试
│   │   └── extension.ts // 语言客户端入口
├── package.json // 扩展清单
└── server // 语言服务端
    └── src
        └── server.ts // 语言服务端入口
```

### 2. 解释“语言客户端”

让我们首先看一下根目录下的`package.json`，它描述了语言客户端的功能。其中有两个有趣的部分。

1. activationEvents：

```json
"activationEvents": [
    "onLanguage:plaintext"
]
```

这个部分告诉VSCode 在打开纯文本文件(例如带有扩展名.txt的文件)时立即激活这个extension。

2. configuration：

```json
"configuration": {
    "type": "object",
    "title": "Example configuration",
    "properties": {
        "languageServerExample.maxNumberOfProblems": {
            "scope": "resource",
            "type": "number",
            "default": 100,
            "description": "Controls the maximum number of problems produced by the server."
        }
    }
}
```

这个部分为VSCode 提供配置设置。这些设置将会在语言服务器启动时和每次更改设置时发送到语言服务器。



现在来看`/client`文件夹中语言客户端实际的代码和相应的`package.json`。`/client/package.json`中有趣的部分是它在`engines`字段里添加了一个 `vscode` 扩展托管API， 以及在 `dependencies` 字段里引入了`vscode-languageclient`库：

```json
"engines": {
    "vscode": "^1.1.18"
},
"dependencies": {
    "vscode-languageclient": "^4.1.4"
}
```

客户端被实现为普通的VS Code扩展，并且可以访问所有VS Code命名空间API。

以下是相应的extension.ts文件的内容，该文件是lsp-sample extension的入口：

```typescript
import * as path from 'path';
import { workspace, ExtensionContext } from 'vscode';

import {
	LanguageClient,
	LanguageClientOptions,
	ServerOptions,
	TransportKind
} from 'vscode-languageclient';

let client: LanguageClient;

export function activate(context: ExtensionContext) {
	// server是Node.js实现的
	let serverModule = context.asAbsolutePath(
		path.join('server', 'out', 'server.js')
	);
	// server默认的debug选项
	// --inspect=6009: 在Node的Inspector模式下运行服务器，以便VS Code可以附加到服务器上进行调试
	let debugOptions = { execArgv: ['--nolazy', '--inspect=6009'] };

	// 如果扩展是在调试模式下启动的，则使用调试服务器选项
	// 否则，将使用运行选项
	let serverOptions: ServerOptions = {
		run: { module: serverModule, transport: TransportKind.ipc },
		debug: {
			module: serverModule,
			transport: TransportKind.ipc,
			options: debugOptions
		}
	};

	// 控制语言客户端的选项
	let clientOptions: LanguageClientOptions = {
		// 在服务器上注册纯文本文档
		documentSelector: [{ scheme: 'file', language: 'plaintext' }],
		synchronize: {
      // 通知服务器有关工作区中包含的“ .clientrc”文件的文件更改
			fileEvents: workspace.createFileSystemWatcher('**/.clientrc')
		}
	};

	// 创建语言客户端，然后启动客户端。
	client = new LanguageClient(
		'languageServerExample',
		'Language Server Example',
		serverOptions,
		clientOptions
	);

	// 启动客户端。 这也将启动服务器
	client.start();
}

export function deactivate(): Thenable<void> | undefined {
	if (!client) {
		return undefined;
	}
	return client.stop();
}
```

### 3. 解释语言服务器

在这个示例中，服务器用TypeScript实现，并使用Node.js执行。因为VSCode 已经附带了一个Node.js runtime，所以没有必要提供你自己的，除非你有非常具体的要求runtime。

语言服务器的源代码在`/server`。`/server/package.json`文件中有趣的部分是:

```json
"dependencies": {
    "vscode-languageserver": "^4.1.3"
}
```

这将拉入`vscode-languageserver`库。

以下是使用提供的简单文本文档管理器的服务器实现，该管理器通过始终将文件的全部内容从VS Code发送到服务器来同步文本文档。

```typescript
import {
	createConnection,
	TextDocuments,
	Diagnostic,
	DiagnosticSeverity,
	ProposedFeatures,
	InitializeParams,
	DidChangeConfigurationNotification,
	CompletionItem,
	CompletionItemKind,
	TextDocumentPositionParams,
	TextDocumentSyncKind,
	InitializeResult
} from 'vscode-languageserver';

import {
	TextDocument
} from 'vscode-languageserver-textdocument';

// 为服务器创建连接。该连接使用Node的IPC作为传输。
// 还包括所有预览/建议的LSP功能。
let connection = createConnection(ProposedFeatures.all);

// 创建一个简单的文本文档管理器。 文本文档管理器仅支持完整文档同步
let documents: TextDocuments<TextDocument> = new TextDocuments(TextDocument);

let hasConfigurationCapability: boolean = false;
let hasWorkspaceFolderCapability: boolean = false;
let hasDiagnosticRelatedInformationCapability: boolean = false;

connection.onInitialize((params: InitializeParams) => {
	let capabilities = params.capabilities;

	// Does the client support the `workspace/configuration` request?
	// If not, we will fall back using global settings
	hasConfigurationCapability = !!(
		capabilities.workspace && !!capabilities.workspace.configuration
	);
	hasWorkspaceFolderCapability = !!(
		capabilities.workspace && !!capabilities.workspace.workspaceFolders
	);
	hasDiagnosticRelatedInformationCapability = !!(
		capabilities.textDocument &&
		capabilities.textDocument.publishDiagnostics &&
		capabilities.textDocument.publishDiagnostics.relatedInformation
	);

	const result: InitializeResult = {
		capabilities: {
			textDocumentSync: TextDocumentSyncKind.Incremental,
			// Tell the client that the server supports code completion
			completionProvider: {
				resolveProvider: true
			}
		}
	};
	if (hasWorkspaceFolderCapability) {
		result.capabilities.workspace = {
			workspaceFolders: {
				supported: true
			}
		};
	}
	return result;
});

connection.onInitialized(() => {
	if (hasConfigurationCapability) {
		// 注册所有配置更改。
		connection.client.register(DidChangeConfigurationNotification.type, undefined);
	}
	if (hasWorkspaceFolderCapability) {
		connection.workspace.onDidChangeWorkspaceFolders(_event => {
			connection.console.log('Workspace folder change event received.');
		});
	}
});

interface ExampleSettings {
	maxNumberOfProblems: number;
}

// 全局设置, 在客户端不支持`workspace/configuration`请求时使用
const defaultSettings: ExampleSettings = { maxNumberOfProblems: 1000 };
let globalSettings: ExampleSettings = defaultSettings;

// 缓存所有打开的文档的设置
let documentSettings: Map<string, Thenable<ExampleSettings>> = new Map();

connection.onDidChangeConfiguration(change => {
	if (hasConfigurationCapability) {
		// Reset all cached document settings
		documentSettings.clear();
	} else {
		globalSettings = <ExampleSettings>(
			(change.settings.languageServerExample || defaultSettings)
		);
	}

	// 重新验证所有打开的文本文档
	documents.all().forEach(validateTextDocument);
});

function getDocumentSettings(resource: string): Thenable<ExampleSettings> {
	if (!hasConfigurationCapability) {
		return Promise.resolve(globalSettings);
	}
	let result = documentSettings.get(resource);
	if (!result) {
		result = connection.workspace.getConfiguration({
			scopeUri: resource,
			section: 'languageServerExample'
		});
		documentSettings.set(resource, result);
	}
	return result;
}

// 仅保留打开文档的设置
documents.onDidClose(e => {
	documentSettings.delete(e.document.uri);
});

// 文本文档的内容已更改。 首次打开文本文档或更改其内容时，将发出此事件。
documents.onDidChangeContent(change => {
	validateTextDocument(change.document);
});

async function validateTextDocument(textDocument: TextDocument): Promise<void> {
	// 在这个简单的示例中，我们获取每个验证运行的设置。
	let settings = await getDocumentSettings(textDocument.uri);

	// 验证器为所有长度大于等于2的大写单词创建诊断
	let text = textDocument.getText();
	let pattern = /\b[A-Z]{2,}\b/g;
	let m: RegExpExecArray | null;

	let problems = 0;
	let diagnostics: Diagnostic[] = [];
	while ((m = pattern.exec(text)) && problems < settings.maxNumberOfProblems) {
		problems++;
		let diagnostic: Diagnostic = {
			severity: DiagnosticSeverity.Warning,
			range: {
				start: textDocument.positionAt(m.index),
				end: textDocument.positionAt(m.index + m[0].length)
			},
			message: `${m[0]} is all uppercase.`,
			source: 'ex'
		};
		if (hasDiagnosticRelatedInformationCapability) {
			diagnostic.relatedInformation = [
				{
					location: {
						uri: textDocument.uri,
						range: Object.assign({}, diagnostic.range)
					},
					message: 'Spelling matters'
				},
				{
					location: {
						uri: textDocument.uri,
						range: Object.assign({}, diagnostic.range)
					},
					message: 'Particularly for names'
				}
			];
		}
		diagnostics.push(diagnostic);
	}

	// 将计算出的诊断信息发送到VSCode。
	connection.sendDiagnostics({ uri: textDocument.uri, diagnostics });
}

connection.onDidChangeWatchedFiles(_change => {
	// 监视的文件在VSCode中有更改
	connection.console.log('We received an file change event');
});

// 该处理程序提供完成补全的初始列表。
connection.onCompletion(
	(_textDocumentPosition: TextDocumentPositionParams): CompletionItem[] => {
		// 参数包含在其中请求代码完成的文本文档的位置。对于示例，我们忽略此信息，并始终提供相同的完成项目。
		return [
			{
				label: 'TypeScript',
				kind: CompletionItemKind.Text,
				data: 1
			},
			{
				label: 'JavaScript',
				kind: CompletionItemKind.Text,
				data: 2
			}
		];
	}
);

// 该处理程序为完成列表中选定的项目解析其他信息。
connection.onCompletionResolve(
	(item: CompletionItem): CompletionItem => {
		if (item.data === 1) {
			item.detail = 'TypeScript details';
			item.documentation = 'TypeScript documentation';
		} else if (item.data === 2) {
			item.detail = 'JavaScript details';
			item.documentation = 'JavaScript documentation';
		}
		return item;
	}
);

//使文本文档管理器在连接上侦听打开，更改和关闭文本文档事件
documents.listen(connection);

// 监听连接
connection.listen();

```

## 参考

https://code.visualstudio.com/api/language-extensions/language-server-extension-guide