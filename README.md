Oracle Forms Test Scripts
=========================

Proof-of-Concept scripts to test Oracle Forms based applications.

OracleFormsTester
-----------------

The `OracleFormsTester/` directory contains a Burp Suite extension that performs decryption, Message parsing and Scanner insertion point selection (Eclipse project).

The files outside this directory are simple utility programs that can help further develop the extension.

To use these programs first unzip the `frmall.jar` archive provided by Oracle Forms and include the resulting directory in the classpath.

* MessageTester.java - Message parsers that allows easy debugging of the deserialization process.
* OracleFormsBruteForce.java - Primitive brute-forcer for encrypted protocol messages.
* OracleFormsSyncingBruteForce.java - Primitive brute-forcer that demonstrates the attack against an out-of-sync cipher stream. 

Further information can be found in my [GWAPT Gold Paper: Automated Security Testing of Oracle Forms Applications](https://www.sans.org/reading-room/whitepapers/testing/automated-security-testing-oracle-forms-applications-35970).

OracleFormsSerializer
---------------------

**You probably want to use this**

This is a new implementation that moves encryption state away from the client and Burp. Instead of juggling with state saving and restore for every possible message we simply kill encryption from the client (so Burp can work with plaintext messages) and reimplement it in a [MitMproxy](https://github.com/mitmproxy/mitmproxy) script.

To use this extension first download `frmall.jar` and unzip it's contents. Then use your favorite Java decompiler to decompile the following classes:
* `oracle.forms.net.EncryptedInputStream`
* `oracle.forms.net.EncryptedOutputStream`

Remove all lines containing an XOR operation, like this: 

```
--- frmall_src_original/oracle/forms/net/EncryptedInputStream.java  2017-08-19 11:16:57.000000000 +0200
+++ frmall_src/oracle/forms/net/EncryptedInputStream.java   2017-08-19 11:29:56.083779579 +0200
@@ -163,7 +163,7 @@
         arrayOfInt[m] = i1; 
         int tmp184_182 = n;
         byte[] tmp184_179 = this.mBuf;
-        tmp184_179[tmp184_182] = ((byte)(tmp184_179[tmp184_182] ^ arrayOfInt[((arrayOfInt[k] + i1) % 256)]));
+        //tmp184_179[tmp184_182] = ((byte)(tmp184_179[tmp184_182] ^ arrayOfInt[((arrayOfInt[k] + i1) % 256)]));
       }   
       this.mI = k;
       this.mJ = m;
```

Then recompile the classes after moving them in their package in the unzipped (compiled) package contents (you'll probably also need to fix some minor errors generated by the decompiler):

```
javac EncryptedInputStream.java
javac EncryptedOutputStream.java
```

Update `frmall.jar` with the updated `.class` files, and delete `META-INF/ORA*` so code signing won't be a problem for JRE. Place the resulting `.jar` file to `/tmp/frmall.jar` (this is a hardcoded path in the MiMproxy script, you can edit the source if you don't like it). Also copy frmall.jar to the `lib/` directory of the OracleFormsSerializer. Then you can compile the Burp extension with the provided Ant script.

Start MitMproxy with the provided script:
```
mitmproxy -s mitmproxy_oracleforms.py -p 8081
```

The script was written for MitMproxy 1.x.x (tested with 1.0.2), other major version will not work!

Configure Burp to use the upstream proxy 127.0.0.1:8081! 

Now you can start your Oracle Form application, configured to use Burp as its proxy. The MitMproxy script will replace `frmall.jar` the client downloads with the patched one, so the client won't encrypt its messages. The OracleFormSerializer extension will then do message serialization for you. Messages will be translated to standard HTTP GET requests in the OracleForms request editor tab with String parameters provided in a query string in the Message body. If you edit these parameters the extension will automatically update the original binary Forms message appropriately (e.g. in Repeater). The extension will also register new insertion points for the Scanner so you can use that too (keep in mind that insertion points provided by Burp will probably break stuff though!).

Common Errors
-------------

* FRM-92095: Oracle Forms won't start until you convince it that Java is still owned by Sun Microsystems... Create a system wide environment variable (as described [here](https://blogs.oracle.com/ptian/solution-for-error-frm-92095:-oracle-jnitiator-version-too-low)): `JAVA_TOOL_OPTIONS='-Djava.vendor="Sun Microsystems Inc."'`
* FRM-92101: `frmall.jar` is cached by the browser so if serve a patched version and then remove the mitmproxy script for some reason (e.g. live demo at a conference...) the browser will then send an `If-Modified-Since` header to the original server so it won't serve the new (unpatched) JAR. As a result the server-side decryption won't work. You can resolve this by removing the mentioned header from the HTTP request. The cause can be of course any other problem resulting in invalid streams being decrypted by the server. 
* `ifError: 11/xxx` on server responses: These messages [instruct the client](https://community.oracle.com/docs/DOC-893120) to wait xxx milliseconds and try to send the request again. This error usually comes up when you try to send requests from multiple threads (this can happen when the legit client and some test tool are running simultaneously). Don't issue multi-threaded requestsas we only have a single keystream to work with, always use a single thread! Another possiblity is that the frequency of your requests is too high and/or the server load is too high. 

Pro Tips
--------

* If something goes wrong you'll probably want to restart MitMproxy so the objects will be reinitialized to a clean state
* Scan only with a single thread
* Disable default Scanner insertion points
