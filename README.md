<div align="center">

## Security and Permissions in \.NET


</div>

### Description

The .NET Framework provides a rich set of permissions and security settings that (when used properly) ensures that your programs only do what their allowed to do. After searching for articles on security and permissions and finding virtually nothing here I decided to create this tutorial to explain the basic concepts. This simple tutorial explains how the Demand and Assert methods work. It also teach's you how to create a simple code group that grants a specific assembly more access to the system. Remember to vote!
 
### More Info
 


<span>             |<span>
---                |---
**Submitted On**   |
**By**             |[David N](https://github.com/Planet-Source-Code/PSCIndex/blob/master/ByAuthor/david-n.md)
**Level**          |Intermediate
**User Rating**    |5.0 (20 globes from 4 users)
**Compatibility**  |C\#
**Category**       |[Security](https://github.com/Planet-Source-Code/PSCIndex/blob/master/ByCategory/security__10-14.md)
**World**          |[\.Net \(C\#, VB\.net\)](https://github.com/Planet-Source-Code/PSCIndex/blob/master/ByWorld/net-c-vb-net.md)
**Archive File**   |[](https://github.com/Planet-Source-Code/david-n-security-and-permissions-in-net__10-3819/archive/master.zip)





### Source Code

<font face="Verdana" size="2">
After noticing virtually nothing on Planet-Source-Code.com about .NET security I decided to write this small tutorial.<br>
<br>
The .NET Framework's security infrastructure is designed to prevent malicious code from gaining access to protected system resources such as the file system or registry. Its
ability to do so lies a lot on the use of permissions. The base assemblies provided in the .NET Framework already make use of permissions for you. This makes sure that your
application can only do what its given permission to do.<br>
<br>
The .NET Framework assigns each assembly a set of permissions that it must adhere to when its loaded. You can look at how it determines what permissions go to what assemblies in
the “.NET Configuration” (Look in “Administrative Tools” in the Control Panel). But for the most part assemblies are granted permissions based on where they came from (Local
computer, Network share, Internet, etc.). You can create a special permission set for assemblies that require more access to the system. But doing so can expose system resources
to malicious code if your not careful.<br>
<br>
This simple tutorial will guide you through the process to gaining access to system resources (in a class library) while running with restricted permissions (in the client
application).<br>
<br>
<b>SITUATION</b>:<br>
Assembly: MyLib.dll<br>
<ul>
<li> Needs access to the file system.</li>
<li> Needs to prevent malicious code from gaining access to the file system.</li>
<ul>
<li> This depends on the design and use of protected system resources within this library.</li>
</ul>
</ul>
Source Code:
<font size="3">
<pre>
<font color="blue">using</font> System;
<font color="blue">using</font> System.IO;
<font color="blue">using</font> System.Security;
<font color="blue">using</font> System.Security.Permissions;
<font color="blue">namespace</font> MyLib {
  <font color="blue">public class</font> MyFileIO {
    <font color="blue">public static int</font> GetFileCount(<font color="blue">string</font> path) {
      <font color="blue">return</font> Directory.GetFiles(path).Length;
    }
  }
}
</pre>
</font>
Assembly: Client.exe
<ul>
<li> Needs to use the MyLib.dll assembly but doesn't need access to the file system directly.</li>
<li> Always running under the LocalIntranet permission set. (Which doesn't allow direct file system access.)</li>
</ul>
Source Code:
<font size="3">
<pre>
<font color="blue">using</font> System;
<font color="blue">using</font> System.Windows.Forms;
<font color="blue">using</font> MyLib;
<font color="blue">namespace</font> Client {
  <font color="blue">public class</font> SimpleUI : Form {
    <font color="blue">public</font> SimpleUI() {
      <font color="blue">this</font>.Text = "Client Application";
      Button btnTest = <font color="blue">new</font> Button();
      btnTest.Text = "Test";
      btnTest.Dock = DockStyle.Fill;
      btnTest.Click += <font color="blue">new</font> EventHandler(<font color="blue">this</font>.OnTest);
      <font color="blue">this</font>.Controls.Add(btnTest);
    }
    <font color="blue">private void</font> OnTest(<font color="blue">object</font> sender, EventArgs e) {
      MessageBox.Show(MyFileIO.GetFileCount("C:\\").ToString());
    }
    <font color="blue">private static void</font> Main(<font color="blue">string</font>[] args) {
      Application.Run(<font color="blue">new</font> SimpleUI());
    }
  }
}
</pre>
</font>
Given the situation you need to give the MyLib.dll extra permissions because the LocalIntranet permission set doesn't allow file system access. You can use the .NET Configuration
utility to do this fairly quickly and easily:<br><br>
<font color="red"><b>WARNING</b>: These are “use at your own risk” instructions.</font><br>
<ol>
<li> Open the “.NET Configuration” (Look in “Administrative Tools” from the Control Panel)</li>
<li> Expand to “Runtime Security Policy\Machine\Code Groups\All_Code\LocalIntranet_Zone”</li>
<li> Click on “LocalIntranet_Zone” and then “Add a Child Code Group”</li>
<li> Give the code group a name: “MyLib Security” (and description if you want.) Click Next.</li>
<li> Change the membership condition to “Hash” (Cheap and simple) and then click “Browse...” and select the MyLib.dll assembly. Click Next.</li>
<li> Select “FullTrust” (You should create a custom permission set to allow minimum access to system resources but this is easier for now) Click Finish.</li>
</ol>
<b>NOTE</b>: To execute your application in the LocalIntranet permission set you have to setup a network share (it <b>can</b> be on the local computer) and then copy the two
assemblies there (MyLib.dll and Client.exe). Then click Start->Run and type something like "\\%COMPUTERNAME%\(share_name)\client.exe". <br>
<b>NOTE:</b>: You can also use my "Secure Execution of .NET Scripts" program to execute this in a restricted permission set. This will require copying the MyLib.dll into the
same folder as that program though.<br>
<br>
Now that MyLib.dll has full access to the system it can freely access the file system. But there are problems when Client.exe tries to use MyLib.dll to access the file system.
Why? Because in Directory.GetFiles() there is something like this:
<font size="3">
<pre>
<font color="green">// Demand PathDiscovery access to the given path.</font>
<font color="blue">new</font> FileIOPermission(FileIOPermissionAccess.PathDiscovery, path).Demand();
</pre>
</font>
<b>NOTE</b>: The Directory.GetFiles() method is <b>only</b> demanding "PathDiscovery" on the path.<br>
<br>
The <b>Demand</b> method is implemented in <i>System.Security.CodeAccessPermission</i> which is also the base class for all permission classes in the .NET Framework. The
<b>Demand</b> method here throws a <i>SecurityException</i> if <b>any</b> of the calling assemblies don't have access to the file system (In this case MyLib.dll has permission but
Client.exe doesn't). So whats the point of giving permissions to MyLib.dll if it doesn't allow Client.exe access to the file system through it? Well thats where the <b>Assert</b>
method comes into play. When the <b>Demand</b> method is used it walks the call stack up until it reachs the Main method or an <b>Assert</b> method. When an <b>Assert</b> method
is encountered the security system in .NET stops looking higher in the call stack (The call stack grows down so the Main method is at the top of the call stack while the
Directory.GetFiles() method is at or near the bottom). So in this case we can place an <b>Assert</b> method in the MyLib.dll so that the <b>Demand</b> in the Directory.GetFiles()
method won't fail. Here's the new source code for MyLib.dll:
<font size="3">
<pre>
<font color="blue">using</font> System;
<font color="blue">using</font> System.IO;
<font color="blue">using</font> System.Security;
<font color="blue">using</font> System.Security.Permissions;
<font color="blue">namespace</font> MyLib {
  <font color="blue">public class</font> MyFileIO {
    <font color="blue">public static int</font> GetFileCount(<font color="blue">string</font> path) {
      <font color="green">// begins a new "frame" with access to the file system.</font>
      <font color="blue">new</font> FileIOPermission(FileIOPermissionAccess.PathDiscovery, path).Assert();
      <font color="blue">int</font> nFileCount = Directory.GetFiles(path).Length;
      <font color="green">// required by .NET to end the "frame"</font>
      CodeAccessPermission.RevertAssert();
      <font color="blue">return</font> nFileCount;
    }
  }
}
</pre>
</font>
<b>NOTE</b>: We only need permission to "PathDiscovery" on this specific path so thats all we're asserting. Asserting more access than this could make us that much more
vulnerable to attack.<br>
<b>NOTE</b>: You have to update your code group in the .NET Configuration with the new hash.
<ol>
<li> Click “MyLib Security” and click “Edit Code Group Properties”</li>
<li> Goto the “Membership Condition” tab.</li>
<li> Click “Import...” and select the newly compiled assembly.</li>
<li> Click “OK”</li>
</ol>
The <i>CodeAccessPermission.RevertAssert()</i> is required as specified in the .NET documentation (Also note that you can only have one <b>Assert</b> per “frame”). By calling
<b>Assert</b> your telling the security system to ignore Client.exe's permission to access the file system (This is where the CAUTION in the .NET documentation for this method
comes into play: <font color="red">“<b>CAUTION</b>: Because calling <b>Assert</b> removes the requirement that <b>all</b> code in the call chain must be granted permission to
access the specified resource, it can open up security vulnerabilities if used incorrectly or inappropriately. Therefore, it should be used with great caution.”</font>).<br>
<br>
Now the Client.exe can use the MyLib.dll without any security exceptions being thrown.<br>
<br>
As for the other methods in the <i>System.Security.CodeAccessPermission</i> class:
<ul>
<li> Deny() : Makes sure that code surrounded by Deny() and RevertDeny() <b>doesn't</b> have access to the permission.</li>
<li> PermitOnly() : Makes sure that code surround by PermitOnly() and ReveryPermitOnly() <b>only</b> has access to the permission.</li>
</ul>
The same methods have been implemented in the <i>System.Security.PermissionSet</i> class so that you can Assert, Deny, Demand, or PermitOnly an entire set of permissions.<br>
<br>
Now that you have a basic understanding of how to give an assembly more access to system resources and allow assemblies with less permissions indirect access to those resources
you can develope more functional and secure code. Just remember that the security of the system depends mainly on trusted assemblies and their use of system resources (Along with
their use of <b>Assert</b>). <b>Please vote!</b>
</font>

