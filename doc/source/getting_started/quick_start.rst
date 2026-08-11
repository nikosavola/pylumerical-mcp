Quick start
===========

Learn how to connect PyLumerical-MCP to your preferred agentic AI client.
Before you begin, install PyLumerical-MCP by following the instructions in
:doc:`Installation <installation>`.

Connect to your agentic client
------------------------------

You can use PyLumerical-MCP with transport over both `Streamable HTTP <https://modelcontextprotocol.io/specification/2025-11-25/basic/transports#streamable-http>`__
and `STDIO <https://modelcontextprotocol.io/specification/2025-11-25/basic/transports#stdio>`__.

Since PyLumerical-MCP uses the open source MCP standard, which enables AI applications to seamlessly integrate with external systems,
you can use it with a wide range of agent-based frameworks and agent harnesses that support MCP.

The following sections show you the important settings for PyLumerical-MCP. For instructions on how to connect,
see your preferred client's documentation.

Configure Streamable HTTP transport
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

To configure Streamable HTTP transport, you can set it through the ``.env`` file.

#. Copy the ``.env.example`` template file and rename it to ``.env``.

#. Set the following variables:

   - ``FASTMCP_TRANSPORT``: Set to ``"streamable-http"``.
   - ``FASTMCP_HOST``: Set to the hostname for the server. For local use, set to ``"127.0.0.1"``.
   - ``FASTMCP_PORT``: Set to the port for the server, such as ``8081``.

#. Start the MCP server by running ``ansys-lumerical-mcp`` from the command line in the environment where the MCP server is installed.

#. Connect to the MCP server from your client.

.. warning::

   The ``execute_python_code`` tool runs arbitrary, unsandboxed Python with
   the same privileges as the MCP server process (file system, network, and
   subprocess access). Streamable HTTP exposes that tool over the network,
   so treat the endpoint accordingly:

   - Keep ``FASTMCP_HOST`` set to ``"127.0.0.1"`` unless a reverse proxy or
     authentication layer terminates connections in front of the server.
   - Never expose ``FASTMCP_PORT`` to an untrusted network (for example, the
     public internet or a shared corporate network) without an
     authentication layer in front of it.

   See `Secure the Streamable HTTP transport`_ for how to add
   authentication.

Secure the Streamable HTTP transport
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

PyLumerical-MCP does not enable authentication by default. The server is
built on `FastMCP <https://github.com/jlowin/fastmcp>`__, which supports
pluggable authentication for its HTTP transports through the ``auth``
argument of ``fastmcp.FastMCP``. ``PyLumericalMCP`` inherits this argument
(via ``ansys.common.mcp.PyAnsysBaseMCP``, which forwards unrecognized keyword
arguments to ``FastMCP.__init__``), so authentication can be enabled without
modifying the class hierarchy.

To require a bearer token, construct a FastMCP ``AuthProvider`` and pass it
as ``auth=`` where ``PyLumericalMCP`` is instantiated, in
``src/ansys/lumerical/mcp/server.py``. For example, using FastMCP's
``JWTVerifier`` with a shared HS256 secret read from the environment:

.. code:: python

    import os

    from fastmcp.server.auth.providers.jwt import JWTVerifier

    auth = JWTVerifier(
        public_key=os.environ["PYLUMERICAL_MCP_BEARER_SECRET"],
        algorithm="HS256",
    )

    app = PyLumericalMCP(
        name="pylumerical-mcp",
        config=config,
        instructions=PYLUMERICAL_SYSTEM_PROMPT,
        auth=auth,
    )

Clients then authenticate by sending ``Authorization: Bearer <token>``, where
``<token>`` is a JWT signed with the same shared secret. For quick local
testing only, FastMCP's ``StaticTokenVerifier`` accepts a plain dictionary of
valid token strings instead of JWTs -- it is explicitly documented upstream
as unsuitable for production use, since tokens are stored in plain text. For
multi-user or production-facing deployments, prefer a real OAuth 2.0 or
JWKS-based provider; see FastMCP's
`authentication documentation <https://gofastmcp.com/servers/auth/authentication>`__
for the full list of supported providers (GitHub, Google, Auth0, WorkOS, and
others).

Authentication is **not currently wired up** in PyLumerical-MCP by default --
the preceding snippet documents the supported path rather than an existing
feature. Turning it into a first-class, configurable option (for example, an
environment variable that selects and configures an ``AuthProvider`` at
startup) is a possible follow-up.

Configure STDIO transport
~~~~~~~~~~~~~~~~~~~~~~~~~

The configuration for STDIO transport depends on your specific client. However, these general guidelines can help:

- Set the target executable to the ``ansys-lumerical-mcp`` in the environment where it is installed.
- Set the transport type to ``stdio``. The specific key can vary by client.
- Leave the argument field empty.

View additional resources
-------------------------

.. grid:: 2 2 3 3

   .. grid-item-card:: :fa:`code` Usage examples
      :link: ../examples/usage_examples
      :link-type: doc

      View example prompts for using PyLumerical-MCP to drive Lumerical tools.

   .. grid-item-card:: :fa:`book` User guide
      :link: ../user_guide/index
      :link-type: doc

      Learn how to use PyLumerical-MCP effectively.

   .. grid-item-card:: :fa:`tools` Available tools
      :link: ../api/ansys/lumerical/mcp/tools/index
      :link-type: doc

      Access references for every tool the server exposes.
