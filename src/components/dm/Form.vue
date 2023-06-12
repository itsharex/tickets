<script setup lang="js">
import { reactive, watch, computed, ref, onMounted  } from 'vue';
import { getQueryString } from '../../../utils/common';
import { Message } from "@arco-design/web-vue";
import { invoke } from '@tauri-apps/api/tauri'
import { useStore } from 'vuex';
import {
    getToken,
    getSign,
    commonTip,
    isSuccess,
    joinMsg,
} from "../../../utils/dm/index.js";

const store = useStore();

const form = reactive({
    cookie: "",
    itemId: "",
    token: "", // 获取 sign 使用
    url: "",
    num: 1,
    retry: 2,
    selectVisitUserList: [] //选择的观演人
});

onMounted(() => {
    initVisitUser()
})

watch(() => form.url, (url) => {
    const search = url.split("html")[1]
    const itemId = getQueryString('itemId', search)
    if(itemId) {
        form.itemId = itemId;
    } else {
        Message.warning("此url无法获取 itemId")
    }
})

const props = defineProps({
    handleSubmit: Function
})

function handleSubmit(e) {
    if(e.errors) {
        return
    }

    // 检查票数和观演人是否一致
    if(Number(form.num) !== form.selectVisitUserList.length) {
        Message.error("购买票数需要与观演人一致")

        return
    }

    store.commit('ADD_FORM', {
        ...e.values,
        cookie: e.values.cookie.trim(),
        // 根据 cookie 解析 token，用来生成 t 和 sign
        token:  getToken(e.values.cookie.trim())
    })
    props.handleSubmit()
}

const visitUserList = computed(() => store.state.dm.visitUserList)
function initVisitUser() {
    // 先从本地加载
    // 如果不存在，则请求接口并保存
    const data = getStore()
    if(Array.isArray(data) && data.length) {
        store.commit("ADD_VISIT_USER", data)
    } else {
        getVisitUser()
    }
}

const isLoading = ref(false)

async function getVisitUser() {
    const data = `{"customerType":"default","dmChannel":"damai@damaih5_h5"}`;
    const [t, sign] = getSign(data, form.token);
    try {
        const res = await invoke("get_user_list", {
            t,
            sign,
            cookie: form.cookie,
            data,
        })

        const parseData = JSON.parse(res)
        if (Array.isArray(parseData.ret) && parseData.ret.length) {
            const message = parseData.ret[0];
            commonTip(message);

            if (isSuccess(message)) {
                console.log('parseData', parseData)
                const result = parseData.data.result
                if(Array.isArray(result) && result.length) {
                    saveStore(result)
                    store.commit("ADD_VISIT_USER", result)
                    form.selectVisitUserList = []
                } else {
                    Message.error("观演人信息获取错误，请设置正确的信息")
                }
            } else {
                Message.error(joinMsg(["获取观演人信息错误", ...parseData.ret]));
            }
        }
    } catch(e) {
        Message.error(joinMsg(["观演人信息获取错误", JSON.stringify(e)]))
    } finally {
        isLoading.value = false
    }
}

function saveStore(data) {
    localStorage.setItem("visitUserList", JSON.stringify(data))
}

function getStore() {
    const visitUserList = localStorage.getItem("visitUserList")
    return visitUserList ? JSON.parse(visitUserList) : []
}

function setLoading() {
    if(!form.cookie) {
        Message.warning("请先填写 cookie")

        return
    }
    isLoading.value = true

    // TODO 更新后，清空选择状态
    getVisitUser()
}

function userChange(valList) {
    store.commit("SET_SELECT_VISIT_USER", valList)
}
</script>

<template>
    <section>
        <a-form
            :model="form"
            :style="{ width: '800px' }"
            @submit="handleSubmit"
        >
            <a-form-item field="cookie" label="cookie" required>
                <template #extra>
                    <div>进入商品页面，http请求头里的cookie</div>
                </template>
                <a-textarea
                    v-model="form.cookie"
                    placeholder="请输入 cookie"
                    allow-clear
                />
            </a-form-item>

            <a-form-item field="url" label="url">
                <a-input v-model="form.url" placeholder="请输入商品详情页url">
                </a-input>
            </a-form-item>

            <a-form-item field="itemId" label="itemId" required>
                <template #extra>
                    <div>进入商品页面，页面路径上的 itemId</div>
                </template>
                <a-input v-model="form.itemId" placeholder="请输入itemId..." />
            </a-form-item>

            <a-form-item field="num" label="购买张数" required>
                <template #extra>
                    <div>需要与选择实名信息一致</div>
                </template>
                <!-- <a-input v-model="form.num" placeholder="请输入购买数量..." /> -->
                <a-input-number
                    v-model="form.num"
                    placeholder="请输入购买数量..."
                    :min="1"
                    model-event="input"
                />
            </a-form-item>
            <a-form-item field="selectVisitUserList" label="观演人" required>
                <!-- <a-input v-model="form.num" placeholder="请输入购买数量..." /> -->
                <a-checkbox-group
                    v-model="form.selectVisitUserList"
                    @change="userChange"
                >
                    <a-checkbox
                        v-for="item in visitUserList"
                        :key="item.maskedIdentityNo"
                        :value="item.maskedIdentityNo"
                    >
                        <!-- {{ item.maskedName }} {{ item.identityTypeName }}
                        {{ item.maskedIdentityNo }} -->

                        <template #checkbox="{ checked }">
                            <a-space
                                align="start"
                                class="custom-checkbox-card"
                                :class="{
                                    'custom-checkbox-card-checked': checked,
                                }"
                            >
                                <div className="custom-checkbox-card-mask">
                                    <div
                                        className="custom-checkbox-card-mask-dot"
                                    />
                                </div>
                                <div>
                                    <div className="custom-checkbox-card-title">
                                        {{ item.maskedName }}
                                    </div>
                                    <a-typography-text type="secondary">
                                        {{ item.maskedIdentityNo }}
                                    </a-typography-text>
                                </div>
                            </a-space>
                        </template>
                    </a-checkbox>
                </a-checkbox-group>

                <a-button
                    size="mini"
                    @click="setLoading"
                    type="primary"
                    :loading="isLoading"
                    >更新</a-button
                >
            </a-form-item>
            <a-form-item field="retry" label="重试次数" required>
                <template #extra>
                    <div>下订单失败的重试次数</div>
                </template>
                <a-input-number
                    v-model="form.retry"
                    placeholder="请输入重试次数"
                    :min="1"
                    :max="3"
                    model-event="input"
                />
            </a-form-item>

            <a-form-item>
                <a-button type="primary" html-type="submit">确定</a-button>
            </a-form-item>
        </a-form>
    </section>
</template>

<style scoped>
.custom-checkbox-card {
    padding: 10px 16px;
    border: 1px solid var(--color-border-2);
    border-radius: 4px;
    width: 250px;
    box-sizing: border-box;
}

.custom-checkbox-card-mask {
    height: 14px;
    width: 14px;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    border-radius: 2px;
    border: 1px solid var(--color-border-2);
    box-sizing: border-box;
}

.custom-checkbox-card-mask-dot {
    width: 8px;
    height: 8px;
    border-radius: 2px;
}

.custom-checkbox-card-title {
    color: var(--color-text-1);
    font-size: 14px;
    font-weight: bold;
    margin-bottom: 8px;
}

.custom-checkbox-card:hover,
.custom-checkbox-card-checked,
.custom-checkbox-card:hover .custom-checkbox-card-mask,
.custom-checkbox-card-checked .custom-checkbox-card-mask {
    border-color: rgb(var(--primary-6));
}

.custom-checkbox-card-checked {
    background-color: var(--color-primary-light-1);
}

.custom-checkbox-card:hover .custom-checkbox-card-title,
.custom-checkbox-card-checked .custom-checkbox-card-title {
    color: rgb(var(--primary-6));
}

.custom-checkbox-card-checked .custom-checkbox-card-mask-dot {
    background-color: rgb(var(--primary-6));
}
</style>