## iOS Basic Security Testing

In the previous chapter, we provided an overview of the iOS platform and described the structure of iOS apps. In this chapter, we'll introduce basic processes and techniques you can use to test iOS apps for security flaws. These basic processes are the foundation for the test cases outlined in the following chapters.

### iOS Testing Setup

#### Host Device

Although you can use a Linux or Windows machine for testing, you'll find that many tasks are difficult or impossible on these platforms. In addition, the Xcode development environment and the iOS SDK are only available for macOS. This means that you'll definitely want to work on macOS for source code analysis and debugging (it also makes black box testing easier).

The following is the most basic iOS app testing setup:

- ideally macOS machine with admin rights
- Wi-Fi network that permits client-to-client traffic
- at least one jailbroken iOS device (of the desired iOS version)
- Burp Suite or other interception proxy tool

##### Setting up Xcode and Command Line Tools

Xcode is an Integrated Development Environment (IDE) for macOS that contains a suite of tools for developing software for macOS, iOS, watchOS, and tvOS. You can [download Xcode for free from the official Apple website](https://developer.apple.com/xcode/ide/ "Apple Xcode IDE"). Xcode will offer you different tools and functions to interact with an iOS device that can be helpful during a penetration test, such as analyzing logs or sideloading of apps.

All development tools are already included within Xcode, but they are not available within your terminal. In order to make them available systemwide, it is recommended to install the Command Line Tools package. This will be handy during testing of iOS apps as some of the tools you will be using later (e.g. objection) are also relying on the availability of this package. You can [download it from the official Apple website](https://developer.apple.com/download/more/ "Apple iOS SDK") or install it straight away from your terminal:

```bash
$ xcode-select --install
```

#### Testing Device

##### Getting the UDID of an iOS device

The UDID is a 40-digit unique sequence of letters and numbers to identify an iOS device. You can find the [UDID of your iOS device via iTunes](http://www.iclarified.com/52179/how-to-find-your-iphones-udid "How to Find Your iPhone's UDID"), by selecting your device and clicking on "Serial Number" in the summary tab. When clicking on this you will iterate through different meta-data of the iOS device including its UDID.

It is also possible to get the UDID via the command line, from a device attached via USB. Install `ideviceinstaller` via brew and use the command `idevice_id -l`:

```bash
$ brew install ideviceinstaller
$ idevice_id -l
316f01bd160932d2bf2f95f1f142bc29b1c62dbc
```

Alternatively you can also use the Xcode command `instruments -s devices`.

##### Testing on a real device (Jailbroken)

You should have a jailbroken iPhone or iPad for running tests. These devices allow root access and tool installation, making the security testing process more straightforward. If you don't have access to a jailbroken device, you can apply the workarounds described later in this chapter, but be prepared for a more difficult experience.

##### Testing on the iOS Simulator

Unlike the Android emulator, which fully emulates the hardware of an actual Android device, the iOS SDK simulator offers a higher-level *simulation* of an iOS device. Most importantly, emulator binaries are compiled to x86 code instead of ARM code. Apps compiled for a real device don't run, making the simulator useless for black box analysis and reverse engineering.

##### Getting Privileged Access

iOS jailbreaking is often compared to Android rooting, but the process is actually quite different. To explain the difference, we'll first review the concepts of "rooting" and "flashing" on Android.

- **Rooting**: This typically involves installing the `su` binary on the system or replacing the whole system with a rooted custom ROM. Exploits aren't required to obtain root access as long as the bootloader is accessible.
- **Flashing custom ROMs**: This allows you to replace the OS that's running on the device after you unlock the bootloader. The bootloader may require an exploit to unlock it.

On iOS devices, flashing a custom ROM is impossible because the iOS bootloader only allows Apple-signed images to be booted and flashed. This is why even official iOS images can't be installed if they aren't signed by Apple, and it makes iOS downgrades only possible for as long as the previous iOS version is still signed.

The purpose of jailbreaking is to disable iOS protections (Apple's code signing mechanisms in particular) so that arbitrary unsigned code can run on the device. The word "jailbreak" is a colloquial reference to all-in-one tools that automate the disabling process.

Cydia is an alternative app store developed by Jay Freeman (aka "saurik") for jailbroken devices. It provides a graphical user interface and a version of the Advanced Packaging Tool (APT). You can easily access many "unsanctioned" app packages through Cydia. Most jailbreaks install Cydia automatically.

Since iOS 11 jailbreaks are introducing [Sileo](https://cydia-app.com/sileo/ "Sileo"), which is a new jailbreak app store for iOS devices. The jailbreak [Chimera](https://chimera.sh/ "Chimera") for iOS 12 is also relying on Sileo as a package manager.

Developing a jailbreak for a given version of iOS is not easy. As a security tester, you'll most likely want to use publicly available jailbreak tools. Still, we recommend studying the techniques that have been used to jailbreak various versions of iOS-you'll encounter many interesting exploits and learn a lot about OS internals. For example, Pangu9 for iOS 9.x [exploited at least five vulnerabilities](https://www.theiphonewiki.com/wiki/Jailbreak_Exploits "Jailbreak Exploits"), including a use-after-free kernel bug (CVE-2015-6794) and an arbitrary file system access vulnerability in the Photos app (CVE-2015-7037).

Some apps attempt to detect whether the iOS device on which they're running is jailbroken. This is because jailbreaking deactivates some of iOS' default security mechanisms. However, there are several ways to get around these detections, and we'll introduce them in the chapters "Reverse Engineering and Tampering on iOS" and "Testing Anti-Reversing Defenses on iOS."

###### Benefits of Jailbreaking

End users often jailbreak their devices to tweak the iOS system's appearance, add new features, and install third-party apps from unofficial app stores. For a security tester, however, jailbreaking an iOS device has even more benefits. They include, but aren't limited to, the following:

- root access to the file system
- possibility of executing applications that haven't been signed by Apple (which includes many security tools)
- unrestricted debugging and dynamic analysis
- access to the Objective-C or Swift runtime

###### Jailbreak Types

There are *tethered*, *semi-tethered*, *semi-untethered*, and *untethered* jailbreaks.

- Tethered jailbreaks don't persist through reboots, so re-applying jailbreaks requires the device to be connected (tethered) to a computer during every reboot. The device may not reboot at all if the computer is not connected.

- Semi-tethered jailbreaks can't be re-applied unless the device is connected to a computer during reboot. The device can also boot into non-jailbroken mode on its own.

- Semi-untethered jailbreaks allow the device to boot on its own, but the kernel patches (or user-land modifications) for disabling code signing aren't applied automatically. The user must re-jailbreak the device by starting an app or visiting a website (not requiring a connection to a computer, hence the term untethered).

- Untethered jailbreaks are the most popular choice for end users because they need to be applied only once, after which the device will be permanently jailbroken.

###### Caveats and Considerations

Jailbreaking an iOS device is becoming more and more complicated because Apple keeps hardening the system and patching the exploited vulnerabilities. Jailbreaking has become a very time-sensitive procedure because Apple stops signing these vulnerable versions relatively soon after releasing a fix (unless the jailbreak benefits from hardware-based vulnerabilities, such as the [limera1n exploit](https://www.theiphonewiki.com/wiki/Limera1n "limera1n exploit") affecting the BootROM of the iPhone 4 and iPad 1). This means that you can't downgrade to a specific iOS version once Apple stops signing the firmware.

If you have a jailbroken device that you use for security testing, keep it as is unless you're 100% sure that you can re-jailbreak it after upgrading to the latest iOS version. Consider getting one (or multiple) spare device(s) (which will be updated with every major iOS release) and waiting for a jailbreak to be released publicly. Apple is usually quick to release a patch once a jailbreak has been released publicly, so you have only a couple of days to downgrade (if it is still signed by Apple) to the affected iOS version and apply the jailbreak.

iOS upgrades are based on a challenge-response process (generating as a result the named SHSH blobs). The device will allow the OS installation only if the response to the challenge is signed by Apple. This is what researchers call a "signing window," and it is the reason you can't simply store the OTA firmware package you downloaded via iTunes and load it onto the device whenever you want to. During minor iOS upgrades, two versions may both be signed by Apple (the latest one, and the previous iOS version). This is the only situation in which you can downgrade the iOS device. You can check the current signing window and download OTA firmware from the [IPSW Downloads website](https://ipsw.me "IPSW Downloads").

###### Which Jailbreaking Tool to Use

Different iOS versions require different jailbreaking techniques. [Determine whether a public jailbreak is available for your version of iOS](https://canijailbreak.com/ "Can I Jailbreak"). Beware of fake tools and spyware, which are often hiding behind domain names that are similar to the name of the jailbreaking group/author.

The jailbreak Pangu 1.3.0 is available for 64-bit devices running iOS 9.0. If you have a device that's running an iOS version for which no jailbreak is available, you can still jailbreak the device if you downgrade or upgrade to the target _jailbreakable_ iOS version (via IPSW download and iTunes). However, this may not be possible if the required iOS version is no longer signed by Apple.

The iOS jailbreak scene evolves so rapidly that providing up-to-date instructions is difficult. However, we can point you to some sources that are currently reliable.

- [Can I Jailbreak?](https://canijailbreak.com/ "Can I Jailbreak?")
- [The iPhone Wiki](https://www.theiphonewiki.com/ "The iPhone Wiki")
- [Redmond Pie](https://www.redmondpie.com/ "Redmone Pie")
- [Reddit Jailbreak](https://www.reddit.com/r/jailbreak/ "Reddit Jailbreak")

> Note that any modification you make to your device is at your own risk. While jailbreaking is typically safe, things can go wrong and you may end up bricking your device. No other party except yourself can be held accountable for any damage.

#### Recommended Tools - iOS Device

Many tools on a jailbroken device can be installed by using Cydia, which is the unofficial AppStore for iOS devices and allows you to manage repositories. One of the most popular repositories is BigBoss, which contains various packages, such as the BigBoss Recommended Tools package. If your Cydia installation isn't pre-configured with this repository, you can add it by navigating to Sources -> Edit, then clicking "Add" in the top left and entering the following URL <http://apt.thebigboss.org/repofiles/cydia/>.

You may also want to add the HackYouriPhone repository to get the AppSync package <http://repo.hackyouriphone.org>.

You can also easily install Frida by adding the following repository to Cydia <https://build.frida.re>.

The following are some useful packages you can install from Cydia to get started:

- adv-cmds: Advanced command line. Includes finger, fingerd, last, lsvfs, md, and ps.
- AppList: Allows developers to query the list of installed apps and provides a preference pane based on the list.
- AppSync Unified: Allows you to sync and install unsigned iOS applications.
- BigBoss Recommended Tools: Installs many useful command line tools for security testing including standard Unix utilities that are missing from iOS, including wget, unrar, less, and sqlite3 client.
- Class-dump: A command line tool for examining the Objective-C runtime information stored in Mach-O files and generates header files with class interfaces.
- Class-dump-Z: A command line tool for examining the Swift runtime information stored in Mach-O files and generates header files with class interfaces.
- Clutch: Used to decrypt an app executable.
- Cycript: Is an inlining, optimizing, Cycript-to-JavaScript compiler and immediate-mode console environment that can be injected into running processes (associated to Substrate).
- [IPA Installer Console](https://cydia.saurik.com/package/com.autopear.installipa/ "IPA Installer Console"): Tool for installing IPA application packages from the command line. Package name is `com.autopear.installipa`.
- Mobile Substrate: A platform that makes developing third-party iOS add-ons easier via dynamic app manipulation or introspection.
- Needle-Agent: This agent is part of the Needle framework and need to be installed on the iOS device.
- PreferenceLoader: A Mobile Substrate-based utility that allows developers to add entries to the Settings application, similar to the SettingsBundles that App Store apps use.

Besides Cydia there are several other open source tools available and should be installed, such as [Introspy](https://github.com/iSECPartners/Introspy-iOS "Introspy-iOS").

Besides Cydia you can also ssh into your iOS device and you can install the packages directly via apt-get, like for example adv-cmds.

```bash
$ apt-get update
$ apt-get install adv-cmds
```

#### Recommended Tools - macOS Device

In order to analyse iOS apps, you should use a macOS device and install the following tools we'll be referring throughout the guide:

- [Burp Suite](https://portswigger.net/burp "Burp Suite"): Is an interception proxy that can be used to analyse the traffic between the app and the API it's talking to.
- [Frida](https://www.frida.re "Frida"): Is a runtime instrumentation framework that lets you inject JavaScript snippets or portions of your own library into native Android and iOS apps.
- [frida-ios-dump](https://github.com/AloneMonkey/frida-ios-dump "frida-ios-dump"): This tools allows you to pull a decrypted IPA from a jailbroken device.
- [Ghidra](https://ghidra-sre.org/ "Ghidra"): Is a software reverse engineering (SRE) suite of tools developed by NSA's Research Directorate.
- [IDB](https://www.idbtool.com "IDBTool"): Is an open source tool to simplify some common tasks for iOS app security assessments and research.
- [ios-deploy](https://github.com/ios-control/ios-deploy "ios-deploy"): Install and debug iPhone apps from the command line, without using Xcode. It can be installed via brew on macOS:

```bash
$ brew install ios-deploy
```

- [iFunbox](http://www.i-funbox.com/ "iFunbox"): File and app management tool that supports iOS.
- [keychain-dumper](https://github.com/mechanico/Keychain-Dumper "keychain-dumper"): A tool to check which keychain items are available to an attacker once an iOS device has been jailbroken.
- [Mobile-Security-Framework - MobSF](https://github.com/MobSF/Mobile-Security-Framework-MobSF "MobSF"):  Is an automated, all-in-one mobile application pen-testing framework that supports also iOS. The easiest way of getting MobSF started is via docker.

```bash
$ docker pull opensecurity/mobile-security-framework-mobsf
$ docker run -it -p 8000:8000 opensecurity/mobile-security-framework-mobsf:latest
```

- [Needle](https://github.com/mwrlabs/needle "Needle"): Is an all-in-one iOS security assessment framework. The [installation guide](https://github.com/mwrlabs/needle/wiki/Installation-Guide "Needle Installation Guide") in the Github wiki contains all the information needed on how to prepare your Kali Linux or macOS and how to install the Needle Agent on your iOS device.
- [Radare2](https://github.com/radare/radare2 "Radare2"): Radare2 is a complete framework for reverse-engineering and analyzing binaries.
- [TablePlus](https://tableplus.io/ "TablePlus"): Tool to inspect and analyse database files, like Sqlite and others.

##### Objection

[Objection](https://github.com/sensepost/objection "Objection on GitHub") is a "runtime mobile exploration toolkit, powered by Frida". Its main goal is to allow security testing on non-rooted or jailbroken devices through an intuitive interface.

Objection achieves this goal by providing you with the tools to easily inject the Frida gadget into an application by repackaging it. This way, you can deploy the repackaged app to the non-jailbroken device by sideloading it and interact with the application as explained in the previous section.

However, Objection also provides a REPL that allows you to interact with the application, giving you the ability to perform any action that the application can perform. A full list of the features of Objection can be found on the project's homepage, but here are a few interesting ones:

- Repackage applications to include the Frida gadget
- Disable SSL pinning for popular methods
- Access application storage to download or upload files
- Execute custom Frida scripts
- Dump the Keychain
- Read plist files

The ability to perform advanced dynamic analysis on non-jailbroken devices is one of the features that makes Objection incredibly useful. It is not always possible to jailbreak the latest version of iOS, or you may have an application with advanced jailbreak detection mechanisms. Furthermore, the included Frida scripts make it very easy to quickly analyze an application, or get around basic security controls.

Finally, in case you do have access to a jailbroken device, Objection can connect directly to the running Frida server to provide all its functionality without needing to repackage the application.

###### Installing Objection

Objection can be installed through pip as described on [Objection's Wiki](https://github.com/sensepost/objection/wiki/Installation "Objection Wiki - Installation").

```shell

$ pip3 install objection

```

If your device is jailbroken, you are now ready to interact with any application running on the device and you can skip to the "Using Objection" section below.

However, if you want to test on a non-jailbroken device, you will first need to include the Frida gadget in the application. The [Objection Wiki](https://github.com/sensepost/objection/wiki/Patching-iOS-Applications "Patching iOS Applications") describes the needed steps in detail, but after making the right preparations, you'll be able to patch an IPA by calling the objection command:

```shell
$ objection patchipa --source my-app.ipa --codesign-signature 0C2E8200Dxxxx
```

Finally, the application needs to be sideloaded and run with debugging communication enabled. Detailed steps can be found on the [Objection Wiki](https://github.com/sensepost/objection/wiki/Running-Patched-iOS-Applications "Running Patched iOS Applications"), but for macOS users it can easily be done by using ios-deploy:

```shell
$ ios-deploy --bundle Payload/my-app.app -W -d
```

###### Using Objection

Starting up Objection depends on whether you've patched the IPA or whether you are using a jailbroken device running Frida-server. For running a patched IPA, objection will automatically find any attached devices and search for a listening frida gadget. However, when using frida-server, you need to explicitly tell frida-server which application you want to analyse.

```shell
# Connecting to a patched IPA
$ objection explore

# Using frida-ps to get the correct application name
$ frida-ps -Ua | grep -i Telegram
983  Telegram

# Connecting to the Telegram app through Frida-server
$ objection --gadget="Telegram" explore
```

Once you are in the Objection REPL, you can execute any of the available commands. Below is an overview of some of the most useful ones:

```shell
# Show the different storage locations belonging to the app
$ env

# Disable popular ssl pinning methods
$ ios sslpinning disable

# Dump the Keychain
$ ios keychain dump

# Dump the Keychain, including access modifiers. The result will be written to the host in myfile.json
$ ios keychain dump --json <myfile.json>

# Show the content of a plist file
$ ios plist cat <myfile.plist>

```

More information on using the Objection REPL can be found on the [Objection Wiki](https://github.com/sensepost/objection/wiki/Using-objection "Using Objection")

### Basic Testing Operations

#### Accessing the Device Shell

One of the most common things you do when testing an app is accessing the device shell. In this section we'll see how to access the iOS shell both remotely from your host computer with/without a USB cable and locally from the device itself.

##### Remote Shell

In contrast to Android where you can easily access the device shell using the adb tool, on iOS you only have the option to access the remote shell via SSH. This also means that your iOS device must be jailbroken in order to connect to its shell from your host computer. For this section we assume that you've properly jailbroken your device and have either Cydia (see screenshot above) or Sileo installed as explained in "Getting Privileged Access". In the rest of the guide we will reference to Cydia, but the same packages should be available in Sileo.

<img src="Images/Chapters/0x06b/cydia.png" alt="iOS App Folder Structure" width="250">

In order to enable SSH access to your iOS device you can install the OpenSSH package. Once installed, be sure to connect both devices to the same Wi-Fi network and take a note of the device IP address, which you can find in the Settings -> Wi-Fi menu and tapping once on the info icon of the network you're connected to.

You can now access the remote device's shell by running `ssh root@<device_ip_address>`, which will log you in as the root user:

```shell
$ ssh root@192.168.197.234
root@192.168.197.234's password:
iPhone:~ root#
```

> press Control + D or type `exit` to quit

When accessing your iOS device via SSH consider the following:

- The default users are `root` and `mobile`.
- The default password for both is `alpine`.

> Remember to change the default password for both users `root` and `mobile` as anyone on the same network can find the IP address of your device and connect via the well-known default password, which will give them root access to your device.

If you forget your password and want to reset it to the default `alpine`:

- Edit the file `/private/etc/master.password` on your jailbroken iOS device (using an on-device shell as shown below)
- Find the lines:
  
  ```bash
  root:xxxxxxxxx:0:0::0:0:System Administrator:/var/root:/bin/sh
  mobile:xxxxxxxxx:501:501::0:0:Mobile User:/var/mobile:/bin/sh
  ```
  
- Change `xxxxxxxxx` to `/smx7MYTQIi2M`
- Save and exit

###### Connect to a Device via SSH over USB

During a real black box test, a reliable Wi-Fi connection may not be available. In this situation, you can use [usbmuxd](https://github.com/libimobiledevice/usbmuxd "usbmuxd") to connect to your device's SSH server via USB.

Usbmuxd is a socket daemon that monitors USB iPhone connections. You can use it to map the mobile device's localhost listening sockets to TCP ports on your host machine. This allows you to conveniently SSH into your iOS device without setting up an actual network connection. When usbmuxd detects an iPhone running in normal mode, it connects to the phone and begins relaying requests that it receives via `/var/run/usbmuxd`.

Connect macOS to an iOS device by installing and starting iproxy:

```shell
$ brew install libimobiledevice
$ iproxy 2222 22
waiting for connection
```

The above command maps port `22` on the iOS device to port `2222` on localhost. With the following command in a new terminal window, you can connect to the device:

```shell
$ ssh -p 2222 root@localhost
root@localhost's password:
iPhone:~ root#
```

You can also connect to your iPhone's USB via [Needle](https://labs.mwrinfosecurity.com/blog/needle-how-to/ "Needle").

##### On-device Shell App

While usually using an on-device shell (terminal emulator) might be very tedious compared to a remote shell, it can prove handy for debugging in case of, for example, network issues or check some configuration. For example, you can install [NewTerm 2](https://repo.chariz.io/package/ws.hbang.newterm2/) via Cydia for this purpose (it supports iOS 6.0 to 12.1.2 at the time of this writing).

In addition, there are a few jailbreaks that explicitly disable incoming SSH *for security reasons*. In those cases, it is very convenient to have an on-device shell app, which you can use to first SSH out of the device with a reverse shell, and then connect from your host computer to it.

Opening a reverse shell over SSH can be done by running the command `ssh -R <remote_port>:localhost:22 <username>@<host_computer_ip>`.

On the on-device shell app run the following command and, when asked, enter the password of the `mstg` user of the host computer:

```bash
ssh -R 2222:localhost:22 mstg@192.168.197.235
```

On your host computer run the following command and, when asked, enter the password of the `root` user of the iOS device:

```bash
$ ssh -p 2222 root@localhost
```

#### Host-Device Data Transfer

##### App Folder Structure

System applications are in the `/Applications` directory. You can use [IPA Installer Console](https://cydia.saurik.com/package/com.autopear.installipa "IPA Installer Console") to identify the installation folder for user-installed apps (available under `/private/var/mobile/Containers/` since iOS 9). Connect to the device via SSH and run the command `ipainstaller` (which does the same thing as `installipa`) as follows:

```shell
iPhone:~ root# ipainstaller -l
...
sg.vp.UnCrackable1

iPhone:~ root# ipainstaller -i sg.vp.UnCrackable1
...
Bundle: /private/var/mobile/Containers/Bundle/Application/A8BD91A9-3C81-4674-A790-AF8CDCA8A2F1
Application: /private/var/mobile/Containers/Bundle/Application/A8BD91A9-3C81-4674-A790-AF8CDCA8A2F1/UnCrackable Level 1.app
Data: /private/var/mobile/Containers/Data/Application/A8AE15EE-DC8B-4F1C-91A5-1FED35258D87
```

The user-installed apps have two main subdirectories (plus the `Shared` subdirectory since iOS 9):

- Bundle
- Data

The Application subdirectory, which is inside the Bundle subdirectory, contains the name of the app. The static installer files are in the Application directory, and all user data is in the Data directory.

The random string in the URI is the application's GUID. Every app installation has a unique GUID. There is no relationship between an app's Bundle GUID and its Data GUID.

##### Copying App Data Files via SSH and SCP

App files are stored in the Data directory. To identify the correct path, SSH into the device and use IPA Installer Console to retrieve the package information (as shown previously):

```shell
iPhone:~ root# ipainstaller -l
...
sg.vp.UnCrackable1

iPhone:~ root# ipainstaller -i sg.vp.UnCrackable1
Identifier: sg.vp.UnCrackable1
Version: 1
Short Version: 1.0
Name: UnCrackable1
Display Name: UnCrackable Level 1
Bundle: /private/var/mobile/Containers/Bundle/Application/A8BD91A9-3C81-4674-A790-AF8CDCA8A2F1
Application: /private/var/mobile/Containers/Bundle/Application/A8BD91A9-3C81-4674-A790-AF8CDCA8A2F1/UnCrackable Level 1.app
Data: /private/var/mobile/Containers/Data/Application/A8AE15EE-DC8B-4F1C-91A5-1FED35258D87
```

You can now simply archive the Data directory and pull it from the device with `scp`:

```shell
iPhone:~ root# tar czvf /tmp/data.tgz /private/var/mobile/Containers/Data/Application/A8AE15EE-DC8B-4F1C-91A5-1FED35258D87
iPhone:~ root# exit
$ scp -P 2222 root@localhost:/tmp/data.tgz .
```

--ToDo: <https://github.com/OWASP/owasp-mstg/issues/1245>

#### Obtaining and Extracting Apps

--ToDo: <https://github.com/OWASP/owasp-mstg/issues/1246>

##### Getting the IPA File from an OTA Distribution Link

During development, apps are sometimes provided to testers via over-the-air (OTA) distribution. In that situation, you'll receive an itms-services link, such as the following:

```http
itms-services://?action=download-manifest&url=https://s3-ap-southeast-1.amazonaws.com/test-uat/manifest.plist
```

You can use the [ITMS services asset downloader](https://www.npmjs.com/package/itms-services "ITMS services asset downloader") tool to download the IPS from an OTA distribution URL. Install it via npm:

```shell
$ npm install -g itms-services
```

Save the IPA file locally with the following command:

```shell
# itms-services -u "itms-services://?action=download-manifest&url=https://s3-ap-southeast-1.amazonaws.com/test-uat/manifest.plist" -o - > out.ipa
```

##### Acquiring the App Binary

1. From an IPA:

   If you have the IPA (probably including an already decrypted app binary), unzip it and you are ready to go. The app binary is located in the main bundle directory (.app), e.g. "Payload/Telegram X.app/Telegram X". See the following subsection for details on the extraction of the property lists.

    > On macOS's Finder, .app directories are opened by right-clicking them and selecting "Show Package Content". On the terminal you can just `cd` into them.

2. From a Jailbroken device:

    If you don't have the original IPA, then you need a jailbroken device where you will install the app (e.g. via App Store). Once installed, you need to extract the app binary from memory and rebuild the IPA file. Because of DRM, the file is encrypted when it is stored on the iOS device, so simply pulling the binary from the Bundle (either through SSH or Objection) will not be successful. The following shows the output of running class-dump on the Telegram app, which was directly pulled from the installation directory of the iPhone:

```shell
$ class-dump Telegram
//
//     Generated by class-dump 3.5 (64 bit) (Debug version compiled Jun  9 2015 22:53:21).
//
//     class-dump is Copyright (C) 1997-1998, 2000-2001, 2004-2014 by Steve Nygard.
//

#pragma mark -

//
// File: Telegram
// UUID: EAF90234-1538-38CF-85B2-91A84068E904
//
//                           Arch: arm64
//                 Source version: 0.0.0.0.0
//            Minimum iOS version: 8.0.0
//                    SDK version: 12.1.0
//
// Objective-C Garbage Collection: Unsupported
//
//                       Run path: @executable_path/Frameworks
//                               = /Frameworks
//         This file is encrypted:
//                                   cryptid: 0x00000001
//                                  cryptoff: 0x00004000
//                                 cryptsize: 0x000fc000
//
```

In order to retrieve the unencrypted version, we can use tools such as [frida-ios-dump](https://github.com/AloneMonkey/frida-ios-dump "Frida-iOS-dump") or [Clutch](https://github.com/KJCracks/Clutch "Clutch"). Both will extract the unencrypted version from memory while the application is running on the device. The stability of both Clutch and Frida can vary depending on your iOS version and Jailbreak method, so it's useful to have multiple ways for extracting the binary. In general, all iOS versions lower than 12 should work with Clutch, while iOS 12+ should work with frida-ios-dump or modify Clutch as discussed later.

###### Using Clutch

After building Clutch as explained on the Clutch GitHub page, push it to the iOS device through scp. Run Cluch with the -i flag to list all installed applications:

```shell

root# ./Clutch -i
2019-06-04 20:16:57.807 Clutch[2449:440427] command: Prints installed applications
Installed apps:
...
5:   Telegram Messenger <ph.telegra.Telegraph>
...

```

Once you have the bundle identifier, you can use Clutch to create the IPA:

```shell

root# ./Clutch -d ph.telegra.Telegraph
2019-06-04 20:19:28.460 Clutch[2450:440574] command: Dump specified bundleID into .ipa file
ph.telegra.Telegraph contains watchOS 2 compatible application. It's not possible to dump watchOS 2 apps with Clutch (null) at this moment.
Zipping Telegram.app
2019-06-04 20:19:29.825 clutch[2465:440618] command: Only dump binary files from specified bundleID
...
Successfully dumped framework TelegramUI!
Zipping WebP.framework
Zipping NotificationContent.appex
Zipping NotificationService.appex
Zipping Share.appex
Zipping SiriIntents.appex
Zipping Widget.appex
DONE: /private/var/mobile/Documents/Dumped/ph.telegra.Telegraph-iOS9.0-(Clutch-(null)).ipa
Finished dumping ph.telegra.Telegraph in 20.5 seconds

```

After copying the IPA file over to the host system and unzipping it, you can see that the Telegram application can now be parsed by class-dump, indicating that it is no longer encrypted:

```shell

$ class-dump Telegram
...
//
//     Generated by class-dump 3.5 (64 bit) (Debug version compiled Jun  9 2015 22:53:21).
//
//     class-dump is Copyright (C) 1997-1998, 2000-2001, 2004-2014 by Steve Nygard.
//

#pragma mark Blocks

typedef void (^CDUnknownBlockType)(void); // return type and parameters are unknown

#pragma mark Named Structures

struct CGPoint {
    double _field1;
    double _field2;
};
...

```

Note: when you use Clutch on iOS 12, please check [Clutch Github issue 228](https://github.com/KJCracks/Clutch/issues/228 "Getting Clutch to run on iOS 12")

###### Using Frida-ios-dump

Frida-ios-dump requires Frida server running on your jailbroken device. It is basically using Frida scripts to dump the decrypted binary from memory onto a file.
Note: before we get started, please note that the Frida-ios-dump tool is not always compatible with the latest version of Frida. Therefore: it might well be that you have to install an older version of Frida on your jailbroken device.
First, make sure that the configuration in `dump.py` is set to either localhost with port 2222 when using iProxy, or to the actual IP address and port of the device from which you want to dump the binary. Next, change the username and password to the ones you use. Now you can safely use the tool to enumerate the apps installed:

```shell
$ ./dump.py -l
 PID  Name             Identifier
----  ---------------  -------------------------------------
 860  Cydia            com.saurik.Cydia
1130  Instellingen     com.apple.Preferences
 685  Mail             com.apple.mobilemail
 834  Telegram         ph.telegra.Telegraph
   -  Aandelen         com.apple.stocks
   ...
```

and you can dump the binary you select:

```shell
$ python dump.py com.highaltitudehacks.DVIAswiftv2
/usr/local/lib/python2.7/site-packages/paramiko/kex_ecdh_nist.py:39: CryptographyDeprecationWarning: encode_point has been deprecated on EllipticCurvePublicNumbers and will be removed in a future version. Please use EllipticCurvePublicKey.public_bytes to obtain both compressed and uncompressed point encoding.
  m.add_string(self.Q_C.public_numbers().encode_point())
/usr/local/lib/python2.7/site-packages/paramiko/kex_ecdh_nist.py:96: CryptographyDeprecationWarning: Support for unsafe construction of public numbers from encoded data will be removed in a future version. Please use EllipticCurvePublicKey.from_encoded_point
  self.curve, Q_S_bytes
/usr/local/lib/python2.7/site-packages/paramiko/kex_ecdh_nist.py:111: CryptographyDeprecationWarning: encode_point has been deprecated on EllipticCurvePublicNumbers and will be removed in a future version. Please use EllipticCurvePublicKey.public_bytes to obtain both compressed and uncompressed point encoding.
  hm.add_string(self.Q_C.public_numbers().encode_point())
Start the target app ph.telegra.Telegraph
Dumping Telegram to /var/folders/0b/5j2l_v0x47j2k6dwqw5c2d040000gn/T
```

#### Installing Apps

When you install an application without using Apple's App Store, this is called sideloading. There are various ways of sideloading which are described below. On the iOS device, the actual installation process is then handled by the installd daemon, which will unpack and install the application. To integrate app services or be installed on an iOS device, all applications must be signed with a certificate issued by Apple. This means that the application can be installed only after successful code signature verification. On a jailbroken phone, however, you can circumvent this security feature with [AppSync](http://repo.hackyouriphone.org/appsyncunified "AppSync"), a package available in the Cydia store. It contains numerous useful applications that leverage jailbreak-provided root privileges to execute advanced functionality. AppSync is a tweak that patches installd, allowing the installation of fake-signed IPA packages.

Different methods exist for installing an IPA package onto an iOS device, which are described in detail below.

> Please note that since iTunes 12.7 it is not longer possible to install apps using iTunes.

##### Cydia Impactor

One tool that is available for Windows, macOS and Linux is [Cydia Impactor](http://www.cydiaimpactor.com/ "Cydia Impactor"). This tool was originally created to jailbreak iPhones, but has been rewritten to sign and install IPA packages to iOS devices via sideloading. The tool can even be used to install APK files to Android devices. A [step by step guide and troubleshooting steps can be found here](https://yalujailbreak.net/how-to-use-cydia-impactor/ "How to use Cydia Impactor").

##### libimobiledevice

On Linux and also macOS, you can alternatively use [libimobiledevice](https://www.libimobiledevice.org/ "libimobiledevice"), a cross-platform software protocol library and a set of tools for native communication with iOS devices. This allows you to install apps over a USB connection by executing ideviceinstaller. The connection is implemented with the USB multiplexing daemon [usbmuxd](https://www.theiphonewiki.com/wiki/Usbmux "Usbmux"), which provides a TCP tunnel over USB.

The package for libimobiledevice will be available in your Linux package manager. On macOS you can install libimobiledevice via brew:

```bash
$ brew install libimobiledevice
```

After the installation you have several new command line tools available, such as `ideviceinfo`, `ideviceinstaller` or `idevicedebug`.

```bash
# The following command will show detailed information about the iOS device connected via USB.
$ ideviceinfo
# The following command will install the IPA to your iOS device.
$ ideviceinstaller -i iGoat-Swift_v1.0-frida-codesigned.ipa
WARNING: could not locate iTunesMetadata.plist in archive!
WARNING: could not locate Payload/iGoat-Swift.app/SC_Info/iGoat-Swift.sinf in archive!
Copying 'iGoat-Swift_v1.0-frida-codesigned.ipa' to device... DONE.
Installing 'OWASP.iGoat-Swift'
Install: CreatingStagingDirectory (5%)
Install: ExtractingPackage (15%)
Install: InspectingPackage (20%)
Install: TakingInstallLock (20%)
Install: PreflightingApplication (30%)
Install: InstallingEmbeddedProfile (30%)
Install: VerifyingApplication (40%)
Install: CreatingContainer (50%)
Install: InstallingApplication (60%)
Install: PostflightingApplication (70%)
Install: SandboxingApplication (80%)
Install: GeneratingApplicationMap (90%)
Install: Complete
# The following command will start the app in debug mode, by providing the bundle name. The bundle name can be found in the previous command after "Installing".
$ idevicedebug -d run OWASP.iGoat-Swift
```

##### ipainstaller

The IPA can also be directly installed on the iOS device via the command line with [ipainstaller](https://github.com/autopear/ipainstaller "IPA Installer"). After copying the file over to the device, for example via scp, you can execute the ipainstaller with the IPA's filename:

```shell
$ ipainstaller App_name.ipa
```

##### ios-deploy

On macOS one more tool can be used on the command line called [ios-deploy](https://github.com/ios-control/ios-deploy "ios-deploy"), to allow installation and debugging of iOS apps from the command line. It can be installed via brew:

```bash
$ brew install ios-deploy
```

After the installation, go into the directory of the IPA you want to install and unzip it as ios-deploy installs an app by using the bundle.

```bash
$ unzip Name.ipa
$ ios-deploy --bundle 'Payload/Name.app' -W -d -v
```

After the app is installed on the iOS device, you can simply start it by adding the `-m` flag which will directly start debugging without installing the application again.

```bash
$ ios-deploy --bundle 'Payload/Name.app' -W -d -v -m
```

##### Xcode

It is also possible to use the Xcode IDE to install iOS apps by doing the following steps:

1. Start Xcode
2. Select "Window/Devices and Simulators"
3. Select the connected iOS device and click on the "+" sign in "Installed Apps".

##### Allow Application Installation on a Non-iPad Device

Sometimes an application can require to be used on an iPad device. If you only have iPhone or iPod touch devices then you can force the application to accept to be installed and used on these kinds of devices. You can do this by changing the value of the property **UIDeviceFamily** to the value **1** in the **Info.plist** file.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>

  <key>UIDeviceFamily</key>
  <array>
    <integer>1</integer>
  </array>

</dict>
</plist>  
```

It is important to note that changing this value will break the original signature of the IPA file so you need to re-sign the IPA, after the update, in order to install it on a device on which the signature validation has not been disabled.

This bypass might not work if the application requires capabilities that are specific to modern iPads while your iPhone or iPod is a bit older.

Possible values for the property [UIDeviceFamily](https://developer.apple.com/library/archive/documentation/General/Reference/InfoPlistKeyReference/Articles/iPhoneOSKeys.html#//apple_ref/doc/uid/TP40009252-SW11 "UIDeviceFamily property") can be found in the Apple Developer documentation.

#### Information Gathering

##### Installed Apps

##### App Basic Information

The following sections describes on how to retrieve basic information of an iOS app, that might be useful during a penetration test.

####### Mobile Security Framework (MobSF)

MobSF is a penetration testing framework that is capable of analysing IPA files and can be used before even installing the app on your testing device.

Once you have MobSF up and running you can open it in your browser by navigating to <http://127.0.0.1:8000>. Simply select the IPA you want to analyse and MobSF will start its job. The bigger the app the longer it takes, but usually you should get some feedback within a few minutes.

After MobSF is done with its analysis, you will receive a one-page overview of all the tests that were executed. While it may look daunting at first, the page is split up into multiple sections, each with their own purpose. Together, all the sections give a good first indication of the attack surface of the application. You can also execute additional actions, such as:

- Download a class-dump, if the app was written in Objective-C; if it is written in Swift no classdump can be created.
- Have access to the Info.plist
- Exceptions in the App Transport Security (ATS) configuration will be raised

There is much more information provided that you should explore, that might be helpful for you.

####### Objection

Once you have installed the app, there is further information to explore, where tools like objection come in handy. Connecting to the application with objection is described in the section "Recommended Tools - Objection".

When using objection you can retrieve different kinds of information, where `env` will show you all the directory information of iGoat.

```shell
OWASP.iGoat-Swift on (iPhone: 10.3.3) [usb] # env

Name               Path
-----------------  -------------------------------------------------------------------------------------------
BundlePath         /var/containers/Bundle/Application/E97D56FE-9C97-47ED-A458-5F1A3BDBE0DB/iGoat-Swift.app
CachesDirectory    /var/mobile/Containers/Data/Application/DF8806A4-F74A-4A6B-BE58-D7FDFF23F156/Library/Caches
DocumentDirectory  /var/mobile/Containers/Data/Application/DF8806A4-F74A-4A6B-BE58-D7FDFF23F156/Documents
LibraryDirectory   /var/mobile/Containers/Data/Application/DF8806A4-F74A-4A6B-BE58-D7FDFF23F156/Library
```

The directories including the UUID will be useful later for analysing the stored data for sensitive data. Other useful commands in objection to retrieve information, such as the classes used in an app, functions of classes or information about the bundle of an app can be found below:

```bash
OWASP.iGoat-Swift on (iPhone: 10.3.3) [usb] # ios hooking list classes
OWASP.iGoat-Swift on (iPhone: 10.3.3) [usb] # ios hooking list class_methods <ClassName>
OWASP.iGoat-Swift on (iPhone: 10.3.3) [usb] # ios bundles list_bundles
```

###### Passionfruit

-- ToDo Passionfruit: <https://github.com/OWASP/owasp-mstg/issues/1249>

##### Accessing App Data

###### Dumping KeyChain Data

###### Objection

-- ToDo: <https://github.com/OWASP/owasp-mstg/issues/1250>

####### Passionfruit (non-Jailbroken)

-- ToDo: <https://github.com/OWASP/owasp-mstg/issues/1250>

####### Keychain-dumper (Jailbroken)

[Keychain-dumper](https://github.com/ptoomey3/Keychain-Dumper/) lets you dump a jailbroken device's KeyChain contents. The easiest way to get the tool is to download the binary from its GitHub repo:

```shell
$ git clone https://github.com/ptoomey3/Keychain-Dumper
$ scp -P 2222 Keychain-Dumper/keychain_dumper root@localhost:/tmp/
$ ssh -p 2222 root@localhost
iPhone:~ root# chmod +x /tmp/keychain_dumper
iPhone:~ root# /tmp/keychain_dumper

(...)

Generic Password
----------------
Service: myApp
Account: key3
Entitlement Group: RUD9L355Y.sg.vantagepoint.example
Label: (null)
Generic Field: (null)
Keychain Data: SmJSWxEs

Generic Password
----------------
Service: myApp
Account: key7
Entitlement Group: RUD9L355Y.sg.vantagepoint.example
Label: (null)
Generic Field: (null)
Keychain Data: WOg1DfuH
```

In newer versions of iOS (iOS 11 and up), additional steps are necessary. See the README.md for more details.
Note that this binary is signed with a self-signed certificate that has a "wildcard" entitlement. The entitlement grants access to *all* items in the Keychain. If you are paranoid or have very sensitive private data on your test device, you may want to build the tool from source and manually sign the appropriate entitlements into your build; instructions for doing this are available in the GitHub repository.

##### Monitoring System Logs

#### Static Analysis

##### Manual Static Analysis

The preferred method of statically analyzing iOS apps involves using the original Xcode project files. Ideally, you will be able to compile and debug the app to quickly identify any potential issues with the source code.

Black box analysis of iOS apps without access to the original source code requires reverse engineering. For example, no decompilers are available for iOS apps (although most commercial and open-source disassemblers can provide a pseudo-source code view of the binary), so a deep inspection requires you to read assembly code. We won't go into too much detail of assembly code in this chapter, but we will revisit the topic in the chapter "Reverse Engineering and Tampering on iOS."

The static analysis instructions in the following chapters are based on the assumption that the source code is available.

##### Automated Static Analysis Tools

Several automated tools for analyzing iOS apps are available; most of them are commercial tools. The free and open source tools [MobSF](https://github.com/MobSF/Mobile-Security-Framework-MobSF "Mobile Security Framework (MobSF)") and [Needle](https://github.com/mwrlabs/needle "Needle") have some static and dynamic analysis functionality. Additional tools are listed in the "Static Source Code Analysis" section of the "Testing Tools" appendix.

Don't shy away from using automated scanners for your analysis - they help you pick low-hanging fruit and allow you to focus on the more interesting aspects of analysis, such as the business logic. Keep in mind that static analyzers may produce false positives and false negatives; always review the findings carefully.

#### Dynamic Analysis

##### Using Non-Jailbroken Devices

###### Injecting Frida into IPA

##### Using Jailbroken Devices

Life is easy with a jailbroken device: not only do you gain easy privileged access to the device, the lack of code signing allows you to use more powerful dynamic analysis techniques. On iOS, most dynamic analysis tools are based on Cydia Substrate, a framework for developing runtime patches that we will cover later, or Frida, a dynamic introspection tool. For basic API monitoring, you can get away with not knowing all the details of how Substrate or Frida work - you can simply use existing API monitoring tools.

###### Using Burp via USB on a Jailbroken Device

In the section "Accessing the Device Shell" we've already learned how we can use iproxy to use SSH via USB. When doing dynamic analysis, it's interesting to use the SSH connection to route our traffic to Burp that is running on our computer. Let's get started:

First we need to use iproxy to make SSH from iOS available on localhost.

```bash
$ iproxy 2222 22
waiting for connection
```

The next step is to make a remote port forwarding of port 8080 on the iOS device to the localhost interface on our computer to port 8080.

```bash
ssh -R 8080:localhost:8080 root@localhost -p 2222
```

You should now be able to reach Burp on your iOS device. Open Safari on iOS and go to 127.0.0.1:8080 and you should see the Burp Suite Page. This would also be a good time to [install the CA certificate](https://support.portswigger.net/customer/portal/articles/1841109-installing-burp-s-ca-certificate-in-an-ios-device "Installing Burp's CA Certificate in an iOS Device") of Burp on your iOS device.

The last step would be to set the proxy globally on your iOS device.

- Go to Settings
- Wi-Fi
- Connect to **any** Wi-Fi (you can literally connect to any Wi-Fi as the traffic for port 80 and 443 will be routed through USB, as we are just using the Proxy Setting for the Wi-Fi so we can set a global Proxy)
- Once connected click on the small blue icon on the right side of the connect Wi-Fi
- Configure your Proxy by selecting Manual
- Type in 127.0.0.1 as Server
- Type in 8080 as Port

Open Safari and go to any webpage, you should see now the traffic in Burp. Thanks @hweisheimer for the [initial idea](https://twitter.com/hweisheimer/status/1095383526885724161 "Port Forwarding via USB on iOS")!

###### Installing Frida

-- ToDo: <https://github.com/OWASP/owasp-mstg/issues/1251>

[Frida](https://www.frida.re "Frida") is a runtime instrumentation framework that lets you inject JavaScript snippets or portions of your own library into native Android and iOS apps. If you've already read the Android section of this guide, you should be quite familiar with this tool.

If you haven't already done so, you need to install the Frida Python package on your host machine:

```shell
$ pip install frida-tools
```

To connect Frida to an iOS app, you need a way to inject the Frida runtime into that app. This is easy to do on a jailbroken device: just install `frida-server` through Cydia. Once it has been installed, the Frida server will automatically run with root privileges, allowing you to easily inject code into any process.

Start Cydia and add Frida's repository by navigating to Manage -> Sources -> Edit -> Add and entering <https://build.frida.re.> You should then be able to find and install the Frida package.

Connect your device via USB and make sure that Frida works by running the `frida-ps` command and the flag '-U'. This should return the list of processes running on the device:

```shell
$ frida-ps -U
PID  Name
---  ----------------
963  Mail
952  Safari
416  BTServer
422  BlueTool
791  CalendarWidget
451  CloudKeychainPro
239  CommCenter
764  ContactsCoreSpot
(...)
```

We will demonstrate a few more uses for Frida below.

##### Method Tracing with Frida

Intercepting Objective-C methods is a useful iOS security testing technique. For example, you may be interested in data storage operations or network requests. In the following example, we'll write a simple tracer for logging HTTP(S) requests made via iOS standard HTTP APIs. We'll also show you how to inject the tracer into the Safari web browser.

In the following examples, we'll assume that you are working on a jailbroken device. If that's not the case, you first need to follow the steps outlined in the previous section to repackage the Safari app.

Frida comes with `frida-trace`, a ready-made function tracing tool. `frida-trace` accepts Objective-C methods via the "-m" flag. You can pass it wildcards as well-given `-[NSURL *]`, for example, `frida-trace` will automatically install hooks on all `NSURL` class selectors. We'll use this to get a rough idea about which library functions Safari calls when the user opens a URL.

Run Safari on the device and make sure the device is connected via USB. Then start `frida-trace` as follows:

```shell
$ frida-trace -U -m "-[NSURL *]" Safari
Instrumenting functions...
-[NSURL isMusicStoreURL]: Loaded handler at "/Users/berndt/Desktop/__handlers__/__NSURL_isMusicStoreURL_.js"
-[NSURL isAppStoreURL]: Loaded handler at "/Users/berndt/Desktop/__handlers__/__NSURL_isAppStoreURL_.js"
(...)
Started tracing 248 functions. Press Ctrl+C to stop.
```

Next, navigate to a new website in Safari. You should see traced function calls on the `frida-trace` console. Note that the `initWithURL:` method is called to initialize a new URL request object.

```shell
           /* TID 0xc07 */
  20313 ms  -[NSURLRequest _initWithCFURLRequest:0x1043bca30 ]
 20313 ms  -[NSURLRequest URL]
(...)
 21324 ms  -[NSURLRequest initWithURL:0x106388b00 ]
 21324 ms     | -[NSURLRequest initWithURL:0x106388b00 cachePolicy:0x0 timeoutInterval:0x106388b80
```

We can look up the declaration of this method on the [Apple Developer Website](https://developer.apple.com/documentation/foundation/nsbundle/1409352-initwithurl?language=objc "Apple Developer Website - initWithURL Instance Method"):

```objc
- (instancetype)initWithURL:(NSURL *)url;
```

The method is called with a single argument of type `NSURL`. According to the [Apple Developer documentation](https://developer.apple.com/documentation/foundation/nsurl?language=objc "Apple Developer Website - NSURL class"), the `NSRURL` class has a property called `absoluteString`, whose value should be the absolute URL represented by the `NSURL` object.

We now have all the information we need to write a Frida script that intercepts the `initWithURL:` method and prints the URL passed to the method. The full script is below. Make sure you read the code and inline comments to understand what's going on.

```python

import sys
import frida


// JavaScript to be injected
frida_code = """

    // Obtain a reference to the initWithURL: method of the NSURLRequest class
    var URL = ObjC.classes.NSURLRequest["- initWithURL"];

    // Intercept the method
    Interceptor.attach(URL.implementation, {
      onEnter: function(args) {

        // We should always initialize an autorelease pool before interacting with Objective-C APIs

        var pool = ObjC.classes.NSAutoreleasePool.alloc().init();

        var NSString = ObjC.classes.NSString;

        // Obtain a reference to the NSLog function, and use it to print the URL value
        // args[2] refers to the first method argument (NSURL *url)

        var NSLog = new NativeFunction(Module.findExportByName('Foundation', 'NSLog'), 'void', ['pointer', '...']);

        NSLog(args[2].absoluteString_());

        pool.release();
      }
    });
"""

process = frida.get_usb_device().attach("Safari")
script = process.create_script(frida_code)
script.on('message', message_callback)
script.load()

sys.stdin.read()

```

Start Safari on the iOS device. Run the above Python script on your connected host and open the device log (we'll explain how to open device logs in the following section). Try opening a new URL in Safari; you should see Frida's output in the logs.

![Frida Xcode Log](Images/Chapters/0x06b/frida-xcode-log.jpg)

Of course, this example illustrates only one of the things you can do with Frida. To unlock the tool's full potential, you should learn to use its [JavaScript API](https://www.frida.re/docs/javascript-api/ "Frida JavaScript API reference"). The documentation section of the Frida website has a [tutorial](https://www.frida.re/docs/ios/ "Frida Tutorial") and [examples](https://www.frida.re/docs/examples/ios/ "Frida examples") for using Frida on iOS.

##### Monitoring Console Logs

Many apps log informative (and potentially sensitive) messages to the console log. The log also contains crash reports and other useful information. You can collect console logs through the Xcode "Devices" window as follows:

1. Launch Xcode.
2. Connect your device to your host computer.
3. Choose Devices from the window menu.
4. Click on your connected iOS device in the left section of the Devices window.
5. Reproduce the problem.
6. Click the triangle-in-a-box toggle located in the lower left-hand corner of the Devices window's right section to view the console log's contents.

To save the console output to a text file, go to the bottom right and click the circular downward-pointing-arrow icon.

![Monitoring console logs through Xcode](Images/Chapters/0x06b/device_console.jpg)

<<<<<<< HEAD
=======
##### Bypassing Certificate Pinning

-- ToDo: <https://github.com/OWASP/owasp-mstg/issues/1252>

"[SSL Kill Switch 2](https://github.com/nabla-c0d3/ssl-kill-switch2 "SSL Kill Switch 2")" is one way to disable certificate pinning. It can be installed via the Cydia store. It will hook on to all high-level API calls and bypass certificate pinning.

The Burp Suite app "[Mobile Assistant](https://portswigger.net/burp/help/mobile_testing_using_mobile_assistant.html "Using Burp Suite Mobile Assistant")" can also be used to bypass certificate pinning.

In some cases, certificate pinning is tricky to bypass. Look for the following when you can access the source code and recompile the app:

- the API calls `NSURLSession`, `CFStream`, and `AFNetworking`
- methods/strings containing words like "pinning," "X.509," "Certificate," etc.

If you don't have access to the source, you can try binary patching or runtime manipulation:

- If OpenSSL certificate pinning is used, you can try [binary patching](https://www.nccgroup.trust/us/about-us/newsroom-and-events/blog/2015/january/bypassing-openssl-certificate-pinning-in-ios-apps/ "Bypassing OpenSSL Certificate Pinning in iOS Apps").
- Applications written with Apache Cordova or Adobe PhoneGap use a lot of callbacks. Look for the callback function that's called on success and manually call it with Cycript.
- Sometimes, the certificate is a file in the application bundle. Replacing the certificate with Burp's certificate may be sufficient, but beware the certificate's SHA sum. If it's hardcoded into the binary, you must replace it too!

Certificate pinning is a good security practice and should be used for all applications that handle sensitive information. [EFF's Observatory](https://www.eff.org/pl/observatory) lists the root and intermediate CAs that major operating systems automatically trust. Please refer to the [map of the roughly 650 organizations that are Certificate Authorities Mozilla or Microsoft trust (directly or indirectly)](https://www.eff.org/files/colour_map_of_CAs.pdf "Map of the 650-odd organizations that function as Certificate Authorities trusted (directly or indirectly) by Mozilla or Microsoft"). Use certificate pinning if you don't trust at least one of these CAs.

It is also possible to bypass SSL Pinning on non-jailbroken devices by using objection. Connecting to the application with objection is described in the section "Recommended Tools - Objection".

By using the following command in objection you can disable SSL Pinning:

```shell
$ ios sslpinning disable
```

More information on this can be found on [Objection's Wiki - Disable SSL Pinning in iOS](https://github.com/sensepost/objection#ssl-pinning-bypass-running-for-an-ios-application "Disable SSL Pinning in iOS")

If you want to get more details about white box testing and typical code patterns, refer to "iOS Application Security" by David Thiel. It contains descriptions and code snippets illustrating the most common certificate pinning techniques.

To get more information about testing transport security, please refer to the section "Testing Network Communication."

>>>>>>> master
### Setting Up a Network Testing Environment

-- ToDo <https://github.com/OWASP/owasp-mstg/issues/1271>

#### Basic Network Monitoring/Sniffing

You can remotely sniff all traffic in real-time on iOS by [creating a Remote Virtual Interface](https://stackoverflow.com/questions/9555403/capturing-mobile-phone-traffic-on-wireshark/33175819#33175819 "Wireshark + OSX + iOS") for your iOS device. First make sure you have Wireshark installed on your macOS machine.

1. Connect your iOS device to your macOS machine via USB.
1. You would need to know the UDID of your iOS device, before you can start sniffing. Check the section "Getting the UDID of an iOS device" on how to retrieve it. Open the Terminal on macOS and enter the following command, filling in the UDID of your iOS device.

```bash
$ rvictl -s <UDID>
Starting device <UDID> [SUCCEEDED] with interface rvi0
```

1. Launch Wireshark and select "rvi0" as the capture interface.
1. Filter the traffic with Capture Filters in Wireshark to display what you want to monitor (for example, all HTTP traffic sent/received via the IP address 192.168.1.1).

```shell
ip.addr == 192.168.1.1 && http
```

The documentation of Wireshark offers many examples for [Capture Filters](https://wiki.wireshark.org/CaptureFilters "Capture Filters") that should help you to filter the traffic to get the information you want.

#### Setting up an Interception Proxy

Burp Suite is an integrated platform for security testing mobile and web applications. Its tools work together seamlessly to support the entire testing process, from initial mapping and analysis of attack surfaces to finding and exploiting security vulnerabilities. Burp Proxy operates as a web proxy server for Burp Suite, which is positioned as a man-in-the-middle between the browser and web server(s). Burp Suite allows you to intercept, inspect, and modify incoming and outgoing raw HTTP traffic.

Setting up Burp to proxy your traffic is pretty straightforward. We assume that you have an iOS device and workstation connected to a Wi-Fi network that permits client-to-client traffic. If client-to-client traffic is not permitted, you can use usbmuxd to connect to Burp via USB.

PortSwigger provides a good [tutorial on setting up an iOS device to work with Burp](https://support.portswigger.net/customer/portal/articles/1841108-configuring-an-ios-device-to-work-with-burp "Configuring an iOS Device to Work With Burp") and a [tutorial on installing Burp's CA certificate to an iOS device](https://support.portswigger.net/customer/portal/articles/1841109-installing-burp-s-ca-certificate-in-an-ios-device "Installing Burp's CA Certificate in an iOS Device").

#### Potential Obstacles

### References

- UIDeviceFamily - <https://developer.apple.com/library/archive/documentation/General/Reference/InfoPlistKeyReference/Articles/iPhoneOSKeys.html#//apple_ref/doc/uid/TP40009252-SW11>

#### Tools

- Burp Suite - <https://portswigger.net/burp/communitydownload>
- Class-dump - <https://github.com/interference-security/ios-pentest-tools/blob/master/class-dump>
- Class-dump-z - <https://github.com/interference-security/ios-pentest-tools/blob/master/class-dump-z>
- Clutch - <https://github.com/KJCracks/Clutch>
- Frida - <https://www.frida.re>
- Frida-ios-dump - <https://github.com/AloneMonkey/frida-ios-dump>
- IDB - <https://www.idbtool.com>
- Introspy - <https://github.com/iSECPartners/Introspy-iOS>
- ipainstaller - <https://github.com/autopear/ipainstaller>
- iProxy - <https://iphonedevwiki.net/index.php/SSH_Over_USB>
- Keychain-dumper - <https://github.com/ptoomey3/Keychain-Dumper/>
- MobSF - <https://github.com/MobSF/Mobile-Security-Framework-MobSF>
- Needle - <https://github.com/mwrlabs/needle>
- Objection - <https://github.com/sensepost/objection>
- Reverse Engineering tools for iOS Apps - <http://iphonedevwiki.net/index.php/Reverse_Engineering_Tools>
- SSL Kill Switch 2 - <https://github.com/nabla-c0d3/ssl-kill-switch2>
- Usbmuxd - <https://github.com/libimobiledevice/usbmuxd>
- Wireshark - <https://www.wireshark.org/download.html>
- Xcode - <https://developer.apple.com/xcode/>
