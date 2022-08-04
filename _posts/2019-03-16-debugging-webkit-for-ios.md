---
title: 'Debugging WebKit for iOS'
date: '2019-03-16T15:24:44+00:00'
author: Guy Lewin
layout: post
redirect_from:
  - /debugging-webkit-for-ios/
tags:
  - crash
  - debug
  - ios
  - lldb
  - 'unable to attach'
  - webcore
  - webkit
  - xcode
---

I found a [bug in WebKit for iOS](https://bugs.webkit.org/show_bug.cgi?id=195537) and wanted to setup a debug environment to find the exact relevant line. This was harder than I thought, so I decided to document the process.

## Collect Initial Crash Dump

I noticed after running the crashing JavaScript code, I got prompted with a "This webpage was reloaded because a problem occurred." error. I went and checked *Settings &gt; Privacy &gt; Analytics &gt; Analytics Data* and found a *com.apple.WebKit.WebContent* entry with the following crash information:

```
...
Exception Type:  EXC_BAD_ACCESS (SIGSEGV)
Exception Subtype: KERN_INVALID_ADDRESS at 0x0000000000000000
VM Region Info: 0 is not in any region.  Bytes before following region: 4372119552
      REGION TYPE                      START - END             [ VSIZE] PRT/MAX SHRMOD  REGION DETAIL
      UNUSED SPACE AT START
--->  
      __TEXT                 0000000104994000-0000000104998000 [   16K] r-x/r-x SM=COW  ...it.WebContent

Termination Signal: Segmentation fault: 11
Termination Reason: Namespace SIGNAL, Code 0xb
Terminating Process: exc handler [30816]
Triggered by Thread:  0

Thread 0 name:  Dispatch queue: com.apple.main-thread
Thread 0 Crashed:
0   WebCore                       	0x00000001b0225e78 0x1aef42000 + 19807864
1   WebCore                       	0x00000001af51f158 0x1aef42000 + 6148440
2   WebCore                       	0x00000001b039bcf8 0x1aef42000 + 21339384
3   WebCore                       	0x00000001b0322a50 0x1aef42000 + 20843088
4   WebCore                       	0x00000001b03927c4 0x1aef42000 + 21301188
5   WebCore                       	0x00000001b03099cc 0x1aef42000 + 20740556
6   WebCore                       	0x00000001b0310034 0x1aef42000 + 20766772
7   WebCore                       	0x00000001b048e35c 0x1aef42000 + 22332252
8   WebCore                       	0x00000001b040d8ec 0x1aef42000 + 21805292
9   WebCore                       	0x00000001b032325c 0x1aef42000 + 20845148
10  WebCore                       	0x00000001b0322a50 0x1aef42000 + 20843088
11  WebCore                       	0x00000001b0322f28 0x1aef42000 + 20844328
12  WebCore                       	0x00000001b0322bdc 0x1aef42000 + 20843484
13  WebCore                       	0x00000001b0323294 0x1aef42000 + 20845204
14  WebCore                       	0x00000001b0322a50 0x1aef42000 + 20843088
15  WebCore                       	0x00000001b0322f28 0x1aef42000 + 20844328
16  WebCore                       	0x00000001b0322bdc 0x1aef42000 + 20843484
17  WebCore                       	0x00000001b0323294 0x1aef42000 + 20845204
18  WebCore                       	0x00000001b0322a50 0x1aef42000 + 20843088
19  WebCore                       	0x00000001b03e6e0c 0x1aef42000 + 21646860
20  WebCore                       	0x00000001b03e4944 0x1aef42000 + 21637444
21  WebCore                       	0x00000001b03e202c 0x1aef42000 + 21626924
22  WebCore                       	0x00000001b03e1ff4 0x1aef42000 + 21626868
23  WebCore                       	0x00000001b03dfe84 0x1aef42000 + 21618308
24  WebCore                       	0x00000001b00bb610 0x1aef42000 + 18322960
25  WebCore                       	0x00000001b015b8e8 0x1aef42000 + 18979048
26  WebCore                       	0x00000001b00bb354 0x1aef42000 + 18322260
27  WebKit                        	0x00000001b629b134 0x1b6135000 + 1466676
28  WebKit                        	0x00000001b629c814 0x1b6135000 + 1472532
29  WebKit                        	0x00000001b61f1430 0x1b6135000 + 771120
30  WebCore                       	0x00000001b016734c 0x1aef42000 + 19026764
31  WebCore                       	0x00000001b0150c5c 0x1aef42000 + 18934876
32  WebCore                       	0x00000001b01ab680 0x1aef42000 + 19306112
33  CoreFoundation                	0x00000001a63e0b80 0x1a6330000 + 723840
34  CoreFoundation                	0x00000001a63e08ac 0x1a6330000 + 723116
35  CoreFoundation                	0x00000001a63e0090 0x1a6330000 + 721040
36  CoreFoundation                	0x00000001a63dad28 0x1a6330000 + 699688
37  CoreFoundation                	0x00000001a63da2e8 0x1a6330000 + 697064
38  Foundation                    	0x00000001a6dde3e0 0x1a6dd6000 + 33760
39  Foundation                    	0x00000001a6e1b1cc 0x1a6dd6000 + 283084
40  libxpc.dylib                  	0x00000001a6099d10 0x1a6085000 + 85264
41  libxpc.dylib                  	0x00000001a609c7a8 0x1a6085000 + 96168
42  com.apple.WebKit.WebContent   	0x00000001049976b4 0x104994000 + 14004
43  libdyld.dylib                 	0x00000001a5e8d050 0x1a5e8c000 + 4176
...
```

It’s clear that this is a null dereference scenario, but finding the bug in the code out of this dump would be a nightmare. So I decided to setup my WebKit for iOS environment.

## System Prerequisites

You have to have:

- Recent Xcode with iPhone SDK
- [Disable SIP](https://www.macworld.co.uk/how-to/mac/how-turn-off-mac-os-x-system-integrity-protection-rootless-3638975/)!
    - I couldn’t tell you how much time I wasted on this one. It was unclear to me that SIP on the Mac host machine would affect attaching to Safari / WebKit **inside** the iPhone simulator. Before disabling SIP I kept getting *error: attach failed: unable to attach* from lldb, and *Could not attach to pid (unable to attach)* trying to attach from Xcode. **Disable SIP solved the debugger attaching errors**

## Get the Code

Generally just follow [the Webkit guide for getting the code](https://webkit.org/getting-the-code/). All you have to run is:

```bash
svn checkout https://svn.webkit.org/repository/webkit/trunk WebKit
```

This will checkout the up-to-date code version, which is probably not very stable. Try updating it once in a while if it’s not stable until you reach a stable enough version using the SVN update command from within the WebKit directory:

```bash
svn up
```

## Compiling + Running WebKit for iOS

All iOS browsers (including Safari, Chrome, Opera, Firefox..) must use the pre-installed iOS WebKit version to get approved into the App Store. The browser used for debugging the WebKit engine on iOS is Safari and there are automated scripts ready to compile and launch everything for you. Simply follow [this guide](https://webkit.org/blog/3457/building-webkit-for-ios-simulator/) from the WebKit blog. (Be sure to pass `--ios-simulator --debug` to every WebKit script you are running)

## Better Crash Logs

Now that you have a running Safari with debug symbols - reproduce the crash. If you manage to get the browser to crash with the "This webpage was reloaded because a problem occurred." message - look at the `~/Library/Logs/DiagnosticReports/` directory on your Mac and check if a log was recently created. It’s file name should end with your Mac’s name and not with the iPhone, which could be a bit confusing. Open the crash dump and you should be able to read it much clearly:

```
...
Exception Type:        EXC_BAD_ACCESS (SIGSEGV)
Exception Codes:       KERN_INVALID_ADDRESS at 0x0000000000000000
Exception Note:        EXC_CORPSE_NOTIFY

Termination Signal:    Segmentation fault: 11
Termination Reason:    Namespace SIGNAL, Code 0xb
Terminating Process:   exc handler [64527]

VM Regions Near 0:
--> 
    __TEXT                 0000000108332000-0000000108334000 [    8K] r-x/rwx SM=COW  /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/Library/CoreSimulator/Profiles/Runtimes/iOS.simruntime/Contents/Resources/RuntimeRoot/System/Library/Frameworks/WebKit.framework/XPCServices/com.apple.WebKit.WebContent.xpc/com.apple.WebKit.WebContent

Application Specific Information:
CoreSimulator 581.2 - Device: iPhone SE For WebKit Development - Runtime: iOS 12.1 (16B91) - DeviceType: iPhone SE

Thread 0 Crashed:: Dispatch queue: com.apple.main-thread
0   com.apple.WebCore             	0x000000026a32c1f7 WebCore::GraphicsContext::platformContext() const + 7
1   com.apple.WebCore             	0x00000002695ff2fc WebCore::RenderThemeIOS::paintFileUploadIconDecorations(WebCore::RenderObject const&, WebCore::RenderObject const&, WebCore::PaintInfo const&, WebCore::IntRect const&, WebCore::Icon*, WebCore::RenderTheme::FileUploadDecorations) + 524
2   com.apple.WebCore             	0x000000026a4b323f WebCore::RenderFileUploadControl::paintObject(WebCore::PaintInfo&, WebCore::LayoutPoint const&) + 2671
3   com.apple.WebCore             	0x000000026a4343af WebCore::RenderBlock::paint(WebCore::PaintInfo&, WebCore::LayoutPoint const&) + 271
4   com.apple.WebCore             	0x000000026a4a9082 WebCore::RenderElement::paintAsInlineBlock(WebCore::PaintInfo&, WebCore::LayoutPoint const&) + 162
5   com.apple.WebCore             	0x000000026a41a687 WebCore::InlineElementBox::paint(WebCore::PaintInfo&, WebCore::LayoutPoint const&, WebCore::LayoutUnit, WebCore::LayoutUnit) + 119
6   com.apple.WebCore             	0x000000026a420cf0 WebCore::InlineFlowBox::paint(WebCore::PaintInfo&, WebCore::LayoutPoint const&, WebCore::LayoutUnit, WebCore::LayoutUnit) + 1056
7   com.apple.WebCore             	0x000000026a5ac9e2 WebCore::RootInlineBox::paint(WebCore::PaintInfo&, WebCore::LayoutPoint const&, WebCore::LayoutUnit, WebCore::LayoutUnit) + 34
8   com.apple.WebCore             	0x000000026a52a122 WebCore::RenderLineBoxList::paint(WebCore::RenderBoxModelObject*, WebCore::PaintInfo&, WebCore::LayoutPoint const&) const + 994
9   com.apple.WebCore             	0x000000026a434b97 WebCore::RenderBlock::paintObject(WebCore::PaintInfo&, WebCore::LayoutPoint const&) + 695
10  com.apple.WebCore             	0x000000026a4343af WebCore::RenderBlock::paint(WebCore::PaintInfo&, WebCore::LayoutPoint const&) + 271
11  com.apple.WebCore             	0x000000026a4347fa WebCore::RenderBlock::paintChild(WebCore::RenderBox&, WebCore::PaintInfo&, WebCore::LayoutPoint const&, WebCore::PaintInfo&, bool, WebCore::RenderBlock::PaintBlockType) + 666
12  com.apple.WebCore             	0x000000026a43452f WebCore::RenderBlock::paintChildren(WebCore::PaintInfo&, WebCore::LayoutPoint const&, WebCore::PaintInfo&, bool) + 95
13  com.apple.WebCore             	0x000000026a434bc3 WebCore::RenderBlock::paintObject(WebCore::PaintInfo&, WebCore::LayoutPoint const&) + 739
14  com.apple.WebCore             	0x000000026a4343af WebCore::RenderBlock::paint(WebCore::PaintInfo&, WebCore::LayoutPoint const&) + 271
15  com.apple.WebCore             	0x000000026a4347fa WebCore::RenderBlock::paintChild(WebCore::RenderBox&, WebCore::PaintInfo&, WebCore::LayoutPoint const&, WebCore::PaintInfo&, bool, WebCore::RenderBlock::PaintBlockType) + 666
16  com.apple.WebCore             	0x000000026a43452f WebCore::RenderBlock::paintChildren(WebCore::PaintInfo&, WebCore::LayoutPoint const&, WebCore::PaintInfo&, bool) + 95
17  com.apple.WebCore             	0x000000026a434bc3 WebCore::RenderBlock::paintObject(WebCore::PaintInfo&, WebCore::LayoutPoint const&) + 739
18  com.apple.WebCore             	0x000000026a4343af WebCore::RenderBlock::paint(WebCore::PaintInfo&, WebCore::LayoutPoint const&) + 271
19  com.apple.WebCore             	0x000000026a500033 WebCore::RenderLayer::paintForegroundForFragmentsWithPhase(WebCore::PaintPhase, WTF::Vector<WebCore::LayerFragment, 1ul, WTF::CrashOnOverflow, 16ul> const&, WebCore::GraphicsContext&, WebCore::RenderLayer::LayerPaintingInfo const&, unsigned int, WebCore::RenderObject*) + 403
20  com.apple.WebCore             	0x000000026a4fd809 WebCore::RenderLayer::paintForegroundForFragments(WTF::Vector<WebCore::LayerFragment, 1ul, WTF::CrashOnOverflow, 16ul> const&, WebCore::GraphicsContext&, WebCore::GraphicsContext&, WebCore::LayoutRect const&, bool, WebCore::RenderLayer::LayerPaintingInfo const&, unsigned int, WebCore::RenderObject*) + 393
21  com.apple.WebCore             	0x000000026a4fa829 WebCore::RenderLayer::paintLayerContents(WebCore::GraphicsContext&, WebCore::RenderLayer::LayerPaintingInfo const&, unsigned int) + 2697
22  com.apple.WebCore             	0x000000026a4fa8e2 WebCore::RenderLayer::paintLayerContents(WebCore::GraphicsContext&, WebCore::RenderLayer::LayerPaintingInfo const&, unsigned int) + 2882
23  com.apple.WebCore             	0x000000026a4f7e01 WebCore::RenderLayer::paint(WebCore::GraphicsContext&, WebCore::LayoutRect const&, WebCore::LayoutSize const&, unsigned int, WebCore::RenderObject*, unsigned int, WebCore::RenderLayer::SecurityOriginPaintPolicy) + 273
24  com.apple.WebCore             	0x000000026a1a64a7 WebCore::FrameView::paintContents(WebCore::GraphicsContext&, WebCore::IntRect const&, WebCore::Widget::SecurityOriginPaintPolicy) + 743
25  com.apple.WebCore             	0x000000026a24e5ee WebCore::ScrollView::paint(WebCore::GraphicsContext&, WebCore::IntRect const&, WebCore::Widget::SecurityOriginPaintPolicy) + 574
26  com.apple.WebCore             	0x000000026a1a619d WebCore::FrameView::traverseForPaintInvalidation(WebCore::GraphicsContext::PaintInvalidationReasons) + 253
27  com.apple.WebKit              	0x0000000108a96e8a WebKit::RemoteLayerTreeDrawingArea::flushLayers() + 340
28  com.apple.WebCore             	0x000000026a259f00 WebCore::ThreadTimers::sharedTimerFiredInternal() + 336
29  com.apple.WebCore             	0x000000026a2a75ef WebCore::timerFired(__CFRunLoopTimer*, void*) + 31
30  com.apple.CoreFoundation      	0x0000000109469f34 __CFRUNLOOP_IS_CALLING_OUT_TO_A_TIMER_CALLBACK_FUNCTION__ + 20
31  com.apple.CoreFoundation      	0x0000000109469b32 __CFRunLoopDoTimer + 1026
32  com.apple.CoreFoundation      	0x000000010946939a __CFRunLoopDoTimers + 266
33  com.apple.CoreFoundation      	0x0000000109463a1c __CFRunLoopRun + 2252
34  com.apple.CoreFoundation      	0x0000000109462e11 CFRunLoopRunSpecific + 625
35  com.apple.Foundation          	0x00000001083ee322 -[NSRunLoop(NSRunLoop) runMode:beforeDate:] + 277
36  com.apple.Foundation          	0x00000001083ee492 -[NSRunLoop(NSRunLoop) run] + 76
37  libxpc.dylib                  	0x000000010ba72812 _xpc_objc_main + 460
38  libxpc.dylib                  	0x000000010ba74cbd xpc_main + 143
39  com.apple.WebKit.WebContent   	0x00000001083332d9 0x108332000 + 4825
40  libdyld.dylib                 	0x000000010b76b575 start + 1
...
```

## Debugging using Xcode

At this point you should already know the area of code you want to put breakpoints in. For convenience reasons - I chose to use Xcode over lldb for debugging. To do that you must configure Xcode to use the command-line-built binaries instead of rebuilding it itself. Do that by following the [Debugging using Xcode section](https://webkit.org/debugging-webkit/#debugging-using-xcode) of the WebKit guide.

Now you can attach Xcode to the right WebKit process within the simulator. **Important notice that also took me a long time to figure out - WebKit has multiple different processes for every tab.** So the process ID printed when you run `Tools/Scripts/run-safari` is **not** the one you want to attach to usually. In my example, I saw my crash is in *WebCore*, so I attached to the *com.apple.WebKit.WebContent.Development* within the simulator. If my bug was related to the networking part of WebKit, I would have attached to a different process. The different process names are documented [here](https://webkit.org/debugging-webkit/#processes).

**You can distinguish the debug versions from the real stable versions by the process names -** WebKit process names that end `.Development` are the debug versions.

## Submitting the Fix

If you found and fixed the bug, follow [WebKit’s guide for submitting a patch](https://webkit.org/contributing-code/).

Good luck!