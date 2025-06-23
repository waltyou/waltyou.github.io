---
layout: post
title: The ConnectionResetError when using AzureOpenAIEmbeddings
date: 2024-06-29 13:39:04
author: admin
comments: true
categories: [GenAI]
tags: [Azure, OpenAI]
---

When using `AzureOpenAIEmbeddings`, met a network error: `Arguments: (ConnectionError(ProtocolError('Connection aborted.', ConnectionResetError(104, 'Connection reset by peer'))),)`. Let's see what happen.

<!-- more -->

---

* Outline
{:toc}
---

## Problem description

Here is the code:

```python
import os
import httpx
from azure.identity import DefaultAzureCredential, get_bearer_token_provider
from langchain_openai import AzureChatOpenAI
from langchain_openai import AzureOpenAIEmbeddings

http_client = httpx.Client(verify=False)
token_provider = get_bearer_token_provider(
    DefaultAzureCredential(), "https://cognitiveservices.azure.com/.default"
)

def get_azure_emb():
    return AzureOpenAIEmbeddings(
        azure_deployment=os.getenv("AZURE_OPENAI_EMBEDDING_DEPLOYMENT_NAME"),
        azure_endpoint=os.getenv("AZURE_OPENAI_ENDPOINT"),
        openai_api_version=os.getenv("OPENAI_API_VERSION"),
        azure_ad_token_provider=token_provider,
        http_client=http_client,
    )

embedding = get_azure_emb()
r = embedding.embed_query("this is a test text")
print(r)
```

Here is the error:
```text
Traceback (most recent call last):
  File "/opt/conda/lib/python3.10/site-packages/urllib3/connectionpool.py", line 715, in urlopen
    httplib_response = self._make_request(
  File "/opt/conda/lib/python3.10/site-packages/urllib3/connectionpool.py", line 404, in _make_request
    self._validate_conn(conn)
  File "/opt/conda/lib/python3.10/site-packages/urllib3/connectionpool.py", line 1060, in _validate_conn
    conn.connect()
  File "/opt/conda/lib/python3.10/site-packages/urllib3/connection.py", line 419, in connect
    self.sock = ssl_wrap_socket(
  File "/opt/conda/lib/python3.10/site-packages/urllib3/util/ssl_.py", line 449, in ssl_wrap_socket
    ssl_sock = _ssl_wrap_socket_impl(
  File "/opt/conda/lib/python3.10/site-packages/urllib3/util/ssl_.py", line 493, in _ssl_wrap_socket_impl
    return ssl_context.wrap_socket(sock, server_hostname=server_hostname)
  File "/opt/conda/lib/python3.10/ssl.py", line 513, in wrap_socket
    return self.sslsocket_class._create(
  File "/opt/conda/lib/python3.10/ssl.py", line 1104, in _create
    self.do_handshake()
  File "/opt/conda/lib/python3.10/ssl.py", line 1375, in do_handshake
    self._sslobj.do_handshake()
ConnectionResetError: [Errno 104] Connection reset by peer
```

## Resolution

Install the tiktoken in your running env: https://stackoverflow.com/questions/76106366/how-to-use-tiktoken-in-offline-mode-computer

## Root cause analysis

Inspired by this [Github comment](https://github.com/langchain-ai/langchain/issues/21575#issuecomment-2204870353), I found the root cause is the code is not running in the public network. 

AzureOpenAIEmbeddings uses `tiktoken` lib to implment a feature called [check_embedding_ctx_length](https://github.com/langchain-ai/langchain/blob/master/libs/partners/openai/langchain_openai/embeddings/base.py#L263): 

```text
Whether to check the token length of inputs and automatically split inputs longer than embedding_ctx_length.
```

By default, this feature is enable. So by following the call chain `embed_documents -> _get_len_safe_embeddings -> self._tokenize`, it will run 
```python
try:
    encoding = tiktoken.encoding_for_model(model_name)
except KeyError:
    encoding = tiktoken.get_encoding("cl100k_base")
```

And this line will throw this error `ConnectionResetError: [Errno 104] Connection reset by peer` if it can't access public network.

So after install tiktoken as offline mode, the error will disappear.

