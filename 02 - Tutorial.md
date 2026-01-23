# Introduction
In part 1 we created am MCP server with simple-mcp and started running prompts with mcphost. In this pat we will explore how to run an MCP server as securely as possible. We will build 3 walls around the simple-mcp process, while allowing the mcp-server to do what it needs to do, even if it needs to behave as root in some cases.

1. Use a least privledges with minimal permissions.
2. Run the process as a systemd service that runs simple-mcp in a container way.
3. Create an SELinux profile that works at the kernel level to give simple-mcp access only to an allow list of binaries, libraries, and other files.

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

# Ring 1: Ownership and Permissions
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

# Ring 2: systemd
Next we will configure a systemd service to run the simple-mcp server. Running under systemd allows proper logging through journalctl, and other conveniences, but for our purposes, it will also mean that we can present a view of the filesystem to the process that is strictly limited to only what it needs. In this way, in the event that someone does get control of the running simple-mcp process, their access is limited. 

## Securing the Filesystem
To run as a systemd service you need to put the configuration files where a systemd service expects to find it, in the /etc/ directory, and then create a service file in the TOML format that:

 1. Describes the services to the system
 1. Defines the user and group under which to run the service
 1. The command to run on startup
 1. The file access that is needed for the service (and only the file access that is needed)
 1. Whether or not the the service is allowed new privilege (this is needed so that zypper list-updates can run as root)

Because we are running it under the mcp user, it means that all of the restrictions and permissions we configured above will apply. However, it is customary and expected that the system file will run in /etc/simple-mcp/ not in the mcp home directory. So first, we will copy the file there as root, and then grant read permissions to the mcp group as we did before.

```bash
# Create the directory
sudo mkdir -p /etc/simple-mcp

# Move the file
sudo mv /home/mcp/simple-mcp.yaml /etc/simple-mcp/

# set permissions and ownership
sudo chown root:mcp /etc/simple-mcp/simple-mcp.yaml
sudo chmod 640 /etc/simple-mcp/simple-mcp.yaml
```

After some trial and error and the assistance of gen AI, here is a service file that does what is needed, with comments inline. The following directory permissions were found to be required for running list-updates as root:
 * /run - zypper drops a file here called zypp.pid to make sure that only one instance of zypper is running. If it can't write the file, it won't run.
 * /tmp - zypper uses this for unpacking compressed files, and storing other things. Note that later we say "PrivateTmp=yes", which means that systemd gives the process its own /tmp directory and so the process can't read tmp content from other programs.
 * /var/cache/zypp - obviously where zypper caches files, like the XML and YAML from the update servers. zypper list-updates won't be able to refresh if this isn't writable.
 * /var/lib/zypp - this is where zypper writes it's dependency data when there is a refresh.
 * /var/lib/rmp - the database of installed software, lockfiles, keys, and such need to be written for zypper to operate.
 * /etc/zypp - if your Suse Customer Care Center token expires, and new one needs to be refreshed and stored here.
 * /var/log - where zypper writes its logs

 All of this is granted access in the ```ReadWritePaths``` variable. Anything outside of these paths is forbidden by ```ProtectSystem=strict```.

```bash

```toml
[Unit]
Description=Simple MCP Server
After=network.target

[Service]
User=mcp
Group=mcp

# Run the fully qualified binary pointing the config file in the standard location
ExecStart=/usr/bin/simple-mcp --config /etc/simple-mcp/simple-mcp.yaml

# Give access only to necessary parts of the file system
ProtectSystem=strict
ReadWritePaths=/var/log /var/cache/zypp /tmp /run /etc/zypp /var/lib/zypp /var/lib/rpm

# Block access to the home dir
ProtectHome=yes

# Put temporary files in a private location
PrivateTmp=yes

# zypper list-updates needs to run as root, so don't block that
NoNewPrivileges=false

[Install]
WantedBy=multi-user.target
```


Write this file to ```/etc/systemd/system/simple-mcp.service``` or use ```sudo vi /etc/systemd/system/simple-mcp.service``` and paste in the TOML.

Now it's time to run the processes under systemd.

For sanities sake, make sure that simple-mcp isn't still running:
```bash
killall simple-mcp
```

The reload the systemd daemon so that it finds the service file:
```bash
sudo systemctl daemon-reload
```

Then run the service:
```bash
sudo systemctl restart simple-mc
```

Then you can check on the status:
```bash
sudo systemctl status simple-mcp
```

You should see some output that implies that the service is running in a healthy way. For example:
```bash
● simple-mcp.service - Simple MCP Server
     Loaded: loaded (/etc/systemd/system/simple-mcp.service; disabled; preset: disabled)
     Active: active (running) since Fri 2026-01-23 17:03:09 CET; 40min ago
 Invocation: 546da938920547e49288bd8a43ee745d
   Main PID: 75285 (simple-mcp)
      Tasks: 6
        CPU: 423ms
     CGroup: /system.slice/simple-mcp.service
             └─75285 /usr/bin/simple-mcp --config /etc/simple-mcp/simple-mcp.yaml
```

Finally, we can check to ensure that our tools are still working. Here is simple-mcp-cli working with our service:

```bash
rick@sles16:/etc/simple-mcp> simple-mcp-cli tool FindIPAddress
2026/01/23 17:46:34 Connected to server: simple-mcp-server
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
       valid_lft 46555sec preferred_lft 46555sec
    inet6 fe80::7e0b:e690:d87a:d104/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
rick@sles16:/etc/simple-mcp> simple-mcp-cli tool ListAllUpdates
2026/01/23 17:46:43 Connected to server: simple-mcp-server
Refreshing service 'SUSE_Linux_Enterprise_Server_16.0_x86_64'.
Loading repository data...
Reading installed packages...
```

Notice that ListAllUpdates includes "Refreshing service", which means that command was able to ru with root.

### Testing The Containment
Let's imagine that somehow a new tool got added to the configuration that tries to read from teh home directory. Running under the mcp user as normal this would totally allowed, it's the mcp user's home dir, afterall. If you want to test out this scenario, you can add this tool to simple-mcp.yaml file:

```yaml
    - name: ListHomeDir
      description: "I should not be allowed to do this"
      command: "ls -la /home/mcp"
      parameters: []
```
Now we do the little systemctl dance to reload the service:

```bash
sudo systemctl daemon-reload
sudo systemctl restart simple-mcp
```

Thencall list tools:
```bash
simple-mcp-cli list-tools
```

As expected you can see the new tool:
```bash
2026/01/23 18:53:40 Connected to server: simple-mcp-server
FindIPAddress
GetResource
ListAllUpdates
ListHomeDir
ListPendingTasks
ListResources
SearchResources
TaskStatus
ping
```

Then we try to call the new tool that has the simple command to list the home dir:

```bash
simple-mcp-cli tool ListHomeDir
```

Aha! But it doesn't work, because system said it doesn't have access!

```bash
2026/01/23 18:53:52 Connected to server: simple-mcp-server
2026/01/23 18:53:52 Tool returned an error: Command failed: command failed: exit status 2. Output: ls: cannot access '/home/mcp': Permission denied
```

## Securing the Network
It is not possible to simply blanket deny access to the network for the mcp serer, because:
 1. The MCP server listens to port 8080 for inbound connections, though only on the local system at address 127.0.0.1.
 1. zypper list-updates requires an outbound connection to a repository mirror. 

However, it is possible to tighten up the use of the network to make it hard for an attacker to turn the mcp server into backdoor, or from moving laterally in your network. We will add some policy to the service file for this. Add the following to the service file.

```toml
# allow networking so zypper can work
PrivateNetwork=no

# only allow inbound connections for tcp and only on port 8080
SocketBindAllow=tcp:8080
SocketBindDeny=any

# stop any outbound access to anywhere in the LAN
IPAddressDeny=192.168.0.0/16
IPAddressDeny=172.16.0.0/12
IPAddressDeny=10.0.0.0/8
IPAddressDeny=fe80::/64
```

Do the little systemctl dance again, and run the tool again:
```bash
sudo systemctl daemon-reload
sudo systemctl restart simple-mcp
simple-mcp-cli tool ListAllUpdates
``` 

And you can see that the mcp server is stil working as expecting. There is a small "gotcha" here, that you should be aware of. Notice that we denied access to any ip address on the local network. It's possible that your server is configured to use a DNS server on the lan, specifically on the router. In that case, you can work through some complicated systemd rules to whitelist only the DNS server, or you can use an external DNS server, such as 8.8.8.8 or 1.1.1.1.

### Testing the Containment
Let's try another experiment to test the containment, but pretending that another tool got added. This one tries to download information from the router.

```bash
    - name: DownloadRouter
      description: "I should not be allowed to do this"
      command: "curl -v --connect-timeout 3 http://192.168.1.1"
      parameters: []
```

If you add that to your simple-mcp.yaml configuration file, do the dand and run the tool again:
```bash
sudo systemctl daemon-reload
sudo systemctl restart simple-mcp
simple-mcp-cli tool DownloadRouter
``` 

We can see that curl fails:
```bash
2026/01/23 22:41:09 Connected to server: simple-mcp-server
2026/01/23 22:41:12 Tool returned an error: Command failed: command failed: exit status 28. Output:   % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0*   Trying 192.168.1.1:80...
  0     0    0     0    0     0      0      0 --:--:--  0:00:02 --:--:--     0* Connection timed out after 3002 milliseconds
  0     0    0     0    0     0      0      0 --:--:--  0:00:03 --:--:--     0
* closing connection #0
```

So the simple-mcp process can be reached and can interact with the internet as zypper needs to, but it can't move around inside your network.

SUSE does actually support using a static ip address for cases where whitelisting specific URL's is required, but this is generally considered to not be worth the problems it causes, especially if your processes systemd services are otherwise properly restricted. For example, in this case, if the LLM or other attack vector does trick the simple-mcp process to download a payload from the internet, that payload will not be able to read any data except from specifically allowed places on the filesystem, will not be able to run a backdoor server, etc... For almost all use cases, giving up the advantages of using DNS and URLs is not worth it, though it is possible. 

# Ring 3: SELinux
Is there even more that can be done to secure the MCP server? Absolutely, yes. The next tool in the toolbox is to use "Security Enhanced Linux (more comonly, SELinux). SELinux comes preinstalled with SLES 16. The way it works is that you create an SELinux policy that tells the kernel to watch the process careful, and only allow the process access to what it should have access to. Even if the user space gets hacked by a bad actor, the kernel is still there applying the policy. Additionally, it comes with auditing and logging tools.