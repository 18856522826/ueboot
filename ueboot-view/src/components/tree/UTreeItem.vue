<template>
    <li role="treeitem"
        :class="classes"
        :draggable="draggable"
        @dragstart.stop="onItemDragStart($event, _self, _self.model)"
        @dragend.stop.prevent="onItemDragEnd($event, _self, _self.model)"
        @dragover.stop.prevent="isDragEnter = true"
        @dragenter.stop.prevent="isDragEnter = true"
        @dragleave.stop.prevent="isDragEnter = false"
        @drop.stop.prevent="handleItemDrop($event, _self, _self.model)">
        <div role="presentation" :class="wholeRowClasses" v-if="isWholeRow">&nbsp;</div>
        <i class="tree-icon tree-ocl" role="presentation" @click="handleItemToggle"></i>
        <div :class="anchorClasses" v-on="events">
            <i :class="{'tree-icon':true,'tree-checkbox': true,'tree-undetermined': this.model.undetermined}" role="presentation" v-if="showCheckbox && !model.loading"></i>
            <slot :vm="this" :model="model">
                <i :class="themeIconClasses" role="presentation" v-if="!model.loading"></i>
                <span v-html="model[textFieldName]"></span>
            </slot>
        </div>
        <ul role="group" ref="group" class="tree-children" v-if="isFolder" :style="groupStyle">
            <u-tree-item v-for="(child, index) in model[childrenFieldName]"
                         :key="index"
                         :data="child"
                         :text-field-name="textFieldName"
                         :value-field-name="valueFieldName"
                         :children-field-name="childrenFieldName"
                         :item-events="itemEvents"
                         :whole-row="wholeRow"
                         :show-checkbox="showCheckbox"
                         :allow-transition="allowTransition"
                         :height="height"
                         :parent-item="model[childrenFieldName]"
                         :draggable="draggable"
                         :drag-over-background-color="dragOverBackgroundColor"
                         :on-item-click="onItemClick"
                         :on-item-toggle="onItemToggle"
                         :on-item-drag-start="onItemDragStart"
                         :on-item-drag-end="onItemDragEnd"
                         :on-item-drop="onItemDrop"
                         :klass="index === model[childrenFieldName].length-1?'tree-last':''">
                <template slot-scope="_">
                    <slot :vm="_.vm" :model="_.model">
                        <i :class="_.vm.themeIconClasses" role="presentation" v-if="!model.loading"></i>
                        <Tooltip max-width="300" :content="_.model.tip" v-if="_.model.tip!==''">
                            <span v-html="_.model[textFieldName]"></span>
                        </Tooltip>
                        <span v-html="_.model[textFieldName]" v-else></span>
                    </slot>
                </template>
            </u-tree-item>
        </ul>
    </li>
</template>
<script>
export default {
  name: 'UTreeItem',
  props: {
    async: { type: Boolean, default: false },
    data: { type: Object, required: true },
    textFieldName: { type: String },
    valueFieldName: { type: String },
    childrenFieldName: { type: String },
    itemEvents: { type: Object },
    wholeRow: { type: Boolean, default: false },
    showCheckbox: { type: Boolean, default: false },
    allowTransition: { type: Boolean, default: true },
    height: { type: Number, default: 24 },
    parentItem: { type: Array },
    draggable: { type: Boolean, default: false },
    dragOverBackgroundColor: { type: String },
    onItemClick: {
      type: Function, default: () => false
    },
    onItemToggle: {
      type: Function, default: () => false
    },
    onItemDragStart: {
      type: Function, default: () => false
    },
    onItemDragEnd: {
      type: Function, default: () => false
    },
    onItemDrop: {
      type: Function, default: () => false
    },
    klass: String
  },
  data () {
    return {
      isHover: false,
      isDragEnter: false,
      model: this.data,
      maxHeight: 0,
      events: {}
    }
  },
  watch: {
    isDragEnter (newValue) {
      if (newValue) {
        this.$el.style.backgroundColor = this.dragOverBackgroundColor
      } else {
        this.$el.style.backgroundColor = 'inherit'
      }
    },
    data (newValue) {
      this.model = newValue
    },
    'model.opened': {
      handler: function (val, oldVal) {
        this.onItemToggle(this, this.model)
        this.handleGroupMaxHeight()
      },
      deep: true
    }
  },
  computed: {
    isFolder () {
      // 有这个标识，代表有子节点，但是异步请求时，不会有子节点数据，但是可以有这个标识
      if (this.model.hasChild) {
        return true
      }
      return this.model[this.childrenFieldName] && this.model[this.childrenFieldName].length > 0
    },
    classes () {
      return [
        { 'tree-node': true },
        { 'tree-open': this.model.opened },
        { 'tree-closed': !this.model.opened },
        { 'tree-leaf': !this.isFolder },
        { 'tree-loading': !!this.model.loading },
        { 'tree-drag-enter': this.isDragEnter },
        { [this.klass]: !!this.klass }
      ]
    },
    anchorClasses () {
      return [
        { 'tree-anchor': true },
        { 'tree-disabled': this.model.disabled },
        { 'tree-selected': this.model.selected },
        { 'tree-hovered': this.isHover }
      ]
    },
    wholeRowClasses () {
      return [
        { 'tree-wholerow': true },
        { 'tree-wholerow-clicked': this.model.selected },
        { 'tree-wholerow-hovered': this.isHover }
      ]
    },
    themeIconClasses () {
      return [
        { 'tree-icon': true },
        { 'tree-themeicon': true },
        { [this.model.icon]: !!this.model.icon },
        { 'tree-themeicon-custom': !!this.model.icon }
      ]
    },

    isWholeRow () {
      if (this.wholeRow) {
        if (this.$parent.model === undefined) {
          return true
        } else if (this.$parent.model.opened) {
          return true
        } else {
          return false
        }
      }
      return false
    },
    groupStyle () {
      return {
        'position': this.model.opened ? '' : 'relative',
        'max-height': this.allowTransition ? this.maxHeight + 'px' : '',
        'transition-duration': this.allowTransition && !this.async ? Math.ceil(this.model[this.childrenFieldName].length / 100) * 300 + 'ms' : '',
        'transition-property': this.allowTransition ? 'max-height' : '',
        'display': this.allowTransition ? 'block' : (this.model.opened ? 'block' : 'none')
      }
    }

  },
  methods: {
    handleItemToggle (e) {
      if (this.isFolder) {
        this.$nextTick(() => {
          this.model.opened = !this.model.opened
        })
        this.onItemToggle(this, this.model)
      }
    },
    handleGroupMaxHeight () {
      if (this.allowTransition) {
        let length = 0
        let childHeight = 0
        if (this.model.opened) {
          length = this.$children.length
          for (let children of this.$children) {
            childHeight += children.maxHeight
          }
        }
        this.maxHeight = length * this.height + childHeight
        if (this.$parent.$options._componentTag === 'u-tree-item') {
          this.$parent.handleGroupMaxHeight()
        }
      }
    },
    handleItemClick (e) {
      if (this.model.disabled) return
      this.model.selected = !this.model.selected
      if (this.$parent.$options._componentTag === 'u-tree-item') {
        this.$parent.handleSetUndetermined()
      }
      this.onItemClick(this, this.model, e)
    },
    // 设置为半选状态
    handleSetUndetermined () {
      // 判断子节点是否有勾选的，且不是全部勾选
      let undetermined = false
      let allSelected = true
      if (this.model.children) {
        this.model.children.forEach((c) => {
          // 子节点存在半选、选中时，父级节点都为半选
          if (c.undetermined) {
            undetermined = true
          }
          if (c.selected) {
            undetermined = true
          } else {
            allSelected = false
          }
        })
      }
      this.model.undetermined = undetermined
      this.model.selected = allSelected
      this.$forceUpdate()
      // 循环调用父级的方法
      if (this.$parent.$options._componentTag === 'u-tree-item') {
        this.$parent.handleSetUndetermined()
      }
    },
    handleItemMouseOver () {
      this.isHover = true
    },
    handleItemMouseOut () {
      this.isHover = false
    },
    handleItemDrop (e, oriNode, oriItem) {
      this.$el.style.backgroundColor = 'inherit'
      this.onItemDrop(e, oriNode, oriItem)
    }
  },
  created () {
    const self = this
    const events = {
      'click': this.handleItemClick,
      'mouseover': this.handleItemMouseOver,
      'mouseout': this.handleItemMouseOut
    }
    for (let itemEvent in this.itemEvents) {
      let itemEventCallback = this.itemEvents[itemEvent]
      if (events.hasOwnProperty(itemEvent)) {
        let eventCallback = events[itemEvent]
        events[itemEvent] = function (event) {
          eventCallback(self, self.model, event)
          itemEventCallback(self, self.model, event)
        }
      } else {
        events[itemEvent] = function (event) {
          itemEventCallback(self, self.model, event)
        }
      }
    }
    this.events = events
  },
  mounted () {
    this.handleGroupMaxHeight()
    this.$nextTick(() => {
      this.$emit('on-mounted', true)
    })
  }
}
</script>
