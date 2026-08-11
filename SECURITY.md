<!--
Copyright (C) 2025 - 2026 ANSYS, Inc. and/or its affiliates.
SPDX-License-Identifier: Apache-2.0


Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
-->

# Security Policy

## Supported Versions

| Version    | Supported          |
| ---------- | ------------------ |
| 0.1.x      | :white_check_mark: |

## Security considerations

The `execute_python_code` tool (`src/ansys/lumerical/mcp/tools.py`) is, by
design, an **unsandboxed arbitrary Python code execution** capability. Code
sent to it by a connected MCP client is `exec`'d in a persistent subprocess
that runs with the same operating-system privileges as the MCP server itself:
full file system access, network access, and the ability to spawn further
subprocesses. This is intentional -- the assistant needs unrestricted access
to the PyLumerical/`lumapi` Python API to drive Lumerical simulations -- but
it means there is no code-level sandbox between "a client can call this tool"
and "a client has a shell on the host machine".

Consequences for anyone deploying or connecting to this server:

- **Only run this server with clients and models you trust.** Granting any
  MCP client access to this server is equivalent to giving that client a
  shell on the host machine. Do not connect it to a client or model whose
  behavior you would not trust with direct shell access.
- **Prompt injection is a realistic attack path.** If the connected LLM is
  ever exposed to untrusted content (web pages, files, output from other MCP
  tools, etc.), that content could attempt to steer the model into calling
  `execute_python_code` with malicious code. This server does not, and by
  design cannot, independently validate or restrict what code is executed --
  the trust boundary is the model and the content it is exposed to, not the
  tool.
- **Network exposure multiplies the risk.** This server can run over the
  Streamable HTTP transport in addition to the default local STDIO
  transport. Exposing that HTTP endpoint on a non-loopback interface without
  an authentication layer in front of it effectively publishes an
  unauthenticated remote code execution endpoint. See
  [Quick start](doc/source/getting_started/quick_start.rst) for guidance on
  binding the HTTP transport safely and adding authentication.

## Reporting a vulnerability

> [!CAUTION]
> Do not use GitHub issues to report any security vulnerabilities.

If you detect a vulnerability, contact the [PyAnsys Core team](mailto:pyansys.core@ansys.com),
mentioning the repository and the details of your finding. The team will address it as soon as possible.

Provide the PyAnsys Core team with this information:

- Any specific configuration settings needed to reproduce the problem
- Step-by-step guidance to reproduce the problem
- The exact location of the problematic source code, including tag, branch, commit, or a direct URL
- The potential consequences of the vulnerability, along with a description of how an attacker could take advantage of the issue
