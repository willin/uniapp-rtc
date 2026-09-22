<template>
	<view class="report-page">
		<!-- 顶部摘要：单卡片总览 -->
		<view class="summary-hero">
			<view class="summary-hero-label">总花费</view>
			<view class="summary-hero-amount">¥{{ formatMoney(totalCost) }}</view>
			<view class="summary-hero-divider"></view>
			<view class="summary-hero-stats">
				<view class="stat-item">
					<view class="stat-value">¥{{ formatMoney3(costPerKm) }}</view>
					<view class="stat-label">每公里</view>
				</view>
				<view class="stat-divider"></view>
				<view class="stat-item">
					<view class="stat-value">¥{{ formatMoney(perPerson) }}</view>
					<view class="stat-label">{{ people }}人人均</view>
				</view>
				<view class="stat-divider"></view>
				<view class="stat-item">
					<view class="stat-value">{{ formatNumber(totalMileage) }}km</view>
					<view class="stat-label">总里程</view>
				</view>
			</view>
		</view>

		<!-- #ifndef MP-WEIXIN || MP-QQ || MP-TOUTIAO -->
		<view class="chart-card chart-card-full">
			<view class="chart-title">
				<view class="chart-title-bar"></view>
				<text>费用占比</text>
			</view>
			<view class="chart-box chart-box-tall">
				<qiun-data-charts
					type="rose"
					:chartData="pieData"
					:opts="pieOpts"
					:reshow="reshow"
				/>
			</view>
		</view>
		<!-- #endif -->
		<!-- #ifdef MP-WEIXIN || MP-QQ || MP-TOUTIAO -->
		<!-- 小程序端：弹框打开时隐藏图表，避免 canvas 原生组件层级穿透盖住弹框 -->
		<view v-if="!showPosterModal" class="chart-card chart-card-full">
			<view class="chart-title">
				<view class="chart-title-bar"></view>
				<text>费用占比</text>
			</view>
			<view class="chart-box chart-box-tall">
				<qiun-data-charts
					type="rose"
					:chartData="pieData"
					:opts="pieOpts"
					:reshow="reshow"
				/>
			</view>
		</view>
		<!-- #endif -->

		<!-- 颜色图例 -->
		<view class="legend-row">
			<view
				v-for="item in visibleCategories"
				:key="item.key"
				class="legend-item"
			>
				<view class="legend-dot" :style="{ backgroundColor: colorOf(item.key) }"></view>
				<text class="legend-text">{{ item.name }}</text>
			</view>
		</view>

		<!-- 费用明细 -->
		<view class="section-card">
			<view class="section-head">
				<view class="section-icon">
					<uni-icons type="list" size="20" color="#10b981" />
				</view>
				<text class="section-title">费用明细</text>
			</view>
			<view class="section-body">
				<view
					v-for="item in visibleCategories"
					:key="item.key"
					class="detail-item"
				>
					<view class="detail-row">
						<view class="detail-dot" :style="{ backgroundColor: colorOf(item.key) }"></view>
						<text class="detail-name">{{ item.name }}</text>
						<text class="detail-percent">{{ percentOf(item.value) }}%</text>
						<text class="detail-amount">¥{{ formatMoney(item.value) }}</text>
					</view>
					<view class="detail-bar-track">
						<view
							class="detail-bar-fill"
							:style="{
								width: percentOf(item.value) + '%',
								backgroundColor: colorOf(item.key)
							}"
						></view>
					</view>
				</view>
			</view>
		</view>

		<!-- 额外支出明细 -->
		<view v-if="extraItems.length" class="section-card">
			<view class="section-head">
				<view class="section-icon">
					<uni-icons type="wallet" size="20" color="#10b981" />
				</view>
				<text class="section-title">额外支出明细</text>
			</view>
			<view class="section-body">
				<view
					v-for="(item, i) in extraItems"
					:key="i"
					class="extra-row"
				>
					<text class="extra-name">{{ item.name }}</text>
					<view class="extra-tag" :class="{ per: item.isPerPerson }">
						{{ item.isPerPerson ? '人均' : '总计' }}
					</view>
					<text class="extra-amount">¥{{ formatMoney(item.total) }}</text>
				</view>
			</view>
		</view>

		<!-- 行程小结 -->
		<view class="trip-meta">
			<text class="trip-meta-text">
				{{ vehicleType }} · {{ days }} 天 · {{ people }} 人 · {{ formatNumber(totalMileage) }} 公里
			</text>
		</view>

		<!-- 分享按钮（仅小程序端） -->
		<view class="share-btn-row">
			<button class="share-btn share-btn-img" @tap="generatePoster">
				<uni-icons type="image" size="18" color="#fff" />
				<text>图片分享</text>
			</button>
			<!-- #ifdef MP-WEIXIN || MP-QQ || MP-TOUTIAO -->
			<button class="share-btn" open-type="share">
				<uni-icons type="redo" size="18" color="#fff" />
				<text>报告分享</text>
			</button>
			<!-- #endif -->
		</view>

		<!-- 海报生成 canvas（离屏） -->
		<canvas
			canvas-id="posterCanvas"
			id="posterCanvas"
			class="poster-canvas"
			:style="{ width: posterWidth + 'px', height: posterHeight + 'px' }"
		></canvas>

		<!-- 海报预览弹窗 -->
		<view v-if="showPosterModal" class="poster-mask" @tap="closePosterModal">
			<view class="poster-modal" @tap.stop>
				<image v-if="posterImage" class="poster-preview" :src="posterImage" mode="widthFix" />
				<view v-else class="poster-loading">
					<uni-icons type="spinner-cycle" size="32" color="#10b981" />
					<text class="poster-loading-text">生成中...</text>
				</view>
				<view class="poster-actions">
					<button class="poster-save-btn" @tap="savePoster">
						<uni-icons type="download" size="16" color="#fff" />
						<text>保存到相册</text>
					</button>
					<button class="poster-close-btn" @tap="closePosterModal">关闭</button>
				</view>
			</view>
		</view>
	</view>
</template>

<script>
	// 5 类别固定配色（在所有图表中保持一致）
	const COLOR_MAP = {
		driving: '#3b82f6', // 行驶成本 - 蓝
		highway:  '#f59e0b', // 高速费   - 琥珀
		hotel:    '#10b981', // 住宿费   - 翡翠
		meal:     '#ec4899', // 餐饮费   - 粉
		extra:    '#8b5cf6'  // 额外支出 - 紫
	}
	const COLOR_LIST = ['driving', 'highway', 'hotel', 'meal', 'extra'].map(k => COLOR_MAP[k])

	export default {
		data() {
			return {
				reshow: false,
				totalMileage: 0,
				people: 1,
				days: 0,
				vehicleType: '',
				totalCost: 0,
				perPerson: 0,
				categories: [],
				extraItems: [],
				// 海报相关
				posterWidth: 600,
				posterHeight: 1000,
				showPosterModal: false,
				posterImage: '',
				posterGenerating: false
			}
		},
		computed: {
			// 每公里行驶成本（仅行驶成本 ÷ 总里程，不含住宿餐饮等）
			costPerKm() {
				if (this.totalMileage <= 0) return 0
				const driving = this.categories.find(c => c.key === 'driving')
				const drivingCost = driving ? Number(driving.value) : 0
				return drivingCost / this.totalMileage
			},
			// 过滤掉金额为 0 的分类（如未走高速时隐藏高速费）
			visibleCategories() {
				return this.categories.filter(c => c.value > 0)
			},
			// 玫瑰图/环形图数据
			pieData() {
				return {
					series: this.visibleCategories
						.map(c => ({
							name: c.name,
							data: Number(c.value.toFixed(2)),
							color: COLOR_MAP[c.key]
						}))
				}
			},
			pieOpts() {
				return {
					color: COLOR_LIST,
					padding: [10, 10, 10, 10],
					animation: true,
					duration: 600,
					legend: { show: false },
					extra: {
						rose: {
							type: 'radius',
							// linearType: 'custom',
							labelColor: '#6b7280'
						}
					},
					subtitle: {
						name: '总计',
						color: '#9ca3af',
						fontSize: 12
					},
					title: {
						name: '¥' + this.formatMoney(this.totalCost),
						fontSize: 14,
						color: '#1f2937'
					}
				}
			},
		},
		onLoad(options) {
			// 优先从 globalData 读取（站内跳转）
			const app = getApp()
			const gd = app && app.globalData && app.globalData.reportData
			if (gd) {
				this.applyData(gd)
				app.globalData.reportData = null
				console.log('[report] data from globalData')
				return
			}
			// 其次从分享链接参数恢复（其他用户点开分享卡片）
			if (options && options.d) {
				try {
					const decoded = this.decodeShareData(options.d)
					this.applyData(decoded)
					console.log('[report] data from share link')
				} catch (e) {
					console.error('[report] decode share data failed:', e)
				}
			}
		},
		onReady() {
			// 布局完成后触发一次图表重绘，确保容器尺寸正确
			// 只设为 true 且不切换回 false，避免反复重绘和动画重放
			this.reshow = true;
		},
		// #ifdef MP-WEIXIN || MP-QQ || MP-TOUTIAO
		onShareAppMessage() {
			const encoded = this.encodeShareData()
			return {
				title: '自驾游预算报告：总花费 ¥' + this.formatMoney(this.totalCost),
				path: '/pages/report/report?d=' + encoded,
				query: 'd=' + encoded
			}
		},
		onShareTimeline() {
			const encoded = this.encodeShareData()
			return {
				title: '自驾游预算报告：总花费 ¥' + this.formatMoney(this.totalCost),
				query: 'd=' + encoded
			}
		},
		// #endif
		methods: {
			applyData(data) {
				// 批量赋值，减少多次响应式触发
				Object.assign(this, {
					totalMileage: data.totalMileage || 0,
					people: data.people || 1,
					days: data.days || 0,
					vehicleType: data.vehicleType || '',
					totalCost: data.totalCost || 0,
					perPerson: data.perPerson || 0,
					categories: (data.categories || []).map(c => ({
						key: c.key,
						name: c.name,
						value: Number(c.value) || 0
					})),
					extraItems: data.extraItems || []
				})
			},
			// 将当前页面数据编码为紧凑 URL 参数
			encodeShareData() {
				const compact = {
					t: this.totalMileage,
					p: this.people,
					d: this.days,
					v: this.vehicleType,
					c: this.categories.map(c => Math.round(c.value * 100) / 100),
					e: this.extraItems.map(item => ({
						n: item.name,
						a: Math.round((item.amount || 0) * 100) / 100,
						p: item.isPerPerson ? 1 : 0,
						t: Math.round((item.total || 0) * 100) / 100
					}))
				}
				return encodeURIComponent(JSON.stringify(compact))
			},
			// 从 URL 参数解码为完整报告数据
			decodeShareData(str) {
				const compact = JSON.parse(decodeURIComponent(str))
				const categoryKeys = ['driving', 'highway', 'hotel', 'meal', 'extra']
				const categoryNames = ['行驶成本', '高速费', '住宿费', '餐饮费', '额外支出']
				const cats = compact.c || []
				const totalCost = cats.reduce((s, v) => s + (Number(v) || 0), 0)
				const people = Math.max(1, compact.p || 1)
				return {
					totalMileage: compact.t || 0,
					people,
					days: compact.d || 0,
					vehicleType: compact.v || '',
					totalCost,
					perPerson: totalCost / people,
					categories: categoryKeys.map((k, i) => ({
						key: k,
						name: categoryNames[i],
						value: Number(cats[i]) || 0
					})),
					extraItems: (compact.e || []).map(item => ({
						name: item.n || '',
						amount: Number(item.a) || 0,
						isPerPerson: !!item.p,
						total: Number(item.t) || 0
					}))
				}
			},
			colorOf(key) {
				return COLOR_MAP[key] || '#9ca3af'
			},
			percentOf(value) {
				if (this.totalCost <= 0) return '0.0'
				return ((value / this.totalCost) * 100).toFixed(1)
			},
			formatMoney(v) {
				const n = Number(v) || 0
				return n.toLocaleString('zh-CN', {
					minimumFractionDigits: 2,
					maximumFractionDigits: 2
				})
			},
			formatNumber(v) {
				const n = Number(v) || 0
				return n.toLocaleString('zh-CN')
			},
			// 金额保留 3 位小数（用于每公里花费）
			formatMoney3(v) {
				const n = Number(v) || 0
				return n.toLocaleString('zh-CN', {
					minimumFractionDigits: 3,
					maximumFractionDigits: 3
				})
			},
			// 格式化日期 YYYY-MM-DD
			formatDate(date) {
				const y = date.getFullYear()
				const m = String(date.getMonth() + 1).padStart(2, '0')
				const d = String(date.getDate()).padStart(2, '0')
				return y + '-' + m + '-' + d
			},
			// 关闭海报弹窗
			closePosterModal() {
				this.showPosterModal = false
				this.posterImage = ''
			},
			// 生成海报图片
			generatePoster() {
				if (this.posterGenerating) return
				this.posterGenerating = true
				this.showPosterModal = true
				this.posterImage = ''

				const W = this.posterWidth
				const visible = this.visibleCategories
				const itemCount = visible.length
				const extraCount = this.extraItems.length
				// 计算海报总高度（与 drawPoster 实际绘制逐段对齐）
				const padTop = 40
				const headerH = 70
				const gap = 16
				const summaryH = 200
				const pieH = 280
				const breakdownH = 50 + itemCount * 62
				const extraH = extraCount > 0 ? 50 + extraCount * 52 : 0
				const footerH = 180
				const padBottom = 30
				const H = padTop
					+ headerH + gap
					+ summaryH + gap
					+ pieH + gap
					+ breakdownH + gap
					+ (extraCount > 0 ? extraH + gap : 0)
					+ footerH + padBottom
				this.posterHeight = H

				// 直接绘制（暂不绘制小程序码图片，排查 drawImage 导致的卡死问题）
				this.$nextTick(() => {
					this.drawPoster(W, H, '')
				})
			},
			// 绘制海报
			drawPoster(W, H, qrPath) {
				try {
				const ctx = uni.createCanvasContext('posterCanvas', this)
				const visible = this.visibleCategories
				const extraItems = this.extraItems || []

				// 背景
				ctx.setFillStyle('#f5f7fa')
				ctx.fillRect(0, 0, W, H)

				let y = 40

				// ===== 顶部标题卡片 =====
				const headerX = 24
				const headerY = y
				const headerW = W - 48
				const headerH = 70
				ctx.setFillStyle('#ffffff')
				this.roundRect(ctx, headerX, headerY, headerW, headerH, 14)
				ctx.fill()

				ctx.setFillStyle('#1f2937')
				ctx.setFontSize(20)
				ctx.setTextBaseline('middle')
				ctx.fillText('自驾游开销报告', headerX + 20, headerY + headerH / 2)

				const dateStr = this.formatDate(new Date())
				ctx.setFillStyle('#9ca3af')
				ctx.setFontSize(13)
				const dateW = ctx.measureText(dateStr).width
				ctx.fillText(dateStr, headerX + headerW - 20 - dateW, headerY + headerH / 2)
				y += headerH + 16

				// ===== 摘要卡片 =====
				const summaryX = 24
				const summaryY = y
				const summaryW = W - 48
				const summaryH = 200
				const grd = ctx.createLinearGradient(summaryX, summaryY, summaryX, summaryY + summaryH)
				grd.addColorStop(0, '#10b981')
				grd.addColorStop(1, '#059669')
				ctx.setFillStyle(grd)
				this.roundRect(ctx, summaryX, summaryY, summaryW, summaryH, 16)
				ctx.fill()

				// 总花费标签
				ctx.setFillStyle('rgba(255,255,255,0.85)')
				ctx.setFontSize(15)
				ctx.setTextAlign('center')
				ctx.fillText('总花费', W / 2, summaryY + 32)

				// 总花费金额
				ctx.setFillStyle('#ffffff')
				ctx.setFontSize(42)
				ctx.fillText('¥' + this.formatMoney(this.totalCost), W / 2, summaryY + 80)

				// 分割线
				ctx.setStrokeStyle('rgba(255,255,255,0.25)')
				ctx.setLineWidth(1)
				ctx.beginPath()
				ctx.moveTo(summaryX + 20, summaryY + 120)
				ctx.lineTo(summaryX + summaryW - 20, summaryY + 120)
				ctx.stroke()

				// 三列指标
				const stats = [
					{ v: '¥' + this.formatMoney3(this.costPerKm), l: '每公里' },
					{ v: '¥' + this.formatMoney(this.perPerson), l: this.people + '人人均' },
					{ v: this.formatNumber(this.totalMileage) + 'km', l: '总里程' }
				]
				const statY1 = summaryY + 150
				const statY2 = summaryY + 175
				stats.forEach((s, i) => {
					const cx = summaryX + (summaryW / 4) * (i + 1)
					ctx.setFillStyle('#ffffff')
					ctx.setFontSize(17)
					ctx.fillText(s.v, cx, statY1)
					ctx.setFillStyle('rgba(255,255,255,0.8)')
					ctx.setFontSize(11)
					ctx.fillText(s.l, cx, statY2)
				})
				ctx.setTextAlign('left')
				y += summaryH + 16

				// ===== 费用占比（玫瑰图：半径随数值变化） =====
				// 卡片背景（含标题 + 玫瑰图 + 图例）
				const pieCardX = 24
				const pieCardY = y
				const pieCardW = W - 48
				const pieCardH = 280
				ctx.setFillStyle('#ffffff')
				this.roundRect(ctx, pieCardX, pieCardY, pieCardW, pieCardH, 14)
				ctx.fill()
				// 标题（与费用明细卡片标题格式统一）
				ctx.setFillStyle('#1f2937')
				ctx.setFontSize(16)
				ctx.fillText('费用占比', pieCardX + 20, pieCardY + 28)
				const pieCenterX = W / 2
				const pieCenterY = pieCardY + 130
				const maxR = 80
				const minR = 28
				// 找出最大值用于归一化半径
				const maxValue = visible.reduce((m, c) => Math.max(m, c.value), 0) || 1
				let startAngle = -Math.PI / 2
				// 先绘制所有扇区，再统一绘制外部标签（避免覆盖）
				visible.forEach(c => {
					const ratio = this.totalCost > 0 ? c.value / this.totalCost : 0
					const endAngle = startAngle + ratio * Math.PI * 2
					const r = minR + (c.value / maxValue) * (maxR - minR)
					ctx.beginPath()
					ctx.moveTo(pieCenterX, pieCenterY)
					ctx.arc(pieCenterX, pieCenterY, r, startAngle, endAngle)
					ctx.closePath()
					ctx.setFillStyle(COLOR_MAP[c.key])
					ctx.fill()
					// 扇区之间加白边分隔
					ctx.setStrokeStyle('#f5f7fa')
					ctx.setLineWidth(2)
					ctx.stroke()
					startAngle = endAngle
				})
				// 外部标签：点 + 引导线 + 百分比文字（与页面 ucharts 风格一致）
				startAngle = -Math.PI / 2
				visible.forEach(c => {
					const ratio = this.totalCost > 0 ? c.value / this.totalCost : 0
					const endAngle = startAngle + ratio * Math.PI * 2
					const r = minR + (c.value / maxValue) * (maxR - minR)
					const midAngle = (startAngle + endAngle) / 2
					// 扇区边缘点
					const dotX = pieCenterX + Math.cos(midAngle) * r
					const dotY = pieCenterY + Math.sin(midAngle) * r
					// 引导线终点（向外延伸一段）
					const lineEndR = maxR + 18
					const lineX = pieCenterX + Math.cos(midAngle) * lineEndR
					const lineY = pieCenterY + Math.sin(midAngle) * lineEndR
					// 文字位置：右侧标签左对齐，右侧右对齐
					const isRight = Math.cos(midAngle) >= 0
					const textX = isRight ? lineX + 6 : lineX - 6
					const textY = lineY
					const pct = this.percentOf(c.value) + '%'
					// 引导线
					ctx.setStrokeStyle(COLOR_MAP[c.key])
					ctx.setLineWidth(1)
					ctx.beginPath()
					ctx.moveTo(dotX, dotY)
					ctx.lineTo(lineX, lineY)
					ctx.stroke()
					// 末端圆点
					ctx.beginPath()
					ctx.arc(lineX, lineY, 2, 0, Math.PI * 2)
					ctx.setFillStyle(COLOR_MAP[c.key])
					ctx.fill()
					// 百分比文字
					ctx.setFillStyle('#374151')
					ctx.setFontSize(11)
					ctx.setTextAlign(isRight ? 'left' : 'right')
					ctx.fillText(pct, textX, textY + 3)
					ctx.setTextAlign('left')
					startAngle = endAngle
				})

				// ===== 图例（居中显示，放在卡片内底部） =====
				ctx.setFontSize(11)
				let legendTotalW = 0
				const legendItems = visible.map(c => {
					const nameW = ctx.measureText(c.name).width
					const itemW = 10 + 4 + nameW
					legendTotalW += itemW
					return { c, nameW, itemW }
				})
				legendTotalW += (visible.length - 1) * 16
				const legendY = pieCardY + pieCardH - 25
				let lx = (W - legendTotalW) / 2
				legendItems.forEach(({ c, nameW }) => {
					ctx.setFillStyle(COLOR_MAP[c.key])
					ctx.fillRect(lx, legendY - 5, 10, 10)
					ctx.setFillStyle('#6b7280')
					ctx.fillText(c.name, lx + 14, legendY + 2)
					lx += 10 + 4 + nameW + 16
				})
				y += pieCardH + 16

				// ===== 费用明细 =====
				ctx.setFillStyle('#ffffff')
				const bdX = 24
				const bdY = y
				const bdW = W - 48
				const bdH = 50 + visible.length * 62
				this.roundRect(ctx, bdX, bdY, bdW, bdH, 14)
				ctx.fill()

				ctx.setFillStyle('#1f2937')
				ctx.setFontSize(16)
				ctx.fillText('费用明细', bdX + 20, bdY + 28)

				visible.forEach((c, i) => {
					const rowY = bdY + 52 + i * 62
					// 色点
					ctx.setFillStyle(COLOR_MAP[c.key])
					ctx.fillRect(bdX + 20, rowY - 6, 10, 10)
					// 名称
					ctx.setFillStyle('#1f2937')
					ctx.setFontSize(14)
					ctx.fillText(c.name, bdX + 40, rowY)
					// 百分比
					const pct = this.percentOf(c.value) + '%'
					ctx.setFillStyle('#9ca3af')
					ctx.setFontSize(12)
					const pctW = ctx.measureText(pct).width
					ctx.fillText(pct, bdX + bdW - 140 - pctW, rowY)
					// 金额
					const amt = '¥' + this.formatMoney(c.value)
					ctx.setFillStyle('#1f2937')
					ctx.setFontSize(15)
					ctx.setTextAlign('right')
					ctx.fillText(amt, bdX + bdW - 20, rowY)
					ctx.setTextAlign('left')
					// 进度条背景
					ctx.setFillStyle('#f3f4f6')
					this.roundRect(ctx, bdX + 40, rowY + 14, bdW - 60, 8, 4)
					ctx.fill()
					// 进度条填充
					ctx.setFillStyle(COLOR_MAP[c.key])
					const fillW = (bdW - 60) * (c.value / (this.totalCost || 1))
					this.roundRect(ctx, bdX + 40, rowY + 14, fillW, 8, 4)
					ctx.fill()
				})
				y += bdH + 16

				// ===== 额外支出明细（仅当有额外支出时） =====
				if (extraItems && extraItems.length > 0) {
					ctx.setFillStyle('#ffffff')
					const exX = 24
					const exY = y
					const exW = W - 48
					const exH = 50 + extraItems.length * 52
					this.roundRect(ctx, exX, exY, exW, exH, 14)
					ctx.fill()

					ctx.setFillStyle('#1f2937')
					ctx.setFontSize(16)
					ctx.fillText('额外支出明细', exX + 20, exY + 28)

					extraItems.forEach((item, i) => {
						const rowY = exY + 50 + i * 52
						// 项目名称
						ctx.setFillStyle('#1f2937')
						ctx.setFontSize(14)
						ctx.fillText(item.name || '未命名', exX + 20, rowY)
						// 人均/总计标签
						const tag = item.isPerPerson ? '人均' : '总计'
						const tagW = ctx.measureText(tag).width
						ctx.setFillStyle(item.isPerPerson ? '#d1fae5' : '#f3f4f6')
						this.roundRect(ctx, exX + exW - 140 - tagW - 16, rowY - 9, tagW + 16, 18, 9)
						ctx.fill()
						ctx.setFillStyle(item.isPerPerson ? '#10b981' : '#6b7280')
						ctx.setFontSize(11)
						ctx.fillText(tag, exX + exW - 140 - tagW - 8, rowY)
						// 金额
						const amt = '¥' + this.formatMoney(item.total)
						ctx.setFillStyle('#1f2937')
						ctx.setFontSize(15)
						ctx.setTextAlign('right')
						ctx.fillText(amt, exX + exW - 20, rowY)
						ctx.setTextAlign('left')
					})
					y += exH + 16
				}

				// ===== 底部小程序码 + 宣传文字（始终绘制文字，图片尽力绘制） =====
				const footerY = y
				const qrSize = 100
				const qrX = W / 2 - qrSize / 2
				// 小程序码（临时注释以排查 drawImage 是否导致生成卡死）
				// if (qrPath) {
				// 	try {
				// 		ctx.drawImage(qrPath, qrX, footerY, qrSize, qrSize)
				// 	} catch (e) {
				// 		console.warn('[poster] drawImage qr fail:', e)
				// 	}
				// }
				// 标题和副标题（无论图片是否成功都绘制）
				ctx.setFillStyle('#1f2937')
				ctx.setFontSize(16)
				ctx.setTextAlign('center')
				ctx.fillText('自驾游预算计算器', W / 2, footerY + qrSize + 24)
				ctx.setFillStyle('#9ca3af')
				ctx.setFontSize(12)
				ctx.fillText('扫码进入小程序，估算你的旅行开销', W / 2, footerY + qrSize + 44)
				ctx.setTextAlign('left')

				console.log('[poster] drawing complete, calling ctx.draw, H=', H)
				ctx.draw(false, () => {
					console.log('[poster] ctx.draw callback fired')
					setTimeout(() => {
						uni.canvasToTempFilePath({
							canvasId: 'posterCanvas',
							width: W,
							height: H,
							destWidth: W * 2,
							destHeight: H * 2,
							success: (res) => {
								this.posterImage = res.tempFilePath
								this.posterGenerating = false
							},
							fail: (err) => {
								console.error('[poster] canvasToTempFilePath fail:', err)
								this.posterGenerating = false
								uni.showToast({ title: '生成失败，请重试', icon: 'none' })
							}
						}, this)
					}, 300)
				})
				// 安全超时：8 秒后若仍在生成中，强制结束避免卡死
				setTimeout(() => {
					if (this.posterGenerating) {
						console.error('[poster] generation timeout')
						this.posterGenerating = false
						uni.showToast({ title: '生成超时，请重试', icon: 'none' })
					}
				}, 8000)
				} catch (e) {
					console.error('[poster] drawPoster error:', e)
					this.posterGenerating = false
					uni.showToast({ title: '生成出错：' + (e.message || e), icon: 'none' })
				}
			},
			// 圆角矩形路径
			roundRect(ctx, x, y, w, h, r) {
				ctx.beginPath()
				ctx.moveTo(x + r, y)
				ctx.lineTo(x + w - r, y)
				ctx.arcTo(x + w, y, x + w, y + r, r)
				ctx.lineTo(x + w, y + h - r)
				ctx.arcTo(x + w, y + h, x + w - r, y + h, r)
				ctx.lineTo(x + r, y + h)
				ctx.arcTo(x, y + h, x, y + h - r, r)
				ctx.lineTo(x, y + r)
				ctx.arcTo(x, y, x + r, y, r)
				ctx.closePath()
			},
			// 保存海报到相册
			savePoster() {
				if (!this.posterImage) return
				uni.saveImageToPhotosAlbum({
					filePath: this.posterImage,
					success: () => {
						uni.showToast({ title: '已保存到相册', icon: 'success' })
					},
					fail: (err) => {
						if (err.errMsg && err.errMsg.indexOf('auth deny') > -1) {
							uni.showModal({
								title: '提示',
								content: '需要您授权保存到相册',
								confirmText: '去授权',
								success: (r) => {
									if (r.confirm) {
										uni.openSetting()
									}
								}
							})
						} else {
							uni.showToast({ title: '保存失败', icon: 'none' })
						}
					}
				})
			}
		}
	}
</script>

<style>
	.report-page {
		min-height: 100vh;
		padding: 24rpx 24rpx 60rpx;
		background-color: var(--bg-page);
		box-sizing: border-box;
	}

	/* ============ 顶部摘要（单卡片总览） ============ */
	.summary-hero {
		background: linear-gradient(135deg, #10b981 0%, #059669 100%);
		border-radius: 28rpx;
		box-shadow: 0 8rpx 28rpx rgba(16, 185, 129, 0.28);
		padding: 36rpx 32rpx 28rpx;
		margin-bottom: 24rpx;
		display: flex;
		flex-direction: column;
		align-items: center;
	}
	.summary-hero-label {
		font-size: 26rpx;
		color: rgba(255, 255, 255, 0.85);
		letter-spacing: 2rpx;
	}
	.summary-hero-amount {
		font-size: 64rpx;
		font-weight: 700;
		color: #fff;
		line-height: 1.3;
		letter-spacing: 1rpx;
		margin: 8rpx 0 20rpx;
	}
	.summary-hero-divider {
		width: 100%;
		height: 1rpx;
		background-color: rgba(255, 255, 255, 0.25);
		margin-bottom: 24rpx;
	}
	.summary-hero-stats {
		width: 100%;
		display: flex;
		align-items: center;
		justify-content: space-around;
	}
	.stat-item {
		flex: 1;
		display: flex;
		flex-direction: column;
		align-items: center;
		gap: 8rpx;
	}
	.stat-value {
		font-size: 30rpx;
		font-weight: 600;
		color: #fff;
	}
	.stat-label {
		font-size: 22rpx;
		color: rgba(255, 255, 255, 0.8);
	}
	.stat-divider {
		width: 1rpx;
		height: 48rpx;
		background-color: rgba(255, 255, 255, 0.25);
		flex-shrink: 0;
	}

	/* ============ 图表区 ============ */
	.chart-card {
		background-color: var(--bg-card);
		border-radius: 24rpx;
		box-shadow: var(--shadow-card);
		padding: 24rpx 16rpx 12rpx;
		display: flex;
		flex-direction: column;
		min-width: 0;
		margin-bottom: 20rpx;
		box-sizing: border-box;
		overflow: hidden;
	}
	.chart-card-full {
		width: 100%;
	}
	.chart-title {
		display: flex;
		align-items: center;
		gap: 12rpx;
		padding: 0 8rpx 12rpx;
	}
	.chart-title-bar {
		width: 8rpx;
		height: 24rpx;
		border-radius: 4rpx;
		background-color: var(--primary);
	}
	.chart-title text {
		font-size: 28rpx;
		font-weight: 600;
		color: var(--text-primary);
	}
	.chart-box {
		width: 100%;
		height: 360rpx;
		box-sizing: border-box;
		overflow: hidden;
	}
	.chart-box-tall {
		height: 460rpx;
	}

	/* ============ 图例 ============ */
	.legend-row {
		display: flex;
		flex-wrap: wrap;
		gap: 12rpx 20rpx;
		padding: 16rpx 24rpx;
		background-color: var(--bg-card);
		border-radius: 20rpx;
		box-shadow: var(--shadow-card);
		margin-bottom: 24rpx;
	}
	.legend-item {
		display: flex;
		align-items: center;
		gap: 8rpx;
	}
	.legend-dot {
		width: 16rpx;
		height: 16rpx;
		border-radius: 4rpx;
	}
	.legend-text {
		font-size: 24rpx;
		color: var(--text-secondary);
	}

	/* ============ 卡片通用 ============ */
	.section-card {
		background-color: var(--bg-card);
		border-radius: 24rpx;
		box-shadow: var(--shadow-card);
		margin-bottom: 24rpx;
		overflow: hidden;
	}
	.section-head {
		display: flex;
		align-items: center;
		padding: 28rpx 32rpx 20rpx;
		border-bottom: 1rpx solid var(--divider);
	}
	.section-icon {
		width: 56rpx;
		height: 56rpx;
		border-radius: 16rpx;
		background-color: var(--primary-tint);
		display: flex;
		align-items: center;
		justify-content: center;
		margin-right: 20rpx;
	}
	.section-title {
		font-size: 32rpx;
		font-weight: 600;
		color: var(--text-primary);
	}
	.section-body {
		padding: 8rpx 32rpx 28rpx;
	}

	/* ============ 费用明细条 ============ */
	.detail-item {
		padding: 22rpx 0;
		border-bottom: 1rpx dashed var(--divider);
	}
	.detail-item:last-child {
		border-bottom: none;
	}
	.detail-row {
		display: flex;
		align-items: center;
		gap: 12rpx;
		margin-bottom: 14rpx;
	}
	.detail-dot {
		width: 16rpx;
		height: 16rpx;
		border-radius: 4rpx;
		flex-shrink: 0;
	}
	.detail-name {
		flex: 1;
		font-size: 28rpx;
		color: var(--text-primary);
	}
	.detail-percent {
		font-size: 24rpx;
		color: var(--text-tertiary);
		min-width: 80rpx;
		text-align: right;
	}
	.detail-amount {
		font-size: 28rpx;
		font-weight: 600;
		color: var(--text-primary);
		min-width: 140rpx;
		text-align: right;
	}
	.detail-bar-track {
		width: 100%;
		height: 14rpx;
		border-radius: 8rpx;
		background-color: var(--bg-input);
		overflow: hidden;
	}
	.detail-bar-fill {
		height: 100%;
		border-radius: 8rpx;
		transition: width 0.4s ease;
	}

	/* ============ 额外支出明细 ============ */
	.extra-row {
		display: flex;
		align-items: center;
		gap: 12rpx;
		padding: 20rpx 0;
		border-bottom: 1rpx dashed var(--divider);
	}
	.extra-row:last-child {
		border-bottom: none;
	}
	.extra-name {
		flex: 1;
		font-size: 28rpx;
		color: var(--text-primary);
	}
	.extra-tag {
		font-size: 20rpx;
		padding: 4rpx 12rpx;
		border-radius: 16rpx;
		background-color: var(--bg-input);
		color: var(--text-tertiary);
	}
	.extra-tag.per {
		background-color: var(--primary-tint);
		color: var(--primary);
	}
	.extra-amount {
		font-size: 28rpx;
		font-weight: 600;
		color: var(--text-primary);
		min-width: 140rpx;
		text-align: right;
	}

	/* ============ 行程小结 ============ */
	.trip-meta {
		text-align: center;
		padding: 8rpx 0 0;
	}
	.trip-meta-text {
		font-size: 24rpx;
		color: var(--text-tertiary);
	}

	/* ============ 分享按钮 ============ */
	.share-btn-row {
		display: flex;
		gap: 20rpx;
		margin-top: 24rpx;
	}
	.share-btn {
		flex: 1;
		display: flex;
		align-items: center;
		justify-content: center;
		gap: 12rpx;
		height: 88rpx;
		border-radius: 24rpx;
		background: linear-gradient(135deg, #10b981 0%, #059669 100%);
		box-shadow: 0 8rpx 24rpx rgba(16, 185, 129, 0.28);
		font-size: 28rpx;
		font-weight: 600;
		color: #fff;
		letter-spacing: 2rpx;
		border: none;
		margin: 0;
	}
	.share-btn-img {
		background: linear-gradient(135deg, #6366f1 0%, #4f46e5 100%);
		box-shadow: 0 8rpx 24rpx rgba(99, 102, 241, 0.28);
	}
	.share-btn::after {
		border: none;
	}
	.share-btn:active {
		opacity: 0.85;
	}

	/* ============ 海报生成 canvas（离屏） ============ */
	.poster-canvas {
		position: fixed;
		left: -9999px;
		top: 0;
		z-index: -1;
	}

	/* ============ 海报预览弹窗 ============ */
	.poster-mask {
		position: fixed;
		top: 0;
		left: 0;
		right: 0;
		bottom: 0;
		background-color: rgba(0, 0, 0, 0.7);
		display: flex;
		align-items: center;
		justify-content: center;
		z-index: 9999;
		padding: 40rpx;
		box-sizing: border-box;
	}
	.poster-modal {
		width: 100%;
		max-width: 560rpx;
		display: flex;
		flex-direction: column;
		gap: 24rpx;
	}
	.poster-preview {
		width: 100%;
		border-radius: 16rpx;
		box-shadow: 0 8rpx 32rpx rgba(0, 0, 0, 0.3);
	}
	.poster-loading {
		height: 600rpx;
		background-color: #fff;
		border-radius: 16rpx;
		display: flex;
		flex-direction: column;
		align-items: center;
		justify-content: center;
		gap: 16rpx;
	}
	.poster-loading-text {
		font-size: 26rpx;
		color: #6b7280;
	}
	.poster-actions {
		display: flex;
		flex-direction: column;
		gap: 16rpx;
	}
	.poster-save-btn {
		display: flex;
		align-items: center;
		justify-content: center;
		gap: 10rpx;
		height: 88rpx;
		border-radius: 24rpx;
		background: linear-gradient(135deg, #10b981 0%, #059669 100%);
		box-shadow: 0 8rpx 24rpx rgba(16, 185, 129, 0.28);
		font-size: 30rpx;
		font-weight: 600;
		color: #fff;
		border: none;
		margin: 0;
	}
	.poster-save-btn::after {
		border: none;
	}
	.poster-close-btn {
		height: 80rpx;
		border-radius: 24rpx;
		background-color: rgba(255, 255, 255, 0.15);
		font-size: 28rpx;
		color: #fff;
		border: none;
		margin: 0;
	}
	.poster-close-btn::after {
		border: none;
	}
</style>
