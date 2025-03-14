<template>
  <div id="picture-management-page">
    <a-flex justify="space-between">
      <h2>Picture management</h2>
      <a-space>
        <a-button type="primary" href="/picture/add" target="_blank">+ Create Picture</a-button>
        <a-button type="primary" href="/picture/add/batch" target="_blank" ghost
          >+ Batch create picture</a-button
        >
      </a-space>
    </a-flex>
    <div style="margin-bottom: 2rem"></div>
    <a-form
      :model="searchParamRef"
      name="search-bar"
      layout="inline"
      autocomplete="off"
      @finish="doSearch"
    >
      <a-form-item label="Key words" name="keywords">
        <a-input
          v-model:value="searchParamRef.searchText"
          allow-clear
          placeholder="Input name or introduction"
        >
        </a-input>
      </a-form-item>

      <a-form-item label="Picture Cateogry" name="category">
        <a-input
          v-model:value="searchParamRef.category"
          allow-clear
          placeholder="Input category name"
        >
        </a-input>
      </a-form-item>
      <a-form-item label="Tag" name="tags">
        <a-select
          v-model:value="searchParamRef.tags"
          mode="tags"
          placeholder="Input tags"
          style="min-width: 10rem"
          allow-clear
        />
      </a-form-item>
      <a-form-item label="Review Status" name="reviewStatus">
        <a-select
          v-model:value="searchParamRef.reviewStatus"
          :options="PIC_REVIEW_STATUS_OPTIONS"
          placeholder="Select review status"
          style="min-width: 10rem"
          allow-clear
        />
      </a-form-item>

      <a-form-item>
        <a-button type="primary" html-type="submit">Search</a-button>
      </a-form-item>
    </a-form>
    <div style="height: 2rem"></div>
    <a-table
      :columns="columns"
      :data-source="dataRef"
      :pagination="pagination"
      :loading="loading"
      :scroll="{ x: 'max-content', y: 'calc(100vh - 27rem)' }"
    >
      <template #bodyCell="{ column, record }">
        <template v-if="column.dataIndex === 'url'">
          <img
            :src="record.thumbnailUrl ?? record.url"
            :width="120"
            loading="lazy"
            style="object-fit: scale-down"
          />
        </template>
        <template v-else-if="column.dataIndex === 'tags'">
          <a-space wrap>
            <a-tag v-for="tag in record.tags || []" :key="tag">{{ tag }}</a-tag>
          </a-space>
        </template>
        <template v-if="column.dataIndex === 'picInfo'">
          <div>type: {{ record.picFormat }}</div>
          <div>width: {{ record.picWidth }}</div>
          <div>height: {{ record.picHeight }}</div>
          <div>scale: {{ record.picScale }}</div>
          <div>size: {{ formatSize(record.picSize) }}</div>
        </template>
        <template v-if="column.dataIndex === 'reviewInfo'">
          <div>Review status: {{ PIC_REVIEW_STATUS_MAP[record.reviewStatus] }}</div>
          <div>Review message: {{ record.reviewMessage }}</div>
          <div>Reviewer: {{ record.reviewerId }}</div>
        </template>

        <template v-else-if="column.dataIndex === 'createTime'">
          {{ dayjs(record.createTime).format('YYYY-MM-DD HH:mm:ss') }}
        </template>
        <template v-else-if="column.dataIndex === 'editTime'">
          {{ dayjs(record.editTime).format('YYYY-MM-DD HH:mm:ss') }}
        </template>
        <template v-else-if="column.key === 'action'">
          <a-space wrap>
            <a-button type="link" :href="`/picture/add?id=${record.id}`" target="_blank"
              >Edit</a-button
            >
            <a-button
              v-if="record.reviewStatus !== PIC_REVIEW_STATUS_ENUM.PASS"
              @click="handleReview(record, PIC_REVIEW_STATUS_ENUM.PASS)"
              type="link"
            >
              Pass
            </a-button>
            <a-button
              v-if="record.reviewStatus !== PIC_REVIEW_STATUS_ENUM.REJECT"
              danger
              @click="handleReview(record, PIC_REVIEW_STATUS_ENUM.REJECT)"
              type="link"
            >
              Reject
            </a-button>
            <a-button type="link" danger @click="doDelete(record.id)">Delete</a-button>
          </a-space>
        </template>
      </template>
    </a-table>
  </div>
</template>
<script lang="ts" setup>
import { type PictureQueryRequest, type PictureVo } from '@/api/api.ts'
import { pictureController } from '@/api/apiFactory'
import { computed, onMounted, reactive, ref } from 'vue'
import dayjs from 'dayjs'
import { message } from 'ant-design-vue'
import type { PagePictureVo } from '@/api/api.ts'
import {
  PIC_REVIEW_STATUS_ENUM,
  PIC_REVIEW_STATUS_MAP,
  PIC_REVIEW_STATUS_OPTIONS,
} from '@/constant/pictureConstant'
import { formatSize } from '@/util'

const columns = [
  {
    title: 'ID',
    dataIndex: 'id',
    width: 80,
  },
  {
    title: 'Picture',
    dataIndex: 'url',
    width: 80,
  },
  {
    title: 'Name',
    dataIndex: 'name',
    width: 100,
  },
  {
    title: 'Introduction',
    dataIndex: 'introduction',
    ellipsis: true,
    width: 160,
  },
  {
    title: 'Category',
    dataIndex: 'category',
    width: 80,
  },
  {
    title: 'Tags',
    dataIndex: 'tags',
    width: 80,
  },
  {
    title: 'Picture info',
    dataIndex: 'picInfo',
    width: 160,
  },
  {
    title: 'Review info',
    dataIndex: 'reviewInfo',
    width: 160,
  },
  {
    title: 'User ID',
    dataIndex: 'userId',
    width: 80,
  },
  {
    title: 'Create time',
    dataIndex: 'createTime',
    width: 160,
  },
  {
    title: 'Edit time',
    dataIndex: 'editTime',
    width: 160,
  },
  {
    title: 'Action',
    key: 'action',
    width: 80,
  },
]

const dataRef = ref<PictureVo[]>([])
const totalRef = ref(0)
const loading = ref(false)
const searchParamRef = reactive<PictureQueryRequest>({
  current: 1,
  pageSize: 5,
  sortField: 'createTime',
  sortOrder: 'desc',
})

const pagination = computed(() => {
  return {
    current: searchParamRef.current,
    pageSize: searchParamRef.pageSize,
    total: totalRef.value,
    showSizeChanger: true,
    showQuickJumper: true,
    showTotal: (total: number) => `Total ${total} items`,
    onChange: (page: number, pageSize: number) => {
      searchParamRef.current = page
      searchParamRef.pageSize = pageSize
      fetchPictureList()
    },
  }
})

onMounted(() => {
  fetchPictureList()
})

const fetchPictureList = async () => {
  loading.value = true
  const res = (await pictureController.listPictureByPage(searchParamRef)) as PagePictureVo
  dataRef.value = res.records ?? []
  totalRef.value = res.total ?? 0
  loading.value = false
}

const doSearch = () => {
  searchParamRef.current = 1
  fetchPictureList()
}

const doDelete = async (id: string) => {
  await pictureController.deletePicture({ id: id })
  message.success('Delete Success')
  fetchPictureList()
}

const handleReview = async (record: PictureVo, reviewStatus: number) => {
  const reviewMessage =
    reviewStatus === PIC_REVIEW_STATUS_ENUM.PASS
      ? 'Administrator operation approved'
      : 'Administrator operation rejected'
  const res = await pictureController.doPictureReview({
    id: record.id,
    reviewStatus,
    reviewMessage,
  })
  if (res) {
    message.success('Review success')
    fetchPictureList()
  } else {
    message.error('Review failed')
  }
}
</script>

<style scoped></style>
