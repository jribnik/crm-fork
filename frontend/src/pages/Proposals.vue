<template>
  <LayoutHeader>
    <template #left-header>
      <ViewBreadcrumbs v-model="viewControls" routeName="Proposals" />
    </template>
    <template #right-header>
      <CustomActions
        v-if="proposalsListView?.customListActions"
        :actions="proposalsListView.customListActions"
      />
      <Button
        variant="solid"
        :label="__('Create')"
        iconLeft="plus"
        @click="createProposal"
      />
    </template>
  </LayoutHeader>
  <ViewControls
    ref="viewControls"
    v-model="proposals"
    v-model:loadMore="loadMore"
    v-model:resizeColumn="triggerResize"
    v-model:updatedPageCount="updatedPageCount"
    doctype="ZeroMark Proposal"
  />
  <ProposalsListView
    v-if="proposals.data && rows.length"
    ref="proposalsListView"
    v-model="proposals.data.page_length_count"
    v-model:list="proposals"
    :rows="rows"
    :columns="columns"
    :options="{
      showTooltip: false,
      resizeColumn: true,
      rowCount: proposals.data.row_count,
      totalCount: proposals.data.total_count,
    }"
    @loadMore="() => loadMore++"
    @columnWidthUpdated="() => triggerResize++"
    @updatePageCount="(count) => (updatedPageCount = count)"
    @applyFilter="(data) => viewControls.applyFilter(data)"
    @applyLikeFilter="(data) => viewControls.applyLikeFilter(data)"
    @likeDoc="(data) => viewControls.likeDoc(data)"
    @selectionsChanged="
      (selections) => viewControls.updateSelections(selections)
    "
  />
  <EmptyState
    v-else-if="proposals.data && !rows.length"
    name="Proposals"
    :icon="FileTextIcon"
  />
</template>
<script setup>
import FileTextIcon from '~icons/lucide/file-text'
import ViewBreadcrumbs from '@/components/ViewBreadcrumbs.vue'
import CustomActions from '@/components/CustomActions.vue'
import LayoutHeader from '@/components/LayoutHeader.vue'
import ProposalsListView from '@/components/ListViews/ProposalsListView.vue'
import ViewControls from '@/components/ViewControls.vue'
import { getMeta } from '@/stores/meta'
import { formatDate } from '@/utils'
import { timestampCell } from '@/composables/useTimelinePreferences'
import { useDoctypeModal } from '@/composables/doctypeModal'
import { ref, computed } from 'vue'
import { useRouter } from 'vue-router'
import EmptyState from '../components/ListViews/EmptyState.vue'

const { getFormattedPercent, getFormattedFloat, getFormattedCurrency } =
  getMeta('ZeroMark Proposal')

const router = useRouter()
const { showModal } = useDoctypeModal()

function createProposal() {
  showModal({
    doctype: 'ZeroMark Proposal',
    callbacks: {
      afterInsert: (d) => {
        router.push({ name: 'Proposal', params: { proposalId: d.name } })
      },
    },
  })
}

const proposalsListView = ref(null)

// proposals data is loaded in the ViewControls component
const proposals = ref({})
const loadMore = ref(1)
const triggerResize = ref(1)
const updatedPageCount = ref(20)
const viewControls = ref(null)

const rows = computed(() => {
  if (
    !proposals.value?.data?.data ||
    !['list', 'group_by'].includes(proposals.value.data.view_type)
  )
    return []
  return proposals.value?.data.data.map((proposal) => {
    let _rows = {}
    proposals.value?.data.rows.forEach((row) => {
      _rows[row] = proposal[row]

      let fieldType = proposals.value?.data.columns?.find(
        (col) => (col.key || col.value) == row,
      )?.type

      if (
        fieldType &&
        ['Date', 'Datetime'].includes(fieldType) &&
        !['modified', 'creation'].includes(row)
      ) {
        _rows[row] = formatDate(proposal[row], '', true, fieldType == 'Datetime')
      }

      if (fieldType && fieldType == 'Currency') {
        _rows[row] = getFormattedCurrency(row, proposal)
      }

      if (fieldType && fieldType == 'Float') {
        _rows[row] = getFormattedFloat(row, proposal)
      }

      if (fieldType && fieldType == 'Percent') {
        _rows[row] = getFormattedPercent(row, proposal)
      }

      if (['modified', 'creation'].includes(row)) {
        _rows[row] = timestampCell(proposal[row])
      }
    })
    return _rows
  })
})

const columns = computed(() => {
  let _columns = proposals.value?.data?.columns || []

  // Set align right for last column
  if (_columns.length) {
    _columns = _columns.map((col, index) => {
      if (index === _columns.length - 1) {
        return { ...col, align: 'right' }
      }
      return col
    })
  }

  return _columns
})
</script>
