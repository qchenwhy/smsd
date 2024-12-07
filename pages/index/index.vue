<template>
	<view class="content">
		<view class="upload-container">
			<button class="upload-btn" @click="chooseFile">选择文件</button>
			<view v-if="selectedFile" class="file-info">
				<text>已选择文件: {{selectedFile.name}}</text>
			</view>
			<button v-if="selectedFile" class="upload-btn primary" @click="uploadFile">上传文件</button>
		</view>
		<view v-if="uploadProgress > 0" class="progress">
			<text>上传进度: {{uploadProgress}}%</text>
		</view>
	</view>
</template>

<script setup>
import { ref } from 'vue'

const selectedFile = ref(null)
const uploadProgress = ref(0)

// 从URI中提取文件名
const getFileNameFromUri = (uri) => {
	try {
		const decodedUri = decodeURIComponent(uri);
		const matches = decodedUri.match(/([^\/]+)$/);
		if (matches && matches[1]) {
			return matches[1];
		}
		return '未知文件';
	} catch (error) {
		console.error('解析文件名出错:', error);
		return '未知文件';
	}
}

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
			intent.setType("*/*");
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
								if (uriString.startsWith('content://')) {
									const Uri = plus.android.importClass('android.net.Uri');
									const ContentUris = plus.android.importClass('android.content.ContentUris');
									const context = plus.android.runtimeMainActivity();
									
									// 尝试使用 DocumentFile 获取真实路径
									const DocumentFile = plus.android.importClass('androidx.documentfile.provider.DocumentFile');
									const documentFile = DocumentFile.fromSingleUri(context, uri);
									
									if (documentFile) {
										console.log('DocumentFile 创建成功');
										const filePath = plus.io.convertLocalFileSystemURL(uriString);
										console.log('转换后的文件路径:', filePath);
										
										selectedFile.value = {
											name: fileName,
											path: filePath
										};
										console.log('已设置选中文件:', selectedFile.value);
									} else {
										throw new Error('无法创建 DocumentFile');
									}
								} else {
									console.log('使用直接路径');
									selectedFile.value = {
										name: fileName,
										path: uriString
									};
									console.log('已设置选中文件:', selectedFile.value);
								}
							} catch (error) {
								console.error('处理文件路径时出错:', error);
								// 尝试直接使用 URI
								selectedFile.value = {
									name: fileName,
									path: uriString
								};
								console.log('使用原始URI作为路径:', selectedFile.value);
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
		// iOS 平台
		else {
			const UTI = ["public.content", "public.data", "public.item"];
			plus.ios.import("UIDocumentPickerViewController")
				.createDocumentPickerViewController(UTI, (picker) => {
					picker.plusLoadSuccess = () => {
						picker.modalPresentationStyle = 0;
					};
					
					picker.plusPickerCallback = (path) => {
						console.log('iOS选择文件路径:', path);
						if (path) {
							const fileName = path.substring(path.lastIndexOf('/') + 1);
							selectedFile.value = {
								name: fileName,
								path: path
							};
							console.log('已设置选中文件:', selectedFile.value);
						}
					};
					
					picker.show();
				});
		}
	} catch (error) {
		console.error('选择文件出错:', error);
		uni.showToast({
			title: '选择文件失败',
			icon: 'none'
		});
	}
	// #endif
	
	// #ifdef H5
	const input = document.createElement('input');
	input.type = 'file';
	input.onchange = (event) => {
		const file = event.target.files[0];
		if (file) {
			selectedFile.value = {
				name: file.name,
				path: URL.createObjectURL(file)
			};
			console.log('已选择文件:', selectedFile.value);
		}
	};
	input.click();
	// #endif
}

const uploadFile = () => {
	if (!selectedFile.value) {
		uni.showToast({
			title: '请先选择文件',
			icon: 'none'
		})
		return
	}

	console.log('开始上传文件:', selectedFile.value);
	const uploadTask = uni.uploadFile({
		url: 'http://110.40.48.222:54977/upload/audio', // 替换为你的服务器上传地址
		filePath: selectedFile.value.path,
		name: 'file',
		success: (res) => {
			console.log('上传成功:', res);
			uni.showToast({
				title: '上传成功',
				icon: 'success'
			})
			selectedFile.value = null
			uploadProgress.value = 0
		},
		fail: (err) => {
			console.error('上传失败:', err);
			uni.showToast({
				title: '上传失败',
				icon: 'none'
			})
		}
	})

	uploadTask.onProgressUpdate((res) => {
		console.log('上传进度:', res.progress);
		uploadProgress.value = res.progress
	})
}
</script>

<style>
.content {
	display: flex;
	flex-direction: column;
	align-items: center;
	padding: 20px;
}

.upload-container {
	width: 100%;
	display: flex;
	flex-direction: column;
	align-items: center;
	gap: 20rpx;
}

.upload-btn {
	width: 80%;
	height: 80rpx;
	line-height: 80rpx;
	text-align: center;
	border-radius: 10rpx;
	background-color: #f0f0f0;
}

.upload-btn.primary {
	background-color: #007AFF;
	color: #fff;
}

.file-info {
	width: 80%;
	padding: 20rpx;
	background-color: #f8f8f8;
	border-radius: 10rpx;
}

.progress {
	margin-top: 30rpx;
	font-size: 28rpx;
	color: #666;
}
</style>
