<template>
  <div class="user">
    <header class="header">
      <div class="userinfo">
        <span>手机：{{ user.tel.slice(-11) }}</span>
        <img :src="user.level" alt="账号等级" />
      </div>
      <div>
        <span v-if="gift.num !== 0" class="gift">
          <img :src="gift.gif" alt="荧光棒" height="16" style="margin-right: 5px" />
          剩余：{{ gift.num }}
        </span>
        <span v-else>
          <el-tag type="success" size="mini">任务已完成</el-tag>
        </span>
      </div>
      <div>
        <el-button type="info" size="mini">切换账号</el-button>
      </div>
    </header>
    <el-alert v-if="info.show" :title="info.title" :type="info.type" :closable="false" center show-icon></el-alert>
    <main class="main">
      <el-scrollbar class="scrollbar">
        <el-table :data="fans" style="width: 100%">
          <el-table-column prop="name" label="主播" align="center"></el-table-column>
          <el-table-column prop="intimacy" label="亲密值" align="center"></el-table-column>
          <el-table-column prop="today" label="今日亲密度" align="center"></el-table-column>
          <el-table-column prop="ranking" label="排名" align="center"></el-table-column>
        </el-table>
      </el-scrollbar>
    </main>
  </div>
</template>
<script lang="ts">
const { app } = require('electron').remote
import { Vue, Component } from 'vue-property-decorator'
const { BrowserWindow } = require('electron').remote
import init from '../function/send'
const CronJob = require('cron').CronJob
const log = require('electron-log')
interface windows {
  [propName: string]: any
}
interface alertInfo {
  title: string
  type: string
  show: boolean
}
@Component({})
export default class user extends Vue {
  [x: string]: any
  win: windows = {}
  info: alertInfo = {
    title: '',
    type: 'info',
    show: false
  }
  get user() {
    return this.$store.state.user
  }
  get gift() {
    return this.$store.state.gift
  }
  get fans() {
    return this.$store.state.fans
  }
  upData() {
    let fans = this.$db.get('fans')
    if (!fans) {
      this.$message({
        type: 'error',
        message: '请在“任务配置”选项卡中先配置任务'
      })
      return
    }
    this.$store
      .dispatch('getgift')
      .then(() => {
        if (this.gift.num > 0) {
          let i = 4
          let timer = setInterval(() => {
            i--
            this.info.title = `${i}秒后开始赠送...`
            this.info.show = true
            if (i === 1) {
              this.info.show = false
              clearInterval(timer)
              init(fans).then(res => {
                if (res) {
                  setTimeout(() => {
                    this.upData()
                    this.$store.commit('isStart', false)
                    let close = this.$db.get('close')
                    if (close) {
                      app.quit()
                    }
                  }, 5000)
                }
              })
            }
          }, 1000)
        } else {
          this.$store.commit('isStart', false)
        }
      })
      .catch(() => {})
  }
  start(type?: string) {
    if (type === 'auto') {
      log.info('自动运行触发')
    }
    this.$store.commit('fans', [])
    if (!this.$store.state.isStart) {
      this.info.title = '正在检测并领取荧光棒...'
      this.info.show = true
      this.$store.commit('isStart', true)
      this.win = new BrowserWindow({
        width: 1200,
        height: 800,
        webPreferences: {
          nodeIntegration: true,
          webSecurity: false,
          enableRemoteModule: true
        },
        resizable: false,
        show: false
      })
      this.win.loadURL('https://www.douyu.com/4120796')
      this.win.on('closed', () => {
        this.upData()
      })
      setTimeout(() => {
        this.win.close()
        this.info.show = false
      }, 10000)
    }
  }
  created() {
    this.start()
    if (this.$db.get('timing') === true && this.$auto === null) {
      const time = this.$db.get('runtime').split(':')
      this.$auto = new CronJob(`${time[2]} ${time[1]} ${time[0]} * * *`, this.start.bind(this, 'auto'))
      this.$auto.start()
      log.info('开启自动运行任务')
      this.$message({
        type: 'success',
        message: '已开启自动运行'
      })
    }
  }
}
</script>
<style lang="scss" scoped>
.user {
  height: 100%;
  padding: 0px;
}
.header {
  height: 45px;
  border-bottom: 1px #ccc solid;
  padding: 0 20px;
  display: flex;
  justify-content: space-between;
  align-items: center;
  .userinfo {
    display: flex;
    align-items: center;
    span {
      margin-right: 10px;
    }
  }
  .gift {
    display: flex;
    justify-content: center;
    align-items: center;
  }
}
.main {
  .scrollbar /deep/ {
    height: calc(100vh - 46px);
    .el-scrollbar__wrap {
      overflow-x: hidden;
    }
  }
}
</style>
