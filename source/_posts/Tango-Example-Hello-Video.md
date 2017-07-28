---
title: Tango Example hello video
date: 2017-07-28 09:24:52
categories: Tango
tags: [Project Tango]
---

## Basic example hello video

完整的代码可以在 [github](https://github.com/googlesamples/tango-examples-c/blob/master/cpp_basic_examples/hello_video) 上找到。

## On tango service connected

范例基于 Android 开发，我们从 JNI 以下的代码说起。当连接到 tango 设备时的回调处理：
``` cpp
JNIEXPORT void JNICALL
Java_com_projecttango_examples_cpp_hellovideo_TangoJniNative_onTangoServiceConnected(
    JNIEnv* env, jobject, jobject binder) {
  app.OnTangoServiceConnected(env, binder);
}
```

``` cpp
void HelloVideoApp::OnTangoServiceConnected(JNIEnv* env, jobject binder) {
  ...
  // Enable color camera from config.
  int ret = TangoConfig_setBool(tango_config_, "config_enable_color_camera",
                                true);
  ret = TangoService_connectOnFrameAvailable(TANGO_CAMERA_COLOR, this,
                                             OnFrameAvailableRouter);
  ...
}
```
从上面的代码可以看到我们启用了 color camera，并注册了 FrameAvailbale 的回调 OnFrameAvailableRouter。

## On frame available

接下来我们看看在回调函数中是如何处理每一帧数据的：
``` cpp
namespace {
constexpr int kTangoCoreMinimumVersion = 9377;
void OnFrameAvailableRouter(void* context, TangoCameraId,
                            const TangoImageBuffer* buffer) {
  hello_video::HelloVideoApp* app =
      static_cast<hello_video::HelloVideoApp*>(context);
  app->OnFrameAvailable(buffer);
}
}  // namespace
```

``` cpp
void HelloVideoApp::OnFrameAvailable(const TangoImageBuffer* buffer) {
  if (current_texture_method_ != TextureMethod::kYuv) {
    return;
  }

  if (yuv_drawable_->GetTextureId() == 0) {
    LOGE("HelloVideoApp::yuv texture id not valid");
    return;
  }

  if (buffer->format != TANGO_HAL_PIXEL_FORMAT_YCrCb_420_SP) {
    LOGE("HelloVideoApp::yuv texture format is not supported by this app");
    return;
  }

  // The memory needs to be allocated after we get the first frame because we
  // need to know the size of the image.
  if (!is_yuv_texture_available_) {
    yuv_width_ = buffer->width;
    yuv_height_ = buffer->height;
    uv_buffer_offset_ = yuv_width_ * yuv_height_;

    yuv_size_ = yuv_width_ * yuv_height_ + yuv_width_ * yuv_height_ / 2;

    // Reserve and resize the buffer size for RGB and YUV data.
    yuv_buffer_.resize(yuv_size_);
    yuv_temp_buffer_.resize(yuv_size_);
    rgb_buffer_.resize(yuv_width_ * yuv_height_ * 3);

    AllocateTexture(yuv_drawable_->GetTextureId(), yuv_width_, yuv_height_);
    is_yuv_texture_available_ = true;
  }

  std::lock_guard<std::mutex> lock(yuv_buffer_mutex_);
  memcpy(&yuv_temp_buffer_[0], buffer->data, yuv_size_);
  swap_buffer_signal_ = true;
}
```
从 tango color camera 获取的是 YUV 格式，单帧数据被保存到成员变量 yuv_temp_buffer_ 中，yuv_temp_buffer_ 的定义如下：
``` cpp
std::vector<uint8_t> yuv_temp_buffer_;
```

## Render YUV

yuv_temp_buffer_ 所保存的数据进行渲染：
``` cpp
void HelloVideoApp::OnDrawFrame() {
  glClearColor(1.0f, 1.0f, 1.0f, 1.0f);
  glClear(GL_DEPTH_BUFFER_BIT | GL_COLOR_BUFFER_BIT);

  if (!is_service_connected_) {
    return;
  }

  if (!is_texture_id_set_) {
    is_texture_id_set_ = true;
    // Connect color camera texture. TangoService_connectTextureId expects a
    // valid texture id from the caller, so we will need to wait until the GL
    // content is properly allocated.
    int texture_id = static_cast<int>(video_overlay_drawable_->GetTextureId());
    TangoErrorType ret = TangoService_connectTextureId(
        TANGO_CAMERA_COLOR, texture_id, nullptr, nullptr);
    if (ret != TANGO_SUCCESS) {
      LOGE(
          "HelloVideoApp: Failed to connect the texture id with error"
          "code: %d",
          ret);
    }
  }

  if (!is_video_overlay_rotation_set_) {
    video_overlay_drawable_->SetDisplayRotation(display_rotation_);
    yuv_drawable_->SetDisplayRotation(display_rotation_);
    is_video_overlay_rotation_set_ = true;
  }

  switch (current_texture_method_) {
    case TextureMethod::kYuv:
      RenderYuv();
      break;
    case TextureMethod::kTextureId:
      RenderTextureId();
      break;
  }
}

void HelloVideoApp::RenderYuv() {
  if (!is_yuv_texture_available_) {
    return;
  }
  {
    std::lock_guard<std::mutex> lock(yuv_buffer_mutex_);
    if (swap_buffer_signal_) {
      std::swap(yuv_buffer_, yuv_temp_buffer_);
      swap_buffer_signal_ = false;
    }
  }

  for (size_t i = 0; i < yuv_height_; ++i) {
    for (size_t j = 0; j < yuv_width_; ++j) {
      size_t x_index = j;
      if (j % 2 != 0) {
        x_index = j - 1;
      }

      size_t rgb_index = (i * yuv_width_ + j) * 3;

      // The YUV texture format is NV21,
      // yuv_buffer_ buffer layout:
      //   [y0, y1, y2, ..., yn, v0, u0, v1, u1, ..., v(n/4), u(n/4)]
      Yuv2Rgb(
          yuv_buffer_[i * yuv_width_ + j],
          yuv_buffer_[uv_buffer_offset_ + (i / 2) * yuv_width_ + x_index + 1],
          yuv_buffer_[uv_buffer_offset_ + (i / 2) * yuv_width_ + x_index],
          &rgb_buffer_[rgb_index], &rgb_buffer_[rgb_index + 1],
          &rgb_buffer_[rgb_index + 2]);
    }
  }

  glBindTexture(GL_TEXTURE_2D, yuv_drawable_->GetTextureId());
  glTexImage2D(GL_TEXTURE_2D, 0, GL_RGB, yuv_width_, yuv_height_, 0, GL_RGB,
               GL_UNSIGNED_BYTE, rgb_buffer_.data());

  yuv_drawable_->Render(glm::mat4(1.0f), glm::mat4(1.0f));
}
```
在上面 RenderYuv 函数告诉了我们更多信息，color camera 获取的是 YUV NV21 格式，如何将它转为 RGB：
``` cpp
// We could do this conversion in a fragment shader if all we care about is
// rendering, but we show it here as an example of how people can use RGB data
// on the CPU.
inline void Yuv2Rgb(uint8_t y_value, uint8_t u_value, uint8_t v_value,
                    uint8_t* r, uint8_t* g, uint8_t* b) {
  float float_r = y_value + (1.370705 * (v_value - 128));
  float float_g =
      y_value - (0.698001 * (v_value - 128)) - (0.337633 * (u_value - 128));
  float float_b = y_value + (1.732446 * (u_value - 128));

  float_r = float_r * !(float_r < 0);
  float_g = float_g * !(float_g < 0);
  float_b = float_b * !(float_b < 0);

  *r = float_r * (!(float_r > 255)) + 255 * (float_r > 255);
  *g = float_g * (!(float_g > 255)) + 255 * (float_g > 255);
  *b = float_b * (!(float_b > 255)) + 255 * (float_b > 255);
}
```
