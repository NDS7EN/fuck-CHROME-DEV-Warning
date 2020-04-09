# 请停用以开发者模式运行的扩展程序 正式版在使用开发者模式的弹窗警告补丁FORK 
# 方便国内用户搜索 与使用 中文机翻


## 支持的浏览器
See below for the custom paths (commandline option).
```javascript
- 支持所有 x64 and x86/x32 bit Chromium-内核浏览器,
- Chrome ✓
- Chromium
- Brave ✓
- New Edge ✓
- Opera?
- Vivaldi
- Blisk
- Colibri
- Epic Browser
- Iron Browser
- Ungoogled Chromium?
.

## 如何使用
使用该程序修改chrome.dll文件，不再显示该对话框






## Gui Screenshot
![Gui Screenshot](https://raw.githubusercontent.com/Ceiridge/Chrome-Developer-Mode-Extension-Warning-Patcher/master/media/guiscreenshot.png)

## Commandline Options (removed, use the gui instead)
All commandline options are removed. ~~**optional** and not required.~~

```bash
ChromeDevExtWarningPatcher.exe [-noDebugPatch] [-noWWWPatch] [-noWarningPatch] [-noWait] [-customPath "C:\Path"]

Explanation:
-noDebugPatch: Disables the patch for the warning of debugging extensions (chrome.debugger)
-noWWWPatch: Disables the patch for re-adding the `https` or `www` in a url, because it matters!
-noWarningPatch: Disables the patch for the warning of developer extensions

-noWait: Disables the almost-pointless wait after finishing

-customPath: Tell the patcher to use another patch to find the chrome.dll file, see below.
```

**Recommended `customPath`s:**
```java
Chrome (default): C:\Program Files (x86)\Google\Chrome\Application
Brave: C:\Program Files (x86)\BraveSoftware\Brave-Browser\Application

Remember: The path always needs to include the version folders of the browser.
Please create a new issue with a path, if you want to contribute to this list.
```

## What is the pattern and what does it patch

### Developer Extension Warning
The Chromium open source project contains a file called `dev_mode_bubble_delegate.cc` and a function called `ShouldIncludeExtension`. Here, it checks if an extension is loaded via the command line or if it is unpacked and if this is true for at least one extension, the dialog will appear.

This means that the search pattern is a signature of the function.

```c++
bool DevModeBubbleDelegate::ShouldIncludeExtension(const Extension* extension) {
  return (extension->location() == Manifest::UNPACKED ||
          extension->location() == Manifest::COMMAND_LINE);
}
```

The tool patches the enum (Manifest) values to 0xFF, so both if statements are never true.
```javascript
Manifest::UNPACKED = 0x4
Manifest::COMMAND_LINE = 0x8
```

### Debugging Warning Patch
The tool is also able to fully remove the debugging warning. You used to be able to disable it with Chrome itself with the `silent-debugger-extension-api` flag, which has been removed in Chrome 79 ( >:( ).

The Chromium open source project contains a file called `global_confirm_info_bar.cc` and a function called `MaybeAddInfoBar`, which is responsible for showing that warning and another one which is not of importance (automation warning; ChromeDriver).

This patch just instantly returns this:

```c++
void GlobalConfirmInfoBar::MaybeAddInfoBar(content::WebContents* web_contents) {
  InfoBarService* infobar_service =
      InfoBarService::FromWebContents(web_contents);
  // WebContents from the tab strip must have the infobar service.
  DCHECK(infobar_service);
  if (base::Contains(proxies_, infobar_service))
    return;

  std::unique_ptr<GlobalConfirmInfoBar::DelegateProxy> proxy(
      new GlobalConfirmInfoBar::DelegateProxy(weak_factory_.GetWeakPtr()));
  GlobalConfirmInfoBar::DelegateProxy* proxy_ptr = proxy.get();
  infobars::InfoBar* added_bar = infobar_service->AddInfoBar(
      infobar_service->CreateConfirmInfoBar(std::move(proxy)));

  // If AddInfoBar() fails, either infobars are globally disabled, or something
  // strange has gone wrong and we can't show the infobar on every tab. In
  // either case, it doesn't make sense to keep the global object open,
  // especially since some callers expect it to delete itself when a user acts
  // on the underlying infobars.
  //
  // Asynchronously delete the global object because the BrowserTabStripTracker
  // doesn't support being deleted while iterating over the existing tabs.
  if (!added_bar) {
    if (!is_closing_) {
      is_closing_ = true;

      base::SequencedTaskRunnerHandle::Get()->PostTask(
          FROM_HERE, base::BindOnce(&GlobalConfirmInfoBar::Close,
                                    weak_factory_.GetWeakPtr()));
    }
    return;
  }

  proxy_ptr->info_bar_ = added_bar;
  proxies_[infobar_service] = proxy_ptr;
  infobar_service->AddObserver(this);
}
```

### WWW/HTTPS/Elision Removal Patch
Chromium also decided to remove `www` and `https` from every url in the omnibox. You were able to disable it with a flag, which has been removed in Chrome 79 ( >:( ). This tool is also able to add these again.

The Chromium open source project contains a file called `chrome_location_bar_model_delegate.cc` and a function called `ShouldPreventElision`, which is responsible for preventing the removal of url extensions.

This is patched by always instantly returning this:

```c++
bool ChromeLocationBarModelDelegate::ShouldPreventElision() const {
#if BUILDFLAG(ENABLE_EXTENSIONS)
  Profile* const profile = GetProfile();
  return profile && extensions::ExtensionRegistry::Get(profile)
                        ->enabled_extensions()
                        .Contains(kPreventElisionExtensionId);
#else
  return false;
#endif
}
```

## How and when to run it
Run it after every chrome/chromium update with administrator rights.
