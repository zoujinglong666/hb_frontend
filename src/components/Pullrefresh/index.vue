<template>
  <div ref="scroller" class="nut-pull-refresh" @touchstart="touchStart" @touchmove="touchMove"
       @touchend="touchEnd">
    <div class="nut-pull-refresh-container" :style="getStyle">
      <div class="nut-pull-refresh-container-topbox" :style="getHeightStyle">
        <Loading
          v-if="state.status == 'loading' && !slots.loading"
        ></Loading>

        <div class="nut-pull-refresh-container-topbox-text">{{ getPullStatus }}</div>

        <slot v-if="state.status == 'pulling'" name="pulling"></slot>
        <slot v-if="state.status == 'loosing'" name="loosing"></slot>
        <slot v-if="state.status == 'loading'" name="loading"></slot>
        <slot v-if="state.status == 'complete'" name="complete"></slot>
      </div>
      <slot></slot>
    </div>
  </div>
</template>
<script lang="ts" setup>
import {computed, CSSProperties, nextTick, reactive, ref, watch} from 'vue'
import {useTouch} from '@/hooks/useTouch.ts'
import {useScrollParent} from '@/hooks/useScrollParent/index.ts'
import Loading from '@/components/ScrollList/components/Loading.vue'

const pxCheck = (value?: string | number): string | undefined => {
  if ( value !== undefined ) {
    return isNaN ( Number ( value ) ) ? String ( value ) : `${value}px`
  }
}
const slots = useSlots ()

type PullRefreshStatus = 'normal' | 'loading' | 'loosing' | 'pulling' | 'complete'
const getScrollTopRoot = (): number => {
  return window.pageYOffset || document.documentElement.scrollTop || document.body.scrollTop || 0
}
const cN = 'NutPullRefresh'
const emit = defineEmits ( ['change', 'refresh', 'update:modelValue'] )
const props = defineProps ( {
  modelValue: {
    type: Boolean,
    default: false
  },

  pullingTxt: {
    type: String,
    default: ''
  },
  loosingTxt: {
    type: String,
    default: ''
  },
  loadingTxt: {
    type: String,
    default: ''
  },

  completeTxt: {
    type: String,
    default: ''
  },
  headHeight: {
    type: [String, Number],
    default: 50
  },

  pullDistance: {
    type: [String, Number],
    default: 80
  },

  duration: {
    type: [String, Number],
    default: 0.5
  },
  completeDuration: {
    type: Number,
    default: 0
  }
} )

const touch: any = useTouch ()
const scroller = ref<HTMLElement> ()
const scrollParent = useScrollParent ( scroller )

const state = reactive<{
  isPullRefresh: Boolean
  distance: Number
  status: PullRefreshStatus
}> ( {
  isPullRefresh: false,
  distance: 0,
  status: 'normal'
} )
const getPullStatus = computed ( () => {
  switch (state.status) {
    case 'pulling':
      return !slots.pulling ? props.pullingTxt || 'pulling' : ''
    case 'loosing':
      return !slots.loosing ? props.loosingTxt || 'loosing' : ''
    case 'loading':
      return !slots.loading ? props.loadingTxt || '' : ''
    case 'complete':
      return !slots.complete ? props.completeTxt || 'complete' : ''
    default:
      break
  }
  return ''
} )

const getStyle = computed ( () => {
  return {
    transitionDuration: `${props.duration}s`,
    transform: state.distance ? `translate3d(0,${state.distance}px, 0)` : ''
  }
} )

const getHeightStyle = computed ( () => {
  const styles: CSSProperties = {}
  if ( props.headHeight != 50 ) styles.height = pxCheck ( props.headHeight )
  return styles
} )

const timing = (distance: number) => {
  const pullDistance = +(props.pullDistance || props.headHeight)
  let moveDistance = distance
  if ( distance > pullDistance ) {
    if ( distance < pullDistance * 2 ) {
      moveDistance = (distance + pullDistance) / 2
    } else {
      moveDistance = pullDistance + distance / 4
    }
  }

  return Math.round ( moveDistance )
}

const setPullStatus = (distance: number, isLoading?: boolean, isComplete?: boolean) => {
  const pullDistance = +(props.pullDistance || props.headHeight)
  state.distance = distance

  if ( isLoading ) {
    state.status = 'loading'
  } else if ( isComplete ) {
    state.status = 'complete'
  } else if ( distance === 0 ) {
    state.status = 'normal'
  } else if ( distance < pullDistance ) {
    state.status = 'pulling'
  } else {
    state.status = 'loosing'
  }

}

const isCanTouch = () => state.status !== 'loading' && state.status !== 'complete'

const isScrollTop = () => {
  if ( scrollParent.value == window ) {
    return getScrollTopRoot () == 0
  } else {
    return scrollParent.value && (scrollParent.value as Element).scrollTop == 0
  }
}

const touchStart = (event: TouchEvent) => {
  if ( isCanTouch () ) {
    if ( isScrollTop () ) {
      touch.start ( event )
      state.isPullRefresh = true
    } else {
      state.distance = 0
      state.isPullRefresh = false
    }
  }
}

const touchMove = (event: TouchEvent) => {
  if ( isCanTouch () ) {
    touch.move ( event )

    const {deltaY} = touch

    if ( (touch as any).isVertical () && deltaY.value > 0 && state.isPullRefresh ) {
      event.preventDefault ()
      setPullStatus ( timing ( deltaY.value ) )
    }
  }
}

const touchEnd = () => {
  if ( state.isPullRefresh && isCanTouch () && touch.deltaY.value ) {
    if ( state.status === 'loosing' ) {
      setPullStatus ( +props.headHeight, true )
      emit ( 'update:modelValue', true )
      nextTick ( () => emit ( 'refresh' ) )
      setTimeout ( () => {
        emit ( 'update:modelValue', false )
      }, 1000 )

    } else {
      setPullStatus ( 0 )
    }
  }

  setTimeout ( () => {
    touch.reset ()
  }, 0 )
}

watch (
  () => props.modelValue,
  (val) => {
    if ( val ) {
      setPullStatus ( +props.headHeight, true )
    } else {
      if ( props.completeDuration === 0 ) setPullStatus ( 0 )
      setPullStatus ( +props.headHeight, false, true )
      setTimeout ( () => {
        setPullStatus ( 0 )
      }, props.completeDuration )
    }
  }
)
</script>

<style lang="less" scoped>
.nut-pull-refresh {
  height: 100%;
  overflow: auto;

  &-container {
    position: relative;
    height: 100%;

    &-topbox {
      position: absolute;
      left: 0;
      width: 100%;
      height: 50px;
      transform: translateY(-100%);
      text-align: center;
      font-size: 14px;
      display: flex;
      align-items: center;
      justify-content: center;

      &-icon {
        margin-right: 4px;
        width: 16px;
        height: 16px;
      }

      &-text {
        font-size: 14px;
        color: #333;
      }
    }
  }
}

</style>
