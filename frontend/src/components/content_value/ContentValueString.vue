<script setup>
import { computed, onMounted, ref, watch } from 'vue'
import { useI18n } from 'vue-i18n'
import ContentToolbar from './ContentToolbar.vue'
import Copy from '@/components/icons/Copy.vue'
import Save from '@/components/icons/Save.vue'
import { useThemeVars } from 'naive-ui'
import { types } from '@/consts/value_view_type.js'
import Close from '@/components/icons/Close.vue'
import Edit from '@/components/icons/Edit.vue'
import { types as redisTypes } from '@/consts/support_redis_type.js'
import { ClipboardSetText } from 'wailsjs/runtime/runtime.js'
import { map, toLower } from 'lodash'
import useConnectionStore from 'stores/connections.js'

const i18n = useI18n()
const themeVars = useThemeVars()

const props = defineProps({
    name: String,
    db: Number,
    keyPath: String,
    ttl: {
        type: Number,
        default: -1,
    },
    value: String,
    size: Number,
    viewAs: {
        type: String,
        default: types.PLAIN_TEXT,
    },
})

const viewOption = computed(() =>
    map(types, (t) => {
        return {
            value: t,
            label: t,
        }
    }),
)
// const viewAs = ref(types.PLAIN_TEXT)

const autoDetectFormat = () => {
    // auto check format when loaded
    // if (IsJson(props.value)) {
    //     viewAs.value = types.JSON
    // } else {
    //     viewAs.value = types.PLAIN_TEXT
    // }
}

onMounted(() => {
    autoDetectFormat()
})
watch(
    () => props.value,
    (value) => {
        autoDetectFormat()
    },
)

const keyType = redisTypes.STRING
const viewLanguage = computed(() => {
    switch (props.viewAs) {
        case types.JSON:
        case types.BASE64_JSON:
            return 'json'
        default:
            return 'plaintext'
    }
})

const onViewTypeUpdate = (viewType) => {
    connectionStore.loadKeyValue(props.name, props.db, props.keyPath, viewType)
}

/**
 * Copy value
 */
const onCopyValue = () => {
    ClipboardSetText(props.value)
        .then((succ) => {
            if (succ) {
                $message.success(i18n.t('dialogue.copy_succ'))
            }
        })
        .catch((e) => {
            $message.error(e.message)
        })
}

const editValue = ref('')
const inEdit = ref(false)
const onEditValue = () => {
    editValue.value = props.value
    inEdit.value = true
}

const onCancelEdit = () => {
    inEdit.value = false
}

/**
 * Save value
 */
const connectionStore = useConnectionStore()
const saving = ref(false)
const onSaveValue = async () => {
    saving.value = true
    try {
        const { success, msg } = await connectionStore.setKey(
            props.name,
            props.db,
            props.keyPath,
            toLower(keyType),
            editValue.value,
            -1,
            props.viewAs,
        )
        if (success) {
            await connectionStore.loadKeyValue(props.name, props.db, props.keyPath)
            $message.success(i18n.t('dialogue.save_value_succ'))
        } else {
            $message.error(msg)
        }
    } catch (e) {
        $message.error(e.message)
    } finally {
        inEdit.value = false
        saving.value = false
    }
}
</script>

<template>
    <div class="content-wrapper flex-box-v">
        <content-toolbar :db="props.db" :key-path="keyPath" :key-type="keyType" :server="props.name" :ttl="ttl" />
        <div class="tb2 flex-box-h">
            <n-text>{{ $t('interface.view_as') }}</n-text>
            <n-select
                :value="props.viewAs"
                :options="viewOption"
                style="width: 200px"
                @update:value="onViewTypeUpdate" />
            <div class="flex-item-expand"></div>
            <n-button-group v-if="!inEdit">
                <n-button :focusable="false" @click="onCopyValue">
                    <template #icon>
                        <n-icon :component="Copy" size="18" />
                    </template>
                    {{ $t('interface.copy_value') }}
                </n-button>
                <n-button plain :focusable="false" @click="onEditValue">
                    <template #icon>
                        <n-icon :component="Edit" size="18" />
                    </template>
                    {{ $t('interface.edit_value') }}
                </n-button>
            </n-button-group>
            <n-button-group v-else>
                <n-button :loading="saving" :focusable="false" plain @click="onSaveValue">
                    <template #icon>
                        <n-icon :component="Save" size="18" />
                    </template>
                    {{ $t('interface.save_update') }}
                </n-button>
                <n-button :loading="saving" :focusable="false" plain @click="onCancelEdit">
                    <template #icon>
                        <n-icon :component="Close" size="18" />
                    </template>
                    {{ $t('common.cancel') }}
                </n-button>
            </n-button-group>
        </div>
        <div class="value-wrapper flex-item-expand flex-box-v">
            <n-scrollbar v-if="!inEdit" class="flex-item-expand">
                <n-code :code="props.value" :language="viewLanguage" show-line-numbers style="cursor: text" word-wrap />
            </n-scrollbar>
            <n-input
                v-else
                v-model:value="editValue"
                :disabled="saving"
                :resizable="false"
                class="flex-item-expand"
                type="textarea" />
        </div>
    </div>
</template>

<style lang="scss" scoped>
.value-wrapper {
    overflow: hidden;
    border-top: v-bind('themeVars.borderColor') 1px solid;
    padding: 5px;
}
</style>
