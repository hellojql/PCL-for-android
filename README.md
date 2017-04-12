# PCL android移植

## 1.编译

### 1.1 环境:`MacOS 10.11.4` + `ndk-r8c`
### 1.2 流程如下：

```
wget http://dl.google.com/android/ndk/android-ndk-r8c-darwin-x86.tar.bz2
tar -jxvf android-ndk-r8c-darwin-x86.tar.bz2
cd android-ndk-r8c
export ANDROID_NDK=$PWD
cd ~work
git clone https://github.com/hirotakaster/pcl-superbuild.git
cd pcl-superbuild
mkdir build
cd build
cmake ../
make -j4 # j4 is prallel build option

```

### 1.3 编译完后，生成如下文件给`android`引用

```
pcl-superbuild/build/CMakeExternals/Install/boost-android/
pcl-superbuild/build/CMakeExternals/Install/flann-android/
pcl-superbuild/build/CMakeExternals/Install/pcl-android/
pcl-superbuild/build/CMakeExternals/Install/engine/

```

## 2. android studio 引用 PCL + OpenCV



### 2.1 Android.mk 文件如下:

```
LOCAL_PATH :=$(call my-dir)


# for PCL library   :begin

PCL_INCLUDE := /Users/netease/PCL/pcl-superbuild/build/CMakeExternals/Install/pcl-android
BOOST_ANDROID_INCLUDE := /Users/netease/PCL/pcl-superbuild/build/CMakeExternals/Install/boost-android
FLANN_INCLUDE := /Users/netease/PCL/pcl-superbuild/build/CMakeExternals/Install/flann-android
EIGEN_INCLUDE := /Users/netease/PCL/pcl-superbuild/build/CMakeExternals/Install/eigen

PCL_STATIC_LIB_DIR := $(PCL_INCLUDE)/lib
BOOST_STATIC_LIB_DIR := $(BOOST_ANDROID_INCLUDE)/lib
FLANN_STATIC_LIB_DIR := $(FLANN_INCLUDE)/lib

PCL_STATIC_LIBRARIES :=     pcl_common pcl_geometry pcl_kdtree pcl_octree pcl_sample_consensus pcl_surface \
                            pcl_features pcl_keypoints pcl_search pcl_tracking pcl_filters pcl_ml \
                            pcl_registration pcl_segmentation pcl_io pcl_io_ply pcl_recognition
BOOST_STATIC_LIBRARIES :=   boost_date_time boost_iostreams boost_regex boost_system \
                            boost_filesystem boost_program_options boost_signals boost_thread
FLANN_STATIC_LIBRARIES :=   flann_s flann_cpp_s


define build_pcl_static
    include $(CLEAR_VARS)
    LOCAL_MODULE:=$1
    LOCAL_SRC_FILES:=$(PCL_STATIC_LIB_DIR)/lib$1.a
    include $(PREBUILT_STATIC_LIBRARY)
endef

define build_boost_static
    include $(CLEAR_VARS)
    LOCAL_MODULE:=$1
    LOCAL_SRC_FILES:=$(BOOST_STATIC_LIB_DIR)/lib$1.a
    include $(PREBUILT_STATIC_LIBRARY)
endef

define build_flann_static
    include $(CLEAR_VARS)
    LOCAL_MODULE:=$1
    LOCAL_SRC_FILES:=$(FLANN_STATIC_LIB_DIR)/lib$1.a
    include $(PREBUILT_STATIC_LIBRARY)
endef

$(foreach module,$(PCL_STATIC_LIBRARIES),$(eval $(call build_pcl_static,$(module))))
$(foreach module,$(BOOST_STATIC_LIBRARIES),$(eval $(call build_boost_static,$(module))))
$(foreach module,$(FLANN_STATIC_LIBRARIES),$(eval $(call build_flann_static,$(module))))

# end


include $(CLEAR_VARS)

NDK_APP_DST_DIR := ../src/main/jniLibs/$(TARGET_ARCH_ABI)

# for opencv :begin

OpenCV_INSTALL_MODULES:=on
OPENCV_CAMERA_MODULES:=off
APP_STL := gnustl_static
OPENCV_LIB_TYPE:=STATIC
include /Users/netease/Downloads/OpenCV-2.4.10-android-sdk/sdk/native/jni/OpenCV.mk

#end

LOCAL_MODULE :=jni-input
LOCAL_CFLAGS =  -DANDROID_NDK_BUILD -D__STDC_FORMAT_MACROS -D__STDC_INT64__
LOCAL_LDLIBS +=  -llog -ldl
LOCAL_LDLIBS += -L$(SYSROOT)/usr/lib -llog
LOCAL_C_INCLUDES += $(LOCAL_PATH)


# for PCL library   :begin

LOCAL_LDFLAGS += -L$(PCL_INCLUDE)/lib  \
                 -L$(BOOST_ANDROID_INCLUDE)/lib \
                 -L$(FLANN_INCLUDE)/lib

LOCAL_C_INCLUDES += $(PCL_INCLUDE)/include/pcl-1.6 \
                    $(BOOST_ANDROID_INCLUDE)/include \
                    $(EIGEN_INCLUDE) \
                    $(FLANN_INCLUDE)/include

LOCAL_STATIC_LIBRARIES   += pcl_common pcl_geometry pcl_kdtree pcl_octree pcl_sample_consensus \
                            pcl_surface pcl_features pcl_io pcl_keypoints pcl_recognition \
                            pcl_search pcl_tracking pcl_filters pcl_io_ply pcl_ml \
                            pcl_registration pcl_segmentation

LOCAL_STATIC_LIBRARIES   += boost_date_time boost_iostreams boost_regex boost_system \
                            boost_filesystem boost_program_options boost_signals \
                            boost_thread


LOCAL_SHARED_LIBRARIES   += flann flann_cpp

LOCAL_CFLAGS += -mfloat-abi=softfp -mfpu=neon -march=armv7 -mthumb -O3

LOCAL_ALLOW_UNDEFINED_SYMBOLS := true


#end

LOCAL_SRC_FILES := XXX1.cpp \ 
                    ...
                   XXX2.cpp

include $(BUILD_SHARED_LIBRARY)

```

### 2.2 Appliacation.mk

```
APP_OPTIM:=release
APP_STL := gnustl_static
APP_CPPFLAGS := -frtti -fexceptions -fno-trapping-math -std=c++11
APP_ABI := armeabi-v7a

```

### 2.3 注意事项

* 同时引用 `PCL`+`OpenCV`生成`so`文件会报如下错误：

```
PCL 1.6.0\include\pcl-1.6\pcl/kdtree/kdtree_flann.h(520): error C2872: 'flann' : ambiguous symbol
              could be 'flann'
              or       'cv::flann'

```
这是由于`PCL`和`OpenCV`的`flann函数`冲突的原因，解决方法：***去掉项目中的 using namespace cv***，同时源码中对应处加上`cv::`即可。

* 如果引用了`c++11`,如 Appliaction.mk 中如下：

```
APP_CPPFLAGS := -frtti -fexceptions -fno-trapping-math -std=c++11

```
编译时会报如下错误：

```
 error: use of deleted function 'boost::shared_ptr<std::vector<int> >::shared_ptr(const boost::shared_ptr<std::vector<int> >&)'
       getIndices () { return (indices_); }

```

这是因为 `boost`库版本太低（1.45），将`boost`库升级至高版本即可，升级方法：

1. 下载相应高版本`boost`库，如`1.55版本`<https://sourceforge.net/projects/boost/files/boost/1.55.0/> ，解压后文件重命名为`1_45_0`(覆盖掉该目录中原有的此文件),放入如下目录：

```
../PCL/pcl-superbuild/build/CMakeExternals/Source/boost

```
2. 修改`CMakeLists.txt`文件，路径如下：

```
../PCL/pcl-superbuild/build/CMakeExternals/Source/boost/CMakeLists.txt

```
修改如下：

```

file(GLOB lib_srcs ${boost_root}/libs/filesystem/v2/src/*.cpp)//修改前

file(GLOB lib_srcs ${boost_root}/libs/filesystem/src/*.cpp)//修改后

```

3. 删除`pcl-superbuild/build/CMakeExternals/Install/boost-android/`
4. 重新编译

```
cd  android-ndk-r8c
export ANDROID_NDK=$PWD
cd pcl-superbuild/build
cmake ../
make -j4 # j4 is prallel build option

```
* `c++11`相关错误：

```
 error: call of overloaded 'isnan(double&)' is ambiguous
 # define pcl_isnan(x)    isnan(x)
 
```
解决方法：打开`../pcl-superbuild/build/CMakeExternals/Install/pcl-android/include/pcl-1.6/pcl/pcl_macros.h`文件，进行如下修改：
```
#elif ANDROID
// Use the math.h macros
# include <math.h>
# define pcl_isnan(x)    isnan(x)  //修改前

# define pcl_isnan(x)    std::isnan(x)  //修改后

```
