<template>
  <div class="container">
    <ul class="example-list">
      <li class="example-item" @click="show">
          <span class="open">{{selectedText}}</span>
      </li>
    </ul>
    <transition name="picker-fade">
      <div class="picker" v-show="state===1" @touchmove.prevent @click="_cancel">
        <transition name="picker-move">
          <div class="picker-panel" v-show="state===1" @click.stop>
            <div class="picker-choose border-bottom-1px">
              <span class="cancel" @click="_cancel">Cancel</span>
              <span class="confirm" @click="_confirm">Confirm</span>
              <h1 class="picker-title">Title</h1>
            </div>
            <div class="picker-content">
              <div class="mask-top border-bottom-1px"></div>
              <div class="mask-bottom border-top-1px"></div>
              <div class="wheel-wrapper" ref="wheelWrapper">
                <div class="wheel" v-for="(data, index) in pickerData" :key="index">
                  <ul class="wheel-scroll">
                    <li
                      v-for="item in data" :key="item.value"
                      :class="{'wheel-disabled-item':item.disabled}"
                      class="wheel-item">{{item.text}}</li>
                  </ul>
                </div>
              </div>
            </div>
            <div class="picker-footer"></div>
          </div>
        </transition>
      </div>
    </transition>
  </div>
</template>

<script type="text/ecmascript-6">
  import BScroll from '@better-scroll/core'
  import Wheel from '@better-scroll/wheel'
  BScroll.use(Wheel)

  const STATE_HIDE = 0
  const STATE_SHOW = 1

  const COMPONENT_NAME = 'picker'
  const EVENT_SELECT = 'select'
  const EVENT_CANCEL = 'cancel'
  const EVENT_CHANGE = 'change'

  const DATA = [
    {
      text: '北京市',
      value: '110000',
      children: [
        {
          text: "北京市",
          value: '110100'
        }
      ]
    },
    {
      text: '天津市',
      value: '120000',
      children: [
        {
          text: "天津市",
          value: '120000'
        }
      ]
    },
    {
      text: '河北省',
      value: '130000',
      children: [
        {
          text: '石家庄市',
          value: '130100'
        },
        {
          text: '唐山市',
          value: '130200'
        },
        {
          text: '秦皇岛市',
          value: '130300'
        },
        {
          text: '邯郸市',
          value: '130400'
        },
        {
          text: '邢台市',
          value: '130500'
        },
        {
          text: '保定市',
          value: '130600'
        },
        {
          text: '张家口市',
          value: '130700'
        },
        {
          text: '承德市',
          value: '130800'
        }
      ]
    },
    {
      text: '山西省',
      value: '140000',
      children: [
        {
          text: '太原市',
          value: '140100'
        },
        {
          text: '大同市',
          value: '140200'
        },
        {
          text: '阳泉市',
          value: '140300'
        },
        {
          text: '长治市',
          value: '140400'
        },
        {
          text: '晋城市',
          value: '140500'
        },
        {
          text: '朔州市',
          value: '140600'
        },
        {
          text: '晋中市',
          value: '140700'
        }
      ]
    }
  ]

  export default {
    name: COMPONENT_NAME,
    data() {
      return {
        state: STATE_HIDE,
        selectedIndex: [0, 0],
        selectedText: 'open',
        pickerData: []
      }
    },
    created () {
      // generate data
      // like [[{text: 'province1', value: '1'}, {text: 'province2', value: '2'}], [{text: 'city1', value: '11'}, {text: 'city2', value: '22'}]
      // pickerData has two array, the first is province collections, second is city collections
      this._loadPickerData(this.selectedIndex, undefined /* no prevSelectedIndex due to instantiating */)
    },
    computed: {
      prevSelectedIndex: {
        get() {
          return this.selectedIndex
        },
        set(v) {
          this.selectedIndex = v
        }
      }
    },
    methods: {
      _loadPickerData (newSelectedIndex, oldSelectedIndex) {
        let provinces
        let cities
        // first instantiated
        if (!oldSelectedIndex) {
          provinces = DATA.map(({ value, text }) => ({ value, text }))
          cities = DATA[newSelectedIndex[0]].children
          this.pickerData = [provinces, cities]
        } else {
          // provinces'index changed, refresh cities data
          if (newSelectedIndex[0] !== oldSelectedIndex[0]) {
            cities = DATA[newSelectedIndex[0]].children
            this.pickerData.splice(1, 1, cities)
            // Since cities data changed
            // refresh better-scroll to recaculate scrollHeight
            this.$nextTick(() => {
              this.wheels[1].refresh()
            })
          }
        }
      },
      _confirm() {
        if (this._isMoving()) {
          return
        }
        this.hide()

        const currentSelectedIndex = this.selectedIndex = this.wheels.map(wheel => {
          return wheel.getSelectedIndex()
        })

        // store array for preventing multi-collecting array dependencies in Vue Source code
        const pickerData = this.pickerData
        const currentSelectedValue =
              this.selectedText =
                pickerData.map((data, index) => {
                  return data[currentSelectedIndex[index]].text
                }).join('-')
        this.$emit(EVENT_SELECT, currentSelectedIndex, currentSelectedValue)
      },
      _cancel() {
        this.hide()
        this.$emit(EVENT_CANCEL)
      },
      _isMoving() {
        return this.wheels.some((wheel) => {
          return wheel.pending
        })
      },
      show() {
        if (this.state === STATE_SHOW) {
          return
        }
        this.state = STATE_SHOW
        if (!this.wheels) {
          this.$nextTick(() => {
            this.wheels = []
            let wheelWrapper = this.$refs.wheelWrapper
            for (let i = 0; i < this.pickerData.length; i++) {
              this._createWheel(wheelWrapper, i)
            }
          })
        } else {
          for (let i = 0; i < this.pickerData.length; i++) {
            this.wheels[i].enable()
            this.wheels[i].wheelTo(this.selectedIndex[i])
          }
        }
      },
      hide() {
        this.state = STATE_HIDE

        for (let i = 0; i < this.pickerData.length; i++) {
          this.wheels[i].disable()
        }
      },
      refresh() {
        this.$nextTick(() => {
          this.wheels.forEach((wheel, index) => {
            wheel.refresh()
          })
        })
      },
      _createWheel(wheelWrapper, i) {
        const wheels = this.wheels
        if (!wheels[i]) {
          wheels[i] = new BScroll(wheelWrapper.children[i], {
            wheel: {
              selectedIndex: this.selectedIndex[i],
              wheelWrapperClass: 'wheel-scroll',
              wheelItemClass: 'wheel-item'
            },
            probeType: 3
          })
          // when any of wheels'scrolling ended , you should refresh data
          wheels[i].on('scrollEnd', () => {
            const currentSelectedIndex = wheels.map(wheel => wheel.getSelectedIndex())
            this._loadPickerData(currentSelectedIndex, this.prevSelectedIndex)
            this.prevSelectedIndex = currentSelectedIndex
            this.$emit(EVENT_CHANGE, i, this.wheels[i].getSelectedIndex())
          })
        } else {
          this.wheels[i].refresh()
        }

        return this.wheels[i]
      }
    }
  }
</script>

<style scoped lang="stylus" rel="stylesheet/stylus">

  /* reset */
  ul
    list-style none
    padding 0

  .example-list
    display: flex
    justify-content: space-between
    flex-wrap: wrap
    margin: 2rem

    .example-item
      background-color white
      padding: 0.8rem
      border: 1px solid rgba(0, 0, 0, .1)
      box-shadow: 0 1px 2px 0 rgba(0,0,0,0.1)
      text-align: center
      margin-bottom: 1rem
      flex: 1
      &.placeholder
        visibility: hidden
        height: 0
        margin: 0
        padding: 0

  .picker
    position: fixed
    left: 0
    top: 0
    z-index: 100
    width: 100%
    height: 100%
    overflow: hidden
    text-align: center
    font-size: 14px
    background-color: rgba(37, 38, 45, .4)
    &.picker-fade-enter, &.picker-fade-leave-active
      opacity: 0
    &.picker-fade-enter-active, &.picker-fade-leave-active
      transition: all .3s ease-in-out

    .picker-panel
      position: absolute
      z-index: 600
      bottom: 0
      width: 100%
      height: 273px
      background: white
      &.picker-move-enter, &.picker-move-leave-active
        transform: translate3d(0, 273px, 0)
      &.picker-move-enter-active, &.picker-move-leave-active
        transition: all .3s ease-in-out
      .picker-choose
        position: relative
        height: 60px
        color: #999
        .picker-title
          margin: 0
          line-height: 60px
          font-weight: normal
          text-align: center
          font-size: 18px
          color: #333
        .confirm, .cancel
          position: absolute
          top: 6px
          padding: 16px
          font-size: 14px
        .confirm
          right: 0
          color: #007bff
          &:active
            color: #5aaaff
        .cancel
          left: 0
          &:active
            color: #c2c2c2
      .picker-content
        position: relative
        top: 20px
        .mask-top, .mask-bottom
          z-index: 10
          width: 100%
          height: 68px
          pointer-events: none
          transform: translateZ(0)
        .mask-top
          position: absolute
          top: 0
          background: linear-gradient(to top, rgba(255, 255, 255, 0.4), rgba(255, 255, 255, 0.8))
        .mask-bottom
          position: absolute
          bottom: 1px
          background: linear-gradient(to bottom, rgba(255, 255, 255, 0.4), rgba(255, 255, 255, 0.8))
      .wheel-wrapper
        display: flex
        padding: 0 16px
        .wheel
          -ms-flex: 1 1 0.000000001px
          -webkit-box-flex: 1
          -webkit-flex: 1
          flex: 1
          -webkit-flex-basis: 0.000000001px
          flex-basis: 0.000000001px
          width: 1%
          height: 173px
          overflow: hidden
          font-size: 18px
          .wheel-scroll
            padding: 0
            margin-top: 68px
            line-height: 36px
            list-style: none
            .wheel-item
              list-style: none
              height: 36px
              overflow: hidden
              white-space: nowrap
              color: #333
              &.wheel-disabled-item
                opacity: .2;
    .picker-footer
      height: 20px
</style>
