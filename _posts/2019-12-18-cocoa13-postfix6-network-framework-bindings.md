---
layout: post
title: "iOS13 PostFix #6: Fixes to Network framework bindings"
tags: ["fix", "binding", "bro-gen", "postfix"]
---
This post continues the series of fixes discovered during compilation of CocoaTouch library and improvements to compiler.
# PostFix #6: Fixes to Network framework bindings


As metntioned in [gitter channel](https://gitter.im/MobiVM/robovm?at=5de4787632df1245cbc8959a) a try to use `NWPathMonitor` fails with `ObjCClassNotFoundException: OS_nw_path_monitor`. The binding of Network framework differs from common approach in way it is completely functional interface. Most of API might be simply grouped into two groups:
- creators: CLASS INST = function_Create()
- usage: function_Usage(CLASS INST, other params)
RoboVM bro compiler allow to use such functions as class memebers, e.g.:
```
function_Usage(CLASS INST, other params)
to
class CLASS {
    @Bridge
    function_Usage( other params)
}
```

Same time bro-gen extracts types defined in framework as protocols (like `NWPathMonitor` is defined as `OS_nw_path_monitor`) and closes binding for it is a `NativeProtocolProxy`. Which shall work in general case but not this.
<!-- more -->
## Other postfixes:
* [PostFix #1: Generic class arguments and @Block parameters]({{ site.baseurl }}{% post_url 2019-10-19-cocoa13-postfix1-generic-blocks %})
* [PostFix #2: Adding support for non-static @Bridge method in enums classes]({{ site.baseurl }}{% post_url 2019-10-20-cocoa13-postfix2-non-static-bridge-methods-in-enum %})
* [PostFix #3: Support for @Block member in structs]({{ site.baseurl }}{% post_url 2019-10-21-cocoa13-postfix3-blocks-in-structs %})
* [PostFix #4: Compilation failed on @Bridge annotate covariant return synthetic method]({{ site.baseurl }}{% post_url 2019-10-22-cocoa13-postfix4-covariant-return-bridge %})
* [PostFix #5: Support for Struct.offsetOf in structs]({{ site.baseurl }}{% post_url 2019-11-17-cocoa13-postfix5-offsetof-in-structs %})
* [PostFix #7: bindings for ios13.2]({{ site.baseurl }}{% post_url 2019-12-23-cocoa13-postfix7-ios-13-2-bindings %})

## Root case
`NativeProtocolProxy` works great presenting protocols as object but in this case it fails runtime with `ObjCClassNotFoundException: OS_nw_path_monitor`. This exception means that there is no such protocol registered in objc runtime. Simple objc runtime code was used to check if protocol is present in runtime:
```objc
-(void) test:(const char *) name {
    NSLog(@"%s == %@", name, objc_getProtocol(name));
}
```
And for all protocols used in binding results are following:
```
OS_nw_object == (null)
OS_nw_advertise_descriptor == <Protocol: 0x7fff89eb9520>
OS_nw_content_context == <Protocol: 0x7fff89eba480>
OS_nw_endpoint == <Protocol: 0x7fff89ef3480>
OS_nw_error == <Protocol: 0x7fff89eba1e0>
OS_nw_interface == <Protocol: 0x7fff89eb9f40>
OS_nw_listener == <Protocol: 0x7fff89f81680>
OS_nw_parameters == <Protocol: 0x7fff89eb9a60>
OS_nw_path == <Protocol: 0x7fff89fcc4a0>
OS_nw_path_monitor == (null)
OS_nw_protocol_definition == <Protocol: 0x7fff89eb9c40>
OS_nw_protocol_metadata == <Protocol: 0x7fff89eb9b80>
OS_nw_protocol_options == <Protocol: 0x7fff89f62780>
OS_nw_protocol_stack == <Protocol: 0x7fff89eb9ac0>
OS_nw_browse_descriptor == <Protocol: 0x7fff89eba240>
OS_nw_browse_result == <Protocol: 0x7fff89f6e480>
OS_nw_browser == <Protocol: 0x7fff89eb9e80>
OS_nw_data_transfer_report == <Protocol: 0x7fff89fec660>
OS_nw_establishment_report == <Protocol: 0x7fff89eb9580>
OS_nw_ethernet_channel == (null)
OS_nw_framer == <Protocol: 0x7fff8a0124c0>
OS_nw_txt_record == <Protocol: 0x7fff89eb9760>
OS_nw_ws_request == <Protocol: 0x7fff89f94a60>
OS_nw_ws_response == <Protocol: 0x7fff89eb9b20>
```

Three protocols are missing in runtime of iOS13.3. Another simple also confirms same:
```objc
    nw_path_monitor_t monitor = nw_path_monitor_create();
    NSLog(@"%@", monitor.class);
    // output >> NWConcrete_nw_path_evaluator
    NSLog(@"%d", [monitor conformsToProtocol:@protocol(OS_nw_path_monitor)]);
    // output >> 0
```

## Workaround
As compilation time protocols are missing at runtime, these names can't be used for bindings. In general we just need to send messages to selectors defined (but not present) protocol of monitor. To achieve this each object being bind to be marked as `@NativeClass(NSObject.class)`. Its a hack way but it works as:
- each java class will consider as it NSObject behind and will not crash due objc class not found;
- calling the objc methods will cause msg_send to use used to send message to selector of instance. And its normal thing in objc;
- Network framework api is function oriented, so invoking it from java will cause function call (even not msg_send) with object instance passed as first parameter.

What will not work with this workaround: any API that returns Network objects as NSObject will not be able to find its Java class (as each java class bind to NSObject) and will marshal it to NSObject. But there is no such api in this framework.

## Other improvements
Beside the fix, Network framework bindings received following improvements:
- creator functions turned into constructors;
- funtion returning C strings wrapped with proper marshaller to return Java string.

## Source code
The fix was delivered as [PR439](https://github.com/MobiVM/robovm/pull/439)

