<script>
import { defineComponent, ref, inject, computed } from "vue";
import { useRouter } from "vue-router";
import { useApi, useStores } from "@directus/extensions-sdk";
import { render } from "micromustache";
import { useI18n } from "vue-i18n";

export default defineComponent({
    emits: ["input"],
    props: {
        label: {
            type: String,
            default: null,
        },
        buttonType: {
            type: String,
            default: null,
        },
        confirm: {
            type: Boolean,
            default: false,
        },
        confirmText: {
            type: String,
            default: null,
        },
        value: {
            type: Boolean,
            default: null,
        },
        icon: {
            type: String,
            default: null,
        },
        url: {
            type: String,
            default: null,
        },
        method: {
            type: String,
            default: "patch",
        },
        body: {
            type: Object,
            default: null,
        },
        disabled: {
            type: Boolean,
            default: false,
        },
        collection: {
            type: String,
            required: true,
        },
        result: {
            type: String,
        },
        headers: {
            type: Array,
        },
        primaryKey: {
            type: [String, Number],
            required: true,
        },
    },
    setup(props) {
        const { useNotificationsStore } = useStores();
        const store = useNotificationsStore();
        const router = useRouter();
        const api = useApi();
        const isLoading = ref(false);
        const confirming = ref(false);
        const values = inject("values");
        const { t } = useI18n();

        const doAction = async () => {
                confirming.value = false;
                isLoading.value = true;

                try {
                    const item = values.value;
                    console.log(item);
                    const url =
                        render(props.url, item) ||
                        `/items/${props.collection}/${props.primaryKey}`;
                    const data = props.body
                        ? maybeJson(render(props.body, item))
                        : false;
                    const method = props.method || "patch";

                    const headers = props.headers
                        ? Object.fromEntries(
                              props.headers.map(({ key, value }) => [
                                  key,
                                  render(value, item),
                              ])
                          )
                        : null;

                    const result = await api.request({
                        method,
                        url,
                        data,
                        headers,
                    });

                    if (props.result === "list") {
                        router.push(`/content/${props.collection}`);
                    } else if (props.result === "reload") {
                        router.go(0);
                    } else {
                        let dismissAction = () => {};

                        if ('replace' in result.data) router.replace(result.data.replace);
                        else if ('push' in result.data) router.push(result.data.push);
                        else if ('go' in result.data) dismissAction = () => {router.go(+result.data.go);};
                        else if ('goto' in result.data) router.push(result.data.goto);
                        
                        store.add({
                            title: result.data.title || "Success",
                            text:
                                result.data.text ||
                                "Action was completed successfully",
                            type: "success",
                            dialog: true,
                            dismissAction: dismissAction,
                        });
                    }
                } catch (error) {
                    console.warn(error);

                    const code =
                        error.response?.data?.errors?.[0]?.extensions?.code ||
                        error.extensions?.code ||
                        "UNKNOWN";

                    const message =
                        error.response?.data?.errors?.[0]?.message ||
                        error.message ||
                        undefined;
                    
                    const hintMatch = (typeof message === 'string') ?
                        message.match(/\$message\$(.+)\$message\$/)
                    : null;

                    if (hintMatch) {
                        store.add({
                            text: hintMatch[1],
                            type: "error",
                            dialog: true,
                        });                    
                    } else {
                        store.add({
                            title: t(`errors.${code}`),
                            text: message,
                            type: "error",
                            dialog: true,
                            error,
                        });
                    }
                } finally {
                    isLoading.value = false;
                }        
        };

        return {
            t,
            isLoading,
            confirming,
            doAction,
            label: computed(() => render(props.label, values.value)),
            click() {
                if (props.confirm) {
                    confirming.value = true;
                } else {
                    doAction();
                }
            },
        };

        function maybeJson(str) {
            try {
                return JSON.parse(str);
            } catch (e) {
                return str;
            }
        }
    },
});
</script>

<template>
    <div>
        <v-button
            :loading="isLoading"
            @click="click"
            :class="buttonType"
            :secondary="buttonType !== 'primary'"
            :icon="!label"
        >
            <v-icon v-if="icon" left :name="icon" />
            {{ label }}
        </v-button>
        <v-dialog v-model="confirming" @esc="confirming = false" @apply="doAction">
            <v-card>
                <v-card-title>{{ confirmText || t('button_confirmation') }}</v-card-title>
                <v-card-actions>
                    <v-button secondary @click="doAction()">
                        {{ t('continue_label') }}
                    </v-button>
                    <v-button @click="confirming = false">{{ t('cancel') }}</v-button>
                </v-card-actions>
            </v-card>
        </v-dialog>
    </div>
</template>

<style scoped>
.info {
    --v-button-background-color: var(--blue);
    --v-button-background-color-hover: var(--blue-125);
    --v-button-color: var(--blue-alt);
    --v-button-color-hover: var(--blue-alt);
}

.success {
    --v-button-background-color: var(--success);
    --v-button-background-color-hover: var(--success-125);
    --v-button-color: var(--success-alt);
    --v-button-color-hover: var(--success-alt);
}

.warning {
    --v-button-background-color: var(--warning);
    --v-button-background-color-hover: var(--warning-125);
    --v-button-color: var(--warning-alt);
    --v-button-color-hover: var(--warning-alt);
}

.danger {
    --v-button-icon-color: var(--white);
    --v-button-background-color: var(--danger);
    --v-button-background-color-hover: var(--danger-125);
    --v-button-color: var(--danger-alt);
    --v-button-color-hover: var(--danger-alt);
}
</style>
