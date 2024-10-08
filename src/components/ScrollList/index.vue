
<script setup lang="ts">
import {nextTick, onMounted, onUnmounted, ref} from 'vue';
import Loading from "@/components/ScrollList/components/Loading.vue";
import {isFunction, uniqBy} from "lodash-es";
import {ResultEnum} from "@/enums/httpEnum";
import {watch} from "vue";
const scrollRef = ref<HTMLElement | null> ( null );
const bodyClass = ref<string> ( '' );
const isPullingDown = ref ( false );
const isRefreshing = ref ( false );
const startY = ref ( 0 );
const startX = ref ( 0 );
const endY = ref ( 0 );
const endX = ref ( 0 );
const distanceY = ref ( 0 );
const distanceX = ref ( 0 );
const loadLock = ref ( false );
const isLoading = ref ( false );
const startTime = ref ( 0 );
const endTime = ref ( 0 );
const DISTANCE_Y_MIN_LIMIT = 80;
const DISTANCE_Y_MAX_LIMIT = 150;
const DEG_LIMIT = 40;
const SPEED_THRESHOLD = 0.5; // 滑动速度阈值
const MAX_CLICK_DURATION = 200; // Max duration for a click (ms)
const MAX_CLICK_MOVEMENT = 10; // Max movement for a click (px)

const start = (e: TouchEvent) => {
  if ( loadLock.value ) return;
  const touch = e.touches[0];
  startY.value = touch.clientY;
  startX.value = touch.clientX;
  startTime.value = e.timeStamp;
  isPullingDown.value = true;
};

const throttledMove = (() => {
  let lastCall = 0;
  return (e: TouchEvent) => {
    const now = Date.now ();
    if ( now - lastCall < 16 ) return; // 60 FPS
    lastCall = now;
    move ( e );
  };
}) ();

const move = (e: TouchEvent) => {
  if ( loadLock.value ) return;

  const touch = e.touches[0];
  endY.value = touch.clientY;
  endX.value = touch.clientX;
  distanceY.value = endY.value - startY.value;
  distanceX.value = endX.value - startX.value;

  if ( distanceY.value < 0 ) return; // Ignore upward movement

  const deg = Math.atan ( Math.abs ( distanceX.value ) / distanceY.value ) * (180 / Math.PI);
  if ( deg > DEG_LIMIT ) {
    startY.value = endY.value;
    startX.value = endX.value;
    return;
  }

  if ( isPullingDown.value && distanceY.value > 40 ) {
    updateBoxStyle ( distanceY.value );
  }

  let percent = (100 - distanceY.value * 0.5) / 100;
  percent = Math.max ( 0.5, percent );
  distanceY.value *= percent;
  if ( distanceY.value > DISTANCE_Y_MAX_LIMIT ) {
    distanceY.value = DISTANCE_Y_MAX_LIMIT;
  }
  updateBoxStyle ( distanceY.value );
};

const end = (e: TouchEvent) => {
  if ( loadLock.value ) return;
  isPullingDown.value = false;
  endTime.value = e.timeStamp;

  const timeDiff = endTime.value - startTime.value;
  const speed = distanceY.value / timeDiff;

  if ( timeDiff < MAX_CLICK_DURATION && Math.abs ( distanceY.value ) < MAX_CLICK_MOVEMENT && Math.abs ( distanceX.value ) < MAX_CLICK_MOVEMENT ) {
    resetBoxStyle ();
    bodyClass.value = '';
    return;
  }

  if ( speed > SPEED_THRESHOLD || endY.value - startY.value < 0 ) {
    resetBoxStyle ();
    bodyClass.value = '';
    return;
  }

  if ( distanceY.value < DISTANCE_Y_MIN_LIMIT ) {
    resetBoxStyle ();
    bodyClass.value = '';
    return;
  }

  loadLock.value = true;
  isLoading.value = true;
  isRefreshing.value = true;
  updateBoxStyle ( 40 );
  bodyClass.value = 'overflowHidden';


  setTimeout ( () => {
    loadLock.value = false;
    isLoading.value = false;
    isRefreshing.value = false;
    resetBoxStyle ();
    bodyClass.value = '';
    emit ( 'refresh' );
  }, 1000 );
};

const updateBoxStyle = (translateY: number) => {
  if ( scrollRef.value ) {
    scrollRef.value.style.transform = `translateY(${translateY}px)`;
    scrollRef.value.style.transition = 'transform 0.3s ease-out';
  }
};

const resetBoxStyle = () => {
  if ( scrollRef.value ) {
    startY.value = 0;
    startX.value = 0;
    endY.value = 0;
    endX.value = 0;
    distanceY.value = 0;
    distanceX.value = 0;
    scrollRef.value.style.transform = 'translateY(0px)';
    scrollRef.value.style.transition = 'transform 0.3s ease-in-out';
  }
};

const addTouchEvent = () => {
  const parentElement = scrollRef.value;
  if ( parentElement ) {
    parentElement.addEventListener ( 'touchstart', start, {passive: false} );
    parentElement.addEventListener ( 'touchmove', throttledMove, {passive: false} );
    parentElement.addEventListener ( 'touchend', end, {passive: false} );
    parentElement.addEventListener ( 'touchcancel', end, {passive: false} );
  }
};

const removeTouchEvent = () => {
  const parentElement = scrollRef.value;
  if ( parentElement ) {
    parentElement.removeEventListener ( 'touchstart', start );
    parentElement.removeEventListener ( 'touchmove', throttledMove );
    parentElement.removeEventListener ( 'touchend', end );
    parentElement.removeEventListener ( 'touchcancel', end );
  }
};



onUnmounted ( () => {
  removeTouchEvent ();
} );

const moreRef = ref ( null );
const nothing = ref ( null );
const hasMore = ref ( true );
const moreLock = ref ( false );

const observer = new IntersectionObserver (
  ([entry]) => {
    if ( entry.isIntersecting ) {
      pullUp ();
    }
  },
  {
    root: null,
    rootMargin: '0px',
    threshold: 0,
  }
);

interface DataListPropsType {

  data?: any[];
  // 自定义api
  api?: (params?: {}) => Promise<any>;
  pageSize?: number,
  params?: NonNullable<unknown>;
  offset?: number
  rowKey?: string,
  refreshing?: boolean,

}

const props = withDefaults ( defineProps<DataListPropsType> (), {
  rowKey: "id",
  pageSize: 5,
  offset: 100,
} );

const DEFAULT_PAGE_SIZE = 10
const list = ref ( [] );
const loading = ref ( false );
const finished = ref ( false );
const refreshing = ref ( false );
const pageNum = ref ( 1 );
const pages = ref ( 1 );
const total = ref ( 0 )
const pullUp = () => {
  if ( !hasMore.value || moreLock.value ) {
    return;
  }
  loadData ();
};
const isLastPage = computed ( () => {
  return pageNum.value >= Number ( pages.value )
} )

watch (
  () => isLastPage.value,
  (newVal) => {
    finished.value = newVal
  },
  {deep: true, immediate: true}
);
const emit = defineEmits ( ['loadMore', 'refresh','update:refreshing','update:list'] );
const loadData = () => {
  emit ( 'loadMore' );
  if ( finished.value ) {
    return
  }
  if ( pageNum.value < Number ( pages.value ) ) {
    pageNum.value += 1;
    getData ( pageNum.value )
  }
};


onMounted ( () => {
  moreRef.value && observer.observe ( moreRef.value );
  nextTick ( addTouchEvent );
  fetchData ();
} );
const fetchData = async () => {
  await getData ( 1 );
}


const getData = async (pageNum: number) => {
  const api = props?.api;
  if ( !api || !isFunction ( api ) ) {
    return;
  }
  try {
    loading.value = true;
    const params = {
      "pageNum": pageNum,
      "pageSize": props.pageSize || 10,
      ...(props.params || {}),

    };
    const {data, code} = await api ( params );
    if ( code == ResultEnum.SUCCESS ) {
      pages.value = Number ( data.pages );
      total.value = Number ( data.total );
      const pageData = data.records as any[];
      if ( pages.value === 1 || data.total <= props.pageSize ) {
        finished.value = true
        list.value = pageData || []
        emit( 'update:list', pageData )
        return
      } else {
        list.value = uniqBy ( (list.value ?? []).concat ( pageData ), props.rowKey );
        emit ( 'update:list', list.value )
      }
    }
  } catch (e) {
    console.log ( e )
  } finally {
    refreshing.value = false;
    loading.value = false;
  }
}
onMounted(() => {
  nextTick(() => {
    if (moreRef.value) {
      observer.observe(moreRef.value);
    }
  });
});


</script>
<template>
  <div class="data-scroll-list" ref="scrollRef">
    <div class="pull-down_text" v-if="isPullingDown && distanceY > 5">
      {{ isPullingDown && distanceY > DISTANCE_Y_MIN_LIMIT ? '松开刷新' : ' 正在下拉刷新...' }}
    </div>
    <div class="pull-down_loading" v-if="isRefreshing">
      <Loading></Loading>
    </div>
    <slot></slot>
    <div class="pull-up">
      <div ref="moreRef" class="pull-up_loading" v-show="!finished"></div>
      <div ref="nothing" class="pull-up_nothing" v-show="finished">没有更多了</div>
    </div>
  </div>
</template>


<style scoped lang="less">
.data-scroll-list {
  position: relative;
  width: 100vw;
  padding-bottom: 60px;

  .pull-down_text {
    position: relative;
    height: 20px;
    text-align: center;
    margin-bottom: 20px;
    color: #717171;
  }

  .pull-down_loading {
    position: relative;
    top: -20px;
    height: 20px;
    display: flex;
    align-items: center;
    justify-content: center;
    line-height: 40px;
  }

  .pull-up {
    height: 40px;
    display: flex;
    align-items: center;
    justify-content: center;
    color: #717171;
    &_loading {
      width: 16px;
      height: 16px;
      border: 1px solid #eee;
      border-radius: 50%;
      border-bottom: 1px solid #717171;
      transform: rotate(0deg);
      animation: loading 1s linear infinite;
    }
  }
}

/* 禁止滚动的样式 */
.no-scroll {
  overflow: hidden;
}

@keyframes loading {
  from {
    transform: rotate(0deg);
  }
  to {
    transform: rotate(360deg);
  }
}
</style>
