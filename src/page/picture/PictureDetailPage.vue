<template>
  <div id="picture-detail-page">
    <a-row :gutter="[16, 16]">
      <!-- Image Display Area -->
      <a-col :sm="24" :md="16" :xl="18">
        <a-card title="Image Preview">
          <a-image style="max-height: 36rem; object-fit: contain" :src="picture.url" />
        </a-card>
      </a-col>
      <!-- Image Information Area -->
      <a-col :sm="24" :md="8" :xl="6">
        <a-card title="Image Information">
          <a-descriptions :column="1">
            <a-descriptions-item label="Author">
              <a-space>
                <a-avatar :size="24" :src="picture.user?.userAvatar" />
                <div>{{ picture.user?.userName }}</div>
              </a-space>
            </a-descriptions-item>
            <a-descriptions-item label="Name">
              {{ picture.name ?? 'Unnamed' }}
            </a-descriptions-item>
            <a-descriptions-item label="Introduction">
              {{ picture.introduction ?? '-' }}
            </a-descriptions-item>
            <a-descriptions-item label="Category">
              {{ picture.category ?? 'Default' }}
            </a-descriptions-item>
            <a-descriptions-item label="Tags">
              <a-tag v-for="tag in picture.tags" :key="tag">
                {{ tag }}
              </a-tag>
            </a-descriptions-item>
            <a-descriptions-item label="Format">
              {{ picture.picFormat ?? '-' }}
            </a-descriptions-item>
            <a-descriptions-item label="Width">
              {{ picture.picWidth ?? '-' }}
            </a-descriptions-item>
            <a-descriptions-item label="Height">
              {{ picture.picHeight ?? '-' }}
            </a-descriptions-item>
            <a-descriptions-item label="Aspect Ratio">
              {{ picture.picScale ?? '-' }}
            </a-descriptions-item>
            <a-descriptions-item label="Size">
              {{ formatSize(picture.picSize) }}
            </a-descriptions-item>
            <a-descriptions-item label="Dominaint Color">
              <a-space>
                {{ picture.picColor ?? '-' }}
                <div
                  v-if="picture.picColor"
                  :style="{
                    backgroundColor: toHexColor(picture.picColor),
                    width: '16px',
                    height: '16px',
                  }"
                ></div>
              </a-space>
            </a-descriptions-item>
          </a-descriptions>
          <a-space wrap>
            <a-button type="primary" @click="doDownload">
              Free Download
              <template #icon>
                <DownloadOutlined />
              </template>
            </a-button>
            <a-button type="primary" ghost @click="doShare">
              Share
              <template #icon>
                <ShareAltOutlined />
              </template>
            </a-button>
            <a-button v-if="canEdit" type="default" @click="doEdit">
              Edit
              <template #icon>
                <EditOutlined />
              </template>
            </a-button>
            <a-button v-if="canDelete" danger @click="doDelete">
              Delete
              <template #icon>
                <DeleteOutlined />
              </template>
            </a-button>
          </a-space>
        </a-card>
      </a-col>
    </a-row>
    <ShareModal title="Share picture" :link="shareLink" ref="shareModalRef" />
  </div>
</template>

<script lang="ts" setup>
import type { PictureVo } from '@/api/api'
import { pictureController } from '@/api/apiFactory'
import { message } from 'ant-design-vue'
import { computed, onMounted, ref } from 'vue'
import { downloadImage, formatSize, toHexColor } from '@/util'
import { useRouter } from 'vue-router'
import {
  DeleteOutlined,
  EditOutlined,
  DownloadOutlined,
  ShareAltOutlined,
} from '@ant-design/icons-vue'
import ShareModal from '@/components/ShareModal.vue'
import { SPACE_PERMISSION_ENUM } from '@/constant/spaceConstant'

const props = defineProps<{
  id: string
}>()
const picture = ref<PictureVo>({})
const router = useRouter()
const shareModalRef = ref()
const shareLink = ref<string>('')

function createPermissionChecker(permission: string) {
  return computed(() => {
    return (picture.value.permissions ?? []).includes(permission)
  })
}

const canEdit = createPermissionChecker(SPACE_PERMISSION_ENUM.PICTURE_EDIT)
const canDelete = createPermissionChecker(SPACE_PERMISSION_ENUM.PICTURE_DELETE)

// Fetch picture details
const fetchPictureDetail = async () => {
  try {
    const res = (await pictureController.getPictureVoById(props.id)) as PictureVo
    if (res) {
      picture.value = res
    } else {
      message.error('Failed to load page details')
    }
  } catch (e: unknown) {
    console.error(e)
    message.error('Failed to load page details')
  }
}

const doEdit = () => {
  router.push(`/picture/add?id=${picture.value.id}&spaceId=${picture.value.spaceId}`)
}
const doDelete = async () => {
  const id = picture.value.id
  if (!id) {
    return
  }
  const res = await pictureController.deletePicture({ id: id })
  if (res) {
    message.success('Delete success')
    router.push('/')
  } else {
    message.error('Delete failed')
  }
}
// Handle download
const doDownload = () => {
  downloadImage(picture.value.url, 'test.png')
}

const doShare = () => {
  shareLink.value = `${window.location.protocol}//${window.location.host}/picture/${picture.value.id}`
  if (shareModalRef.value) {
    shareModalRef.value.openModal()
  }
}

onMounted(() => {
  fetchPictureDetail()
})
</script>

<style scoped></style>
