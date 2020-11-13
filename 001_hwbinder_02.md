# HIDL示例--C++服务创建与Client验证

## 1. hal文件创建及环境准备

### 1.1 创建demo目录

mkdir -p vendor/lixd/interfaces/demo/1.0/default

### 1.2 创建IDemo.hal文件，填充内容

vim vendor/lixd/interfaces/demo/1.0/IDemo.hal

写一个接口为getHelloString，传入参数类型为string，返回值generates 也为string类型:
```
package vendor.lixd.demo@1.0;
interface IDemo {
    getHelloString(string name) generates (string result);
};
```

### 1.3 给demo配置一个Android.bp

vim vendor/lixd/interfaces/demo/Android.bp

下面的hidl_package_root 用来指向hal的目录，hidl编译时，需要用到该变量，内容如下：

```
subdirs = [
    "*"
]

hidl_package_root {
    name: "vendor.lixd.demo",
    path: "vendor/lixd/interfaces/demo",
}
```

### 1.4 编译hidl-gen 工具

make -j16 hidl-gen

我们的Android工程一般都编译过了，不需要再执行此命令。

### 1.5 hidl-gen相关执行过程

#### 1.5.1 制作一个shell脚本

vim vendor/lixd/interfaces/demo/update-all.sh

code:
```

#!/bin/bash

HAL_PACKAGES=(
    "vendor.lixd.demo@1.0"
)

HAL_PACKAGE_ROOT=vendor.lixd.demo
HAL_PATH_ROOT=vendor/lixd/interfaces/demo
HAL_OUTPUT_CODE_PATH=vendor/lixd/interfaces/demo/1.0/default/
HASH_FILE=$HAL_PATH_ROOT/current.txt

for pkg in "${HAL_PACKAGES[@]}"
do
    echo "Generating hash for $pkg interface"
    echo "" >> $HASH_FILE
    hidl-gen -L hash -r $HAL_PACKAGE_ROOT:$HAL_PATH_ROOT -r android.hardware:hardware/interfaces -r android.hidl:system/libhidl/transport $pkg  >> $HASH_FILE

    echo "Updating $pkg Android.bp"
    hidl-gen -L androidbp -r $HAL_PACKAGE_ROOT:$HAL_PATH_ROOT -r android.hidl:system/libhidl/transport $pkg

    echo "Updating $pkg code's Android.bp"
    hidl-gen -o $HAL_OUTPUT_CODE_PATH -Landroidbp-impl -r $HAL_PACKAGE_ROOT:$HAL_PATH_ROOT -randroid.hidl:system/libhidl/transport $pkg
    echo "Updating $pkg code"
    hidl-gen -o $HAL_OUTPUT_CODE_PATH -Lc++-impl -r $HAL_PACKAGE_ROOT:$HAL_PATH_ROOT -randroid.hidl:system/libhidl/transport $pkg
done
```

上面这个脚本，生成四个文件：

* hal文件对应的hash--current.txt //哈希是一种旨在防止意外更改接口并确保接口更改经过全面审查的机制

* hal文件对应的Android.bp

* default中demo对应代码的Android.bp

* default中demo的代码--Demo.cpp、Demo.h

此时的目录结构如下：
```
vendor/lixd/interfaces/demo/
├── 1.0
│   ├── default
│   └── IDemo.hal
├── Android.bp
└── update-all.sh
```

#### 1.5.2 执行shell脚本

./vendor/lixd/interfaces/demo/update-all.sh

执行结果：
```
Generating hash for vendor.lixd.demo@1.0 interface
Updating vendor.lixd.demo@1.0 Android.bp
Updating vendor.lixd.demo@1.0 code's Android.bp
Updating vendor.lixd.demo@1.0 code
```

命令执行后，会在vendor/lixd/interfaces/demo/1.0/中生成Android.bp，在vendor/lixd/interfaces/demo/1.0/default/中生成源码Demo.cpp、Demo.h，此时的目录结构如下：
```
vendor/lixd/interfaces/demo/
├── 1.0
│   ├── Android.bp
│   ├── default
│   │   ├── Android.bp
│   │   ├── Demo.cpp
│   │   └── Demo.h
│   └── IDemo.hal
├── Android.bp
├── current.txt
└── update-all.sh
```

#### 1.5.3 编译hal

mmm vendor/lixd/interfaces/demo/1.0/

编译完成后，生成如下文件：

1） Android 的jar包，供JAVA层使用

out/.../product/framework/vendor.lixd.demo-V1.0-java.jar

out/.../product/framework/vendor.lixd.demo-V1.0-java-shallow.jar

2） 系统库so，供Native层调用- C++

out/.../product/lib/vendor.lixd.demo@1.0-adapter-helper.so

out/.../product/lib/vendor.lixd.demo@1.0.so

out/.../product/lib/vendor.lixd.demo@1.0-vts.driver.so

out/.../product/lib/vendor.lixd.demo@1.0-vts.profiler.so

## 2. Demo服务实现

### 2.1 Demo的接口实现

#### 2.1.1 Demo.h

由[1.5.2]的脚本自动生成，不需要做特殊处理
```
// FIXME: your file license if you have one

#pragma once

#include <vendor/lixd/demo/1.0/IDemo.h>
#include <hidl/MQDescriptor.h>
#include <hidl/Status.h>

namespace vendor {
namespace lixd {
namespace demo {
namespace V1_0 {
namespace implementation {

using ::android::hardware::hidl_array;
using ::android::hardware::hidl_memory;
using ::android::hardware::hidl_string;
using ::android::hardware::hidl_vec;
using ::android::hardware::Return;
using ::android::hardware::Void;
using ::android::sp;

struct Demo : public IDemo {
    // Methods from ::vendor::lixd::demo::V1_0::IDemo follow.
    Return<void> getHelloString(const hidl_string& name, getHelloString_cb _hidl_cb) override;

    // Methods from ::android::hidl::base::V1_0::IBase follow.

};

// FIXME: most likely delete, this is only for passthrough implementations
// extern "C" IDemo* HIDL_FETCH_IDemo(const char* name);

}  // namespace implementation
}  // namespace V1_0
}  // namespace demo
}  // namespace lixd
}  // namespace vendor

```

#### 2.1.1 Demo.cpp

Demo.cpp也是由[1.5.2]的脚本自动生成，不需要做特殊处理
```
// FIXME: your file license if you have one

#include "Demo.h"

namespace vendor {
namespace lixd {
namespace demo {
namespace V1_0 {
namespace implementation {

// Methods from ::vendor::lixd::demo::V1_0::IDemo follow.
Return<void> Demo::getHelloString(const hidl_string& name, getHelloString_cb _hidl_cb) {
    // TODO implement

    //实现函数具体内容, 这里是自己添加的
    char buf[100];
    ::memset(buf, 0x00, 100);
    ::snprintf(buf, 100, "Hello , %s", name.c_str());
    hidl_string result(buf);
    _hidl_cb(result);
    //实现函数具体内容，这里是自己添加的

    return Void();
}


// Methods from ::android::hidl::base::V1_0::IBase follow.

//IDemo* HIDL_FETCH_IDemo(const char* /* name */) {
    //return new Demo();
//}
//
}  // namespace implementation
}  // namespace V1_0
}  // namespace demo
}  // namespace lixd
}  // namespace vendor
```

#### 2.1.3 创建服务程序Service.cpp

vim vendor/lixd/interfaces/demo/1.0/default/Service.cpp

code:
```
#include <android-base/logging.h>
#include <hidl/HidlTransportSupport.h>
#include <vendor/lixd/demo/1.0/IDemo.h>

#include <hidl/LegacySupport.h>
#include "Demo.h"
using android::hardware::configureRpcThreadpool;
using android::hardware::joinRpcThreadpool;
using vendor::lixd::demo::V1_0::implementation::Demo;

int main() {
    //1.和"dev/hwbinder" 进行通信，设置最大的线程个数为4
    configureRpcThreadpool(4, true);

    Demo demo;
    //2.注册服务
    auto status = demo.registerAsService();
    CHECK_EQ(status, android::OK) << "Failed to register demo HAL implementation";

    //3.把当前线程加入到线程池
    joinRpcThreadpool();
    return 0;
}
```

#### 2.1.4 添加demo service启动的rc文件

vim vendor/lixd/interfaces/demo/1.0/default/vendor.lixd.demo@1.0-service.rc

code:
```
service vendor_lixd_demo /vendor/bin/hw/vendor.lixd.demo@1.0-service
    class hal
    user  system
    group system
```

#### 2.1.5修改default的Android.bp

编译service服务

code：
```
cc_binary {
    name: "vendor.lixd.demo@1.0-service",
    relative_install_path: "hw",
    defaults: ["hidl_defaults"],
    proprietary: true,
    init_rc: ["vendor.lixd.demo@1.0-service.rc"],
    srcs: [
           "Service.cpp",
          ],
    shared_libs: [
        "libbase",
        "liblog",
        "libdl",
        "libutils",
        "libhardware",
        "libhidlbase",
        "libhidltransport",
        "vendor.lixd.demo@1.0",
        "vendor.lixd.demo@1.0-impl",
    ],
}
```

#### 2.1.6 配置

为了让服务可以被客户端访问到，需要添加manifest

vim vendor/lixd/interfaces/demo/1.0/default/manifest.xml

code:
```
<manifest version="1.0" type="device">
    <hal format="hidl">
        <name>vendor.lixd.demo</name>
        <transport>hwbinder</transport>
        <version>1.0</version>
        <interface>
            <name>IDemo</name>
            <instance>default</instance>
        </interface>
    </hal>
</manifest>
```

创建一个 moduleconfig.mk ，把manifest.xml的内容加入到系统的manifext.xml

vim vendor/lixd/interfaces/demo/1.0/default/moduleconfig.mk

code:
```
$(eval LOCAL_MK_PATH := $(lastword $(MAKEFILE_LIST)))
$(eval DEVICE_MANIFEST_FILE += $(dir $(LOCAL_MK_PATH))manifest.xml)
$(warning DEVICE_MANIFEST_FILE = $(DEVICE_MANIFEST_FILE))
```

整个系统版本编译时，最终可以在/vendor/etc/vintf/manifest.xml中，看到我们这里添加的manifest.xml里面的内容。

#### 2.1.7 编译

mmm vendor/lixd/interfaces/demo/1.0/default/

生成文件：

1）服务进程

out/.../vendor/bin/hw/vendor.lixd.demo@1.0-service

2）rc文件：

out/.../vendor/etc/init/vendor.lixd.demo@1.0-service.rc

3）implement库

out/.../vendor/lib/hw/vendor.lixd.demo@1.0-impl.so

## 3.Native层Client进行测试

### 3.1 创建Client源文件

mkdir -p vendor/lixd/hal_demo/cpp/

vim vendor/lixd/hal_demo/cpp/hal_demo_test.cpp

code:
```
#include <android-base/logging.h>
#include <hidl/HidlTransportSupport.h>
#include <vendor/lixd/demo/1.0/IDemo.h>
#include <hidl/LegacySupport.h>

using vendor::lixd::demo::V1_0::IDemo;
using android::sp;
using android::hardware::hidl_string;

int main() {
    //1.获取到Demo的服务对象
    android::sp<IDemo> service = IDemo::getService();

    if(service == nullptr) {
        printf("Failed to get service\n");
        return -1;
    }

    //2.通过服务对象，获取服务的getHelloString() hidl接口实现
    service->getHelloString("lixd", [&](hidl_string result) {
                printf("%s\n", result.c_str());
        });

    return 0;
}
```

### 3.2 配置编译环境

vim vendor/lixd/hal_demo/cpp/Android.bp

code:
```
cc_binary {
    name: "hal_demo_test",
    relative_install_path: "hw",
    defaults: ["hidl_defaults"],
    proprietary: true,
    srcs: [
           "hal_demo_test.cpp",
          ],
    shared_libs: [
        "libbase",
        "liblog",
        "libdl",
        "libutils",
        "libhardware",
        "libhidlbase",
        "libhidltransport",
        "vendor.lixd.demo@1.0",
    ],
}
```

### 3.3 编译

mmm vendor/lixd/hal_demo/cpp/

生成文件：

out\...\vendor\bin\hw\hal_demo_test

### 3.4 验证方法：

手动把vendor/etc/vintf/manifest.xml pull到本地，增加如下内容，再push到机器的vendor/etc/vintf/中。

manifest.xml新增内容：
```
<hal format="hidl">
    <name>vendor.lixd.demo</name>
    <transport>hwbinder</transport>
    <version>1.0</version>
    <interface>
        <name>IDemo</name>
        <instance>default</instance>
    </interface>
    <fqname>@1.0::IDemo/demo</fqname>
</hal>
```

push 以下文件:
```
vendor/bin/hw/hal_demo_test
vendor/bin/hw/vendor.lixd.demo@1.0-service
vendor/etc/init/vendor.lixd.demo@1.0-service.rc
vendor/lib/vendor.lixd.demo@1.0.so
vendor/lib/hw/vendor.lixd.demo@1.0-impl.so
product/lib/vendor.lixd.demo@1.0.so
```

以上步骤完成后，重启机器。

服务端执行：

/vendor/bin/hw # ./vendor.lixd.demo@1.0-service  &

客户端执行：
/vendor/bin/hw # ./hal_demo_test

输出结果：
Hello , lixd
