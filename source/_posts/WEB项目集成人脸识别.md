---
title: WEB项目集成人脸识别
url: WEB项目集成人脸识别
date: 2024-10-22 16:32:48
categories:
- 项目实践
tags:
- 项目实践
---

前端开发的一大痛点就是参考资料太多，css，h5 标准，浏览器的各种规范等等。AI 的出现真的很大程度上缓解了这些问题，让我这种半桶水的前端也能够快速的定位到我需要的文档。

<!-- more -->

项目中需要让客户使用人脸识别验证身份，很自然的想到如下实现思路：

1. 浏览器调用摄像头，获得视频
2. 视频截图，发往后端
3. 后端将照片和系统现有用户比对

# 浏览器调用摄像头

托 AI 的福，很快就查到了关键词：摄像头调用使用 `navigator.mediaDevices` 相关接口，视频播放使用 `video` 元素。

# 视频截图

直接用 `canvas` 把 `video` 内容画上来，当成图片发给后端就好。

# 后端比对

使用 github 的 [face_recognition](https://github.com/ageitgey/face_recognition) 仓库，该仓库可以使用命令行进行人脸比对，用 go 稍微封装一下就好了。

再次感谢无数开源作者为世界做出的贡献。

# 前端代码示例

用 `vue2` 做了一个简易的人脸识别弹窗：

```vue
<template>
  <div class="app-container">
    选择您的摄像头设备
    <el-button-group>
      <el-button v-for="(camera, x) in cameras" :key="x" @click="playThis(camera)">{{ camera.label }}</el-button>
    </el-button-group>
    <video id="video"></video >
    <canvas id="photo" width="640" height="420" />
  </div>
</template>

<script>
export default {
  data() {
    return {
      cameras: [], // 摄像头列表
      video_dom: undefined,
      timeHandler: undefined
    }
  },
  mounted() {
    const self = this
    this.video_dom = document.getElementById('video')
    navigator.mediaDevices.enumerateDevices().then(function(devices) {
      devices.forEach(device => {
        if (device.kind === 'videoinput') {
          self.cameras.push(device)
        }
      })
    }).catch(function(err) {
      alert('你的浏览器无法获摄像头设备列表')
      this.$emit('cancel')
    })
  },
  methods: {
    playThis(camera) {
      this.atClose()
      const self = this
      navigator.mediaDevices.getUserMedia({ video: { deviceId: camera.deviceId } }).then(function(stream) {
        const video = document.getElementById('video')
        video.srcObject = stream
        video.play()
        self.toPhoto()
      }).catch(function(err) {
        console.log('获取摄像头媒体流时出现错误：', err)
        alert('你的浏览器无法获取此设备流')
        self.$emit('cancel')
      })
    },
    toPhoto() {
      let self = this
      const canvas = document.getElementById('photo')
      const context = canvas.getContext('2d')
      const video = document.getElementById('video')
      this.timeHandler = setTimeout(function() {
        context.drawImage(video, 0, 0, 640, 420)
        let img = canvas.toDataURL('image/jpg')
        img = img.split(',')[1] // 照片的 base64
        // TODO 发到后端做人脸识别判断

        self.toPhoto()
      },300)
    },
    atClose() {
      clearTimeout(this.timeHandler)
      const video = document.getElementById('video')
      if (video && video.srcObject) {
        let stream = video.srcObject
        let tracks = stream.getTracks()
        tracks.forEach(function(track) {
            track.stop()
        })
        video.srcObject = null
      }
    }
  }
}
</script>

<style lang="scss">
#video {
  width: 640px;
  height: 420px;
  background: #333;
  margin-top: 20px;
}
#photo {
  width: 640px;
  height: 420px;
  display: none;
}
</style>
```