<template>
  <div class="slide slide-infinite">
    <Loading v-if="props.loading && props.list.length === 0" />
    <div
      class="slide-list flex-direction-column"
      ref="slideListEl"
      @pointerdown.prevent="touchStart"
      @pointermove.prevent="touchMove"
      @pointerup.prevent="touchEnd"
    >
      <SlideItem
        v-for="(item, index) in renderList"
        :key="item.aweme_id"
        :data-index="(curIndex = index + renderPos.start)"
        :style="{
          top: top + 'px'
        }"
      >
        <slot :item="item" :index="curIndex"></slot>
      </SlideItem>
    </div>
  </div>
</template>

<script setup lang="tsx">
import { computed, onMounted, reactive, ref, watch } from 'vue'
import {
  SlideType,
  slideInit,
  slideReset,
  slideTouchEnd,
  slideTouchMove,
  slideTouchStart
} from './slide'
import SlideItem from './SlideItem.vue'
import Loading from './Loading.vue'

const props = defineProps({
  index: {
    type: Number,
    default: () => {
      return -1
    }
  },
  list: {
    type: Array,
    default: () => {
      return []
    }
  },
  //页面中同时存在多少个SlideItem
  virtualTotal: {
    type: Number,
    default: () => 5
  },
  loading: {
    type: Boolean,
    default: () => false
  },
  active: {
    type: Boolean,
    default: () => false
  }
})

const emit = defineEmits(['update:index', 'loadMore', 'refresh'])

const slideListEl = ref<HTMLDivElement>(null)

const state = reactive({
  homeRefresh: 60,
  judgeValue: 20,
  type: SlideType.VERTICAL_INFINITE,
  localIndex: props.index,
  needCheck: true,
  next: false,
  isDown: false,
  start: { x: 0, y: 0, time: 0 },
  move: { x: 0, y: 0 },
  wrapper: { width: 0, height: 0, childrenLength: 0 }
})

watch(
  () => props.active,
  (newVal) => {
    //当激活此页时，如果list为空，那么向上发射事件通知父组件请求数据
    if (newVal && !props.list.length) {
      return emit('refresh')
    }
  },
  {
    immediate: true
  }
)

watch(
  () => props.index,
  (newVal) => {
    state.localIndex = newVal
  }
)

onMounted(() => {
  slideInit(slideListEl.value, state)
})

const half = computed(() => {
  return (props.virtualTotal - 1) / 2
})

const renderPos = computed(() => {
  let start = 0
  if (state.localIndex > half.value) {
    start = state.localIndex - half.value
  }
  let end = start + props.virtualTotal
  if (end >= props.list.length) {
    end = props.list.length
    start = end - props.virtualTotal
  }
  if (start < 0) start = 0

  return {
    start,
    end
  }
})

const renderList = computed(() => {
  const { start, end } = renderPos.value
  return props.list.slice(start, end)
})

function touchStart(e) {
  slideTouchStart(e, slideListEl.value, state)
}

function touchMove(e) {
  slideTouchMove(e, slideListEl.value, state, canNext)
}

const top = ref(0)

function touchEnd(e) {
  let isNext = state.move.y < 0
  if (state.localIndex === 0 && !isNext && state.move.y > state.homeRefresh + state.judgeValue) {
    emit('refresh')
  }
  slideTouchEnd(e, state, canNext, (isNext) => {
    // let half = (props.virtualTotal - 1) / 2
    if (props.list.length > props.virtualTotal) {
      //手指往上滑(即列表展示下一条内容)
      if (isNext) {
        //删除最前面的 `dom` ，然后在最后面添加一个 `dom`
        if (
          state.localIndex > props.list.length - props.virtualTotal &&
          state.localIndex > half.value
        ) {
          emit('loadMore')
        }
        if (state.localIndex > half.value && state.localIndex < props.list.length - half.value) {
          top.value = (state.localIndex - half.value) * state.wrapper.height
        }
      } else {
        if (
          state.localIndex >= half.value &&
          state.localIndex < props.list.length - (half.value + 1)
        ) {
          top.value = (state.localIndex - half.value) * state.wrapper.height
        }
      }
      state.wrapper.childrenLength = slideListEl.value.children.length
    }
  })
  slideReset(e, slideListEl.value, state, emit)
}

function canNext(state, isNext: boolean) {
  return !(
    (state.localIndex === 0 && !isNext) ||
    (state.localIndex === props.list.length - 1 && isNext)
  )
}
</script>

<style scoped lang="less">
.slide {
  touch-action: none;
  height: 100%;
  width: 100%;
  transition: height 0.3s;
  position: relative;
  overflow: hidden;

  .slide-infinite {
    z-index: 1;
    margin-top: 0;
    transition: all 0.3s;
  }

  .slide-list {
    height: 100%;
    width: 100%;
    display: flex;
    position: relative;
  }
}
</style>
