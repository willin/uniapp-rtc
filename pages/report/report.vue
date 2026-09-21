<template>
	<view class="report-page">
		<!-- 顶部摘要：预计总花费 / 人均花费 -->
		<view class="summary-row" :class="{ 'summary-single': people <= 1 }">
			<view class="summary-card">
				<view class="summary-label">预计总花费</view>
				<view class="summary-amount">¥{{ formatMoney(totalCost) }}</view>
				<view class="summary-sub">总里程 {{ formatNumber(totalMileage) }} 公里</view>
			</view>
			<view v-if="people > 1" class="summary-card">
				<view class="summary-label">人均花费</view>
				<view class="summary-amount">¥{{ formatMoney(perPerson) }}</view>
				<view class="summary-sub">总花费 ÷ {{ people }} 人</view>
			</view>
		</view>

		<!-- 图表区：费用占比 + 费用对比（纵向堆叠，让每个图表更大更清晰） -->
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
		<view class="chart-card chart-card-full">
			<view class="chart-title">
				<view class="chart-title-bar"></view>
				<text>费用对比</text>
			</view>
			<view class="chart-box chart-box-tall">
				<qiun-data-charts
					type="column"
					:chartData="columnData"
					:opts="columnOpts"
					:reshow="reshow"
				/>
			</view>
		</view>

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
		<!-- #ifdef MP-WEIXIN || MP-QQ || MP-TOUTIAO -->
		<button class="share-btn" open-type="share">
			<uni-icons type="redo" size="18" color="#fff" />
			<text>分享报告</text>
		</button>
		<!-- #endif -->
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
				extraItems: []
			}
		},
		computed: {
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
					legend: { show: false },
					extra: {
						rose: {
							type: 'radius',
							linearType: 'custom',
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
			// 垂直柱状图数据
			columnData() {
				return {
					categories: this.visibleCategories.map(c => c.name),
					series: [{
						name: '费用',
						data: this.visibleCategories.map(c => ({
							value: Number(c.value.toFixed(2)),
							color: COLOR_MAP[c.key]
						}))
					}]
				}
			},
			columnOpts() {
				return {
					color: COLOR_LIST,
					padding: [20, 10, 0, 10],
					legend: { show: false },
					xAxis: {
						disableGrid: true,
						fontSize: 9,
						rotateLabel: false,
						lineHeight: 12,
						disabled: false
					},
					yAxis: {
						disabled: true,
						disableGrid: true
					},
					extra: {
						column: {
							type: 'group',
							width: 18,
							barBorderRadius: [4, 4, 0, 0]
						}
					},
					labelLayout: {
						hideOverlap: true
					}
				}
			}
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
			// 页面首次渲染完成后触发图表重绘
			this.$nextTick(() => {
				this.reshow = true
				setTimeout(() => { this.reshow = false }, 200)
			})
		},
		// #ifdef MP-WEIXIN || MP-QQ || MP-TOUTIAO
		onShareAppMessage() {
			const encoded = this.encodeShareData()
			return {
				title: '自驾游预算报告：总花费 ¥' + this.formatMoney(this.totalCost),
				path: '/pages/report/report?d=' + encoded
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
				this.totalMileage = data.totalMileage || 0
				this.people = data.people || 1
				this.days = data.days || 0
				this.vehicleType = data.vehicleType || ''
				this.totalCost = data.totalCost || 0
				this.perPerson = data.perPerson || 0
				this.categories = (data.categories || []).map(c => ({
					key: c.key,
					name: c.name,
					value: Number(c.value) || 0
				}))
				this.extraItems = data.extraItems || []
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

	/* ============ 顶部摘要 ============ */
	.summary-row {
		display: flex;
		gap: 20rpx;
		margin-bottom: 24rpx;
	}
	.summary-card {
		flex: 1;
		background-color: var(--bg-card);
		border-radius: 24rpx;
		box-shadow: var(--shadow-card);
		padding: 28rpx 24rpx;
		display: flex;
		flex-direction: column;
		gap: 8rpx;
	}
	/* 仅一人时，总花费卡片占满全宽，文字居中 */
	.summary-single .summary-card {
		padding: 36rpx 32rpx;
		align-items: center;
		text-align: center;
	}
	.summary-single .summary-amount {
		font-size: 52rpx;
	}
	.summary-label {
		font-size: 24rpx;
		color: var(--text-tertiary);
	}
	.summary-amount {
		font-size: 44rpx;
		font-weight: 700;
		color: var(--primary);
		line-height: 1.2;
		letter-spacing: 0.5rpx;
	}
	.summary-sub {
		font-size: 22rpx;
		color: var(--text-secondary);
		margin-top: 4rpx;
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
	.share-btn {
		display: flex;
		align-items: center;
		justify-content: center;
		gap: 12rpx;
		height: 88rpx;
		margin-top: 24rpx;
		border-radius: 24rpx;
		background: linear-gradient(135deg, #10b981 0%, #059669 100%);
		box-shadow: 0 8rpx 24rpx rgba(16, 185, 129, 0.28);
		font-size: 30rpx;
		font-weight: 600;
		color: #fff;
		letter-spacing: 2rpx;
		border: none;
	}
	.share-btn::after {
		border: none;
	}
	.share-btn:active {
		opacity: 0.85;
	}
</style>
