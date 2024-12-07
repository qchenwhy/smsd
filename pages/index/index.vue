<template>
	<view class="content">
		<!-- 日期选择器 -->
		<view class="date-picker">
			<picker mode="date" :value="selectedDate" @change="onDateChange">
				<view class="picker-text">
					选择日期: {{selectedDate}}
				</view>
			</picker>
		</view>
		
		<!-- 文件列表 -->
		<view class="file-list" v-if="fileList.length > 0">
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
		
		<!-- 上传进度 -->
		<view v-if="isUploading" class="upload-progress">
			<view class="progress-info">
				<text>正在上传: {{currentUploadFile}}</text>
				<text>{{currentFileProgress}}%</text>
			</view>
			<progress :percent="currentFileProgress" active stroke-width="3" />
			<view class="total-progress">
				<text>总进度: {{totalProgress}}%</text>
				<progress :percent="totalProgress" active stroke-width="3" />
			</view>
		</view>
		
		<!-- 上传按钮 -->
		<button class="upload-btn" 
				:disabled="!hasSelectedFiles || isUploading"
				:class="{'primary': hasSelectedFiles && !isUploading}"
				@tap="uploadSelectedFiles">
			{{ isUploading ? '上传中...' : '上传选中文件' }}
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
            
            // 设置初始目录
            const initialUri = DocumentsContract.buildDocumentUri(
                "com.android.externalstorage.documents",
                "primary:Sounds/CallRecord"
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
                const files = pickedDir.listFiles()
                console.log('文件总数:', files.length)
                
                // 清空当前列表
                fileList.value = []
                
                // 遍历文件
                for (let i = 0; i < files.length; i++) {
                    const file = files[i]
                    const fileName = file.getName()
                    
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
                                console.log('添加文件到列表:', fileName)
                            }
                        }
                    } catch (error) {
                        console.log('跳过不符合格式的文件:', fileName, error)
                        continue
                    }
                }
                
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
    }
}

// 上传文件
async function uploadSelectedFiles() {
	if (!hasSelectedFiles.value || isUploading.value) return
	
	const selectedFiles = fileList.value.filter(file => file.selected)
	const totalFiles = selectedFiles.length
	let uploadedFiles = 0
	
	isUploading.value = true
	
	try {
		for (const file of selectedFiles) {
			currentUploadFile.value = file.name
			currentFileProgress.value = 0
			
			await new Promise((resolve, reject) => {
				try {
					const main = plus.android.runtimeMainActivity()
					const Uri = plus.android.importClass('android.net.Uri')
					const uri = Uri.parse(file.uri)
					
					// 直接使用uri进行上传
					const uploadTask = uni.uploadFile({
						url: 'http://110.40.48.222:54977/upload/audio',
						filePath: file.uri,
						name: 'file',
						formData: {
							'filename': file.name
						},
						header: {
							'Content-Type': 'multipart/form-data'
						},
						success: (res) => {
							console.log('文件上传成功:', file.name, res)
							resolve(res)
						},
						fail: (err) => {
							console.error('文件上传失败:', file.name, err)
							reject(err)
						}
					})
					
					uploadTask.onProgressUpdate((res) => {
						currentFileProgress.value = res.progress
						totalProgress.value = Math.round((uploadedFiles * 100 + res.progress) / totalFiles)
					})
				} catch (error) {
					console.error('处理文件失败:', error)
					reject(error)
				}
			})
			
			uploadedFiles++
		}
		
		uni.showToast({
			title: '所有文件上传完成',
			icon: 'success'
		})
	} catch (error) {
		console.error('上传错误:', error)
		uni.showToast({
			title: '上传过程中出现错误',
			icon: 'none'
		})
	} finally {
		isUploading.value = false
		currentUploadFile.value = ''
		currentFileProgress.value = 0
		totalProgress.value = 0
		loadFileList()
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
	background-color: #ffffff;
	border-radius: 10rpx;
	margin-bottom: 20rpx;
}

.picker-text {
	font-size: 32rpx;
	color: #333;
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
	display: flex;
	justify-content: space-between;
	margin-bottom: 10rpx;
	font-size: 28rpx;
	color: #666;
}

.total-progress {
	margin-top: 20rpx;
}

.total-progress text {
	font-size: 28rpx;
	color: #666;
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
</style>