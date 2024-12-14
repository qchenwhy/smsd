<template>
	<view class="container">
		<view class="joystick-wrapper">
			<view class="joystick-container" id="joystick" @touchstart.stop.prevent="onTouchStart" @touchmove.stop.prevent="onTouchMove" @touchend.stop.prevent="onTouchEnd">
				<!-- 最外层圈（分区） -->
				<view class="outer-ring">
					<view v-for="(section, index) in 6" :key="index" 
						:class="['outer-section', `section-${index}`, activeSection === index && showInnerCircle ? 'active' : '']">
					</view>
				</view>
				<!-- 图标容器 -->
				<view class="icons-container">
				    <view v-for="(icon, index) in currentIcons" :key="index"
				        :class="['icon-wrapper', `icon-pos-${index}`, { 'show': showInnerCircle }]">
				        <image v-if="!showText || activeSection !== (index+1)%6"
				            :src="icon.src" 
				            class="icon"
				            mode="aspectFit"
				        />
						<text v-if="showText && activeSection === (index+1)%6" class="icon-text">{{ icon.text }}</text>
				    </view>
				</view>
				<!-- 内圈（限制区域） -->
				<view class="inner-circle" :class="{ 'show-circle': showInnerCircle }"></view>
				<!-- 可移动的球 -->
				<view class="inner-ball" :style="ballStyle"></view>
			</view>
		</view>
		
		<!-- 蒙层 -->
		<view class="overlay" v-if="showDialog" @click="hideDialogs"></view>
		
		<!-- 对话框 -->
		<view class="dialog" :class="{ 'dialog-show': showDialog }">
			<view class="dialog-content">
				<view class="dialog-header">
					<text>当日计划</text>
				</view>
				<scroll-view class="card-list" scroll-y>
					<view 
						class="card" 
						v-for="(item, index) in cardList" 
						:key="index"
						:class="{ 'card-highlight': item.isHighlight }"
					>
						<view class="card-header">
							<text class="card-title">{{ item.title }}</text>
							<text class="card-time">{{ item.time }}</text>
						</view>
						<view class="card-body">
							<text class="card-description">{{ item.description }}</text>
						</view>
						<view class="card-footer">
							<view class="card-status" :class="item.status.toLowerCase()">
								{{ item.status }}
							</view>
							<view class="card-tags">
								<text 
									class="card-tag" 
									v-for="(tag, tagIndex) in item.tags" 
									:key="tagIndex"
								>
									{{ tag }}
								</text>
							</view>
						</view>
					</view>
				</scroll-view>
			</view>
			<view class="close-icon" @click="hideDialogs">
				<image src="/static/icons/close.svg" mode="aspectFit" />
			</view>
		</view>
		
		<!-- 输入框 -->
		<view class="input-box" :class="{ 'input-show': showDialog }">
			<view class="input-container">
				<view class="input-icon-wrapper" @click="toggleInputMode">
					<view class="icon-flip-container" :class="{ 'flip': isVoiceMode }">
						<view class="icon-flipper">
							<image 
								class="input-icon front"
								src="/static/icons/voice.svg"
							/>
							<image 
								class="input-icon back"
								src="/static/icons/keyboard.svg"
							/>
						</view>
					</view>
				</view>
				<view class="input-content">
					<view class="text-input" :class="{ 'slide-up': isVoiceMode }">
						<input type="text" v-model="inputText" placeholder="请输入内容" />
					</view>
					<view class="voice-button" :class="{ 'slide-up-in': isVoiceMode }" @touchstart="startVoiceInput" @touchend="endVoiceInput">
						按住说话
					</view>
				</view>
				<view class="send-icon-wrapper" :class="{ 'fade-in': inputText.length > 0 }">
					<image 
						class="send-icon"
						src="/static/icons/send.svg"
						@click="sendMessage"
					/>
				</view>
			</view>
		</view>
	</view>
</template>

<script setup>
import { ref, computed, onMounted } from 'vue';

// 一级菜单
const icons = [  
	{ src: '/static/icons/icon1.svg', text: '工作' },    // 上
	{ src: '/static/icons/icon2.svg', text: '设置' },    // 右上
	 { src: '/static/icons/icon6.svg', text: '更多' },  // 右下
	{ src: '/static/icons/icon4.svg', text: '学习' },    // 下
	{ src: '/static/icons/icon5.svg', text: '财务' } ,
	{ src: '/static/icons/icon3.svg', text: '生活' }
];

// 二级菜单配置
const subMenus = {
	'工作': [
		{ src: '/static/icons/work1.svg', text: '长期规划' },    // 上
		{ src: '/static/icons/work2.svg', text: '短期目标' },    // 左上
		{ src: '/static/icons/work3.svg', text: '内容上传' },    // 左下
		{ src: '/static/icons/work4.svg', text: '资讯获取' },    // 下
		{ src: '/static/icons/work5.svg', text: '返回' }         // 右上
	],
	'设置': [
		{ src: '/static/icons/setting1.svg', text: '系统设置' },
		{ src: '/static/icons/setting2.svg', text: '个人信息' },
		{ src: '/static/icons/setting3.svg', text: '通知设置' },
		{ src: '/static/icons/setting4.svg', text: '隐私设置' },
		{ src: '/static/icons/setting5.svg', text: '返回' }
	],
	'更多': [
		{ src: '/static/icons/more1.svg', text: '帮助中心' },
		{ src: '/static/icons/more2.svg', text: '关于我们' },
		{ src: '/static/icons/more3.svg', text: '意见反馈' },
		{ src: '/static/icons/more4.svg', text: '版本信息' },
		{ src: '/static/icons/more5.svg', text: '返回' }
	],
	'学习': [
		{ src: '/static/icons/study1.svg', text: '课程学习' },
		{ src: '/static/icons/study2.svg', text: '资料下载' },
		{ src: '/static/icons/study3.svg', text: '笔记管理' },
		{ src: '/static/icons/study4.svg', text: '学习计划' },
		{ src: '/static/icons/study5.svg', text: '返回' }
	],
	'财务': [
		{ src: '/static/icons/finance1.svg', text: '收支统计' },
		{ src: '/static/icons/finance2.svg', text: '预算管理' },
		{ src: '/static/icons/finance3.svg', text: '账单明细' },
		{ src: '/static/icons/finance4.svg', text: '财务报表' },
		{ src: '/static/icons/finance5.svg', text: '返回' }
	],
	'生活': [
		{ src: '/static/icons/life1.svg', text: '日程安排' },
		{ src: '/static/icons/life2.svg', text: '健康管理' },
		{ src: '/static/icons/life3.svg', text: '生活记录' },
		{ src: '/static/icons/life4.svg', text: '休闲娱乐' },
		{ src: '/static/icons/life5.svg', text: '返回' }
	]
};

// 三级菜单配置
const thirdLevelMenus = {
	'短期目标': [
		{ src: '/static/icons/daily1.svg', text: '当日计划' },
		{ src: '/static/icons/daily2.svg', text: '当日进度' },
		{ src: '/static/icons/daily3.svg', text: '当日总结' },
		{ src: '/static/icons/daily4.svg', text: '评价建议' },
		{ src: '/static/icons/daily5.svg', text: '返回上一层' },
		{ src: '/static/icons/daily6.svg', text: '设置' }
	]
};

// 当前菜单状态
const currentMenu = ref('main');
const currentSubMenu = ref(''); // 记录当前二级菜单项
const currentIcons = computed(() => {
	if (currentMenu.value === 'main') {
		return icons;
	} else if (currentSubMenu.value && thirdLevelMenus[currentSubMenu.value]) {
		return thirdLevelMenus[currentSubMenu.value];
	} else {
		return subMenus[currentMenu.value] || icons;
	}
});

const ballPosition = ref({ x: 0, y: 0 });
const activeSection = ref(-1);
const isDragging = ref(false);
const joystickRect = ref({ width: 0, height: 0, left: 0, top: 0 });
const showInnerCircle = ref(false);
const showText = ref(false);
let pressTimer = null;
let textTimer = null;

// 对话框状态
const showDialog = ref(false);

// 卡片数据
const cardList = ref([
	{
		title: '上午会议',
		time: '09:30 - 10:30',
		description: '讨论新项目进展情况，确定下一步开发计划',
		status: 'Pending',
		tags: ['重要', '团队'],
		isHighlight: false
	},
	{
		title: '代码评审',
		time: '11:00 - 12:00',
		description: '评审前端新功能代码，确保代码质量',
		status: 'InProgress',
		tags: ['技术', '评审'],
		isHighlight: false
	},
	{
		title: '客户沟通',
		time: '14:00 - 15:00',
		description: '与客户讨论需求变更，确认新功能细节',
		status: 'Completed',
		tags: ['客户', '需求'],
		isHighlight: false
	}
])

// 检查时间是否在范围内
const isTimeInRange = (timeRange) => {
    const [start, end] = timeRange.split(' - ');
    const now = new Date();
    const currentHours = now.getHours();
    const currentMinutes = now.getMinutes();
    
    const [startHours, startMinutes] = start.split(':').map(Number);
    const [endHours, endMinutes] = end.split(':').map(Number);
    
    const currentTime = currentHours * 60 + currentMinutes;
    const startTime = startHours * 60 + startMinutes;
    const endTime = endHours * 60 + endMinutes;
    
    return currentTime >= startTime && currentTime <= endTime;
}

// 更新卡片高亮状态
const updateCardHighlight = () => {
    cardList.value.forEach(card => {
        card.isHighlight = isTimeInRange(card.time);
    });
}

// 模拟从后台获取数据的函数
const fetchCardData = async () => {
	try {
		// 这里应该是实际的API调用
		// const response = await fetch('your-api-endpoint')
		// cardList.value = await response.json()
		
		// 暂时使用模拟数据
		console.log('已获取卡片数据')
	} catch (error) {
		console.error('获取数据失败:', error)
	}
}

// 显示对话框和输入框
const showDialogs = () => {
	showDialog.value = true;
	updateCardHighlight(); // 打开对话框时更新卡片高亮状态
};

// 隐藏对话框和输入框
const hideDialogs = () => {
	showDialog.value = false;
};

// 计算球的位置样式
const ballStyle = computed(() => {
	return {
		transform: `translate(${ballPosition.value.x}px, ${ballPosition.value.y}px)`
	}
});

// 输入框状态
const isVoiceMode = ref(false);
const inputText = ref('');

const toggleInputMode = () => {
	isVoiceMode.value = !isVoiceMode.value
	inputText.value = '' // 切换模式时清空输入
}

const startVoiceInput = () => {
	console.log('开始语音输入')
	// 这里添加语音输入的逻辑
}

const endVoiceInput = () => {
	console.log('结束语音输入')
	// 这里添加语音输入结束的逻辑
}

const sendMessage = () => {
	console.log('发送消息:', inputText.value)
	inputText.value = '' // 发送后清空输入
}

onMounted(() => {
	// 获取摇杆容器的位置和尺寸信息
	const query = uni.createSelectorQuery();
	query.select('#joystick').boundingClientRect(data => {
		joystickRect.value = data;
	}).exec();
	
	// 获取初始数据
	fetchCardData();
});

// 计算角度所在的区域（0-5）
const calculateSection = (angle) => {
	// 将角度转换为0-360范围
	let normalizedAngle = (angle + 360) % 360;
	// 每个区域60度，计算当前角度属于哪个区域
	let section = Math.floor(normalizedAngle / 60);
	// 调整区域映射，使其与UI布局一致（反转旋转方向）
	const sectionMap = [2, 1, 0, 5, 4, 3];
	return sectionMap[section];
};

// 触摸开始
const onTouchStart = (e) => {
	isDragging.value = true;
	updatePosition(e);
	
	// 设置定时器，0.3秒后显示内圈
	pressTimer = setTimeout(() => {
		showInnerCircle.value = true;
		// 0.5秒后显示文字
		textTimer = setTimeout(() => {
			showText.value = true;
		}, 500);
	}, 300);
};

// 触摸移动
const onTouchMove = (e) => {
	if (!isDragging.value) return;
	
	const prevSection = activeSection.value;
	updatePosition(e);
	console.log(activeSection.value)
	// 如果区域改变，重置文字显示定时器
	if (prevSection !== activeSection.value) {
		showText.value = false;
		console.log()
		if (textTimer) {
			clearTimeout(textTimer);
		}
		if (showInnerCircle.value) {
			textTimer = setTimeout(() => {
				showText.value = true;
			}, 500);
		}
	}
};

// 触摸结束
const onTouchEnd = () => {
	if (showInnerCircle.value) {
		handleMenuSelect();
	}
	showInnerCircle.value = false;
	showText.value = false;
	if (textTimer) {
		clearTimeout(textTimer);
	}
	ballPosition.value = { x: 0, y: 0 };
	activeSection.value = -1;
	
	// 清除定时器
	if (pressTimer) {
		clearTimeout(pressTimer);
		pressTimer = null;
	}
	if (textTimer) {
		clearTimeout(textTimer);
		textTimer = null;
	}
};

// 处理菜单选择
const handleMenuSelect = () => {
	if (activeSection.value !== -1) {
		const sectionIndex = (activeSection.value + 5) % 6;
		
		if (currentMenu.value === 'main') {
			// 主菜单选择逻辑
			const selectedIcon = icons[sectionIndex];
			if (subMenus[selectedIcon.text]) {
				currentMenu.value = selectedIcon.text;
				showInnerCircle.value = false;
				showText.value = false;
				if (textTimer) {
					clearTimeout(textTimer);
				}
				ballPosition.value = { x: 0, y: 0 };
				activeSection.value = -1;
				setTimeout(() => {
					showInnerCircle.value = true;
				}, 100);
			}
		} else if (currentSubMenu.value && thirdLevelMenus[currentSubMenu.value]) {
			// 三级菜单选择逻辑
			const selectedIcon = thirdLevelMenus[currentSubMenu.value][sectionIndex];
			if (selectedIcon) {
				if (selectedIcon.text === '返回上一层') {
					currentSubMenu.value = '';
					showInnerCircle.value = false;
					showText.value = false;
					if (textTimer) {
						clearTimeout(textTimer);
					}
					setTimeout(() => {
						showInnerCircle.value = true;
					}, 100);
				} else if (selectedIcon.text === '当日计划') {
					showDialogs();
				}
			}
		} else {
			// 二级菜单选择逻辑
			const selectedIcon = subMenus[currentMenu.value][sectionIndex];
			if (selectedIcon) {
				if (selectedIcon.text === '返回') {
					currentMenu.value = 'main';
					showInnerCircle.value = false;
					showText.value = false;
					if (textTimer) {
						clearTimeout(textTimer);
					}
					setTimeout(() => {
						showInnerCircle.value = true;
						setTimeout(() => {
							showInnerCircle.value = false;
						}, 500);
					}, 100);
				} else if (selectedIcon.text === '短期目标' && thirdLevelMenus[selectedIcon.text]) {
					currentSubMenu.value = selectedIcon.text;
					showInnerCircle.value = false;
					showText.value = false;
					if (textTimer) {
						clearTimeout(textTimer);
					}
					setTimeout(() => {
						showInnerCircle.value = true;
					}, 100);
				} else if (selectedIcon.text === '内容上传') {
					// 处理内容上传选项
					navigateToUpload();
				}
			}
		}
	}
};

// 添加uni-app的页面跳转方法
const navigateToUpload = () => {
	uni.navigateTo({
		url: '/pages/index/index'
	});
};

// 更新位置和活动区域
const updatePosition = (e) => {
	const centerX = joystickRect.value.width / 2;
	const centerY = joystickRect.value.height / 2;
	
	// 计算触摸点相对于中心的位置
	const touch = e.touches[0];
	let x = touch.clientX - joystickRect.value.left - centerX;
	let y = touch.clientY - joystickRect.value.top - centerY;
	
	// 计算距离和角度
	const distance = Math.sqrt(x * x + y * y);
	const maxDistance = 5; // 限制移动距离为5rpx
	
	// 限制移动范围
	if (distance > maxDistance) {
		const ratio = maxDistance / distance;
		x *= ratio;
		y *= ratio;
	}
	
	// 更新球的位置
	ballPosition.value = { x, y };
	
	// 只有当距离大于某个阈值时才更新区域
	if (distance > 2) {
		// 计算角度，使用-y因为屏幕坐标系y轴向下为正
		const angle = Math.atan2(-y, x) * (180 / Math.PI);
		activeSection.value = calculateSection(angle);
	} else {
		// 在中心位置时不激活任何区域
		activeSection.value = -1;
	}
};
</script>

<style scoped>
.container {
	position: fixed;
	left: 0;
	right: 0;
	top: 0;
	bottom: 0;
	display: flex;
	justify-content: center;
	align-items: center;
	background-color: #f5f5f5;
}

.joystick-wrapper {
	width: 650rpx;
	height: 650rpx;
	position: relative;
}

.joystick-container {
	position: absolute;
	width: 100%;
	height: 100%;
	left: 0;
	top: 0;
}

.outer-ring {
	position: absolute;
	width: 100%;
	height: 100%;
	border-radius: 50%;
	overflow: hidden;
	z-index: 1;
}

.inner-circle {
	position: absolute;
	width: 300rpx;
	height: 300rpx;
	left: 175rpx;
	top: 175rpx;
	border-radius: 50%;
	z-index: 2;
	background-color: #f5f5f5;
	opacity: 0;
	border: 2rpx solid transparent;
	transition: all 0.3s ease;
}

.inner-circle.show-circle {
	opacity: 1;
	border-color: #ccc;
}

.outer-section {
	position: absolute;
	width: 50%;
	height: 50%;
	transform-origin: 100% 100%;
}

.section-0 { transform: rotate(0deg) skew(30deg); }     /* 右上 */
.section-1 { transform: rotate(60deg) skew(30deg); }    /* 右下 */
.section-2 { transform: rotate(120deg) skew(30deg); }   /* 下 */
.section-3 { transform: rotate(180deg) skew(30deg); }   /* 左下 */
.section-4 { transform: rotate(240deg) skew(30deg); }   /* 左上 */
.section-5 { transform: rotate(300deg) skew(30deg); }   /* 上 */

.outer-section.active {
	background: radial-gradient(
		circle at 100% 100%,
		rgba(0, 255, 0, 0.6) 0%,
		rgba(0, 255, 0, 0.55) 20%,
		rgba(0, 255, 0, 0.22) 30%,
		rgba(0, 255, 0, 0.15) 40%,
		rgba(0, 255, 0, 0.12) 50%,
		rgba(0, 255, 0, 0.08) 60%,
		rgba(0, 255, 0, 0.03) 70%,
		rgba(0, 255, 0, 0.00) 80%,
		rgba(0, 255, 0, 0) 100%
	);
}

.inner-ball {
	position: absolute;
	width: 280rpx;
	height: 280rpx;
	background-color: #3498db;
	border-radius: 50%;
	left: 185rpx;
	top: 185rpx;
	transform: translate(0, 0);
	transition: transform 0.1s ease;
	z-index: 3;
}

.icons-container {
    position: absolute;
    width: 100%;
    height: 100%;
    left: 0;
    top: 0;
    z-index: 2;
}

.icon-wrapper {
    position: absolute;
    width: 60rpx;
    height: 60rpx;
    opacity: 0;
    transition: all 0.3s ease;
    display: flex;
    justify-content: center;
    align-items: center;
}

.icon-wrapper.show {
    opacity: 1;
}

.icon {
    width: 100%;
    height: 100%;
    transition: opacity 0.3s ease;
}

.icon-text {
    font-size: 28rpx;
    color: #333;
    white-space: nowrap;
    transition: opacity 0.3s ease;
    font-weight: bold;
}

/* 计算每个图标的位置 */
.icon-pos-0 {
    left: 295rpx;
    top: 60rpx;
}

.icon-pos-1 {
    right: 120rpx;
    top: 180rpx;
}

.icon-pos-2 {
    right: 120rpx;
    bottom: 180rpx;
}

.icon-pos-3 {
    left: 295rpx;
    bottom: 60rpx;
}

.icon-pos-4 {
    left: 120rpx;
    bottom: 180rpx;
}

.icon-pos-5 {
    left: 120rpx;
    top: 180rpx;
}

/* 蒙层样式 */
.overlay {
	position: fixed;
	top: 0;
	left: 0;
	width: 100%;
	height: 100%;
	background-color: rgba(0, 0, 0, 0.5);
	z-index: 100;
}

/* 对话框样式 */
.dialog {
	position: fixed;
	top: -100%;
	left: 5%;
	width: 90%;
	height: calc(100% - 300rpx);
	background-color: white;
	border-radius: 20rpx;
	z-index: 101;
	transition: transform 0.3s ease-in-out;
	display: flex;
	flex-direction: column;
	box-sizing: border-box;
	padding: 20rpx 0;
}

.dialog-show {
	transform: translateY(calc(400rpx + 100%));
}

.dialog-content {
	flex: 1;
	display: flex;
	flex-direction: column;
	background-color: #f5f5f5;
	border-radius: 20rpx;
	overflow: hidden;
	width: 100%;
	box-sizing: border-box;
}

.dialog-header {
	padding: 30rpx;
	background-color: white;
	font-size: 32rpx;
	font-weight: bold;
	box-shadow: 0 2rpx 10rpx rgba(0, 0, 0, 0.1);
}

.card-list {
	flex: 1;
	padding: 20rpx;
	overflow-y: auto;
	box-sizing: border-box;
	width: 100%;
}

.card {
	background-color: white;
	border-radius: 16rpx;
	padding: 20rpx;
	margin-bottom: 20rpx;
	box-shadow: 0 2rpx 10rpx rgba(0, 0, 0, 0.05);
	transition: transform 0.3s ease;
	box-sizing: border-box;
	width: 100%;
}

.card:active {
	transform: scale(0.98);
}

.card-highlight {
	border-left: 8rpx solid #007AFF;
}

.card-header {
	display: flex;
	justify-content: space-between;
	align-items: center;
	margin-bottom: 16rpx;
}

.card-title {
	font-size: 32rpx;
	font-weight: bold;
	color: #333;
}

.card-time {
	font-size: 24rpx;
	color: #666;
}

.card-body {
	margin-bottom: 16rpx;
}

.card-description {
	font-size: 28rpx;
	color: #666;
	line-height: 1.5;
}

.card-footer {
	display: flex;
	justify-content: space-between;
	align-items: center;
}

.card-status {
	padding: 4rpx 16rpx;
	border-radius: 100rpx;
	font-size: 24rpx;
}

.card-status.pending {
	background-color: #FFF3E0;
	color: #FF9800;
}

.card-status.inprogress {
	background-color: #E3F2FD;
	color: #2196F3;
}

.card-status.completed {
	background-color: #E8F5E9;
	color: #4CAF50;
}

.card-tags {
	display: flex;
	gap: 10rpx;
}

.card-tag {
	padding: 4rpx 16rpx;
	background-color: #F5F5F5;
	border-radius: 100rpx;
	font-size: 24rpx;
	color: #666;
}

.close-icon {
	position: absolute;
	top: 20rpx;
	right: 20rpx;
	width: 60rpx;
	height: 60rpx;
	z-index: 102;
	display: flex;
	align-items: center;
	justify-content: center;
	background-color: rgba(0, 0, 0, 0.1);
	border-radius: 50%;
}

.close-icon image {
	width: 40rpx;
	height: 40rpx;
}

/* 输入框样式 */
.input-box {
	position: fixed;
	bottom: -120rpx;
	left: 5%;
	width: 90%;
	height: 120rpx;
	background-color: #f5f5f5;
	border-radius: 20rpx;
	z-index: 102;
	transition: transform 0.3s ease-in-out;
	padding: 10rpx 20rpx;
	box-sizing: border-box;
}

.input-container {
	display: flex;
	align-items: center;
	height: 100%;
	background-color: white;
	border-radius: 10rpx;
	padding: 0 20rpx;
}

.input-icon-wrapper {
	width: 60rpx;
	height: 60rpx;
	perspective: 1000rpx;
}

.icon-flip-container {
	position: relative;
	width: 100%;
	height: 100%;
	transition: transform 0.6s;
	transform-style: preserve-3d;
}

.icon-flip-container.flip {
	transform: rotateX(180deg);
}

.icon-flipper {
	position: relative;
	width: 100%;
	height: 100%;
	transform-style: preserve-3d;
}

.input-icon {
	width: 48rpx;
	height: 48rpx;
	position: absolute;
	left: 50%;
	top: 50%;
	transform: translate(-50%, -50%);
	backface-visibility: hidden;
}

.input-icon.front {
	z-index: 2;
}

.input-icon.back {
	transform: translate(-50%, -50%) rotateX(180deg);
}

.input-content {
	flex: 1;
	position: relative;
	height: 80rpx;
	margin: 0 20rpx;
	overflow: hidden;
}

.text-input, .voice-button {
	position: absolute;
	width: 100%;
	height: 100%;
	transition: transform 0.3s ease-in-out;
}

.text-input {
	transform: translateY(0);
	display: flex;
	align-items: center;
}

.text-input input {
	width: 100%;
	height: 100%;
	font-size: 28rpx;
}

.text-input.slide-up {
	transform: translateY(-100%);
}

.voice-button {
	transform: translateY(100%);
	background-color: #f5f5f5;
	border-radius: 8rpx;
	display: flex;
	align-items: center;
	justify-content: center;
	font-size: 28rpx;
	color: #666;
}

.voice-button.slide-up-in {
	transform: translateY(0);
}

.voice-button:active {
	background-color: #e0e0e0;
}

.send-icon-wrapper {
	width: 60rpx;
	height: 60rpx;
	opacity: 0;
	transition: opacity 0.3s ease-in-out;
	display: flex;
	align-items: center;
	justify-content: center;
}

.send-icon {
	width: 48rpx;
	height: 48rpx;
}

.fade-in {
	opacity: 1;
}

.input-show {
	transform: translateY(-140rpx);
}
</style>
