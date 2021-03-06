<template>
  <div class="options-container px-2 pb-2 scroll-y">
    <div class="flex pb-1">
      <Button type="primary" class="w-6r" @click="onCreate">添加</Button>
      <Button type="primary" class="w-6r ml-1" :disabled="selectedRows.length < 1" @click="update">更新</Button>
      <Button type="warning" class="w-6r ml-1" @click="remove" :disabled="selectedRows.length < 1">删除</Button>
      <div class="ml-auto flex-inline flex-ai-center">
        <Checkbox :value="appConfig.autoUpdateSubscribes" @on-change="onUpdateChange">自动更新</Checkbox>
        <div v-if="appConfig.autoUpdateSubscribes" class="flex-inline flex-ai-center cycle-wrapper">
          <span>每&nbsp;</span>
          <Input :value="cycle.number" :maxlength="2" number @input="onChangeCycleNumber"/>
          <Select :value="cycle.unit" @input="onChangeCycleUnit">
            <Option value="hour">时</Option>
            <Option value="day">天</Option>
            <Option value="week">周</Option>
          </Select>
          <span>&nbsp;更新</span>
        </div>
      </div>
    </div>
    <div class="flex pb-1">
      <Input v-model="group" class="mr-2 create-input" placeholder="请输入订阅名称" ref="inputGroup" />
      <Input
        class="mr-2 url-input"
        :class="{ 'input-error': urlError }"
        v-model="url"
        placeholder="请输入合法的URL并回车"
        icon="plus"
        ref="inputUrl"
        @keyup.enter.native="save"
        @keyup.esc.native="cancel"
        @on-blur="cancel"
      />
    </div>
    <Table
      stripe
      border
      :columns="columns"
      :data="tableData"
      size="small"
      :loading="loading"
      no-data-text="暂无订阅服务器"
      height="280"
      @on-selection-change="selectRows"
      @on-row-dblclick="onRowDBClick"
    ></Table>
  </div>
</template>
<script>
import { mapState, mapMutations, mapActions } from 'vuex'
import { request, isSubscribeContentValid, somePromise } from '../../../shared/utils'

const URL_REGEX = /^https?:\/\/[-A-Za-z0-9+&@#/%?=~_|!:,.;]+[-A-Za-z0-9+&@#/%=~_|]/
// 单位对应的小时倍数
const unitMap = {
  hour: 1,
  day: 24,
  week: 168,
}
export default {
  data () {
    return {
      group: '',
      url: '',
      showNewUrl: false,
      urlError: false,
      loading: false,
      columns: [
        { type: 'selection', width: 54, align: 'center' },
        {
          title: '订阅地址',
          key: 'URL',
          render: (h, params) => {
            const self = this
            const isEditing = params.index === this.editingRowIndex
            let element
            if (isEditing) {
              element = h('i-input', {
                ref: 'url',
                props: {
                  value: this.editingRowUrl,
                  placeholder: '请输入新的订阅服务器的URL',
                },
                attrs: {
                  id: 'editing-input',
                },
                on: {
                  'on-blur' () {
                    self.cancelEditing()
                  },
                },
                nativeOn: {
                  keyup (e) {
                    if (e.keyCode === 13) {
                      const url = self.editingRowUrl
                      // 未发生改变
                      if (url === self.appConfig.serverSubscribes[params.index].URL) {
                        self.cancelEditing()
                        return
                      }
                      self.loading = true
                      if (URL_REGEX.test(url)) {
                        self
                          .requestSubscribeUrl(url)
                          .then(res => {
                            self.loading = false
                            const configs = isSubscribeContentValid(res)
                            if (configs.length) {
                              const clone = self.appConfig.serverSubscribes.slice()
                              clone.splice(params.index, 1)
                              clone.splice(params.index, 0, {
                                URL: url,
                                Group: self.appConfig.serverSubscribes[params.index].URL,
                              })
                              self.updateConfig({
                                serverSubscribes: clone,
                                configs: self.appConfig.configs.concat(configs),
                              })
                            }
                          })
                          .catch(() => {
                            self.loading = false
                          })
                        self.cancelEditing()
                      } else {
                        self.editingUrlError = true
                      }
                    } else if (e.keyCode === 27) {
                      self.cancelEditing()
                    }
                  },
                },
              })
            } else {
              element = params.row.URL
            }
            return h('div', [element])
          },
        },
        { title: '组名', key: 'Group', width: 320 },
      ],
      selectedRows: [],
      editingRowIndex: -1,
      editingRowUrl: '',
      editingUrlError: false,
    }
  },
  computed: {
    ...mapState(['appConfig']),
    tableData () {
      return this.appConfig.serverSubscribes
    },
    cycle () {
      const interval = this.appConfig.subscribeUpdateInterval
      const cycle = { number: 1, unit: 'hour' }
      if (interval % 24 === 0) {
        if (interval % 168 === 0) {
          cycle.unit = 'week'
          cycle.number = interval / 168
        } else {
          cycle.unit = 'day'
          cycle.number = interval / 24
        }
      } else {
        cycle.number = interval
      }
      return cycle
    },
  },
  watch: {
    url () {
      if (this.urlError) {
        this.urlError = false
      }
    },
    editingRowUrl () {
      if (this.editingUrlError) {
        this.editingUrlError = false
      }
    },
  },
  methods: {
    ...mapMutations(['updateView']),
    ...mapActions(['updateConfig', 'updateConfigs', 'updateSubscribes']),
    selectRows (rows) {
      this.selectedRows = rows
    },
    onRowDBClick (row, index) {
      if (this.editingRowIndex < 0) {
        this.editingRowIndex = index
        this.editingRowUrl = row.URL
      }
      this.$nextTick(() => {
        // hack，使用ref不行...
        document
          .getElementById('editing-input')
          .querySelector('input')
          .focus()
      })
    },
    onUpdateChange (v) {
      this.updateConfig({ autoUpdateSubscribes: v })
    },
    onChangeCycleNumber (v) {
      const value = parseInt(v) || 1
      this.updateConfig({
        subscribeUpdateInterval: value * unitMap[this.cycle.unit],
      })
    },
    onChangeCycleUnit (v) {
      this.updateConfig({
        subscribeUpdateInterval: this.cycle.number * unitMap[v],
      })
    },
    update () {
      this.loading = true
      this.updateSubscribes(this.selectedRows).then(updatedCount => {
        this.loading = false
        this.$Message.success(`已更新${updatedCount}个节点`)
      })
    },
    remove () {
      const removeGroup = this.selectedRows.map(row => row.Group)
      const clone = this.appConfig.serverSubscribes.filter(config => removeGroup.indexOf(config.Group) < 0)
      this.updateConfig({ serverSubscribes: clone })
      this.selectedRows = []
    },
    // 同时使用electron的net和fetch api请求
    requestSubscribeUrl (url) {
      return somePromise([request(url, true), fetch(url).then(res => res.text())])
    },
    // 根据订阅返回的节点数据更新 V2ray 配置项
    updateSubscribedConfigs (configs) {
      const group = configs[0].group
      const notInGroup = this.appConfig.configs.filter(config => config.group !== group)
      this.updateConfigs(notInGroup.concat(configs))
    },
    onCreate () {
      // this.showNewUrl = true
      const group = this.group
      // const url = this.url
      if (this.appConfig.serverSubscribes.every(serverSubscribe => {
        return serverSubscribe.Group !== group
      })) {
        this.save()
      }
      this.$nextTick(() => {
        this.$refs.inputGroup.focus()
      })
    },
    cancel () {
      this.showNewUrl = false
      this.url = ''
      this.urlError = false
      this.updateView({ active: false })
    },
    cancelEditing () {
      this.editingRowIndex = -1
      this.editingRowUrl = ''
      this.editingUrlError = false
    },
    save () {
      if (URL_REGEX.test(this.url)) {
        this.loading = true
        const group = this.group
        const url = this.url
        // 未发生改变
        if (
          this.appConfig.serverSubscribes.every(serverSubscribe => {
            return serverSubscribe.Group !== group
          })
        ) {
          this.requestSubscribeUrl(url)
            .then(res => {
              this.loading = false
              const configs = isSubscribeContentValid(res)
              if (configs) {
                configs.forEach(config => {
                  config.group = group
                })
                const clone = this.appConfig.serverSubscribes.slice()
                clone.push({ URL: url, Group: group })
                this.updateConfig({
                  serverSubscribes: clone,
                  configs: this.appConfig.configs.concat(configs),
                })
              }
            })
            .catch(() => {
              this.loading = false
            })
        } else {
          this.loading = false
        }
        this.cancel()
      } else {
        this.urlError = true
      }
    },
  },
  mounted () {
    // 支持初始化打开新增输入框
    if (this.$store.state.view.active) {
      this.showNewUrl = true
      setTimeout(() => {
        this.$refs.inputGroup.focus()
      }, 300)
    }
  },
}
</script>
<style lang="stylus">
.options-container
  .cycle-wrapper
    .ivu-input-wrapper
      width auto
    .ivu-input
      padding 0
      width 24px
      height 24px
      border-left none
      border-top none
      border-right none
      border-radius 0
      text-align center
      &:focus
        box-shadow none
    .ivu-select
      width 24px
      height 24px
      &.ivu-select-visible
        .ivu-select-selection
          box-shadow none
      .ivu-select-selection
        height 24px
        border-left none
        border-top none
        border-right none
        border-radius 0
      .ivu-select-selected-value
        margin-right 0
        padding 0
        height 24px
        text-align center
        line-height 24px
      .ivu-select-arrow
        display none
      .ivu-select-dropdown
        .ivu-select-item
          padding-left 0
          padding-right 0
          text-align center
</style>
