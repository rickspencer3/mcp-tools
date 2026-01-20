# Introduction
In part 1 we created am MCP server with simple-mcp and started running prompts with mcphost. In this part we will go a little further, and investigate how to safely allow an mcp server access to more priveledged commands and information. Typically, configuration changes and reading logs, etc... is not permitted as a normal user, but you certainly do want to be running an mcp server as root!

For example, imagine that you want your mcp server to be able to check if there are outstanding updates. Your server can run the ```zypper list-updates``` command, which works, but it will not run ```zypper refresh``` automatically as a normal user, as that requires root privledges. That means that your mcp server won't have access to the latest update information. Arguably, ```zypper refresh``` is not much of risky command to run, so we will give the mcp server the ability to run that command, and only that specifici command, as root.

## Conceptual Overview
### simple-mcp-hardenning
For example purposes, I used an LLM to generate a simple-mcp configuration that can be used by an LLM to ask questions about the security posture of a server. This configuration is called simple-mcp-hardenning.yaml.

It has the following tool for checking for out of date packages:

```yaml
    # discover ip addresses
    - name: ListAllUpdates
      description: "Get a list of all updates currently available"
      command: "zypper list-updates"
      parameters: []
```

## Ownership and Permissions
In order to run that tool as root, we will do the following:

 1. Create an mcp group
 1. Create an mcp user and add it to that group
 1. Optionally, add your existing user to the mcp group
 1. Give the mcp user the capabilities it needs
 1. Put the configuration files for simple-mcp in /home/mcp
 1. Set the ownerhsip of the config files to root and give the mcp user only permission to READ the file
 1. Run simple-mcp and mcphost as the mcp user

In this way, your mcp-server is protected from prompt injection attacks, because it will be blocked from doing anything that you haven't given it expllicit permission to do. For example, a prompt injection can't trick the mcp server into changing its own configuration, because the server doesn't have permission to change that file.

There are other prompt-injection risks that we will mitigate later when we create an SELinux profile and Systemd service file that keeps the simple-mcp process from creating new files or spawning new processes.

