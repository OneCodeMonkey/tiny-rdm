<script setup>
import { validType } from '@/consts/support_redis_type.js'
import useDialog from 'stores/dialog.js'
import Delete from '@/components/icons/Delete.vue'
import Edit from '@/components/icons/Edit.vue'
import Refresh from '@/components/icons/Refresh.vue'
import Timer from '@/components/icons/Timer.vue'
import RedisTypeTag from '@/components/common/RedisTypeTag.vue'
import { useI18n } from 'vue-i18n'
import IconButton from '@/components/common/IconButton.vue'
import useConnectionStore from 'stores/connections.js'
import Copy from '@/components/icons/Copy.vue'
import { ClipboardSetText } from 'wailsjs/runtime/runtime.js'

const props = defineProps({
    server: String,
    db: Number,
    keyType: {
        type: String,
        validator(value) {
            return validType(value)
        },
        default: 'STRING',
    },
    keyPath: String,
    ttl: {
        type: Number,
        default: -1,
    },
})

const dialogStore = useDialog()
const connectionStore = useConnectionStore()
const i18n = useI18n()

const onReloadKey = () => {
    connectionStore.loadKeyValue(props.server, props.db, props.keyPath)
}

const onCopyKey = () => {
    ClipboardSetText(props.keyPath)
        .then((succ) => {
            if (succ) {
                $message.success(i18n.t('dialogue.copy_succ'))
            }
        })
        .catch((e) => {
            $message.error(e.message)
        })
}

const onDeleteKey = () => {
    $dialog.warning(i18n.t('dialogue.remove_tip', { name: props.keyPath }), () => {
        connectionStore.deleteKey(props.server, props.db, props.keyPath).then((success) => {
            if (success) {
                $message.success(i18n.t('dialogue.delete_key_succ', { key: props.keyPath }))
            }
        })
    })
}
</script>

<template>
    <div class="content-toolbar flex-box-h">
        <n-input-group>
            <redis-type-tag :type="props.keyType" size="large" />
            <n-input v-model:value="props.keyPath">
                <template #suffix>
                    <icon-button :icon="Refresh" size="18" t-tooltip="interface.reload" @click="onReloadKey" />
                </template>
            </n-input>
            <icon-button :icon="Copy" border size="18" t-tooltip="interface.copy_key" @click="onCopyKey" />
        </n-input-group>
        <n-button-group>
            <n-tooltip>
                <template #trigger>
                    <n-button :focusable="false" @click="dialogStore.openTTLDialog(props.ttl)">
                        <template #icon>
                            <n-icon :component="Timer" size="18" />
                        </template>
                        <template v-if="ttl < 0">
                            {{ $t('interface.forever') }}
                        </template>
                        <template v-else>{{ ttl }} {{ $t('common.second') }}</template>
                    </n-button>
                </template>
                TTL
            </n-tooltip>
            <icon-button
                :icon="Edit"
                border
                size="18"
                t-tooltip="interface.rename_key"
                @click="dialogStore.openRenameKeyDialog(props.server, props.db, props.keyPath)" />
        </n-button-group>
        <n-tooltip>
            <template #trigger>
                <n-button>
                    <template #icon>
                        <n-icon :component="Delete" size="18" @click="onDeleteKey" />
                    </template>
                </n-button>
            </template>
            {{ $t('interface.delete_key') }}
        </n-tooltip>
    </div>
</template>

<style lang="scss" scoped>
.content-toolbar {
    align-items: center;
    gap: 5px;
}
</style>
