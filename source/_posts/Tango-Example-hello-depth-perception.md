---
title: Tango Example hello depth perception
date: 2017-07-28 10:03:28
categories: Tango
tags: [Project Tango, Point Cloud]
---

## Basic example hello depth perception

完整的代码可以在 [github](https://github.com/googlesamples/tango-examples-c/blob/master/cpp_basic_examples/hello_depth_perception) 上找到。

## On tango service connected

范例基于 Android 开发，我们从 JNI 以下的代码说起。当连接到 tango 设备时的回调处理：
``` cpp
JNIEXPORT void JNICALL
Java_com_projecttango_examples_cpp_hellodepthperception_TangoJniNative_onTangoServiceConnected(
    JNIEnv* env, jobject, jobject binder) {
  app.OnTangoServiceConnected(env, binder);
}
```

``` cpp
void HelloVideoApp::OnTangoServiceConnected(JNIEnv* env, jobject binder) {
  ...
  // Enable Depth Perception.
  TangoErrorType err =
      TangoConfig_setBool(tango_config_, "config_enable_depth", true);
  if (err != TANGO_SUCCESS) {
    LOGE(
        "HelloDepthPerceptionApp::OnTangoServiceConnected,"
        "config_enable_depth() failed with error code: %d.",
        err);
    std::exit(EXIT_SUCCESS);
  }

  // Need to specify the depth_mode as XYZC.
  err = TangoConfig_setInt32(tango_config_, "config_depth_mode",
                             TANGO_POINTCLOUD_XYZC);
  if (err != TANGO_SUCCESS) {
    LOGE(
        "Failed to set 'depth_mode' configuration flag with error"
        " code: %d",
        err);
    std::exit(EXIT_SUCCESS);
  }

  // Attach the OnPointCloudAvailable callback to the OnPointCloudAvailable
  // function defined above. The callback will be called every time a new
  // point cloud is acquired, after the service is connected.
  err = TangoService_connectOnPointCloudAvailable(OnPointCloudAvailable);
  if (err != TANGO_SUCCESS) {
    LOGE(
        "HelloDepthPerceptionApp::OnTangoServiceConnected,"
        "Failed to connect to point cloud callback with error code: %d",
        err);
    std::exit(EXIT_SUCCESS);
  }

  // Connect to the Tango Service, the service will start running:
  // point clouds can be queried and callbacks will be called.
  err = TangoService_connect(this, tango_config_);
  if (err != TANGO_SUCCESS) {
    LOGE(
        "HelloDepthPerceptionApp::OnTangoServiceConnected,"
        "Failed to connect to the Tango service with error code: %d",
        err);
    std::exit(EXIT_SUCCESS);
  }
}
```
从上面的代码可以看到我们启用了 depth 并设定格式为 XYZC，然后注册了 PointCloudAvailable 的回调 OnPointCloudAvailable。

## On point cloud available

接下来我们看看在回调函数中如何处理点云数据：
``` cpp
namespace {
// The minimum Tango Core version required from this application.
constexpr int kTangoCoreMinimumVersion = 9377;

// This function logs point cloud data from OnPointCloudAvailable callbacks.
//
// @param context, this will be a pointer to a HelloDepthPerceptionApp
//        instance on which to call callbacks. This parameter is hidden
//        since it is not used.
// @param *point_cloud, point cloud data to log.
void OnPointCloudAvailable(void* /*context*/,
                           const TangoPointCloud* point_cloud) {
  // Number of points in the point cloud.
  float average_depth;

  // Calculate the average depth.
  average_depth = 0;
  // Each xyzc point has 4 coordinates.
  for (size_t i = 0; i < point_cloud->num_points; ++i) {
    average_depth += point_cloud->points[i][2];
  }
  if (point_cloud->num_points) {
    average_depth /= point_cloud->num_points;
  }

  // Log the number of points and average depth.
  LOGI("HelloDepthPerceptionApp: Point count: %d. Average depth (m): %.3f",
       point_cloud->num_points, average_depth);
}
}  // anonymous namespace
```
点云数据被存储在一个叫做  TangoPointCloud 的结构体数组中，Tango SDK 提供了其描述文档。

## struct TangoPointCloud

SDK 所提供的 TangoPointCloud，完整信息请查看 [官方文档](https://developers.google.com/tango/apis/c/reference/struct/tango-point-cloud)，下面只重点说明几个公共属性。

### num_points
``` cpp
uint32_t TangoPointCloud::num_points
```
The number of points in depth_data_buffer populated successfully.

This is variable with each call to the function, and is returned as number of (x,y,z,c) points populated.

### points
``` cpp
float(* TangoPointCloud::points)[4]
```
An array of packed {X, Y, Z, C} values.

{X, Y, Z} is a coordinate triplet (in meters). C is a confidence value, in the range of [0, 1], where 1 corresponds to full confidence. +Z points in the direction of the camera's optical axis, perpendicular to the plane of the camera. +X points toward the user's right, and +Y points toward the bottom of the screen. The origin is the focal center of the depth camera.

### timestamp
``` cpp
double TangoPointCloud::timestamp
```
Time of capture of the depth data for this struct (in seconds).

### version
``` cpp
uint32_t TangoPointCloud::version
```
An integer denoting the version of the structure.
