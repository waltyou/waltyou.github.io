---
layout: post
title: 用kotlin在一个intellij 插件中实现一个MCP client 
date: 2024-06-29 13:39:04
author: admin
comments: true
categories: [GenAI]
tags: [MCP, MCP Client, Intellij Plugin, OpenAI, Kotlin]
---

MCP 最近很火，自己也为公司内部的Code Assistant Tool 实现了一个MCP Client。因为网上几乎全是教大家怎么写MCP Server的，很少有关于MCP Client，kotlin 实现的就更少了。所以我觉得有必要记录一下过程，以及遇到的问题，希望可以帮助大家。


<!-- more -->

---

* 目录
{:toc}
---

## 背景

因为我需要给一个kotlin写的Intellij plugin添加MCP client功能，所以需要参考官方代码库 [Kotlin-sdk](https://github.com/modelcontextprotocol/kotlin-sdk)。但是官方代码中[kotlin-mcp-client](https://github.com/modelcontextprotocol/kotlin-sdk/blob/main/samples/kotlin-mcp-client/src/main/kotlin/io/modelcontextprotocol/sample/client/MCPClient.kt)有几个问题不符合我的需求：

1. LLM 调用的是 Anthropic 的API，但是我希望是OpenAI compatible API
2. 只支持 stdio 的MCP server，没有sse 的例子
3. 初始化 stdio的代码在intellij plugin中运行时，无法正确获取完整的 PATH 环境变量，导致cmd找不到
4. 没有从MCP Server 配置文件中获取server配置、初始化server、管理server的代码

## 问题及解决方案

### 1. MCP Client + OpenAI compatible API + Kotlin

网上大多数 MCP Client + OpenAI compatible API 的组合，大都是Python or Typescript 写的，kotlin的很少，几乎没有找到，而且可能因为MCP的概念也很新，主流llm训练数据里可能都没有相关代码，所以生成的效果很差，几番尝试后，只能自己动手、丰衣足食了。

#### #1. define needed types

```kotlin
import io.modelcontextprotocol.kotlin.sdk.Tool
import io.modelcontextprotocol.kotlin.sdk.client.Client
import io.modelcontextprotocol.kotlin.sdk.shared.Transport

enum class McpServerStatus {
    CONNECTED,
    CONNECTING,
    DISCONNECTED
}

enum class McpServerSource {
    GLOBAL,
    PROJECT
}

data class McpConnection(
    val server: McpServer,
    val client: Client,
    val transport: Transport
)

data class McpServer(
    val name: String,
    val config: ServerConfig,
    var status: McpServerStatus,
    var error: String? = null,
    var tools: List<Tool>? = null,
    var disabled: Boolean? = null,
    val timeout: Long? = null,
    val source: McpServerSource? = null,
    val projectPath: String? = null
)

data class OpenaiTool(
    val type: String = "function",
    val function: McpFunction,
)

data class McpFunction(
    val name: String,
    val description: String,
    val parameters: McpParameters,
)

data class McpParameters(
    val type: String = "object",
    val properties: Map<String, McpProperty>,
    val required: List<String>,
)

data class McpProperty(
    val type: String,
    val description: String,
    val enum: List<String>? = null,
)

data class ChatMessage(
    val role: String,
    val content: String,
)

data class OpenAIRequestBody(
    val model: String,
    val messages: List<ChatMessage>,
    val tools: List<OpenaiTool>? = null,
)
```

#### #2. code to call OpenAI Compatible API

这里主要是要按规范解析出response中 `tool_calls` 字段，剩下的就是常规操作。

```kotlin
suspend fun sendSuspendToOpenAI(
    model: String,
    chatHistory: List<ChatMessage>,
    tools: List<OpenaiTool>? = null,
): JsonObject {
    val requestBodyJsonStr: String =
        gson.toJson(
            OpenAIRequestBody(
                model = model,
                messages = chatHistory,
                tools = tools,
            ),
        )
    println(requestBodyJsonStr)
    val llmResponse: HttpResponse =
        client.post(OPENAI_ENDPOINT) {
            contentType(ContentType.Application.Json)
            setBody(requestBodyJsonStr)
        }

    val json: JsonObject =
        Gson().fromJson(llmResponse.bodyAsText(), JsonObject::class.java)
    println(json)

    // Extract the response content from the OpenAI format
    val choices = json.getAsJsonArray("choices")
    if (choices != null && choices.size() > 0) {
        val firstChoice = choices[0].asJsonObject
        val message = firstChoice.getAsJsonObject("message")

        // Check if there's a tool call in the response
        if (message.has("tool_calls") && !message.get("tool_calls").isJsonNull) {
            val toolCalls = message.getAsJsonArray("tool_calls")
            if (toolCalls != null && toolCalls.size() > 0) {
                return toolCalls[0].asJsonObject
            }
        }
        return message
    }

    return json
}
```

#### #3. MCP Client call LLM with Tools

这一步的重点在于 func `newFun`, 它把 mcp 规范中定义的 tool 类型转换成 OpenaiTool 类型。

```kotlin
// hold all mcp server's connnection
val connections = mutableListOf<McpConnection>()

//Get a list of all available MCP tools across all connected servers, then convert them to OpenaiTool 
private fun getAvailableTools(): List<OpenaiTool> {
  return connections
    .filter { it.server.status == McpServerStatus.CONNECTED }
    .flatMap { connection ->
        val serverTools = connection.server.tools ?: emptyList()
        val serverName = connection.server.name

        serverTools.map { tool ->
            OpenaiTool(
                function = newFun(tool, serverName)
            )
        }
    }
}

private fun newFun(tool: Tool, serverName: String) =
  McpFunction(
      name = serverName + "." + tool.name,
      description = tool.description ?: "",
      parameters =
          McpParameters(
              properties =
                  (
                      tool.inputSchema.properties as?
                          JsonObject
                      )?.mapValues { (_, value) ->
                          McpProperty(
                              type = "string",
                              description =
                                  value.toString(),
                          )
                      }
                      ?: emptyMap(),
              required =
                  tool.inputSchema.required
                      ?: emptyList(),
          ),
  )

/**
* Send a message to the LLM with MCP tools.
*
* @param session The chat session to send
* @return Triple of (response, isFunction, success)
*/
suspend fun send(
  session: List<ChatMessage>,
  model: String = "gpt-4o-mini"
): Triple<String, Boolean, Boolean> {
  try {
      if (connections.isEmpty()) {
          return Triple(
              "No MCP servers connected. Please check your configuration.",
              false,
              false
          )
      }
      val tools = getAvailableTools()
      val response =
          LLMClient.sendSuspendToOpenAI(
              model = model,
              chatHistory = session,
              tools = tools,
          )
      // if it is function call, we need to call the function
      if (response.has("type") && response.get("type").asString == "function") {
          val functionCall = response.get("function").asJsonObject
          val functionName = functionCall.get("name").asString
          val tool = tools.find { it.function.name == functionName }
          if (tool != null) {
              functionCall.addProperty("description", tool.function.description)
              return Triple(functionCall.toString(), true, true)
          } else {
              return Triple("tool is not found", false, true)
          }
      }
      return Triple(response.get("content").asString, false, true)
  } catch (e: Exception) {
      LOG.error("Error processing MCP query: ${e.message}", e)
      return Triple("Error processing query: ${e.message}", false, false)
  }
}
```

#### #4. MCP Client call MCP Server's tool

```kotlin
/**
* Execute a function through MCP.
*
* @param funcStr The function string to execute
* @return Pair of (response, success)
*/
suspend fun executeFunc(funcStr: String): Pair<String, Boolean> {
  return try {
      if (connections.isEmpty()) {
          return Pair("No MCP servers connected. Please check your configuration.", false)
      }

      val tools = getAvailableTools()
      val function: com.google.gson.JsonObject =
          Gson().fromJson(funcStr, com.google.gson.JsonObject::class.java)
      val functionName = function.get("name").asString
      // Check if the function exists in the available tools
      if (tools.none { it.function.name == functionName }) {
          throw IllegalArgumentException("Function $functionName not found in available tools")
      }
      val names = functionName.split(".")
      val serverName = names[0]
      val realFuncName = names[1]

      val connection = findConnection(serverName)
          ?: return Pair("No connection found for server: $serverName", false)

      val argumentsString = function.get("arguments").asString
      val argumentsJson = JsonParser.parseString(argumentsString).asJsonObject
      val arguments = jsonObjectToMap(argumentsJson)

      val toolResponse =
          connection.client.callTool(
              name = realFuncName,
              arguments,
          )

      if (toolResponse == null) {
          return Pair("The function $functionName returned no result", false)
      }

      val toolText =
          (toolResponse.content.firstOrNull() as? TextContent)?.text
              ?: "null"
      val success = toolResponse.isError?.let { !it } ?: true
      return Pair(
          "The function %s's result is:\n %s".format(functionName, toolText),
          success
      )
  } catch (e: Exception) {
      LOG.error("Error executing MCP function: ${e.message}", e)
      Pair("Error processing query: ${e.message}", false)
  }
}
```

这里需要着重说一下func `jsonObjectToMap`, 它用来把JsonObject 转换成 Map<String, Any> 类型，麻烦的地方就是 `Any` 这里，code在转换的时候还是要判断一下类型的，这一点不如python 和 ts的方便简单。

```kotlin
private fun jsonObjectToMap(jsonObject: com.google.gson.JsonObject): Map<String, Any> {
    val map = mutableMapOf<String, Any>()

    for ((key, value) in jsonObject.entrySet()) {
        map[key] = convertJsonElement(value)
    }

    return map
}

private fun convertJsonElement(element: JsonElement): Any =
    when {
        element.isJsonPrimitive -> {
            if (element.asJsonPrimitive.isNumber) {
                element.asNumber
            } else if (element.asJsonPrimitive.isBoolean) {
                element.asBoolean
            } else {
                element.asString
            }
        }

        element.isJsonObject -> jsonObjectToMap(element.asJsonObject)
        element.isJsonArray -> element.asJsonArray
        else -> Unit
    }

```

### 2. 初始化 sse 类型的 MCP Server 

代码不是很符合直觉，它并不是通过初始化一个类来获取，而是调用一个给 ktor HttpClient 配置的静态函数 mcpSseTransport。

https://github.com/modelcontextprotocol/kotlin-sdk/blob/main/src/commonMain/kotlin/io/modelcontextprotocol/kotlin/sdk/client/KtorClient.kt#L18

```kotlin
/**
 * Returns a new SSE transport for the Model Context Protocol using the provided HttpClient.
 *
 * @param urlString Optional URL of the MCP server.
 * @param reconnectionTime Optional duration to wait before attempting to reconnect.
 * @param  requestBuilder Optional lambda to configure the HTTP request.
 * @return A [SSEClientTransport] configured for MCP communication.
 */
public fun HttpClient.mcpSseTransport(
    urlString: String? = null,
    reconnectionTime: Duration? = null,
    requestBuilder: HttpRequestBuilder.() -> Unit = {},
): SseClientTransport = SseClientTransport(this, urlString, reconnectionTime, requestBuilder)
```

实际我的代码片段如下:

```kotlin
private fun createSseTransport(c: SseConfig, name: String, source: McpServerSource): Transport {
    // SSE connection
    val httpClient =
        HttpClient(CIO) {
            install(SSE) // Install SSE plugin
        }

    val transport = httpClient.mcpSseTransport(
        urlString = c.url.removeSuffix("/sse"),
        reconnectionTime = c.timeout.seconds,
    ) {
        c.headers?.forEach { (key, value) ->
            header(key, value)
        }
    }
    transport.onError {
        LOG.error("Transport error for \"$name\": ${it.message}", it)
        val connection = findConnection(name, source)
        if (connection != null) {
            connection.server.status = McpServerStatus.DISCONNECTED
            appendErrorMessage(connection, it.message)
        }
        notifyUIOfServerChanges()
    }
    return transport
}
```

这里有个坑，kotlin sdk会默认给sse server的 url 的结尾加上 `sse`，代码看[这里](https://github.com/modelcontextprotocol/kotlin-sdk/blob/main/src/commonMain/kotlin/io/modelcontextprotocol/kotlin/sdk/client/SSEClientTransport.kt#L56), 但是python sdk 、ts sdk的就不会。需要自己额外处理一下，我通过 `c.url.removeSuffix("/sse")` 移除了url末尾的`sse`。

### 3. 在intellij plugin，使用 GeneralCommandLine 初始化 stdio mcp server

官方例子中，初始化 stdio server的时候，用的是`ProcessBuilder`，但是在intellij plugin中，使用它的时候，总是无法正确获取完整 PATH 环境变量，导致无法正确启动stdio mcp server，经过各种调试和查询，发现应该使用`GeneralCommandLine` 替代它。原因如下(deepseek 的回答)：

```text
在Java中执行外部命令或启动进程时，`GeneralCommandLine`和`ProcessBuilder`是两种不同的工具，它们的主要区别和适用场景如下：

1. **来源与依赖**
   - **ProcessBuilder**  
     属于Java标准库（`java.lang.ProcessBuilder`），自Java 1.5引入，无需额外依赖。
   - **GeneralCommandLine**  
     是JetBrains开发的第三方类（如IntelliJ IDEA插件开发中的`com.intellij.execution.configurations.GeneralCommandLine`），需引入JetBrains相关库。

2. **API设计**
   - **ProcessBuilder**  
     提供基础的链式配置，但参数需通过列表或数组设置，不够直观：
     ```java
     ProcessBuilder pb = new ProcessBuilder("cmd", "/c", "echo", "Hello");
     pb.directory(new File("C:/"));
     Process p = pb.start();
     ```
   - **GeneralCommandLine**  
     支持更流畅的链式调用，参数处理更直观：
     ```java
     GeneralCommandLine cmd = new GeneralCommandLine()
         .withExePath("cmd")
         .withParameters("/c", "echo", "Hello")
         .withWorkDirectory("C:/");
     Process process = cmd.createProcess();
     ```

3. **功能与安全性**
   - **参数安全性**  
     `ProcessBuilder`需手动处理参数分隔（如空格、特殊字符），而`GeneralCommandLine`可能自动处理转义，避免注入风险。
   - **环境变量与路径**  
     `ProcessBuilder`通过`environment()`返回`Map`供修改，`GeneralCommandLine`提供`addEnvironmentVariable(key, value)`等更直观的方法。
   - **输出处理**  
     `ProcessBuilder`需手动重定向流（如`redirectOutput`），而`GeneralCommandLine`可能封装了流捕获或回调机制。

4. **跨平台支持**
   - **ProcessBuilder**  
     需开发者自行处理平台差异（如Windows用`cmd /c`，Linux用`/bin/sh`）。
   - **GeneralCommandLine**  
     可能内置跨平台逻辑（如自动选择Shell），但具体实现取决于库版本。

5. **异常处理**
   - **ProcessBuilder**  
     抛出`IOException`，需手动处理低级异常。
   - **GeneralCommandLine**  
     可能封装异常为更友好的类型，并提供详细错误信息。

6. **适用场景**
   - **选择ProcessBuilder**  
     - 项目需避免第三方依赖。
     - 需要底层控制（如自定义流重定向）。
     - 已熟悉Java标准API。
   - **选择GeneralCommandLine**  
     - 开发IntelliJ插件或使用JetBrains库。
     - 需要简洁、安全的API减少样板代码。
     - 优先考虑开发效率和代码可读性。

7. 总结对比表

| 特性                | ProcessBuilder                          | GeneralCommandLine                     |
|---------------------|-----------------------------------------|----------------------------------------|
| **来源**            | Java标准库                              | JetBrains第三方库                      |
| **依赖**            | 无需                                    | 需引入JetBrains库                      |
| **API易用性**       | 基础，需手动处理参数                    | 链式调用，参数处理更安全               |
| **跨平台支持**      | 需手动处理                              | 可能内置逻辑                           |
| **输出处理**        | 需手动重定向                            | 可能封装高级功能                       |
| **异常处理**        | 抛出IOException                         | 可能更友好的异常封装                   |
| **适用场景**        | 无依赖需求或底层控制                    | 快速开发、代码简洁性优先               |

根据项目需求选择：优先标准化和轻量级使用`ProcessBuilder`；追求开发效率和安全性且可接受依赖时，选择`GeneralCommandLine`。
```

所以我初始化stdio的code如下：

```kotlin
fun createProcess(c: StdioConfig): Process {
    // Build the command based on the file extension of the server script
    val command = buildList {
        add(c.command)
        addAll(c.args ?: emptyList())
    }
    val pb = GeneralCommandLine(command)
    val env = pb.environment
    c.env?.forEach { (key, value) ->
        env[key] = value
    }
    pb.workDirectory = File(c.cwd)
    return pb.createProcess()
}

private fun createStdioTransport(
    c: StdioConfig,
    name: String,
    source: McpServerSource
): Transport {
    val process = createProcess(c)
    // Setup I/O transport using the process streams
    val transport = StdioClientTransport(
        input = process.inputStream.asSource().buffered(),
        output = process.outputStream.asSink().buffered()
    )
    // Set up stdio specific error handling
    transport.onError {
        val connection = findConnection(name, source)
        if (connection != null) {
            connection.server.status = McpServerStatus.DISCONNECTED
            appendErrorMessage(connection, it.message)
        }
        notifyUIOfServerChanges()
    }
    transport.onClose {
        val connection = findConnection(name, source)
        if (connection != null) {
            connection.server.status = McpServerStatus.DISCONNECTED
        }
        notifyUIOfServerChanges()
    }
    return transport
}
```

### 4. MCP server 配置文件的管理功能

这个主要是是参考了roo code的实现 [McpHub.ts](https://github.com/RooVetGit/Roo-Code/blob/main/src/services/mcp/McpHub.ts)，它是ts版本的，我把它转成了kotlin版本，这个代码有点长，只列出一些核心流程吧。

1. 在intellij plugin 插件初始化的时候，获取 project level和global level的 mcp setting 文件的内容
2. 根据配置文件内容，初始化mcp server connection，获取server的name、tool list、status 等其他需要的信息
3. 通过 val connections 持有所有的server connection
4. 用户可以通过intellij plugin 的UI 查看所有server 的name、tool list、status 等信息
5. 用户通过intellij plugin 的UI 对mcp server setting 文件进行增删改后，需要同步更新 val connections 以及 setting 文件内容
6. 用户如果直接用file editor对setting 文件进行了修改，需要同步修改到 val connections 以及 intellij plugin 的UI
7. 用户通过MCP client发送消息之前，获取所有的tool list，然后call llm，获取llm 建议调用的tool name以及具体参数，接着执行 mcp server tool 获取结果，最后在把tool 的结果，传回llm，获取最终答案。当然这些步骤是可以循环的，因为有可能需要循环多个不同的tool。


## 写在最后

整个mcp client的实现过程，大概用了3周的时间，比我预期的慢了好多，因为现如今在LLM的加持下，开发的效率已经大大的提升。然而 LLM 在这个feature上的帮助其实并不大，要么是生成的代码几乎不可用，或者就是兜兜转转、各种尝试却无法解决问题。最后不得不手写代码，预计80%代码都是手写（从工业文明倒退回了农耕文明，非常难受，如果是python或者ts， 估计95%的都是LLM写的）。究其原因，可能是因为在 LLM 的训练数据中，intellij plugin 开发的代码太少了，LLM 不是很擅长。

最后总结一下经验就是：

1. 尽量使用流行的语言去开发项目，比如python 或者 ts，LLM 会更擅长，这样子开发效率也会事半功倍
2. 现阶段的LLM 可能还是很难根据**旧**的知识或者代码，产生**新**的知识或者代码