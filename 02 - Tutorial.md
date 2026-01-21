# Introduction
In part 1 we created am MCP server with simple-mcp and started running prompts with mcphost. In this part we will go a little further, and investigate how to safely allow an mcp server access to more priveledged commands and information. Typically, configuration changes and reading logs, etc... is not permitted as a normal user, but you certainly do want to be running an mcp server as root!

For example, imagine that you want your mcp server to be able to check if there are outstanding updates. Your server can run the ```zypper list-updates``` command, which works, but it will not run ```zypper refresh``` automatically as a normal user, as that requires root privledges. That means that your mcp server won't have access to the latest update information. Arguably, ```zypper refresh``` is not much of risky command to run, so we will give the mcp server the ability to run that command, and only that specifici command, as root.

## Conceptual Overview
### simple-mcp-hardenning
For example purposes, I used an LLM to generate a simple-mcp configuration that can be used by an LLM to ask questions about the security posture of a server. This configuration is called simple-mcp-hardenning.yaml.

It has the following tool for checking for out of date packages:

```yaml
    - name: ListAllUpdates
      description: "Get a list of all updates currently available"
      command: "sudo /usr/bin/zypper list-updates"
      parameters: []
```

For simplicity, add this to the tools section of simple-mcp.yaml.

# Ownership and Permissions
In order to safely run that tool as root, but also mitigate other attacks, we will set up ownership and permisions in a least privledges way. There are other prompt-injection risks that we will mitigate later when we create an SELinux profile and Systemd service file that keeps the simple-mcp process from creating new files or spawning new processes.

## Overview of the setup.
 1. Create an mcp group
 1. Create an mcp user and add it to that group
 1. Optionally, add your existing user to the mcp group
 1. Give the mcp user the capabilities it needs
 1. Put the configuration file for simple-mcp in /home/mcp
 1. Set the ownerhsip of the config file to root and give the mcp group only permission to READ the file
 1. Run simple-mcp as the mcp user

This is part of mitigating prompt injection attacks, because simple-mcp will be blocked from doing anything that you haven't given it expllicit permission to do. For example, a prompt injection can't trick the mcp server into changing its own configuration, because the server doesn't have permission to change that file.

## Step by Step
Create the mcp group:
```bash
sudo groupadd mcp
```

Then create the mcp user and them to the mcp group:
```bash
sudo useradd -r -m -g mcp -s /bin/bash mcp
```

Optionally, add yourself to the mcp group. This will make it easier to look at the same logs and such that the mcp user can see. You'll need to login and logout for this to take effect.
```bash
sudo usermod -aG mcp $USER
```

Now grant the mcp user the abilty to run the zypper list-updates command (only that command) which, when run as root, will do an implicit refresh as desired. It's generally good practive to specify the full path to the binary so the process can't be tricked into running a different binary root. Additionally, the "NOEXEC" here means that no new processes can be launched, which ensures that or shell can be launched by zypper.
```bash
echo "mcp ALL=(root) NOPASSWD:NOEXEC: /usr/bin/zypper list-updates" | sudo tee /etc/sudoers.d/mcp
```

Then copy the configuration files that you want to use to mcp user's directory.
```bash
sudo cp simple-mcp.yaml /home/mcp
```

This will write the files in the mcp user's home directory, but they will be owned by root. This is desirable, because it means that the mcp user can't be tricked into modifying the files because it doesn't own those files. However, it's not so good because the mcp user also doesn't have permissions to read those files, We fix this by granting group ownership to the mcp group with chown, and then providing read only permissions with chmod. 

```bash
sudo chown root:mcp /home/mcp/simple-mcp.yaml
sudo chmod 640 /home/mcp/simple-mcp.yaml
```

Finally, we can run simple-mcp as the mcp user. We'll fully qualify all the paths so that you know you are running the binary you think you are and using the config you think you are.
```bash
sudo -u mcp /usr/bin/simple-mcp --config /home/mcp/simple-mcp.yaml
```

Now we can see that our new tools is running.

```bash
2026/01/20 19:42:09 Configuration loaded successfully from /home/mcp/simple-mcp.yaml
2026/01/20 19:42:09 Task store initialized.
2026/01/20 19:42:09 Cached 2 resource definitions.
2026/01/20 19:42:09 MCP Server simple-mcp-server with API v1 created.
2026/01/20 19:42:09 Registered built-in tool: ping
2026/01/20 19:42:09 Registered built-in tool: ListPendingTasks
2026/01/20 19:42:09 Registered built-in tool: TaskStatus
2026/01/20 19:42:09 Registered built-in tool: ListResources
2026/01/20 19:42:09 Registered built-in tool: GetResource
2026/01/20 19:42:09 Registered built-in tool: SearchResources
2026/01/20 19:42:09 Registered tool: FindIPAddress
2026/01/20 19:42:09 Registered tool: ListAllUpdates
2026/01/20 19:42:09 Registered resource: simple-mcp://system/overview (dynamic: false)
2026/01/20 19:42:09 Registered resource: simple-mcp://system/os-release (dynamic: true)
2026/01/20 19:42:09 Creating Streamable HTTP server...
2026/01/20 19:42:09 MCP server starting, listening on localhost:8080/mcp ...
```

But is it working? Will zypper list-tools run as root? We can try poking it with simple-mcp-client using our normal user:
```bash
simple-mcp-cli tool ListAllUpdates
```

Produces the following output:
```bash
2026/01/20 19:59:49 Connected to server: simple-mcp-server
Refreshing service 'SUSE_Linux_Enterprise_Server_16.0_x86_64'.
Loading repository data...
Reading installed packages...
No updates found.
```

Success! Notice that it says "Refreshing server ... " This means that the command was run as root without the password. Does this mean that the mcp user has root? No, if you try any other command with sudo, it will get blocked awaiting a password:

```bash
sudo -u mcp sudo ls /
```

Produces the following output:
```bash
We trust you have received the usual lecture from the local System
Administrator. It usually boils down to these three things:

    #1) Respect the privacy of others.
    #2) Think before you type.
    #3) With great power comes great responsibility.

For security reasons, the password you type will not be visible.

[sudo] password for root: 
```

Without the password, the process is stopped.

Same thing if the mcp user tries to write the file:

```bash
sudo -u mcp touch /home/mcp/simple-mcp.yaml
```

You see that the mcp user cannot touch the file:
```bash
[sudo] password for rick: 
touch: cannot touch '/home/mcp/simple-mcp.yaml': Permission denied
```

But it can read it. The following will print the contents of the file:
```bash
sudo -u mcp cat /home/mcp/simple-mcp.yaml
```

But does mcphost work? Let's try it:
```bash
mcphost --config simple-mcp-host.yaml --prompt "are all of the packages up to date?"
```

And it works!
```bash
    Executing simple-mcp__ListAllUpdates (23:21)                                                     
    
    Refreshing service 'SUSE_Linux_Enterprise_Server_16.0_x86_64'.
    Loading repository data...
    Reading installed packages...
    No updates found.

    All packages are up to date.
        gemini-2.5-flash (23:21)
  ```

