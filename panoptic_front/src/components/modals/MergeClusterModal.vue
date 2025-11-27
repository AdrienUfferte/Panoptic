<script setup lang="ts">
import CenteredImage from '@/components/images/CenteredImage.vue'
import TagBadge from '@/components/tagtree/TagBadge.vue'
import { DbCommit, ImagePropertyValue, Instance, InstancePropertyValue, ModalId, Property, PropertyMode, PropertyType } from '@/data/models'
import { useDataStore } from '@/data/dataStore'
import { useModalStore } from '@/data/modalStore'
import { usePanopticStore } from '@/data/panopticStore'
import { isTag } from '@/utils/utils'
import { computed, reactive, ref } from 'vue'
import { useI18n } from 'vue-i18n'
import Modal2 from './Modal2.vue'

const data = useDataStore()
const modal = useModalStore()
const panoptic = usePanopticStore()
const { t } = useI18n()

interface MergeOption {
    key: string
    value: any
    sourceImageId?: number
}

interface MergeRow {
    property: Property
    options: MergeOption[]
    conflict: boolean
}

const selectedImageId = ref<number | null>(null)
const mergeRows = ref<MergeRow[]>([])
const selectedOptions = reactive<{ [propertyId: number]: string }>({})
const merging = ref(false)
const error = ref<string | null>(null)

const groupData = ref<{ images: Instance[] } | null>(null)

const images = computed(() => groupData.value?.images ?? [])
const baseImage = computed(() => images.value.find(img => img.id === selectedImageId.value) ?? images.value[0])

function resetState() {
    mergeRows.value = []
    error.value = null
    Object.keys(selectedOptions).forEach(k => delete selectedOptions[k])
}

function onShow() {
    const payload = modal.getData(ModalId.MERGE_CLUSTER) as { images?: Instance[] }
    groupData.value = { images: payload?.images ?? [] }
    selectedImageId.value = images.value[0]?.id ?? null
    resetState()
    buildRows()
}

function onHide() {
    resetState()
    groupData.value = null
}

function stringifyValue(value: any, property: Property) {
    if (value === undefined || value === null) return '__undefined__'
    if (Array.isArray(value)) {
        return JSON.stringify([...value].sort())
    }
    if (value instanceof Date) {
        return value.toISOString()
    }
    if (property.type === PropertyType.color) {
        return String(value)
    }
    return JSON.stringify(value)
}

function formatValue(value: any, property: Property): string {
    if (value === undefined || value === null) return ''
    if (Array.isArray(value)) {
        return value.join(', ')
    }
    if (value instanceof Date) {
        return value.toISOString()
    }
    if (property.type === PropertyType.checkbox) {
        return value ? t('common.yes') : t('common.no')
    }
    return String(value)
}

function getTags(value: any, property: Property) {
    const res = []
    const ids = Array.isArray(value) ? value : [value]
    ids.forEach(id => {
        const tag = property.tags?.[id]
        if (tag) {
            res.push(tag)
        }
    })
    return res
}

function combinedLabel(values: any[], property: Property) {
    const defined = values.filter(v => v !== undefined && v !== null)
    if (!defined.length) return t('none')
    if (isTag(property.type)) {
        return defined
            .map(v => getTags(v, property).map(t => t.value).join(' -- '))
            .filter(v => v.length)
            .join(' -- ')
    }
    return defined.map(v => formatValue(v, property)).join(' -- ')
}

function combineValues(values: any[], property: Property) {
    const defined = values.filter(v => v !== undefined && v !== null)
    if (!defined.length) return undefined

    if (isTag(property.type)) {
        const merged = new Set<number>()
        defined.forEach(v => {
            if (Array.isArray(v)) {
                v.forEach(id => merged.add(id))
            } else {
                merged.add(v)
            }
        })
        return Array.from(merged)
    }

    return defined.map(v => formatValue(v, property)).join(' -- ')
}

function buildRows() {
    const groupImages = images.value
    if (!groupImages.length) {
        mergeRows.value = []
        return
    }

    const rows: MergeRow[] = []
    const properties = data.propertyList.filter(p => p.id > 0 && !p.computed &&
        ![PropertyType._folders, PropertyType._id, PropertyType._sha1, PropertyType._ahash, PropertyType._width, PropertyType._height].includes(p.type as PropertyType))

    for (const property of properties) {
        const values = groupImages.map(img => img.properties[property.id])
        if (!values.some(v => v !== undefined && v !== null)) continue

        const options: MergeOption[] = []
        values.forEach((value, index) => {
            if (value === undefined || value === null) return
            options.push({ key: String(groupImages[index].id), value, sourceImageId: groupImages[index].id })
        })

        const definedKeys = values
            .filter(v => v !== undefined && v !== null)
            .map(v => stringifyValue(v, property))
        const conflict = new Set(definedKeys).size > 1

        if (conflict) {
            options.push({ key: 'combined', value: combineValues(values, property) })
        }

        if (!options.length) continue

        const defaultKey = selectedOptions[property.id] ?? (conflict ? 'combined' : options[0].key)
        selectedOptions[property.id] = defaultKey
        rows.push({ property, options, conflict })
    }

    mergeRows.value = rows
}

async function confirmMerge() {
    if (!baseImage.value) return
    merging.value = true
    error.value = null

    try {
        const newInstance: Instance = { ...baseImage.value, id: -1 }
        const commit: DbCommit = { instances: [newInstance], instanceValues: [], imageValues: [] }

        for (const row of mergeRows.value) {
            const key = selectedOptions[row.property.id]
            const option = row.options.find(o => o.key === key)
            if (!option || option.value === undefined || option.value === null) continue

            if (row.property.mode === PropertyMode.id) {
                const value: InstancePropertyValue = {
                    propertyId: row.property.id,
                    instanceId: newInstance.id,
                    value: option.value
                }
                commit.instanceValues.push(value)
            } else {
                const value: ImagePropertyValue = {
                    propertyId: row.property.id,
                    sha1: newInstance.sha1,
                    value: option.value
                }
                commit.imageValues.push(value)
            }
        }

        await data.sendCommit(commit)
        panoptic.hideModal(ModalId.MERGE_CLUSTER)
    } catch (e) {
        console.error(e)
        error.value = t('modals.merge_cluster.error')
    }

    merging.value = false
}

const someConflict = computed(() => mergeRows.value.some(r => r.conflict))

</script>

<template>
    <Modal2 :id="ModalId.MERGE_CLUSTER" :width="1000" :height="700" @show="onShow" @hide="onHide">
        <template #title>
            {{ $t('modals.merge_cluster.title') }}
        </template>
        <template #content>
            <div class="h-100 d-flex flex-column">
                <div class="mb-2 text-secondary">{{ $t('modals.merge_cluster.description') }}</div>
                <div class="image-row mb-3" v-if="images.length">
                    <div v-for="img in images" :key="img.id" class="image-choice"
                        :class="{ active: img.id === selectedImageId }"
                        @click="selectedImageId = img.id">
                        <CenteredImage :image="img" :width="140" :height="110" :border="img.id === selectedImageId ? 3 : 1" />
                        <div class="text-center mt-1 small">
                            <b>{{ img.name }}</b>
                            <div class="text-secondary">ID: {{ img.id }}</div>
                        </div>
                    </div>
                </div>
                <div v-else class="text-secondary">{{ $t('modals.merge_cluster.empty_cluster') }}</div>

                <div class="flex-grow-1 overflow-auto">
                    <div v-for="row in mergeRows" :key="row.property.id" class="property-row">
                        <div class="d-flex align-items-center mb-1">
                            <div class="property-name">{{ row.property.name }}</div>
                            <div v-if="row.conflict" class="conflict ms-2">{{ $t('modals.merge_cluster.conflict') }}</div>
                        </div>
                        <div class="option-row">
                            <label v-for="option in row.options" :key="option.key" class="option-card">
                                <input type="radio" :name="'prop-' + row.property.id" :value="option.key"
                                    v-model="selectedOptions[row.property.id]" />
                                <div class="option-header">
                                    <span v-if="option.sourceImageId">{{ $t('modals.merge_cluster.keep_from', {
                                        name: data.instances[option.sourceImageId]?.name ?? option.sourceImageId
                                    }) }}</span>
                                    <span v-else>{{ $t('modals.merge_cluster.combine_choice') }}</span>
                                </div>
                                <div class="value-preview">
                                    <template v-if="isTag(row.property.type)">
                                        <template v-if="getTags(option.value, row.property).length">
                                            <TagBadge v-for="tag in getTags(option.value, row.property)"
                                                :key="tag.id" :id="tag.id" />
                                        </template>
                                        <span v-else class="text-secondary">{{ $t('modals.merge_cluster.empty_value') }}</span>
                                    </template>
                                    <template v-else>
                                        <span v-if="formatValue(option.value, row.property)">{{ formatValue(option.value, row.property) }}</span>
                                        <span v-else class="text-secondary">{{ $t('modals.merge_cluster.empty_value') }}</span>
                                    </template>
                                </div>
                                <div v-if="!option.sourceImageId" class="combined-preview">
                                    {{ combinedLabel(images.map(img => img.properties[row.property.id]), row.property) }}
                                </div>
                            </label>
                        </div>
                    </div>
                </div>

                <div class="d-flex mt-3 align-items-center">
                    <div class="text-secondary" v-if="someConflict">{{ $t('modals.merge_cluster.conflict_hint') }}</div>
                    <div class="text-danger" v-if="error">{{ error }}</div>
                    <div class="flex-grow-1"></div>
                    <div class="bb me-2" @click="panoptic.hideModal(ModalId.MERGE_CLUSTER)">{{ $t('cancel') }}</div>
                    <div class="bb" @click="confirmMerge" :class="{ disabled: merging || !baseImage }">
                        <span v-if="merging" class="spinner-border spinner-border-sm text-primary"></span>
                        <span v-else>{{ $t('modals.merge_cluster.confirm') }}</span>
                    </div>
                </div>
            </div>
        </template>
    </Modal2>
</template>

<style scoped>
.image-row {
    display: flex;
    gap: 12px;
    flex-wrap: wrap;
}

.image-choice {
    border: 2px solid var(--border-color);
    padding: 6px;
    border-radius: 6px;
    cursor: pointer;
    width: 160px;
}

.image-choice.active {
    border-color: var(--main-color);
    box-shadow: 0 0 0 2px rgba(var(--main), 0.2);
}

.property-row {
    padding: 8px 0;
    border-bottom: 1px solid var(--border-color);
}

.property-name {
    font-weight: bold;
}

.conflict {
    font-size: 11px;
    color: #c92a2a;
    border: 1px solid #c92a2a;
    padding: 1px 6px;
    border-radius: 4px;
}

.option-row {
    display: flex;
    flex-wrap: wrap;
    gap: 8px;
}

.option-card {
    border: 1px solid var(--border-color);
    border-radius: 6px;
    padding: 8px;
    min-width: 220px;
    max-width: 320px;
    display: flex;
    flex-direction: column;
    gap: 6px;
}

.option-card input[type="radio"] {
    align-self: flex-start;
}

.option-header {
    font-weight: 600;
}

.value-preview {
    min-height: 22px;
}

.combined-preview {
    font-size: 12px;
    color: var(--text-color);
}

.bb.disabled {
    pointer-events: none;
    opacity: 0.6;
}
</style>
