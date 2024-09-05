<template>
	<view style="position: relative; display: flex; justify-content: center">
		<canvas class="newCanvasClass" :style="defineStyle" canvas-id="newCanvas" id="newCanvas"></canvas>
		<image :src="bgImage" :style="defineStyle" v-if="bgImage"></image>
		<image src="@/static/R-C.png" :style="defineStyle" v-else></image>
		<canvas
			class="defCanvasClass"
			:style="defineStyle"
			canvas-id="defCanvas"
			id="defCanvas"
			@touchstart="penStart"
			@touchmove="penMove"
			@touchend="penEnd"
		></canvas>
		<view :class="['canvasHuabi']"></view>
	</view>
</template>

<script>
// 绘制点集合
let drawFigures = [];
// 保存操作的像素数据
let imageData = [];
// 总绘制点的集合
let totalDrawFigures = [];

export default {
	data() {
		return {
			curStep: -1, // 当前步数
			myCanvasContext: null,
			// 画布背景图片
			bgImage: null,
			topLeftPoint: {
				// 画布左上角点
				left: 0,
				top: 0,
			},
			// 尺寸
			size: {
				width: 375,
				height: 450,
			},
			// 原图片尺寸
			orgSize: {
				width: 375,
				height: 450,
			},
			startX: 0, // 前一个绘制节点X轴
			startY: 0, // 前一个绘制节点Y轴
			canMaskImg: null,
			blockCanvas: null,
		};
	},
	props: {
		// 画笔颜色
		lineColor: {
			type: String,
			default: 'rgba(8, 0, 255, 0.1)',
		},
		// 画笔宽度
		lineWidth: {
			type: Number,
			default: 15,
		},
		containerHeight: {
			type: Number,
			default: 450,
		},
		containerWidth: {
			type: Number,
			default: uni.getSystemInfoSync().screenWidth,
		},
	},

	watch: {
		bgImage() {
			this.handleCanvasImg();
		},
	},
	computed: {
		defineStyle() {
			return `width:${this.size.width}px;height:${this.size.height}px;`;
		},
	},
	methods: {
		handleInit() {},
		handleCanvasImg() {
			this.myCanvasContext = uni.createCanvasContext('defCanvas', this);
			// 绘制图片
			this.myCanvasContext.drawImage(this.bgImage, 0, 0, parseInt(this.size.width), parseInt(this.size.height));
			let pattern = this.myCanvasContext.createPattern(this.bgImage, 'no-repeat');
			this.myCanvasContext.fillStyle = pattern;
			this.myCanvasContext.draw();

			// 让线条圆润
			this.myCanvasContext.setLineCap('round');
			this.myCanvasContext.strokeStyle = this.lineColor;
			this.myCanvasContext.setFillStyle(this.lineColor);
			this.myCanvasContext.setLineWidth(this.lineWidth);
		},
		beforePenStart(fn, e) {
			const _this = this;
			if (!this.bgImage) {
				this.handleQuShuiYing(fn);
			} else {
				fn(e);
			}
		},
		// penStart
		penStart(e) {
			const _this = this;
			this.beforePenStart((e) => {
				// #ifdef MP-WEIXIN
				const x = e.touches[0].pageX - this.topLeftPoint.left;
				const y = e.touches[0].pageY - this.topLeftPoint.top;
				// #endif
				// #ifndef MP-WEIXIN
				const x = e.touches[0].x;
				const y = e.touches[0].y;
				// #endif
				_this.myCanvasContext && _this.myCanvasContext.beginPath();
				_this.startX = x;
				_this.startY = y;

				_this.drawCurve(x, y);
				drawFigures = drawFigures.filter((figure, index) => index <= _this.curStep);
				drawFigures.push({
					lineColor: _this.lineColor,
					lineWidth: _this.lineWidth,
					points: [
						{
							x,
							y,
						},
					],
				});
			}, e);
		},
		/**
		 * 滑动事件
		 * @param {Object} e
		 */
		penMove(e) {
			if (!this.bgImage) return;
			if (this.intervalId) {
				clearInterval(this.intervalId);
				this.intervalId = null;
			}
			console.log(e, '===>e');
			this.intervalId = setInterval((_) => (this.canDraw = true), 100);
			// #ifdef MP-WEIXIN
			// const x = e.touches[0].pageX - this.topLeftPoint.left;
			// const y = e.touches[0].pageY - this.topLeftPoint.top;
			const x = e.mp.touches[0].x;
			const y = e.mp.touches[0].y;
			// #endif
			// #ifndef MP-WEIXIN
			const x = e.touches[0].x;
			const y = e.touches[0].y;
			// #endif
			console.log(x, y);
			this.addToPoints(x, y);
			this.drawCurve(x, y);
			this.myCanvasContext.draw(true);
		},
		// 重新上传图片
		resetImg() {
			let _this = this;
			uni.chooseImage({
				sizeType: ['compressed', 'original'],
				count: 1,
				success: (img) => {
					uni.getImageInfo({
						src: img.tempFilePaths[0],
						success: (res) => {
							const containerWidth = uni.getSystemInfoSync().screenWidth;
							const containerHeight = 450;
							const imageWidth = res.width;
							const imageHeight = res.height;
							const aspectRatio = imageWidth / imageHeight;
							let displayWidth, displayHeight;

							if (containerWidth / containerHeight > aspectRatio) {
								displayHeight = containerHeight;
								displayWidth = containerHeight * aspectRatio;
							} else {
								displayWidth = containerWidth;
								displayHeight = containerWidth / aspectRatio;
							}
							this.bgImage = res.path;
							this.size = {
								width: displayWidth,
								height: displayHeight,
							};
							this.orgSize = {
								width: imageWidth,
								height: imageHeight,
							};
							console.log(this.size, this.orgSize, '112');
						},
					});
				},
			});
		},
		/**
		 * 添加到点数组
		 * @param {Object} x
		 * @param {Object} y
		 */
		addToPoints(x, y) {
			if (!this.bgImage) return;
			const drawFigure = drawFigures[this.curStep + 1];
			drawFigure.points.push({
				x,
				y,
			});
			if (drawFigure.points.length == 1) {
				drawFigure.points.push({
					x,
					y,
				});
			} else {
				drawFigure.points[1] = {
					x,
					y,
				};
			}
			return drawFigures;
		},
		handleModifySize(res) {
			const containerWidth = uni.getSystemInfoSync().screenWidth;
			const containerHeight = 450;
			const imageWidth = res.width;
			const imageHeight = res.height;
			const aspectRatio = imageWidth / imageHeight;
			let displayWidth, displayHeight;

			if (containerWidth / containerHeight > aspectRatio) {
				displayHeight = containerHeight;
				displayWidth = containerHeight * aspectRatio;
			} else {
				displayWidth = containerWidth;
				displayHeight = containerWidth / aspectRatio;
			}
			return {
				displayWidth,
				displayHeight,
				imageWidth,
				imageHeight,
			};
		},
		handleQuShuiYing(fn) {
			let _this = this;
			uni.chooseImage({
				sizeType: ['compressed', 'original'],
				count: 1,
				success: (img) => {
					uni.getImageInfo({
						src: img.tempFilePaths[0],
						success: (res) => {
							const { displayWidth, displayHeight, imageWidth, imageHeight } = this.handleModifySize(res);
							this.bgImage = res.path;
							this.size = {
								width: displayWidth,
								height: displayHeight,
							};
							this.orgSize = {
								width: imageWidth,
								height: imageHeight,
							};
							this.$nextTick(() => {
								fn();
							});
						},
					});
				},
			});
		},
		// 画出
		drawCurve(x, y) {
			this.myCanvasContext && this.myCanvasContext.beginPath();
			this.myCanvasContext.moveTo(this.startX, this.startY);
			this.myCanvasContext.strokeStyle = this.lineColor;
			this.myCanvasContext.lineTo(x, y);
			this.myCanvasContext.stroke();
			this.startX = x;
			this.startY = y;
		},
		/**
		 * 离开屏幕事件
		 * @param {Object} e
		 */
		penEnd(e) {
			// #ifdef MP-WEIXIN
			const x = e.changedTouches[0].pageX - this.topLeftPoint.left;
			const y = e.changedTouches[0].pageY - this.topLeftPoint.top;
			// #endif
			// #ifndef MP-WEIXIN
			const x = e.changedTouches[0].x;
			const y = e.changedTouches[0].y;
			// #endif
			const totalPoints = this.addToPoints(x, y);
			totalDrawFigures.push(...totalDrawFigures, ...totalPoints);
			this.drawCurve(x, y);
			this.handleTotalPoints(totalDrawFigures);
		},
		// 处理totalPoints
		handleTotalPoints(regions) {
			this.blockCanvas = uni.createCanvasContext('newCanvas', this);
			// 设置画布为黑色背景
			this.blockCanvas.setFillStyle('black');
			this.blockCanvas.fillRect(0, 0, this.size.width, this.size.height);
			this.blockCanvas.setStrokeStyle('white');
			this.blockCanvas.setLineJoin('round'); // 使线条连接处更平滑
			this.blockCanvas.setLineCap('round'); // 使线条末端更平滑

			// 遍历每段路径并绘制
			regions.forEach((region) => {
				this.blockCanvas.setLineWidth(region.lineWidth);
				const points = region.points;
				this.blockCanvas.beginPath();
				this.blockCanvas.moveTo(points[0].x, points[0].y);
				for (let i = 1; i < points.length; i++) {
					this.blockCanvas.lineTo(points[i].x, points[i].y);
				}
				this.blockCanvas.stroke();
			});
			this.blockCanvas.draw(
				false,
				() => {
					uni.canvasToTempFilePath(
						{
							canvasId: 'newCanvas',
							width: this.size.width,
							height: this.size.height,
							fileType: 'png',
							destWidth: this.orgSize.width, // 生成图片的实际宽度，可以放大或缩小
							destHeight: this.orgSize.height, // 生成图片的实际高度
							success: (res) => {
								// 将 imageData 发送给后端或返回
								console.log(res, '===>res');
								this.canMaskImg = res.tempFilePath;
							},
							fail: function (err) {
								console.log(err);
							},
						},
						this
					);
				},
				this
			);
		},
		// 重置操作
		redo() {
			this.myCanvasContext.clearRect(0, 0, parseInt(this.size.width * 2) / 2, parseInt(this.size.height * 2) / 2);
			drawFigures = [];
			imageData = [];
			totalDrawFigures = [];
			this.curStep = -1;
			this.myCanvasContext.drawImage(this.bgImage, 0, 0, parseInt(this.width * 2) / 2, parseInt(this.height * 2) / 2);
			this.myCanvasContext.draw(true);
			this.handleTotalPoints(drawFigures);
			this.blockCanvas.setFillStyle('black');
			this.myCanvasContext.draw(true);
		},
		/**
		 * 保存画布内容
		 */
		saveCanvas() {
			const _that = this;
			return new Promise((resolve, reject) => {
				// 定时器线程中执行绘制，否则因绘制占用主线程导致加载动画不显示
				resolve({
					maskImg: this.canMaskImg,
					orgImg: this.bgImage,
				});
			});
		},
	},
};
</script>

<style lang="scss" scoped>
.newCanvasClass {
	position: absolute;
	left: 50%;
	top: 50%;
	transform: translate(-50%, -50%);
	visibility: hidden;
}
.defCanvasClass {
	position: absolute;
	left: 50%;
	top: 50%;
	transform: translate(-50%, -50%);
}
</style>
