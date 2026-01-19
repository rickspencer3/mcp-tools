# Introduction
This tutorial will show system administrators how to use SLES 16.0 to expose their expertise and knowledge to the LLM of their choice. The LLM can be running via ollama on the server itself, hosted somewhere else on your network, or via a cloud provider or similar provider.

This is made possible by 2 innovations from SUSE.
simple-mcp: This is a tool that exposes the Model Context Protocol, but its behavior is defined entirely in yaml, with the yaml defining the MCP interface, as well as the shell commands run in response.
mcphost: This is a tool that manages connections between multiple MCP servers and an LLM.

This tutorial assumes that you are logged into a SLES 16.0 server that has Internet access.
MCP Concepts
Here is a very quick overview of how MCP works. MCP stands for “Model Context Protocol.” It is a simple protocol that most LLMs can read. It typically provides 2 different interfaces:


 * Resources: This is relatively static information that the LLM can use as context.
 * Tools: These are commands that the LLM can run to get new information, possibly parameterized, and even to take actions directly.

When the LLM interacts with an MCP server, the MCP server sends JSON back that describes the resources and tools that are available to it.  This distinction between resources and tools can be a bit fuzzy, but that’s okay. LLMs can usually figure it out if enough information is provided via the JSON.

There is a third interface called “prompts”, but that is not much used, and is it out of scope for this tutorial.
Set Up
In order to use both mcphost and simple-mcp, we will use the science-mcp repository in SUSE’s Open Build Service. It’s a simple matter of adding the repository, and then installing the packages.

```
sudo zypper addrepo https://download.opensuse.org/repositories/science:/machinelearning:/mcp/SLE_16.0/ science-mcp
sudo zypper install mcphost simple-mcp
```

## A Quick Tour
Note that while the packages are installed, they are not running by default. You can create service files to manage their lifecycle later, but while we are developing, it is much simpler to start the services from the CLI as needed. 

simple-mcp comes with a man page, so you might want to run that and read through it to get the lay of the land.

```man simple-mcp```

For example, as you develop your own servers, the example config file may be useful:
```cat /etc/simple-mcp/simple-mcp.yaml```

# Configure
## A Note of Caution
simple-mcp is designed to be consumed by mcphost on the local machine, and so should not be directly exposed to the Internet! For this reason, always use the defaults, and always run it at 127.0.0.1 only, so no attackers can try to reach it from outside the machine! In a later tutorial we will expose mcphost in a safe way. 

## Configure simple-mcp
Now that simple-mcp and mcphost are installed, we can go ahead and configure them. We will start with some very simple example configuration. First, we’ll add a simple-mcp yaml configuration. When simple-mcp runs, it will look for a file called simple-mcp.yaml in the local directory. So create such a file, and you can add this content:

``` yaml
apiVersion: v1
kind: DynamicContextSource
metadata:
  name: simple-mcp-server
spec:
  # 1. RESOURCES (System Data)
  resources:
    - uri: "simple-mcp://system/overview"
      description: "What this MCP context source is for."
      content: |
        Simple test

    - uri: "simple-mcp://system/os-release"
      description: "OS identification."
      command: "cat /etc/os-release"
      intervalSeconds: 3600


  # 2. TOOLS (Commands) - READ ONLY
  contextItems:
    # discover ip addresses
    - name: FindIPAddress
      description: "Get the IP address of the server."
      command: "ip addr show"
      parameters: []
```

Let’s take a closer look.

``` yaml
Defining a Resource
    - uri: "simple-mcp://system/overview" # [0]
      description: "What this MCP context source is for." # [1]
      content: |
        Simple test # [2]
``` 

[0] For each resource, you provide a URI
[1] The description is very important, as this allows the LLM to know when and if to use the resource to build context
[2] The content is what is sent back to the LLM should the LLM request this resource. As you can see, the content can also be the result of a command, such as "cat /etc/os-release".

### Defining a Tool
In simple-mcp, tools are in the contextItem list. Here is a very simple example of a tool that provides information back to the LLM:

``` yaml
  contextItems:
    # discover ip addresses
    - name: FindIPAddress # [0]
      description: "Get the IP address of the server." # [1]
      command: "ip addr show" # [2]
      parameters: [] #[3]
```

[0] Instead of a URI, you provide the name of the tool. The LLM will try to understand the name. Remember that your user here is an LLM, not a human, so you can craft the name in a way that makes sense to an LLM. 
[1] Again, the description field is very important, as this will determine if the LLM believes that it can use the tool to respond to the prompts. Again, remember that your user here is an LLM, not a human.
[2] This is the command that the tool runs. The output of the command will be returned to the LLM.
[3] You can pass a list of parameters to the command as well. You can see an example of how to use this in the sample configuration file:

``` yaml
    - name: PackageVersion
      description: "Gets the installed version of a specific RPM package."
      command: "rpm -q {{.package}}"
      parameters: ["package"]
```

## Test Out Simple MCP
First, you need to run simple-mcp. Assuming that you used the default location for the configuration, and you don’t mind using the default address to run it on, just run it:

``` sh
rick@sles16:~> simple-mcp
2026/01/17 15:40:39 Configuration loaded successfully from ./simple-mcp.yaml
2026/01/17 15:40:39 Task store initialized.
2026/01/17 15:40:39 Cached 2 resource definitions.
2026/01/17 15:40:39 MCP Server simple-mcp-server with API v1 created.
2026/01/17 15:40:39 Registered built-in tool: ping
2026/01/17 15:40:39 Registered built-in tool: ListPendingTasks
2026/01/17 15:40:39 Registered built-in tool: TaskStatus
2026/01/17 15:40:39 Registered built-in tool: ListResources
2026/01/17 15:40:39 Registered built-in tool: GetResource
2026/01/17 15:40:39 Registered tool: FindIPAddress
2026/01/17 15:40:39 Registered resource: simple-mcp://system/overview (dynamic: false)
2026/01/17 15:40:39 Registered resource: simple-mcp://system/overview (dynamic: false)
2026/01/17 15:40:39 Registered resource: simple-mcp://system/os-release (dynamic: true)
2026/01/17 15:40:39 Creating Streamable HTTP server...
2026/01/17 15:40:39 MCP server starting, listening on :8080/mcp ...
```

At this point, you have an MCP server running and available locally! It is not yet exposed to an LLM, but we can still interact with it to test it out, using simple-mcp-cli. Here we list the tools, and then call one to view the output that tool will send to the LLM when it is hooked up:

``` sh
rick@sles16:~> simple-mcp-cli -server 127.0.0.1:8080 list-tools
2026/01/17 16:34:36 Connected to server: simple-mcp-server
FindIPAddress
GetResource
ListPendingTasks
ListResources
TaskStatus
ping
rick@sles16:~> simple-mcp-cli -server 127.0.0.1:8080 tool FindIPAddress
2026/01/17 16:34:41 Connected to server: simple-mcp-server
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute 
       valid_lft forever preferred_lft forever
2: enp1s0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc pfifo_fast state DOWN group default qlen 1000
    link/ether b0:41:6f:0a:08:61 brd ff:ff:ff:ff:ff:ff
    altname enxb0416f0a0861
3: wlo1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 58:1c:f8:a8:2c:99 brd ff:ff:ff:ff:ff:ff
    altname wlp2s0
    altname wlx581cf8a82c99
    inet 192.168.1.7/24 brd 192.168.1.255 scope global dynamic noprefixroute wlo1
       valid_lft 50867sec preferred_lft 50867sec
    inet6 fe80::7e0b:e690:d87a:d104/64 scope link noprefixroute
```

# Use mcphost
## Configuraiton
We now have a running mcp server, but there is no way for it to talk to an LLM. That’s where mcphost comes in. mcphost has the following responsibilities:

 1. Manage a connection with an LLM
 1. Manage authorization if you are exposing the LLM externally
 1. Manages a list of MCP servers so that an LLM can pick and choose the resources and tools from all of the available functionality. 

mcphost is easy to configure. Here is a configuration that uses my Gemini account and adds the simple-mcp server I am already running to the list of servers.

``` yaml
# MCPHost Configuration File
# All command-line flags can be configured here

# Application settings
system-prompt: "You have full access to the knowledge available to Gemini. Answer questions concisely, in a formal manner"
model: "google:gemini-2.5-flash"
provider-api-key: "<your key>"

# MCP Servers configuration
mcpServers:
  simple-mcp:
    type: "remote"
    url: "http://127.0.0.1:8080/mcp"
```

As you can see, there are 2 main sections, the overall setting, and then a list of mcp server configurations. There are many other settings available. Most likely, you will be interested in how to configure for other LLMs. 

<where to find other example configurations>
## Running mcphost
Now that mcphost is configured, we are ready to actually run it and start doing some inference! To run mcphost, we just tell it what configuration to use, and supply a prompt. For example:

```mcphost --config simple-mcp-host.yaml --prompt "what is the servers ip address?"```

And so gemini uses the tool provided to respond. mcphost prints out a data structure that includes some summary about how it ran, but also, the answer.
``` sh

 Model loaded: google (gemini-2.5-flash)                                                                                                                                                                                                                                                     MCPHost System (16:47)                                                                                                                                                                                                                                                                     

Loaded 6 tools from MCP servers                                                                                                                                                                                                                                                             MCPHost System (16:47)                                                                                                                                                                                                                                                                     

what is the servers ip address?                                                                                                                                                                                                                                                             rick (16:47) 
     Executing simple-mcp__FindIPAddress (16:47)                                                                                                                                                                                                                                             

1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000                                                                                                                                                                                            
link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00                                                                                                                                                                                                                                   inet 127.0.0.1/8 scope host lo                                                                                                                                                                                                                                                          valid_lft forever preferred_lft forever                                                                                                                                                                                                                                              inet6 ::1/128 scope host noprefixroute                                                                                                                                                                                                                                                          valid_lft forever preferred_lft forever                                                                                                                                                                                                                                        

2: enp1s0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc pfifo_fast state DOWN group default qlen 1000                                                                                                                                                                                 

link/ether b0:41:6f:0a:08:61 brd ff:ff:ff:ff:ff:ff                                                                                                                                                                                                                                      altname enxb0416f0a0861                                                                                                                                                                                                                                                                 

3: wlo1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000                                                                                                                                                                                           ... (truncated)                                                                                                                                                                                                                                                                             

The server's IP address is 192.168.1.7.                                                                                                                                                                                                                                                     

gemini-2.5-flash (16:47)
```

# Conclusion
In this part, we learned how to use configuration to create a simple MCP server, and connect it to an LLM via mcphost. In the next chapter we will learn how to:


Create tools that can allow the LLM to safely make controlled changes.   
Safely expose mcphost so it can be used by an external agent.                                                                                                                                                                                         
