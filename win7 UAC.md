Win7 UAC的安全、兼容及权限

1、Win7 UAC的安全、兼容及权限
Posted on 2010-11-24 22:42 edwardlewiswe 阅读(401) 评论(0) 编辑 收藏 
网上关于这个问题讨论较多，但也不外乎几种方法。总结一下，如附中。顺便了解一个UAC。 

UAC，全称User Account Control（用户帐户控制） 

System Safe Monitor（主机入侵防御系统）

UAC是如何工作的[3]

我们可以简单的把UAC当作权限临时重分配的工具。在默认情况下，所有的非系统核心进程都只拥有标准权限，这一权限不能对系统关键区域进行修改。对于一个程序，如果它当中含有提权申请，则在运行时会弹出UAC窗口要求提权。如果用户允许，则程序暂时性的获得了最高权限，可以对系统关键区域进行更改；如果用户拒绝，则程序被拒绝执行。而如果程序中没有提权申请，则系统会让程序运行于标准权限下。同时，对于所有程序，都可以用“以管理员身份运行”的方式手动提权。而即便病毒感染了系统，它也处于UAC的监视之下，这使得病毒的反清除行为会受到很大阻碍。正是凭借这一机制，UAC成为了一道重要的系统防火墙。

在 Windows7（NT6.x系统）中，系统取消了对移动设备Autorun.inf的支持。

用户界面特权隔离[5]

在早期的Windows操作系统中，在同一用户下运行的所有进程有着相同的安全等级，拥有相同的权限。例如，一个进程可以自由地发送一个Windows消息到另外一个进程的窗口。从Windows Vista开始，当然也包括Windows 7，对于某些Windows消息，这一方式再也行不通了。进程(或者其他的对象)开始拥有一个新的属性——特权等级(Privilege Level)。一个特权等级较低的进程不再可以向一个特权等级较高的进程发送消息，虽然他们在相同的用户权限下运行。这就是所谓的用户界面特权隔离 (User Interface Privilege Isolation，UIPI)。

UIPI的引入，最大的目的是防止恶意代码发送消息给那些拥有较高权限的窗口以对其进行攻击，从而获取较高的权限等等。

UIPI的运行机制

在Windows 7中，当UAC(User Account Control)启用的时候，UIPI的运行可以得到最明显的体现。在UAC中，当一个管理员用户登录系统后，操作系统会创建两个令牌对象(Token Object)：第一个是管理员令牌，拥有大多数特权(类似于Windows Vista之前的System中的用户)，而第二个是一个经过过滤后的简化版本，只拥有普通用户的权限。

默认情况下，以普通用户权限启动的进程拥有普通特权等级(UIPI的等级划分为低等级(low)，普通(normal)，高等级(high)，系统 (system))。同样的，以管理员权限运行的进程，例如，用户右键单击选择“以管理员身份运行”或者是通过添加“runas”参数调用 ShellExecute运行的进程，这样的进程就相应地拥有一个较高(high)的特权等级。

这将导致系统会运行两种不同类型，不同特权等级的进程(当然，从技术上讲这两个进程都是在同一用户下)。我们可以使用Windows Sysinternals工具集中的进程浏览器(Process Explorer)查看各个进程的特权等级。[6]

所以，当你发现你的进程之间Windows消息通信发生问题时，不妨使用进程浏览器查看一下两个进程之间是否有合适的特权等级。

UIPI所带来的限制

正如我们前文所说，等级的划分，是为了防止以下犯上。所以，有了用户界面特权隔离，一个运行在较低特权等级的应用程序的行为就受到了诸多限制，它不可以：

►验证由较高特权等级进程创建的窗口句柄

►通过调用SendMessage和PostMessage向由较高特权等级进程创建的窗口►发送Windows消息

►使用线程钩子处理较高特权等级进程

►使用普通钩子(SetWindowsHookEx)监视较高特权等级进程

►向一个较高特权等级进程执行DLL注入

但是，一些特殊Windows消息是容许的。因为这些消息对进程的安全性没有太大影响。这些Windows消息包括：

0x000 - WM_NULL

0x003 - WM_MOVE

0x005 - WM_SIZE

0x00D - WM_GETTEXT

0x00E - WM_GETTEXTLENGTH

0x033 - WM_GETHOTKEY

0x07F - WM_GETICON

0x305 - WM_RENDERFORMAT

0x308 - WM_DRAWCLIPBOARD

0x30D - WM_CHANGECBCHAIN

0x31A - WM_THEMECHANGED

0x313, 0x31B (WM_???)

修复UIPI问题

基于Windows Vista之前的操作系统行为所设计的应用程序，可能希望Windows消息能够在进程之间自由的传递，以完成一些特殊的工作。当这些应用程序在 Windows 7上运行时，因为UIPI机制，这种消息传递被阻断了，应用程序就会遇到兼容性问题。为了解决这个问题，Windows Vista引入了一个新的API函数ChangeWindowMessageFilter[7]。利用这个函数，我们可以添加或者删除能够通过特权等级隔离的 Windows消息。这就像拥有较高特权等级的进程，设置了一个过滤器，允许通过的Windows消息都被添加到这个过滤器的白名单，只有在这个白名单上的消息才允许传递进来。

如果我们想容许一个消息可以发送给较高特权等级的进程，我们可以在较高特权等级的进程中调用ChangeWindowMessageFilter函数，以 MSGFLT_ADD作为参数将消息添加进消息过滤器的白名单。同样的，我们也可以以MSGFLT_REMOVE作为参数将这个消息从白名单中删除。

一个示例

对于系统消息的处理，接受消息的进程需要将该消息加入到白名单中，可以通过下面的代码实现：

需要在高权限程序开始的地方加入以下代码,指定什么消息可以接受

 

代码
 1 typedef BOOL (WINAPI *_ChangeWindowMessageFilter)( UINT , DWORD);
 2 
 3 BOOL CVistaMsgRecvApp::AllowMeesageForVista(UINT uMessageID, BOOL bAllow)//注册Vista全局消息
 4  {
 5 BOOL bResult = FALSE;
 6 HMODULE hUserMod = NULL;
 7  //vista and later
 8 hUserMod = LoadLibrary( L"user32.dll" );
 9 if( NULL == hUserMod )
10 {
11 return FALSE;
12 }
13 _ChangeWindowMessageFilter pChangeWindowMessageFilter = (_ChangeWindowMessageFilter)GetProcAddress( hUserMod, "ChangeWindowMessageFilter" );
14 if( NULL == pChangeWindowMessageFilter )
15 {
16 AfxMessageBox(_T("create windowmessage filter failed"));
17 return FALSE;
18 }
19 bResult = pChangeWindowMessageFilter( uMessageID, bAllow ? 1 : 2 );//MSGFLT_ADD: 1, MSGFLT_REMOVE: 2
20 if( NULL != hUserMod )
21 {
22 FreeLibrary( hUserMod );
23 }
24 return bResult;
25 }

对于自定义消息，通常是指大于WM_USER的消息，我们首先必须在系统中注册该消息，然后在调用上面的代码:
 


show sourceview sourceprint?
1	#define WM_MYNEWMESSAGE (WM_USER + 999)<BR>UINT uMsgBall=::RegisterWindowMessage (WM_MYNEWMESSAGE )<BR>if(!uMsgBall)<BR>return FALSE;<BR> 注册消息通过RegisterWindowMessage实现，函数的参数就是你需要注册的消息值。
 

此时，低等级的进程就可以像高等级的进程发送消息了。

附 Win7下如何让应用程序以管理员身份进行安装执行

1、法一：

runas /profile /env /user:mydomain\admin "mmc %windir%\system32\dsa.msc"

我感觉这种方法不靠谱。

2、法二：

通过manifest文件使VC应用程序获得管理员权限

这种方法还不错[1]。

<security>

<requestedPrivileges>

<requestedExecutionLevel level="requireAdministrator" uiAccess="false"/>

</requestedPrivileges>

</security>

3、法三：

起名为setup，win7会自己提升权限，增加manifest文件。还有一种方法是改注册表。HKEY_CURRENT_USER\Software \Microsoft\Windows NT\CurrentVersion\AppCompatFlags\Layers

添加一个字符串值 名称就是你的程序的路径和名字，值为RUNASADMIN。

也可以做成服务。

4、其它方法：

还有的网友说：有个开源的项目叫做RunAs，就是用来以指定用户来运行程序的项目，可以参考

参考网址和更多阅读

[1] http://hi.baidu.com/crowreturns/blog/item/f5e7cefd7546a284b801a07e.html

[2] http://www.cnblogs.com/sun8134/archive/2009/10/30/1593025.html

[3] 360 论坛

[4] http://topic.csdn.net/u/20100203/14/d98d8310-4971-47d1-94b1-9cdfbf159b4f.html

[5] http://blog.csdn.net/jinhill/archive/2010/07/21/5752870.aspx

[6] Sysinternals Utilities Index

http://technet.microsoft.com/en-us/sysinternals/bb545027.aspx

[7] ChangeWindowMessageFilter

http://msdn.microsoft.com/en-us/library/ms632675%28VS.85%29.aspx

[ Using the ChangeWindowMessageFilter function is not recommended, as it has process-wide scope. Instead, use the ChangeWindowMessageFilterEx function to control access to specific windows as needed. ChangeWindowMessageFilter may not be supported in future versions of Windows.]

［8］http://it.chinawin.net/softwaredev/article-b6f9.html
