<template>
  <LayoutHeader v-if="proposal.doc">
    <template #left-header>
      <Breadcrumbs :items="breadcrumbs">
        <template #prefix="{ item }">
          <Icon v-if="item.icon" :icon="item.icon" class="mr-2 h-4" />
        </template>
      </Breadcrumbs>
    </template>
    <template #right-header>
      <CustomActions
        v-if="proposal._actions?.length"
        :actions="proposal._actions"
      />
      <Button
        v-if="canDelete"
        :label="__('Delete')"
        theme="red"
        size="sm"
        iconLeft="trash-2"
        @click="deleteProposal()"
      />
    </template>
  </LayoutHeader>
  <div v-if="proposal.doc" class="flex h-full flex-col overflow-hidden">
    <div class="border-b p-5">
      <div class="truncate text-3xl-medium text-ink-gray-9">
        {{ title }}
      </div>
    </div>
    <div
      v-if="sections.data"
      class="flex flex-1 flex-col overflow-hidden max-w-2xl"
    >
      <SidePanelLayout
        :sections="sections.data"
        doctype="ZeroMark Proposal"
        :docname="proposal.doc.name"
        @reload="sections.reload"
      />
    </div>
  </div>
  <ErrorPage
    v-else-if="errorTitle"
    :errorTitle="errorTitle"
    :errorMessage="errorMessage"
  />
  <DeleteLinkedDocModal
    v-if="showDeleteLinkedDocModal"
    v-model="showDeleteLinkedDocModal"
    :doctype="'ZeroMark Proposal'"
    :docname="props.proposalId"
    name="Proposals"
  />
</template>

<script setup>
import ErrorPage from '@/components/ErrorPage.vue'
import SidePanelLayout from '@/components/SidePanelLayout.vue'
import Icon from '@/components/Icon.vue'
import LayoutHeader from '@/components/LayoutHeader.vue'
import DeleteLinkedDocModal from '@/components/DeleteLinkedDocModal.vue'
import CustomActions from '@/components/CustomActions.vue'
import { useDocument } from '@/data/document'
import { getSettings } from '@/stores/settings'
import { globalStore } from '@/stores/global'
import { getMeta } from '@/stores/meta'
import { getView } from '@/utils/view'
import { setupCustomizations } from '@/utils'
import {
  Breadcrumbs,
  usePageMeta,
  createResource,
  toast,
  call,
} from 'frappe-ui'
import { useTelemetry } from 'frappe-ui/frappe'
import { computed, ref, watch, onMounted } from 'vue'
import { useRoute, useRouter } from 'vue-router'

const props = defineProps({
  proposalId: { type: String, required: true },
})

const { brand } = getSettings()
const { $dialog, $socket } = globalStore()
const { doctypeMeta } = getMeta('ZeroMark Proposal')
const { capture } = useTelemetry()

const route = useRoute()
const router = useRouter()

const errorTitle = ref('')
const errorMessage = ref('')

const showDeleteLinkedDocModal = ref(false)

const {
  document: proposal,
  permissions,
  scripts,
  triggerOnRender,
} = useDocument('ZeroMark Proposal', props.proposalId)

const canDelete = computed(() => permissions.data?.permissions?.delete || false)

onMounted(async () => {
  if (proposal.doc) await triggerOnRender()
})

const breadcrumbs = computed(() => {
  let items = [{ label: __('Proposals'), route: { name: 'Proposals' } }]

  if (route.query.view || route.query.viewType) {
    let view = getView(
      route.query.view,
      route.query.viewType,
      'ZeroMark Proposal',
    )
    if (view) {
      items.push({
        label: __(view.label),
        icon: view.icon,
        route: {
          name: 'Proposals',
          params: { viewType: route.query.viewType },
          query: { view: route.query.view },
        },
      })
    }
  }

  items.push({
    label: title.value,
    route: { name: 'Proposal', params: { proposalId: props.proposalId } },
  })
  return items
})

const title = computed(() => {
  let t = doctypeMeta.value?.title_field || 'name'
  return proposal.doc?.[t] || props.proposalId
})

usePageMeta(() => {
  return {
    title: title.value,
    icon: brand.favicon,
  }
})

async function deleteProposal() {
  showDeleteLinkedDocModal.value = true
}

const sections = createResource({
  url: 'crm.fcrm.doctype.crm_fields_layout.crm_fields_layout.get_sidepanel_sections',
  cache: ['sidePanelSections', 'ZeroMark Proposal'],
  params: { doctype: 'ZeroMark Proposal' },
  auto: true,
})

// Setup custom actions from Form Scripts
watch(
  () => proposal.doc,
  async (_doc) => {
    if (scripts.data?.length) {
      let s = await setupCustomizations(scripts.data, {
        doc: _doc,
        $dialog,
        $socket,
        router,
        toast,
        updateField: proposal.setValue.submit,
        createToast: toast.create,
        deleteDoc: deleteProposal,
        call,
      })
      proposal._actions = s.actions || []
    }
  },
  { once: true },
)
</script>
