---
layout: post
title: Implementing an MCP Client in an IntelliJ Plugin using Kotlin
date: 2024-06-29 13:39:04
author: admin
comments: true
categories: [GenAI]
tags: [MCP, MCP Client, Intellij Plugin, OpenAI, Kotlin]
---

MCP has been gaining popularity recently, and I've implemented an MCP Client for our company's internal Code Assistant Tool. Since there are plenty of tutorials about writing MCP Servers but very few about MCP Clients, especially implementations in Kotlin, I feel it's necessary to document the process and the challenges I encountered, hoping it will help others.


<!-- more -->

---

* Content
{:toc}
---

## Background

I needed to add MCP client functionality to an IntelliJ plugin written in Kotlin, so I referenced the official codebase [Kotlin-sdk](https://github.com/modelcontextprotocol/kotlin-sdk). However, the official [kotlin-mcp-client](https://github.com/modelcontextprotocol/kotlin-sdk/blob/main/samples/kotlin-mcp-client/src/main/kotlin/io/modelcontextprotocol/sample/client/MCPClient.kt) had several issues that didn't meet my requirements:

1. The LLM calls Anthropic's API, but I wanted to use an OpenAI compatible API
2. It only supports stdio MCP servers, with no examples for SSE
3. When running in an IntelliJ plugin, the stdio initialization code couldn't correctly get the complete PATH environment variable, causing cmd not to be found
4. There was no code for getting server configurations from MCP Server config files, initializing servers, or managing servers

## Problems and Solutions

### 1. MCP Client + OpenAI compatible API + Kotlin

Most MCP Client + OpenAI compatible API combinations found online are written in Python or TypeScript, with very few in Kotlin. Since MCP is also a relatively new concept, mainstream LLM training data probably doesn't include related code, resulting in poor generation quality. After several attempts, I had to implement it myself.

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

The main focus here is properly parsing the `tool_calls` field in the response according to the specification. The rest is standard operation.

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

The key point in this step is the `newFun` function, which converts MCP-spec tool types to OpenaiTool types.

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

The tricky part here is the `jsonObjectToMap` function that converts JsonObject to Map<String, Any> type. The `Any` type makes this more complicated than in Python or TypeScript, as the code needs to check types during conversion.

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

### 2. Initializing SSE type MCP Server

The code isn't very intuitive - instead of initializing a class to get an instance, it calls a static function `mcpSseTransport` that configures a ktor HttpClient.

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

Here's my actual code snippet:

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

There's a catch here - the Kotlin SDK will automatically append "sse" to the end of the SSE server's URL (see code [here](https://github.com/modelcontextprotocol/kotlin-sdk/blob/main/src/commonMain/kotlin/io/modelcontextprotocol/kotlin/sdk/client/SSEClientTransport.kt#L56)), but Python and TypeScript SDKs don't. I handled this by removing the "sse" suffix with `c.url.removeSuffix("/sse")`.

### 3. Using GeneralCommandLine to initialize stdio mcp server in IntelliJ plugin

In the official example, `ProcessBuilder` is used to initialize stdio servers. However, when using it in an IntelliJ plugin, it couldn't correctly get the complete PATH environment variable, preventing stdio MCP servers from starting properly. After debugging and research, I found that `GeneralCommandLine` should be used instead. Here's why (according to DeepSeek):

```text
When executing external commands or starting processes in Java, GeneralCommandLine and ProcessBuilder are two different tools with distinct differences and use cases:

1. **Origin and Dependencies**
   - **ProcessBuilder**  
     Part of Java standard library (java.lang.ProcessBuilder), introduced in Java 1.5, no extra dependencies needed.
   - **GeneralCommandLine**  
     Developed by JetBrains (com.intellij.execution.configurations.GeneralCommandLine), requires JetBrains libraries.

2. **API Design**
   - **ProcessBuilder**  
     Provides basic chained configuration, but parameters must be set via lists or arrays:
     ```java
     ProcessBuilder pb = new ProcessBuilder("cmd", "/c", "echo", "Hello");
     pb.directory(new File("C:/"));
     Process p = pb.start();
     ```
   - **GeneralCommandLine**  
     Supports more fluid chaining, more intuitive parameter handling:
     ```java
     GeneralCommandLine cmd = new GeneralCommandLine()
         .withExePath("cmd")
         .withParameters("/c", "echo", "Hello")
         .withWorkDirectory("C:/");
     Process process = cmd.createProcess();
     ```

3. **Features and Security**
   - **Parameter Safety**  
     ProcessBuilder requires manual parameter separation (spaces, special chars), GeneralCommandLine may handle escaping automatically, preventing injection risks.
   - **Environment Variables and Paths**  
     ProcessBuilder exposes environment() returning a Map for modification, GeneralCommandLine provides more intuitive methods like addEnvironmentVariable(key, value).
   - **Output Handling**  
     ProcessBuilder requires manual stream redirection (redirectOutput), while GeneralCommandLine may encapsulate stream capture or callbacks.

4. **Cross-platform Support**
   - **ProcessBuilder**  
     Developers must handle platform differences (Windows cmd /c vs Linux /bin/sh).
   - **GeneralCommandLine**  
     May include built-in cross-platform logic (e.g., automatic shell selection), depending on library version.

5. **Exception Handling**
   - **ProcessBuilder**  
     Throws IOException, requiring manual low-level exception handling.
   - **GeneralCommandLine**  
     May wrap exceptions in friendlier types with more detailed error information.

6. **Use Cases**
   - **Choose ProcessBuilder when:**  
     - Project must avoid third-party dependencies
     - Need low-level control (e.g., custom stream redirection)
     - Familiar with Java standard APIs
   - **Choose GeneralCommandLine when:**  
     - Developing IntelliJ plugins or using JetBrains libraries
     - Need concise, safe APIs to reduce boilerplate
     - Prioritize development efficiency and code readability
```

So here's my code for initializing stdio:

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

### 4. MCP server configuration file management

This mainly references Roo's implementation in [McpHub.ts](https://github.com/RooVetGit/Roo-Code/blob/main/src/services/mcp/McpHub.ts). I converted it from TypeScript to Kotlin. The code is quite long, so I'll just list the core workflow:

1. When initializing the IntelliJ plugin, get the contents of MCP setting files at both project and global levels
2. Based on config file contents, initialize MCP server connections, getting server names, tool lists, status, and other needed information
3. Hold all server connections in the `connections` variable
4. Users can view all servers' names, tool lists, status, etc. through the IntelliJ plugin UI
5. When users add/delete/modify MCP server settings through the plugin UI, need to sync updates to both `connections` and settings files
6. When users directly modify setting files with file editor, need to sync changes to `connections` and plugin UI
7. Before sending messages via MCP client, get all tool lists, then call LLM, get suggested tool name and parameters from LLM, execute MCP server tool to get results, then pass tool results back to LLM for final answer. These steps can be cycled as multiple different tools may be needed.


## Final Thoughts

The whole MCP client implementation process took about 3 weeks, much slower than expected, as development efficiency has greatly improved with LLM support nowadays. However, LLM wasn't very helpful for this feature - either generating almost unusable code or going in circles with various attempts that couldn't solve the problem. In the end, I had to write the code manually, estimating 80% was hand-written (feeling like regressing from industrial to agricultural civilization - very uncomfortable. If using Python or TypeScript, probably 95% would be LLM-written). The reason might be that there's too little IntelliJ plugin development code in LLM training data, making it less proficient.

Final lessons learned:

1. Try to use popular languages like Python or TypeScript for projects - LLMs are more proficient with them, making development much more efficient
2. Current LLMs may still struggle to generate **new** knowledge or code based on **old** knowledge or code