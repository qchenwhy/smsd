<template>
	<view class="content">
		<!-- 日期选择器 -->
		<view class="date-picker">
			<picker mode="date" :value="selectedDate" @change="onDateChange">
				<view class="picker-text">
					选择日期: {{selectedDate}}
				</view>
			</picker>
			<view class="file-select-btn" @tap="chooseFile">
				<text class="iconfont icon-file"></text>
			</view>
		</view>
		
		<!-- 文件列表 -->
		<view class="file-container">
			<!-- 加载中状态 -->
			<view v-if="isLoading" class="loading-container">
				<view class="loading-content">
					<text class="loading-text">正在搜索文件...</text>
				</view>
			</view>
			
			<!-- 文件列表 -->
			<template v-else>
				<view v-if="fileList.length > 0" class="file-list">
					<view class="list-header">
						<checkbox :checked="allSelected" @tap="toggleAllSelect">全选</checkbox>
						<text>文件列表</text>
					</view>
					<scroll-view scroll-y="true" class="list-content">
						<view v-for="(file, index) in fileList" :key="index" class="file-item">
							<checkbox :checked="file.selected" @tap="toggleSelect(index)"></checkbox>
							<text class="file-name">{{file.name}}</text>
							<text class="file-time">{{formatTime(file.time)}}</text>
						</view>
					</scroll-view>
				</view>
				<view v-else class="no-files">
					<text>当前日期没有录音文件</text>
				</view>
			</template>
		</view>
		
		<!-- 上传进度 -->
		<view v-if="isUploading" class="upload-progress">
			<view class="progress-info">
				<view class="progress-item">
					<text>正在上传: {{currentUploadFile}}</text>
					<view class="progress-bar">
						<view class="progress-inner" :style="{transform: `scaleX(${currentFileProgress / 100})`}"></view>
					</view>
					<text class="progress-text">{{currentFileProgress}}%</text>
				</view>
				<view class="progress-item">
					<text>总进度: {{totalProgress}}%</text>
					<view class="progress-bar">
						<view class="progress-inner" :style="{transform: `scaleX(${totalProgress / 100})`}"></view>
					</view>
				</view>
			</view>
		</view>
		
		<!-- 上传按钮 -->
		<button class="upload-btn" 
				:disabled="!hasSelectedFiles || isUploading || isLoading"
				:class="{'primary': hasSelectedFiles && !isUploading && !isLoading}"
				@tap="uploadSelectedFiles">
			{{ isUploading ? '上传中...' : isLoading ? '加载中...' : '上传选中文件' }}
		</button>
	</view>
</template>

<script setup>
import { ref, computed, onMounted } from 'vue'

// 日期选择
const selectedDate = ref(formatDate(new Date()))
const fileList = ref([])
const isUploading = ref(false)
const currentUploadFile = ref('')
const currentFileProgress = ref(0)
const totalProgress = ref(0)
const authorizedUri = ref('')
const isLoading = ref(false)
const uploadedFiles = ref(0)

// 计算属性
const allSelected = computed(() => {
	return fileList.value.length > 0 && fileList.value.every(file => file.selected)
})

const hasSelectedFiles = computed(() => {
	return fileList.value.some(file => file.selected)
})

// 日期格式化
function formatDate(date) {
	const year = date.getFullYear()
	const month = String(date.getMonth() + 1).padStart(2, '0')
	const day = String(date.getDate()).padStart(2, '0')
	return `${year}-${month}-${day}`
}

// 时间格式化
function formatTime(timestamp) {
	const date = new Date(timestamp)
	const hours = String(date.getHours()).padStart(2, '0')
	const minutes = String(date.getMinutes()).padStart(2, '0')
	return `${hours}:${minutes}`
}

// 日期选择处理
function onDateChange(e) {
	selectedDate.value = e.detail.value
	loadFileList()
}

// 选择处理
function toggleAllSelect() {
	const newState = !allSelected.value
	fileList.value.forEach(file => file.selected = newState)
}

function toggleSelect(index) {
	fileList.value[index].selected = !fileList.value[index].selected
}

// 请求权限
async function requestPermission() {
    return new Promise((resolve, reject) => {
        try {
            console.log('开始请求权限')
            const main = plus.android.runtimeMainActivity()
            const Intent = plus.android.importClass('android.content.Intent')
            const Uri = plus.android.importClass('android.net.Uri')
            const DocumentsContract = plus.android.importClass('android.provider.DocumentsContract')
            
            const intent = new Intent(Intent.ACTION_OPEN_DOCUMENT_TREE)
            
            // 添加权限标志
            intent.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION | 
                          Intent.FLAG_GRANT_WRITE_URI_PERMISSION |
                          Intent.FLAG_GRANT_PREFIX_URI_PERMISSION |
                          Intent.FLAG_GRANT_PERSISTABLE_URI_PERMISSION)
            
            // 设置初始目录为 Sounds
            const initialUri = DocumentsContract.buildDocumentUri(
                "com.android.externalstorage.documents",
                "primary:Sounds"
            )
            intent.putExtra(DocumentsContract.EXTRA_INITIAL_URI, initialUri)
            
            // 设置新的回调
            main.onActivityResult = (requestCode, resultCode, data) => {
                console.log('收到权限请求结果:', requestCode, resultCode)
                if (requestCode === 1) {
                    if (resultCode === -1 && data) {
                        try {
                            const uri = data.getData()
                            const uriString = uri.toString()
                            console.log('获取到URI:', uriString)
                            
                            // 直接保存URI并设置权限标志
                            authorizedUri.value = uriString
                            uni.setStorageSync('authorizedUri', uriString)
                            
                            // 使用 addFlags 设置权限
                            const flags = Intent.FLAG_GRANT_READ_URI_PERMISSION | 
                                        Intent.FLAG_GRANT_WRITE_URI_PERMISSION
                            main.getApplicationContext().grantUriPermission(
                                main.getPackageName(),
                                uri,
                                flags
                            )
                            
                            console.log('权限保存成功')
                            resolve(true)
                        } catch (error) {
                            console.error('处理权限结果失败:', error)
                            resolve(false)
                        }
                    } else {
                        console.log('用户取消了权限请求')
                        resolve(false)
                    }
                }
            }
            
            main.startActivityForResult(intent, 1)
        } catch (error) {
            console.error('请求权限失败:', error)
            reject(error)
        }
    })
}

// 检查已有权限
async function checkPermission() {
    try {
        if (plus.os.name.toLowerCase() === 'android') {
            const savedUri = uni.getStorageSync('authorizedUri')
            console.log('检查已保存的URI:', savedUri)
            
            if (savedUri) {
                const main = plus.android.runtimeMainActivity()
                const Uri = plus.android.importClass('android.net.Uri')
                const DocumentFile = plus.android.importClass('androidx.documentfile.provider.DocumentFile')
                
                try {
                    const uri = Uri.parse(savedUri)
                    const pickedDir = DocumentFile.fromTreeUri(main, uri)
                    
                    // 尝试列出文件来验证权限
                    if (pickedDir && pickedDir.exists()) {
                        try {
                            pickedDir.listFiles()
                            console.log('权限验证成功')
                            authorizedUri.value = savedUri
                            return true
                        } catch (error) {
                            console.log('列出文件失败，权限可能已失效:', error)
                        }
                    }
                } catch (error) {
                    console.log('解析URI失败:', error)
                }
            }
        }
        return false
    } catch (error) {
        console.error('检查权限失败:', error)
        return false
    }
}

// 加载文件列表
async function loadFileList() {
    console.log('开始加载文件列表, 日期:', selectedDate.value)
    isLoading.value = true
    try {
        if (plus.os.name.toLowerCase() === 'android') {
            // 检查权限
            let hasPermission = await checkPermission()
            console.log('权限检查结果:', hasPermission)
            
            if (!hasPermission) {
                hasPermission = await requestPermission()
                console.log('权限请求结果:', hasPermission)
                if (!hasPermission) {
                    throw new Error('未获得权限')
                }
                
                // 重新检查权限
                hasPermission = await checkPermission()
                if (!hasPermission) {
                    throw new Error('权限验证失败')
                }
            }
            
            // 使用已授权的URI访问文件
            const main = plus.android.runtimeMainActivity()
            const Uri = plus.android.importClass('android.net.Uri')
            const DocumentFile = plus.android.importClass('androidx.documentfile.provider.DocumentFile')
            
            console.log('使用URI访问文件:', authorizedUri.value)
            const uri = Uri.parse(authorizedUri.value)
            const pickedDir = DocumentFile.fromTreeUri(main, uri)
            
            if (pickedDir && pickedDir.exists()) {
                console.log('成功访问目录')
                // 清空当前列表
                fileList.value = []
                
                // 递归遍历目录
                async function traverseDirectory(dir) {
                    const files = dir.listFiles()
                    for (let i = 0; i < files.length; i++) {
                        const file = files[i]
                        if (file.isDirectory()) {
                            await traverseDirectory(file)
                        } else {
                            const fileName = file.getName()
                            const mimeType = file.getType()
                            
                            // 检查是否为音频文件
                            if (mimeType && mimeType.startsWith('audio/')) {
                                try {
                                    const parts = fileName.split('_')
                                    if (parts.length >= 2 && parts[1].length >= 8) {
                                        const fileDate = parts[1].substring(0, 8)
                                        const selectedDateFormatted = selectedDate.value.replace(/-/g, '')
                                        
                                        if (fileDate === selectedDateFormatted) {
                                            fileList.value.push({
                                                name: fileName,
                                                uri: file.getUri().toString(),
                                                time: file.lastModified(),
                                                selected: true
                                            })
                                        }
                                    }
                                } catch (error) {
                                    console.log('跳过不符合格式的文件:', fileName)
                                    continue
                                }
                            }
                        }
                    }
                }
                
                await traverseDirectory(pickedDir)
                
                // 按时间排序
                fileList.value.sort((a, b) => b.time - a.time)
                console.log('最终文件列表数量:', fileList.value.length)
                
                if (fileList.value.length === 0) {
                    uni.showToast({
                        title: '当前日期没有录音文件',
                        icon: 'none'
                    })
                }
            } else {
                console.error('目录访问失败')
                throw new Error('无法访问目录')
            }
        }
    } catch (error) {
        console.error('加载文件列表失败:', error)
        uni.showToast({
            title: '加载文件列表失败',
            icon: 'none'
        })
    } finally {
        isLoading.value = false
    }
}

// 从URI中获取文件名
const getFileNameFromUri = (uriString) => {
    try {
        // 处理 content:// URI
        if (uriString.startsWith('content://')) {
            const matches = uriString.match(/[^\/]+$/);
            if (matches) {
                let fileName = decodeURIComponent(matches[0]);
                // 如果文件名包含 primary: 前缀，去除它
                if (fileName.startsWith('primary:')) {
                    const parts = fileName.split('/');
                    fileName = parts[parts.length - 1];
                }
                return fileName;
            }
        }
        // 处理普通路径
        const parts = uriString.split('/');
        return decodeURIComponent(parts[parts.length - 1]);
    } catch (error) {
        console.error('解析文件名出错:', error);
        return '未知文件名';
    }
}

// 选择文件
const chooseFile = async () => {
    // #ifdef APP-PLUS
    try {
        console.log('开始选择文件');
        
        // 检查并请求权限（仅Android）
        if (plus.os.name.toLowerCase() === 'android') {
            await new Promise((resolve, reject) => {
                plus.android.requestPermissions(
                    ['android.permission.READ_EXTERNAL_STORAGE', 'android.permission.WRITE_EXTERNAL_STORAGE'],
                    function(e) {
                        console.log('权限请求成功');
                        resolve();
                    },
                    function(e) {
                        console.log('权限请求失败');
                        reject(new Error('权限请求失败'));
                    }
                );
            });
        }

        // Android 平台
        if (plus.os.name.toLowerCase() === 'android') {
            const main = plus.android.runtimeMainActivity();
            const Intent = plus.android.importClass('android.content.Intent');
            const intent = new Intent(Intent.ACTION_GET_CONTENT);
            intent.setType("audio/*");  // 只显示音频文件
            intent.addCategory(Intent.CATEGORY_OPENABLE);
            
            const REQUEST_CODE = 1;
            
            // 设置文件选择结果处理
            main.onActivityResult = (requestCode, resultCode, data) => {
                console.log('收到文件选择结果:', requestCode, resultCode);
                if (requestCode === REQUEST_CODE) {
                    const RESULT_OK = -1;
                    if (resultCode === RESULT_OK && data) {
                        try {
                            const uri = data.getData();
                            const uriString = plus.android.invoke(uri, 'toString');
                            console.log('文件URI:', uriString);
                            
                            // 从URI中获取文件名
                            const fileName = getFileNameFromUri(uriString);
                            console.log('解析的文件名:', fileName);
                            
                            // 获取文件路径
                            try {
                                const context = plus.android.runtimeMainActivity();
                                const DocumentFile = plus.android.importClass('androidx.documentfile.provider.DocumentFile');
                                const documentFile = DocumentFile.fromSingleUri(context, uri);
                                
                                if (documentFile) {
                                    console.log('DocumentFile 创建成功');
                                    const filePath = plus.io.convertLocalFileSystemURL(uriString);
                                    console.log('转换后的文件路径:', filePath);
                                    
                                    // 获取文件大小
                                    const fileSize = plus.android.invoke(documentFile, 'length');
                                    console.log('文件大小:', fileSize);
                                    
                                    // 添加到文件列表
                                    fileList.value.push({
                                        name: fileName,
                                        path: filePath,
                                        size: fileSize,
                                        time: new Date().getTime(),
                                        selected: true
                                    });
                                    console.log('已添加文件到列表:', fileName);
                                } else {
                                    throw new Error('无法创建 DocumentFile');
                                }
                            } catch (error) {
                                console.error('处理文件路径时出错:', error);
                                uni.showToast({
                                    title: '无法访问文件',
                                    icon: 'none'
                                });
                            }
                        } catch (error) {
                            console.error('处理文件选择结果出错:', error);
                            uni.showToast({
                                title: '处理文件失败',
                                icon: 'none'
                            });
                        }
                    }
                }
            };
            
            // 启动文件选择器
            main.startActivityForResult(intent, REQUEST_CODE);
        }
    } catch (error) {
        console.error('文件选择错误:', error);
        uni.showToast({
            title: '选择文件失败',
            icon: 'none'
        });
    }
    // #endif
}

// 上传文件
async function uploadFile(file) {
    return new Promise((resolve, reject) => {
        const uploadTask = uni.uploadFile({
            url: 'http://110.40.48.222:54977/upload/audio',
            filePath: file.path,
            name: 'file',
            formData: {
                'filename': file.name
            },
            success: (res) => {
                console.log('文件上传成功:', file.name, res);
                try {
                    if (res.statusCode === 200) {
                        const response = JSON.parse(res.data);
                        if (response && response.success) {
                            resolve(response);
                        } else {
                            reject(new Error(response?.message || '上传失败'));
                        }
                    } else {
                        reject(new Error(`上传失败，状态码：${res.statusCode}`));
                    }
                } catch (error) {
                    console.error('处理上传响应出错:', error);
                    reject(error);
                }
            },
            fail: (err) => {
                console.error('上传失败:', err);
                reject(err);
            }
        });

        // 监听上传进度
        if (uploadTask) {
            uploadTask.onProgressUpdate((res) => {
                console.log('上传进度:', {
                    file: file.name,
                    fileProgress: res.progress,
                    completed: res.totalBytesSent,
                    total: res.totalBytesExpectedToSend
                });
                
                // 更新当前文件进度
                currentFileProgress.value = res.progress;
                
                // 更新总进度
                const totalFiles = fileList.value.filter(f => f.selected).length;
                const completedFiles = uploadedFiles.value;
                totalProgress.value = Math.floor((completedFiles * 100 + res.progress) / totalFiles);
            });
        }
    });
}

// 上传文件
async function uploadSelectedFiles() {
    if (!hasSelectedFiles.value || isUploading.value || isLoading.value) return;
    
    try {
        isUploading.value = true;
        currentUploadFile.value = '';
        currentFileProgress.value = 0;
        totalProgress.value = 0;
        uploadedFiles.value = 0;
        
        const selectedFiles = fileList.value.filter(file => file.selected);
        
        for (const file of selectedFiles) {
            try {
                currentUploadFile.value = file.name;
                await uploadFile(file);
                uploadedFiles.value++;
            } catch (error) {
                console.error('上传错误:', error);
                uni.showToast({
                    title: `${file.name} 上传失败`,
                    icon: 'none'
                });
            }
        }
    } catch (error) {
        console.error('批量上传错误:', error);
        uni.showToast({
            title: '上传过程中出错',
            icon: 'none'
        });
    } finally {
        isUploading.value = false;
        currentUploadFile.value = '';
        currentFileProgress.value = 0;
    }
}

// 页面加载时获取文件列表
onMounted(() => {
	loadFileList()
})
</script>

<style>
.content {
	display: flex;
	flex-direction: column;
	padding: 20rpx;
	height: 100vh;
}

.date-picker {
	padding: 20rpx;
	background-color: #fff;
	border-radius: 10rpx;
	margin-bottom: 20rpx;
	display: flex;
	align-items: center;
	justify-content: space-between;
}

.picker-text {
	font-size: 32rpx;
	color: #333;
}

.file-select-btn {
	width: 80rpx;
	height: 80rpx;
	display: flex;
	align-items: center;
	justify-content: center;
	background-color: #f8f8f8;
	border-radius: 50%;
}

.file-select-btn .iconfont {
	font-size: 40rpx;
	color: #007AFF;
}

.file-container {
	flex: 1;
	display: flex;
	flex-direction: column;
}

.loading-container {
	flex: 1;
	display: flex;
	justify-content: center;
	align-items: center;
	background-color: #ffffff;
	border-radius: 10rpx;
	margin-bottom: 20rpx;
}

.loading-content {
	display: flex;
	flex-direction: column;
	align-items: center;
	padding: 40rpx;
}

.loading-text {
	font-size: 28rpx;
	color: #666;
}

.file-list {
	flex: 1;
	background-color: #ffffff;
	border-radius: 10rpx;
	margin-bottom: 20rpx;
	overflow: hidden;
}

.list-header {
	padding: 20rpx;
	border-bottom: 1rpx solid #eee;
	display: flex;
	align-items: center;
}

.list-header text {
	margin-left: 20rpx;
	font-size: 32rpx;
	color: #333;
}

.list-content {
	height: calc(100% - 100rpx);
}

.file-item {
	padding: 20rpx;
	display: flex;
	align-items: center;
	border-bottom: 1rpx solid #eee;
}

.file-name {
	flex: 1;
	margin-left: 20rpx;
	font-size: 28rpx;
	color: #333;
}

.file-time {
	font-size: 24rpx;
	color: #999;
	margin-left: 20rpx;
}

.no-files {
	padding: 40rpx;
	text-align: center;
	color: #999;
	background-color: #ffffff;
	border-radius: 10rpx;
	margin-bottom: 20rpx;
}

.upload-progress {
	background-color: #ffffff;
	border-radius: 10rpx;
	padding: 20rpx;
	margin-bottom: 20rpx;
}

.progress-info {
	padding: 20rpx;
	background-color: #fff;
	border-radius: 10rpx;
	margin: 20rpx;
}

.progress-item {
	margin-bottom: 20rpx;
}

.progress-item text {
	font-size: 28rpx;
	color: #333;
	margin-bottom: 10rpx;
	display: block;
}

.progress-text {
	text-align: right;
	font-size: 24rpx;
	color: #666;
	margin-top: 4rpx;
}

.progress-bar {
	width: 100%;
	height: 6rpx;
	background-color: #f0f0f0;
	border-radius: 3rpx;
	overflow: hidden;
}

.progress-inner {
	width: 100%;
	height: 100%;
	background-color: #007AFF;
	border-radius: 3rpx;
	transform-origin: left;
	transition: transform 0.3s linear;
	will-change: transform;
}

.upload-btn {
	width: 100%;
	height: 88rpx;
	line-height: 88rpx;
	text-align: center;
	border-radius: 10rpx;
	background-color: #f0f0f0;
	color: #999;
}

.upload-btn.primary {
	background-color: #007AFF;
	color: #fff;
}

.upload-btn:disabled {
	opacity: 0.6;
}

@font-face {
	font-family: "iconfont";
	src: url('data:application/x-font-woff2;charset=utf-8;base64,d09GMgABAAAAAALYAAsAAAAABpwAAAKLAAEAAAAAAAAAAAAAAAAAAAAAAAAAAAAAHEIGVgCCcAqBPIE8ATYCJAMICwYABCAFhG0HMBvdBciemjwJBEUUAKAoHmCwNf7s3d0nqiZR06gnkUQzJEKGRIhQKYlE6IQO5f9fa+cD/4kjkVVFlVAh0ft9L7O7OmKRhEQIhRAJ0QqhQqK+WbG5OXxeOL3UF8hnW5dYk2ZNmgI4HtAAjbEtssIu5B38D2MXvMDzBGoNmpRsLK9sAk4ZMC4Q9wJOAWeZUqnKVShULM0s4gkKVekh/QjgMfp+/IeiOEmhYCZtPSqXgdkvwDc4tf/XulwECD4fcJtEhmlAJm5WJk5hgkmptbCxYlBcVQW/AJv///9DQVREtf76eJFCtB0yO2Ank1sCX8wkBL7YSQh9k2tnV0hwBzwC3nBkd3Vlb1NVR1tz48r+8kJ/c0vbvvKD7Y0H9jRXNe9p6WjurNnf2FVf2V67Z+f2XbVxHXFXnJu7t6W5s6GmtbOhfU9dZ0PnvqbOhqrGhvb9zZ2N7Xuad+7cXnUu7o1zc3e3Zz7YOLZnb+WB+o7qXXs76g/uaWrYW9fZVLW3oaOxvbaqobGz5sCOXXEXnJu7uyPzYZBNw+jFjPtP5oVr/qb5YL8/e/LBwt9ZCQDf/+b5C/DFzPz/JwBPXc3fBF/MKvhlFqWC36ZB4JdpxJVVqFWL/5udk0CtnwR1XlqGa5eVKqpUKoGTqvwJFGqNIjNzGgpV5qBQawFqTVm1ukIfIoQyB1M+AYRGr5Co9wYKjb4iM/MbFNrDHwo1kQy1dkbLigOmwzDRYUViiOgHzRQVSt0iHXYN+T5DPqcQpZwSPkNfYYdYLHFY4hZ1iDnGKL0oQoUoQw5xHhFFEXJRGkVF6TzKFGXIoigdFaJ0UeQBAKYGhREtTEEYhNAPaEyhBCXdRvL5ayDfzyCeJkEok0MsPgN9Cg5EMQnvwJKu0KycynKlR9ERigglEMcQDuLxEEIhEeQVSUNRGqpPRTEiC+VCpUNKiKQLRW0PANRSt3hGuY1CKFFCS0n3pKU0NQA=') format('woff2');
}

.iconfont {
	font-family: "iconfont" !important;
	font-style: normal;
	-webkit-font-smoothing: antialiased;
	-moz-osx-font-smoothing: grayscale;
}

.icon-file:before {
	content: "\e601";
}
</style>