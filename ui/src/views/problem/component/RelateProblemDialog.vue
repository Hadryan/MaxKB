<template>
  <el-dialog
    :title="$t('views.problem.relateParagraph.title')"
    v-model="dialogVisible"
    width="80%"
    class="paragraph-dialog"
    destroy-on-close
    :close-on-click-modal="false"
    :close-on-press-escape="false"
  >
    <el-row v-loading="loading">
      <el-col :span="6">
        <el-scrollbar height="500" wrap-class="paragraph-scrollbar">
          <div class="bold title align-center p-24 pb-0">
            {{ $t('views.problem.relateParagraph.selectDocument') }}
          </div>
          <div class="p-8" style="padding-bottom: 8px">
            <el-input
              v-model="filterDoc"
              :placeholder="$t('views.problem.relateParagraph.placeholder')"
              prefix-icon="Search"
              clearable
            />
            <common-list
              :data="documentList"
              class="mt-8"
              @click="clickDocumentHandle"
              :default-active="currentDocument"
            >
              <template #default="{ row }">
                <span class="flex lighter align-center">
                  <auto-tooltip :content="row.name">
                    {{ row.name }}
                  </auto-tooltip>
                  <el-badge
                    :value="associationCount(row.id)"
                    type="primary"
                    v-if="associationCount(row.id)"
                    class="paragraph-badge ml-4"
                  />
                </span>
              </template>
            </common-list>
          </div>
        </el-scrollbar>
      </el-col>
      <el-col :span="18" class="border-l">
        <el-scrollbar height="500" wrap-class="paragraph-scrollbar">
          <div class="p-24" style="padding-bottom: 8px; padding-top: 16px">
            <div class="flex-between mb-16">
              <div class="bold title align-center">
                {{ $t('components.selectParagraph.title') }}
                <el-text>
                  （{{ $t('views.problem.relateParagraph.selectedParagraph') }}：{{
                    associationCount(currentDocument)
                  }}
                  {{ $t('views.problem.relateParagraph.count') }}）
                </el-text>
              </div>
              <el-input
                v-model="search"
                :placeholder="$t('common.search')"
                class="input-with-select"
                style="width: 260px"
                @change="searchHandle"
              >
                <template #prepend>
                  <el-select v-model="searchType" placeholder="Select" style="width: 80px">
                    <el-option :label="$t('common.title')" value="title" />
                    <el-option :label="$t('common.content')" value="content" />
                  </el-select>
                </template>
              </el-input>
            </div>
            <el-empty v-if="paragraphList.length == 0" :description="$t('common.noData')" />

            <InfiniteScroll
              v-else
              :size="paragraphList.length"
              :total="paginationConfig.total"
              :page_size="paginationConfig.page_size"
              v-model:current_page="paginationConfig.current_page"
              @load="getParagraphList"
              :loading="loading"
            >
              <template v-for="(item, index) in paragraphList" :key="index">
                <CardBox
                  shadow="hover"
                  :title="item.title || '-'"
                  :description="item.content"
                  class="paragraph-card cursor mb-16"
                  :class="isAssociation(item.id) ? 'selected' : ''"
                  :showIcon="false"
                  @click="associationClick(item)"
                >
                </CardBox>
              </template>
            </InfiniteScroll>
          </div>
        </el-scrollbar>
      </el-col>
    </el-row>
    <template #footer v-if="isMul">
      <div class="dialog-footer">
        <el-button @click="dialogVisible = false"> {{ $t('common.cancel') }}</el-button>
        <el-button type="primary" @click="mulAssociation"> {{ $t('common.confirm') }} </el-button>
      </div>
    </template>
  </el-dialog>
</template>
<script setup lang="ts">
import { ref, watch, reactive } from 'vue'
import { useRoute } from 'vue-router'
import problemApi from '@/api/problem'
import paragraphApi from '@/api/paragraph'
import useStore from '@/stores'
import { MsgSuccess } from '@/utils/message'
import { t } from '@/locales'
const { problem, document } = useStore()

const route = useRoute()
const {
  params: { id } // datasetId
} = route as any

const emit = defineEmits(['refresh'])

const dialogVisible = ref<boolean>(false)

const loading = ref(false)
const documentList = ref<any[]>([])
const cloneDocumentList = ref<any[]>([])
const paragraphList = ref<any[]>([])
const currentProblemId = ref<String>('')
const currentMulProblemId = ref<string[]>([])

// 回显
const associationParagraph = ref<any[]>([])

const currentDocument = ref<String>('')
const search = ref('')
const searchType = ref('title')
const filterDoc = ref('')
// 批量
const isMul = ref(false)

const paginationConfig = reactive({
  current_page: 1,
  page_size: 50,
  total: 0
})

function mulAssociation() {
  const data = {
    problem_id_list: currentMulProblemId.value,
    paragraph_list: associationParagraph.value.map((item) => ({
      paragraph_id: item.id,
      document_id: item.document_id
    }))
  }
  problemApi.postMulAssociationProblem(id, data, loading).then(() => {
    MsgSuccess(t('views.problem.tip.relatedSuccess'))
    dialogVisible.value = false
  })
}

function associationClick(item: any) {
  if (isMul.value) {
    if (isAssociation(item.id)) {
      associationParagraph.value.splice(associationParagraph.value.indexOf(item.id), 1)
    } else {
      associationParagraph.value.push(item)
    }
  } else {
    if (isAssociation(item.id)) {
      problem
        .asyncDisassociationProblem(
          id,
          item.document_id,
          item.id,
          currentProblemId.value as string,
          loading
        )
        .then(() => {
          getRecord(currentProblemId.value)
        })
    } else {
      problem
        .asyncAssociationProblem(
          id,
          item.document_id,
          item.id,
          currentProblemId.value as string,
          loading
        )
        .then(() => {
          getRecord(currentProblemId.value)
        })
    }
  }
}

function searchHandle() {
  paginationConfig.current_page = 1
  paragraphList.value = []
  currentDocument.value && getParagraphList(currentDocument.value)
}

function clickDocumentHandle(item: any) {
  paginationConfig.current_page = 1
  paragraphList.value = []
  currentDocument.value = item.id
  getParagraphList(item.id)
}

function getDocument() {
  document.asyncGetAllDocument(id, loading).then((res: any) => {
    cloneDocumentList.value = res.data
    documentList.value = res.data
    currentDocument.value = cloneDocumentList.value?.length > 0 ? cloneDocumentList.value[0].id : ''
    currentDocument.value && getParagraphList(currentDocument.value)
  })
}

function getParagraphList(documentId: String) {
  paragraphApi
    .getParagraph(
      id,
      (documentId || currentDocument.value) as string,
      paginationConfig,
      search.value && { [searchType.value]: search.value },
      loading
    )
    .then((res) => {
      paragraphList.value = [...paragraphList.value, ...res.data.records]
      paginationConfig.total = res.data.total
    })
}

// 已关联分段
function getRecord(problemId: String) {
  problemApi.getDetailProblems(id as string, problemId as string, loading).then((res) => {
    associationParagraph.value = res.data
  })
}

function associationCount(documentId: String) {
  return associationParagraph.value.filter((item) => item.document_id === documentId).length
}
function isAssociation(paragraphId: String) {
  return associationParagraph.value.some((option) => option.id === paragraphId)
}

watch(dialogVisible, (bool) => {
  if (!bool) {
    documentList.value = []
    cloneDocumentList.value = []
    paragraphList.value = []
    associationParagraph.value = []
    isMul.value = false

    currentDocument.value = ''
    search.value = ''
    searchType.value = 'title'
    emit('refresh')
  }
})

watch(filterDoc, (val) => {
  paragraphList.value = []
  documentList.value = val
    ? cloneDocumentList.value.filter((item) => item.name.includes(val))
    : cloneDocumentList.value
  currentDocument.value = documentList.value?.length > 0 ? documentList.value[0].id : ''
})

const open = (problemId: any) => {
  getDocument()
  if (problemId.length == 1) {
    currentProblemId.value = problemId[0]
    getRecord(problemId)
  } else if (problemId.length > 1) {
    currentMulProblemId.value = problemId
    isMul.value = true
  }
  dialogVisible.value = true
}

defineExpose({ open })
</script>
<style lang="scss" scoped>
.paragraph-card {
  position: relative;
}
.paragraph-badge {
  .el-badge__content {
    height: auto;
    display: table;
  }
}
</style>
