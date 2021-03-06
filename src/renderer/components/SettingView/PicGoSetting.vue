<template>
  <div id="picgo-setting">
    <div class="view-title">
      PicGo设置
    </div>
    <el-row class="setting-list">
      <el-col :span="15" :offset="4">
        <el-row>
        <el-form
          label-width="120px"
          label-position="right"
          size="small"
        >
          <el-form-item
            label="修改快捷键"
          >
            <el-button type="primary" round size="mini" @click="keyBindingVisible = true">点击设置</el-button>
          </el-form-item>
          <el-form-item
            label="自定义链接格式"
          >
            <el-button type="primary" round size="mini" @click="customLinkVisible = true">点击设置</el-button>
          </el-form-item>
          <el-form-item
            label="打开更新助手"
          >
            <el-switch
              v-model="form.updateHelper"
              active-text="开"
              inactive-text="关"
              @change="updateHelperChange"
            ></el-switch>
          </el-form-item>
          <el-form-item
            label="开机自启"
          >
            <el-switch
              v-model="form.autoStart"
              active-text="开"
              inactive-text="关"
              @change="handleAutoStartChange"
            ></el-switch>
          </el-form-item>
          <el-form-item
            label="上传前重命名"
          >
            <el-switch
              v-model="form.rename"
              active-text="开"
              inactive-text="关"
            ></el-switch>
          </el-form-item>
          <el-form-item
            label="选择显示的图床"
          >
            <el-checkbox-group
              v-model="form.showPicBedList"
              @change="handleShowPicBedListChange"
            >
              <el-checkbox
                v-for="(item, index) in picBed"
                :label="item.name"
              ></el-checkbox>
            </el-checkbox-group>
          </el-form-item>
        </el-form>
        </el-row>
      </el-col>
    </el-row>
    <el-dialog
      title="修改快捷键"
      :visible.sync="keyBindingVisible"
      :modal-append-to-body="false"
    >
      <el-form
        label-width="80px"
      >
        <el-form-item
          label="快捷上传"
        >
          <el-input 
            class="align-center"
            @keydown.native.prevent="keyDetect('upload', $event)"
            v-model="shortKey.upload"
            :autofocus="true"
          ></el-input>
        </el-form-item>
      </el-form>
      <span slot="footer">
        <el-button @click="cancelKeyBinding">取消</el-button>
        <el-button type="primary" @click="confirmKeyBinding">确定</el-button>
      </span>
    </el-dialog>
    <el-dialog
      title="自定义链接格式"
      :visible.sync="customLinkVisible"
      :modal-append-to-body="false"
    >
      <el-form
        label-position="top"
        :model="customLink"
        ref="customLink"
        :rules="rules"
      >
        <el-form-item
          label="用占位符$url来表示url的位置"
          prop="value"
        >
          <el-input 
            class="align-center"
            v-model="customLink.value"
            :autofocus="true"
          ></el-input>
        </el-form-item>
      </el-form>
      <div>
        如[]($url)
      </div>
      <span slot="footer">
        <el-button @click="cancelCustomLink">取消</el-button>
        <el-button type="primary" @click="confirmCustomLink">确定</el-button>
      </span>
    </el-dialog>
  </div>
</template>
<script>
import db from '../../../datastore'
import keyDetect from 'utils/key-binding'
export default {
  name: 'picgo-setting',
  data () {
    const customLinkRule = (rule, value, callback) => {
      if (!/\$url/.test(value)) {
        return callback(new Error('必须含有$url'))
      } else {
        return callback()
      }
    }
    return {
      form: {
        updateHelper: this.$db.get('picBed.showUpdateTip').value(),
        showPicBedList: [],
        autoStart: this.$db.get('picBed.autoStart').value() || false,
        rename: this.$db.get('picBed.rename').value() || false
      },
      picBed: this.$picBed,
      keyBindingVisible: false,
      customLinkVisible: false,
      customLink: {
        value: db.read().get('customLink').value() || '$url'
      },
      shortKey: {
        upload: db.read().get('shortKey.upload').value()
      },
      rules: {
        value: [
          { validator: customLinkRule, trigger: 'blur' }
        ]
      }
    }
  },
  created () {
    this.form.showPicBedList = this.picBed.map(item => {
      if (item.visible) {
        return item.name
      }
    })
  },
  methods: {
    keyDetect (type, event) {
      this.shortKey[type] = keyDetect(event).join('+')
    },
    cancelKeyBinding () {
      this.keyBindingVisible = false
      this.shortKey = db.read().get('shortKey').value()
    },
    confirmKeyBinding () {
      const oldKey = db.read().get('shortKey').value()
      db.read().set('shortKey', this.shortKey).write()
      this.keyBindingVisible = false
      this.$electron.ipcRenderer.send('updateShortKey', oldKey)
    },
    cancelCustomLink () {
      this.customLinkVisible = false
      this.customLink.value = db.read().get('customLink').value() || '$url'
    },
    confirmCustomLink () {
      this.$refs.customLink.validate((valid) => {
        if (valid) {
          db.read().set('customLink', this.customLink.value).write()
          this.customLinkVisible = false
          this.$electron.ipcRenderer.send('updateCustomLink')
        } else {
          return false
        }
      })
    },
    updateHelperChange (val) {
      this.$db.read().set('picBed.showUpdateTip', val).write()
    },
    handleShowPicBedListChange (val) {
      const list = this.picBed.map(item => {
        if (!val.includes(item.name)) {
          item.visible = false
        } else {
          item.visible = true
        }
        return item
      })
      this.$db.read().set('picBed.list', list).write()
    },
    handleAutoStartChange (val) {
      this.$db.read().set('picBed.autoStart', val).write()
      this.$electron.ipcRenderer.send('autoStart', val)
    }
  },
  beforeDestroy () {
    this.$electron.ipcRenderer.removeAllListeners('autoStart')
  }
}
</script>
<style lang='stylus'>
.el-message
  left 60%
.view-title
  color #eee
  font-size 20px
  text-align center
  margin 20px auto
#picgo-setting
  .sub-title
    font-size 14px
  .setting-list
    height 360px
    box-sizing border-box
    overflow-y auto
    overflow-x hidden
  .setting-list
    .el-form
      label  
        line-height 32px
        padding-bottom 0
        color #eee
      .el-button-group
        width 100%
        .el-button
          width 50%
      .el-input__inner
        border-radius 19px
      .el-radio-group
        margin-left 25px
      .el-switch__label
        color #eee
        &.is-active
          color #409EFF
      .el-icon-question
        font-size 20px
        float right
        margin-top 9px
        color #eee
        cursor pointer
        transition .2s color ease-in-out
        &:hover
          color #409EFF
      .el-checkbox-group
        label
          margin-right 30px
          width 100px
      .el-checkbox+.el-checkbox
        margin-right 30px
        margin-left 0
      .confirm-button
        width 100%
</style>