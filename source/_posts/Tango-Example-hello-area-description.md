---
title: Tango Example hello area description
date: 2017-07-28 10:26:42
categories: Tango
tags: [Project Tango]
---

## Basic example hello depth perception

完整的代码可以在 [github](https://github.com/googlesamples/tango-examples-c/blob/master/cpp_basic_examples/hello_area_description) 上找到。

## On tango service connected

范例基于 Android 开发，我们从 JNI 以下的代码说起。当连接到 tango 设备时的回调处理：
``` cpp
JNIEXPORT void JNICALL
Java_com_projecttango_examples_cpp_helloareadescription_TangoJniNative_onTangoServiceConnected(
    JNIEnv* env, jobject, jobject binder, jboolean is_area_learning_enabled,
    jboolean is_loading_area_description) {
  app.OnTangoServiceConnected(env, binder, is_area_learning_enabled,
                              is_loading_area_description);
}
```

``` cpp
void AreaLearningApp::OnTangoServiceConnected(
    JNIEnv* env, jobject binder, bool is_area_learning_enabled,
    bool is_loading_area_description) {
  ...

  if (is_area_learning_enabled || is_loading_area_description) {
    // Here, we'll configure the service to run in the way we'd want. For this
    // application, we'll start from the default configuration
    // (TANGO_CONFIG_DEFAULT). This enables basic motion tracking capabilities.
    tango_config_ = TangoService_getConfig(TANGO_CONFIG_DEFAULT);
    if (tango_config_ == nullptr) {
      LOGE("AreaLearningApp: Failed to get default config form");
      std::exit(EXIT_SUCCESS);
    }

    int ret = TangoConfig_setBool(tango_config_, "config_enable_learning_mode",
                                  is_area_learning_enabled);
    if (ret != TANGO_SUCCESS) {
      LOGE(
          "AreaLearningApp: config_enable_learning_mode failed with error"
          "code: %d",
          ret);
      std::exit(EXIT_SUCCESS);
    }

    // If load ADF, load the most recent saved ADF.
    if (is_loading_area_description) {
      std::vector<std::string> adf_list;
      GetAdfUuids(&adf_list);
      if (!adf_list.empty()) {
        std::string adf_uuid = adf_list.back();
        std::ostringstream adf_str_stream;
        adf_str_stream << "Number of ADFs:" << adf_list.size()
                       << ", Loaded ADF: " << adf_uuid;
        loaded_adf_string_ = adf_str_stream.str();
        ret = TangoConfig_setString(tango_config_,
                                    "config_load_area_description_UUID",
                                    adf_uuid.c_str());
        if (ret != TANGO_SUCCESS) {
          LOGE("AreaLearningApp: get ADF UUID failed with error code: %d", ret);
        }
      }
    }

    // Setting up the frame pair for the onPoseAvailable callback.
    TangoCoordinateFramePair pairs[3] = {
        {TANGO_COORDINATE_FRAME_START_OF_SERVICE,
         TANGO_COORDINATE_FRAME_DEVICE},
        {TANGO_COORDINATE_FRAME_AREA_DESCRIPTION,
         TANGO_COORDINATE_FRAME_DEVICE},
        {TANGO_COORDINATE_FRAME_AREA_DESCRIPTION,
         TANGO_COORDINATE_FRAME_START_OF_SERVICE}};

    // Attach onPoseAvailable callback.
    // The callback will be called after the service is connected.
    ret = TangoService_connectOnPoseAvailable(3, pairs, onPoseAvailableRouter);
    if (ret != TANGO_SUCCESS) {
      LOGE(
          "AreaLearningApp: Failed to connect to pose callback with error"
          "code: %d",
          ret);
      std::exit(EXIT_SUCCESS);
    }

    // Connect to the Tango Service, the service will start running:
    // point clouds can be queried and callbacks will be called.
    ret = TangoService_connect(this, tango_config_);
    if (ret != TANGO_SUCCESS) {
      LOGE(
          "AreaLearningApp::OnTangoServiceConnected,"
          "Failed to connect to the Tango service with error code: %d",
          ret);
      std::exit(EXIT_SUCCESS);
    }
  }
}
```
相较与 color 和 depth，pose 的初始设置更复杂，其中有大量是关于 ADF（Automatic Direction Finder：自动方位搜寻器）的，实际代码还另有大量处理 ADF 的函数被定义和实现，请参考原始代码。

我们这里重点关注 PoseAvailable 的回调 OnPoseAvailableRouter。

## On pose available

接下来我们看看在回调函数中如何处理 pose 数据：
``` cpp
namespace {
// The minimum Tango Core version required from this application.
constexpr int kTangoCoreMinimumVersion = 9377;
const int kVersionStringLength = 128;

// This function routes onPoseAvailable callbacks to the application object for
// handling.
//
// @param context, context will be a pointer to a AreaLearningApp
//        instance on which to call callbacks.
// @param pose, pose data to route to onPoseAvailable function.
void onPoseAvailableRouter(void* context, const TangoPoseData* pose) {
  hello_area_description::AreaLearningApp* app =
      static_cast<hello_area_description::AreaLearningApp*>(context);
  app->onPoseAvailable(pose);
}
}  // namespace

void AreaLearningApp::onPoseAvailable(const TangoPoseData* pose) {
  std::lock_guard<std::mutex> lock(pose_mutex_);
  pose_data_.UpdatePose(*pose);
}
```
Pose 数据被保存在成员变量 pose_data_ 中，它的定义：
``` cpp
  // pose_data_ handles all pose onPoseAvailable callbacks, onPoseAvailable()
  // in this object will be routed to pose_data_ to handle.
  PoseData pose_data_;
```

## PoseData

这里出现的 PoseData 并非 SDK 提供，其完整定义如下。
``` cpp
// PoseData holds all pose related data. E.g. pose position, rotation and time-
// stamp. It also produce the debug information strings.
class PoseData {
 public:
  PoseData();
  ~PoseData();

  // Update current pose and previous pose.
  //
  // @param pose: pose data of current frame.
  void UpdatePose(const TangoPoseData& pose_data);

  // Reset all saved pose data.
  void ResetPoseData();

  // Get pose data in current frame.
  //
  // @return: curent pose data.
  TangoPoseData GetCurrentPoseData();

  // Check if the device is relocalized.
  //
  // @return: relocalized flag.
  bool IsRelocalized() { return is_relocalized_; }

 private:
  // Relocalized flag, the relocalization is determined by the pose in start of
  // service with respect to ADF turns to valid.
  bool is_relocalized_ = false;

  TangoPoseData adf_T_device_pose_;
  TangoPoseData start_service_T_device_pose_;
};
```

``` cpp
PoseData::PoseData() {}

PoseData::~PoseData() {}

void PoseData::UpdatePose(const TangoPoseData& pose_data) {
  // We check the frame pair received in the pose_data instance, and store it
  // in the proper cooresponding local member.
  //
  // The frame pairs are the one we registred in the
  // TangoService_connectOnPoseAvailable functions during the setup of the
  // Tango configuration file.
  if (pose_data.frame.base == TANGO_COORDINATE_FRAME_START_OF_SERVICE &&
      pose_data.frame.target == TANGO_COORDINATE_FRAME_DEVICE) {
    start_service_T_device_pose_ = pose_data;
  } else if (pose_data.frame.base == TANGO_COORDINATE_FRAME_AREA_DESCRIPTION &&
             pose_data.frame.target == TANGO_COORDINATE_FRAME_DEVICE) {
    adf_T_device_pose_ = pose_data;
  } else if (pose_data.frame.base == TANGO_COORDINATE_FRAME_AREA_DESCRIPTION &&
             pose_data.frame.target ==
                 TANGO_COORDINATE_FRAME_START_OF_SERVICE) {
    is_relocalized_ = (pose_data.status_code == TANGO_POSE_VALID);
  } else {
    return;
  }
}

void PoseData::ResetPoseData() { is_relocalized_ = false; }

TangoPoseData PoseData::GetCurrentPoseData() {
  if (is_relocalized_) {
    return adf_T_device_pose_;
  } else {
    return start_service_T_device_pose_;
  }
}
```

## TangoPoseData

SDK 所提供的 TangoPoseData，完整信息请查看 [官方文档](https://developers.google.com/tango/apis/c/reference/struct/tango-pose-data)，下面只重点说明几个公共属性。

### frame

``` cpp
TangoCoordinateFramePair TangoPoseData::frame
```
The pair of coordinate frames for this pose.

We retrieve a pose for a target coordinate frame (such as the Tango device) against a base coordinate frame (such as a learned area).

### orientation

``` cpp
double TangoPoseData::orientation[4]
```
Orientation, as a quaternion, of the pose of the target frame with reference to the base frame.

Specified as (x,y,z,w) where RotationAngle is in radians:

``` cpp
x = RotationAxis.x * sin(RotationAngle / 2)
y = RotationAxis.y * sin(RotationAngle / 2)
z = RotationAxis.z * sin(RotationAngle / 2)
w = cos(RotationAngle / 2)
```

### status_code

``` cpp
TangoPoseStatusType TangoPoseData::status_code
```
The status of the pose, according to the pose lifecycle.

### timestamp

``` cpp
double TangoPoseData::timestamp
```
The timestamp of the pose estimate, in seconds.

### translation

``` cpp
double TangoPoseData::translation[3]
```
Translation, ordered x, y, z, of the pose of the target frame with reference to the base frame.
