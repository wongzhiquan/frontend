<template>
  <div v-loading="loading" element-loading-text="正在初始化.... 请确保屏幕处于解锁显示状态" style="width: 100%">
    <el-alert
      v-if="showAlert"
      style="position: fixed"
      :closable="false"
      title="远程连接已断开"
      type="error"
      show-icon
    />
    <!--画布-->
    <div style="width: 100%; min-height: 200px">
      <canvas id="iosControllerCanvas" style="width: 100%" />
    </div>
    <div style="margin-top: 2px" align="center">
      <ios-controller-buttom :ios-websocket="iosWebsocket" />
    </div>
  </div>
</template>

<script>
import IosControllerButtom from './IosControllerButtom'

export default {
  components: {
    IosControllerButtom
  },
  data() {
    return {
      loading: false,
      showAlert: false,
      iosWebsocket: null,
      touchDown: {
        operation: 'd',
      },
      touchMove: {
        operation: 'm',
      },
      touchUp: {
        operation: 'u'
      }
    }
  },
  computed: {
    agentIp() {
      return this.$store.state.device.agentIp
    },
    agentPort() {
      return this.$store.state.device.agentPort
    },
    deviceId() {
      return this.$store.state.device.id
    },
    username() {
      return this.$store.state.user.name
    }
  },
  // 关闭控制窗口，组件销毁前
  beforeDestroy() {
    this.iosWebsocket.close()
  },
  mounted() {
    this.loading = true
    const canvas = document.getElementById('iosControllerCanvas')
    const g = canvas.getContext('2d')
    const BLANK_IMG = 'data:image/gif;base64,R0lGODlhAQABAAAAACH5BAEKAAEALAAAAAABAAEAAAICTAEAOw=='
    const URL = window.URL || window.webkitURL

    // iosWebsocket
    this.iosWebsocket = new WebSocket('ws://' + this.agentIp + ':' + this.agentPort + '/ios/' + this.deviceId + '/user/' + this.username + '/project/' + this.$store.state.project.id)
    this.iosWebsocket.onclose = () => {
      this.showAlert = true
      this.loading = false
    }
    this.iosWebsocket.onerror = () => {
      this.showAlert = true
      this.loading = false
    }
    this.iosWebsocket.onmessage = (message) => {
      if (message.data instanceof Blob) {
        let blob = new Blob([message.data], { type: 'image/jpeg' })
        let url = URL.createObjectURL(blob)
        let img = new Image()
        img.src = url
        img.onload = () => {
          canvas.width = img.width
          canvas.height = img.height
          g.drawImage(img, 0, 0)

          img.onload = null
          img.src = BLANK_IMG
          img = null
          blob = null

          URL.revokeObjectURL(url)
          url = null
        }
      } else {
        console.log('iosWebsocket-onmessage', message.data)
        if (message.data && message.data.indexOf('appiumSessionId') !== -1) {
          this.loading = false
          this.$store.dispatch('device/setAppiumSessionId', JSON.parse(message.data).appiumSessionId)
        }
      }
    }
    let isMouseDown = false
    // 当鼠标处于按下的状态移出画布,这个时候体验不好，需要在移出的时候，发送鼠标抬起事件,并将鼠标状态设为抬起
    canvas.onmouseleave = () => {
      if (isMouseDown) {
        this.iosWebsocket.send(JSON.stringify(this.touchUp))
        isMouseDown = false
      }
    }
    // 鼠标按下
    canvas.onmousedown = (e) => {
      isMouseDown = true
      const rect = canvas.getBoundingClientRect()
      this.touchDown.x = parseInt((e.clientX - rect.left) / rect.width * canvas.width)
      this.touchDown.y = parseInt((e.clientY - rect.top) / rect.height * canvas.height)
      this.touchDown.width = canvas.width
      this.touchDown.height = canvas.height
      this.iosWebsocket.send(JSON.stringify(this.touchDown))
    }
    // 鼠标抬起
    canvas.onmouseup = () => {
      isMouseDown = false
      this.iosWebsocket.send(JSON.stringify(this.touchUp))
    }
    // 鼠标移动
    canvas.onmousemove = (e) => {
      // 鼠标按下才发送移动事件,防止在画布上移动鼠标也发送移动事件
      if (isMouseDown) {
        const rect = canvas.getBoundingClientRect()
        this.touchMove.x = parseInt((e.clientX - rect.left) / rect.width * canvas.width)
        this.touchMove.y = parseInt((e.clientY - rect.top) / rect.height * canvas.height)
        this.touchMove.width = canvas.width
        this.touchMove.height = canvas.height
        this.iosWebsocket.send(JSON.stringify(this.touchMove))
      }
    }
  }
}
</script>
