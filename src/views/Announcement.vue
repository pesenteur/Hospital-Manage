<template>
    <div class="container">
        <el-header>
            <el-row>
                <el-col :span="4">
                    <el-icon size="30">
                        <Notification/>
                    </el-icon>
                    <span style="margin-left: 5px"><strong>编辑公告</strong></span>
                </el-col>
                <el-col :span="2" :offset="15">
                    <el-button type="primary" @click="publishNotification">发布为通知</el-button>
                </el-col>
                <el-col :span="2" :offset="1">
                    <el-button type="primary" @click="publishNews">发布为新闻</el-button>
                </el-col>
            </el-row>
        </el-header>
        <el-input placeholder="请输入标题" v-model="title" style="margin-top: 10px;margin-bottom: 10px"/>
        <el-row>
            <el-col :span="4">
                <el-upload
                        v-model:file-list="fileList"
                        action="http://azure.pesenteur.eu.org:5555/api/uploadImage"
                        class="upload-demo"
                        name="image"
                        list-type="picture"
                        accept="image/*"
                        :on-success="storeImage"
                        :limit="1"
                        :headers="header"
                        ref="imageUploader"
                >
                    <el-button type="primary">点击上传封面图片</el-button>
                </el-upload>
            </el-col>
            <el-col :span="5">
                <p style="margin-top: 3px">是否选择此图为轮播图：</p>
            </el-col>
            <el-col :span="4">
                <el-radio-group v-model="type" style="margin-left: 5px">
                    <el-radio label="1">是</el-radio>
                    <el-radio label="0">否</el-radio>
                </el-radio-group>
            </el-col>
        </el-row>
        <div style="border: 1px solid #ccc ">
            <Toolbar
                    style="border-bottom: 1px solid #ccc"
                    :editor="editorRef"
                    :defaultConfig="toolbarConfig"
                    mode="default"
            />
            <Editor
                    style="height: 500px; overflow-y: hidden;"
                    v-model="valueHtml"
                    :defaultConfig="editorConfig"
                    mode="default"
                    @onCreated="handleCreated"
            />
        </div>
        <el-tabs v-model="message">
            <el-tab-pane :label="`已发布通知(${state.notification.length})`" name="first">
                <el-table :data="state.notification" style="width: 100%" ref="notificationRef">
                    <el-table-column type="selection" width="55"/>
                    <el-table-column label="标题" prop="title" width="180"/>
                    <el-table-column label="日期" prop="date" width="180"/>
                    <el-table-column label="概要" prop="short_info"/>
                </el-table>
                <div class="handle-row">
                    <el-button type="primary" @click="deleteNotification">删除通知</el-button>
                </div>
            </el-tab-pane>
            <el-tab-pane :label="`已发布新闻(${state.news.length})`" name="second">
                <template v-if="message === 'second'">
                    <el-table :data="state.news" style="width: 100%" ref="newsRef">
                        <el-table-column type="selection" width="55"/>
                        <el-table-column label="标题" prop="title" width="180"/>
                        <el-table-column label="日期" prop="date" width="180"/>
                        <el-table-column label="概要" prop="short_info"/>
                    </el-table>
                    <div class="handle-row">
                        <el-button type="primary" @click="deleteNews">删除新闻</el-button>
                    </div>
                </template>
            </el-tab-pane>
        </el-tabs>
    </div>
</template>

<script setup>
import '@wangeditor/editor/dist/css/style.css'; // 引入 css

import {onBeforeUnmount, ref, shallowRef, inject, onMounted, nextTick} from 'vue';
import {Editor, Toolbar} from '@wangeditor/editor-for-vue';
import {Notification} from "@element-plus/icons-vue";
import {ElMessage} from "element-plus";
import useCustomLoading from "@/utils/loading.js";

const $api = inject('$api');
const title = ref('');
// 编辑器实例，必须用 shallowRef
const editorRef = shallowRef();
const message = ref('first');
const state = ref({
    notification: [],
    news: []
});
const notificationRef = ref()
const newsRef = ref()

// 内容 HTML
const valueHtml = ref('');
const toolbarConfig = {};
const editorConfig = {placeholder: '请输入内容...', MENU_CONF: {}};
const imageUrl = ref('');
const type = ref('1');
const header = {
    "Authorization": localStorage.getItem('TOKEN')
};

// 组件销毁时，也及时销毁编辑器
onBeforeUnmount(() => {
    const editor = editorRef.value;
    if (editor == null) return;
    editor.destroy();
});

const handleCreated = (editor) => {
    editorRef.value = editor; // 记录 editor 实例，重要！
};

const fileList = ref([])

const storeImage = (res, file, fileList) => {
    if (res.result !== '1') {
        console.log(res);
        ElMessage.error('上传失败');
        return;
    }
    ElMessage.success('上传成功');
    imageUrl.value = res.data.url;
}

editorConfig.MENU_CONF['uploadImage'] = {
    server: 'http://azure.pesenteur.eu.org:5555/api/uploadImage',
    fieldName: 'image',
    headers: {
        "Authorization": localStorage.getItem('TOKEN')
    },
}

const imageUploader = ref()

async function publishNotification() {
    useCustomLoading().start({
        fullscreen: true,
        text: '加载中，请稍后'
    });
    const editor = editorRef.value;
    if (editor == null) return;
    const html = editor.getHtml();
    const summary = editor.getText().slice(0, 100);
    console.log(html);
    if (imageUrl.value === '') {
        ElMessage.error('请上传封面图片');
        return;
    }
    if (title.value === '') {
        ElMessage.error('请输入标题');
        return;
    }
    if (summary === '') {
        ElMessage.error('请输入通知内容');
        return;
    }
    const res = await $api.announcement.publishNotification(imageUrl.value, html, title.value, type.value, summary);
    if (res.result !== '1') {
        ElMessage.error('发布失败');
        return;
    }
    ElMessage.success('发布成功');
    title.value = '';
    editor.setHtml('');
    type.value = '1';
    imageUploader.value.clearFiles();
    await getList();
}

async function publishNews() {
    useCustomLoading().start({
        fullscreen: true,
        text: '加载中，请稍后'
    });
    const editor = editorRef.value;
    if (editor == null) return;
    const html = editor.getHtml();
    const summary = editor.getText().slice(0, 100);
    console.log(html);
    if (imageUrl.value === '') {
        ElMessage.error('请上传封面图片');
        return;
    }
    if (title.value === '') {
        ElMessage.error('请输入标题');
        return;
    }
    if (summary === '') {
        ElMessage.error('请输入新闻内容');
        return;
    }
    const res = await $api.announcement.publishNews(imageUrl.value, html, title.value, type.value, summary);
    if (res.result !== '1') {
        ElMessage.error('发布失败');
        return;
    }
    ElMessage.success('发布成功');
    title.value = '';
    editor.setHtml('');
    type.value = '1';
    imageUploader.value.clearFiles();
    await getList();
}

async function deleteNotification() {
    useCustomLoading().start({
        fullscreen: true,
        text: '加载中，请稍后'
    });
    let notificationId = [];
    for (let i = 0; i < notificationRef.value.getSelectionRows().length; i++) {
        notificationId.push(notificationRef.value.getSelectionRows()[i].id);
    }
    const res = await $api.announcement.deleteNotification(notificationId);
    if (res.result !== '1') {
        ElMessage.error('删除失败');
        return;
    }
    ElMessage.success('删除成功');
    await getList();
}

async function deleteNews() {
    useCustomLoading().start({
        fullscreen: true,
        text: '加载中，请稍后'
    });
    let newsId = [];
    for (let i = 0; i < newsRef.value.getSelectionRows().length; i++) {
        newsId.push(newsRef.value.getSelectionRows()[i].id);
    }
    const res = await $api.announcement.deleteNews(newsId);
    if (res.result !== '1') {
        ElMessage.error('删除失败');
        return;
    }
    ElMessage.success('删除成功');
    await getList();
}

async function getList() {
    let res = await $api.announcement.getNotificationList();
    if (res.result !== '1') {
        console.log('获取通知列表失败');
        return;
    }
    state.value.notification = res.data;
    res = await $api.announcement.getNewsList();
    if (res.result !== '1') {
        console.log('获取新闻列表失败');
        return;
    }
    state.value.news = res.data;
    await nextTick();
    useCustomLoading().end();
}

onMounted(async () => {
    await getList();
})
</script>
