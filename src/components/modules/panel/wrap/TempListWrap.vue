<!--
 * @Author: ShawnPhang
 * @Date: 2021-08-27 15:16:07
 * @Description: 模板列表
 * @LastEditors: ShawnPhang <https://m.palxp.cn>
 * @Date: 2024-03-06 21:16:00
-->
<template>
  <div class="wrap">
    <search-header v-model="state.searchKeyword" @change="cateChange" />
    <el-divider v-show="state.title" style="margin-top: 1.7rem" content-position="left">
      <span style="font-weight: bold">{{ state.title }}</span>
    </el-divider>

    <ul ref="listRef" v-infinite-scroll="load" class="infinite-list" :infinite-scroll-distance="150" style="overflow: auto">
      <img-water-fall :listData="state.list" @select="selectItem" />
      <div v-show="state.loading" class="loading"><i class="el-icon-loading"></i> 拼命加载中</div>
      <div v-show="state.loadDone" class="loading">全部加载完毕</div>
    </ul>
  </div>
</template>

<script lang="ts" setup>
import { reactive, ref } from 'vue'
import api from '@/api'

import { LocationQueryValue, useRoute, useRouter } from 'vue-router'
// import chooseType from './components/chooseType.vue'
// import editModel from './components/editModel.vue'
import searchHeader from './components/searchHeader.vue'
import useConfirm from '@/common/methods/confirm'
// import { useSetupMapGetters } from '@/common/hooks/mapGetters'
import imgWaterFall from './components/imgWaterFall.vue'
import { IGetTempListData } from '@/api/home'
import { useControlStore, useCanvasStore, useUserStore, useHistoryStore, useWidgetStore, useForceStore } from '@/store'
import { storeToRefs } from 'pinia'

type TState = {
  loading: boolean
  loadDone: boolean
  list: IGetTempListData[]
  title: string
  searchKeyword: string
}

type TPageOptions = {
  page: number
  pageSize: number
  cate: number | string
  state?: string
}

const listRef = ref<HTMLElement | null>(null)
const route = useRoute()
const router = useRouter()

const controlStore = useControlStore()

const userStore = useUserStore()
const pageStore = useCanvasStore()
const widgetStore = useWidgetStore()
const forceStore = useForceStore()
const state = reactive<TState>({
  loading: false,
  loadDone: false,
  list: [],
  title: '推荐模板',
  searchKeyword: '',
})

// const { tempEditing } = useSetupMapGetters(['tempEditing'])
const { dHistoryParams } = storeToRefs(useHistoryStore())

const pageOptions: TPageOptions = { page: 0, pageSize: 20, cate: 1 }
const { cate, edit } = route.query
cate && (pageOptions.cate = (cate as LocationQueryValue) ?? 1)
// edit && store.commit('managerEdit', true)
edit && userStore.managerEdit(true)

// onMounted(async () => {})

const load = async (init: boolean = false, stat?: string) => {
  stat && (pageOptions.state = stat)

  if (init && listRef.value) {
    listRef.value.scrollTop = 0
    state.list = []
    pageOptions.page = 0
    state.loadDone = false
  }
  if (state.loadDone || state.loading) {
    return
  }

  state.loading = true
  pageOptions.page += 1

  const res = await api.home.getTempList({ search: state.searchKeyword, ...pageOptions })
  res.list.length <= 0 && (state.loadDone = true)
  state.list = state.list.concat(res.list)
  setTimeout(() => {
    state.loading = false
    checkHeight()
  }, 100)
}

function cateChange(type: any) {
  state.title = type.name
  const init = pageOptions.cate != type.id
  pageOptions.cate = type.id
  load(init, pageOptions.state)
}

function checkHeight() {
  if (!listRef.value) return
  // 检查高度是否占满，否则继续请求下一页
  const isLess = listRef.value.offsetHeight > (listRef.value.firstElementChild as HTMLElement)?.offsetHeight
  isLess && load()
}

async function selectItem(item: IGetTempListData) {
  controlStore.setShowMoveable(false) // 清理掉上一次的选择框

  if (dHistoryParams.value.length > 0) {
    const isPass = await useConfirm('提示', '使用模板后，当前页面将会被替换，是否继续', 'warning')
    if (!isPass) {
      return false
    }
  }
  userStore.managerEdit(false)
  widgetStore.setDWidgets([])
  setTempId(item.id)

  let result = null
  if (!item.data) {
    const res = await api.home.getTempDetail({ id: item.id })
    result = JSON.parse(res.data)
  } else {
    result = JSON.parse(item.data)
  }
  const { page, widgets } = result
  pageStore.setDPage(page)
  widgetStore.setTemplate(widgets)
  setTimeout(() => {
    forceStore.setZoomScreenChange()
  }, 300)
  widgetStore.selectWidget({
    uuid: '-1',
  })
}

function setTempId(tempId: number | string) {
  const { id } = route.query
  router.push({ path: '/home', query: { tempid: tempId, id }, replace: true })
}

defineExpose({
  load,
  cateChange,
  listRef,
})
</script>

<style lang="less" scoped>
.wrap {
  width: 100%;
  height: 100%;
}

.infinite-list {
  height: 100%;
  margin-top: 1rem;
  padding-bottom: 150px;
  scrollbar-width: none; /* Firefox */
  -ms-overflow-style: none; /* IE 10+ */
}
.infinite-list::-webkit-scrollbar {
  display: none; /* Chrome Safari */
}
// .list {
//   width: 100%;
//   padding: 4px 0 0 10px;
//   &__img {
//     cursor: pointer;
//     width: 128px;
//     height: auto;
//     position: relative;
//     &-mask {
//       opacity: 0;
//       width: 100%;
//       height: 100%;
//       background: rgba(0, 0, 0, 0.12);
//       position: absolute;
//       z-index: 1;
//       top: 0;
//       left: 0;
//     }
//   }
//   &__img:hover {
//     background: rgba(0, 0, 0, 0.04);
//   }
//   &__img:hover > &__img-mask {
//     opacity: 1;
//   }
// }

.loading {
  padding-top: 1rem;
  text-align: center;
  font-size: 14px;
  color: #999;
}
</style>
