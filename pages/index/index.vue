<template>
	<view class="trip-page">

		<!-- 基础行程 -->
		<view class="section-card">
			<view class="section-head">
				<view class="section-icon">
					<uni-icons type="map" size="20" color="#10b981" />
				</view>
				<text class="section-title">基础行程</text>
			</view>

			<view class="section-body">
				<view class="row">
					<view class="row-icon">
						<uni-icons type="map-pin-ellipse" size="18" color="#10b981" />
					</view>
					<text class="row-label">行驶里程（去程）</text>
					<view class="row-input-group">
						<input class="row-input" type="number" v-model="form.mileage" />
						<text class="row-suffix">公里</text>
					</view>
				</view>

				<view class="row row-tap" @tap="toggleRoundTrip">
					<view class="row-icon">
						<uni-icons type="loop" size="18" color="#10b981" />
					</view>
					<view class="row-text">
						<text class="row-label">是否往返</text>
						<text class="row-tip">勾选后总里程按去程里程 × 2 计算</text>
					</view>
					<view class="check-circle" :class="{ active: form.isRoundTrip }">
						<uni-icons v-if="form.isRoundTrip" type="checkmarkempty" size="14" color="#fff" />
					</view>
				</view>

				<view class="row row-tap" @tap="toggleHighway">
					<view class="row-icon">
						<uni-icons type="flag" size="18" color="#10b981" />
					</view>
					<view class="row-text">
						<text class="row-label">是否走高速</text>
						<text class="row-tip">勾选后按 0.5 元/公里估算高速通行费</text>
					</view>
					<view class="check-circle" :class="{ active: form.useHighway }">
						<uni-icons v-if="form.useHighway" type="checkmarkempty" size="14" color="#fff" />
					</view>
				</view>
			</view>
		</view>

		<!-- 车辆类型与能耗 -->
		<view class="section-card">
			<view class="section-head">
				<view class="section-icon">
					<uni-icons type="gear" size="20" color="#10b981" />
				</view>
				<text class="section-title">车辆类型与能耗</text>
			</view>

			<view class="section-body">
				<view class="pill-tabs">
					<view
						v-for="(tab, i) in vehicleTabs"
						:key="i"
						class="pill-tab"
						:class="{ active: vehicleTab === i }"
						@tap="vehicleTab = i"
					>
						<uni-icons
							:type="vehicleIcons[i]"
							size="16"
							:color="vehicleTab === i ? '#10b981' : 'currentColor'"
						/>
						<text>{{ tab }}</text>
					</view>
				</view>

				<!-- 油车 -->
				<view v-if="vehicleTab === 0" class="tab-panel">
					<view class="row">
						<view class="row-icon"><uni-icons type="fire" size="18" color="#10b981" /></view>
						<text class="row-label">百公里油耗</text>
						<view class="row-input-group">
							<input class="row-input" type="number" v-model="gas.consumption" />
							<text class="row-suffix">升</text>
						</view>
					</view>
					<view class="row">
						<view class="row-icon"><uni-icons type="wallet" size="18" color="#10b981" /></view>
						<text class="row-label">当前油价</text>
						<view class="row-input-group">
							<input class="row-input" type="number" v-model="gas.price" />
							<text class="row-suffix">元/升</text>
						</view>
					</view>
				</view>

				<!-- 电车 -->
				<view v-if="vehicleTab === 1" class="tab-panel">
					<view class="row">
						<view class="row-icon"><uni-icons type="loop" size="18" color="#10b981" /></view>
						<text class="row-label">百公里电耗</text>
						<view class="row-input-group">
							<input class="row-input" type="number" v-model="ev.consumption" />
							<text class="row-suffix">kWh</text>
						</view>
					</view>
					<view class="row">
						<view class="row-icon"><uni-icons type="wallet" size="18" color="#10b981" /></view>
						<text class="row-label">平均电价</text>
						<view class="row-input-group">
							<input class="row-input" type="number" v-model="ev.price" />
							<text class="row-suffix">元/度</text>
						</view>
					</view>
				</view>

				<!-- 混动车 -->
				<view v-if="vehicleTab === 2" class="tab-panel">
					<view class="row">
						<view class="row-icon"><uni-icons type="loop" size="18" color="#10b981" /></view>
						<text class="row-label">百公里电耗</text>
						<view class="row-input-group">
							<input class="row-input" type="number" v-model="hybrid.evConsumption" />
							<text class="row-suffix">kWh</text>
						</view>
					</view>
					<view class="row">
						<view class="row-icon"><uni-icons type="wallet" size="18" color="#10b981" /></view>
						<text class="row-label">平均电价</text>
						<view class="row-input-group">
							<input class="row-input" type="number" v-model="hybrid.evPrice" />
							<text class="row-suffix">元/度</text>
						</view>
					</view>
					<view class="row">
						<view class="row-icon"><uni-icons type="fire" size="18" color="#10b981" /></view>
						<text class="row-label">亏电油耗</text>
						<view class="row-input-group">
							<input class="row-input" type="number" v-model="hybrid.fuelConsumption" />
							<text class="row-suffix">升</text>
						</view>
					</view>
					<view class="row">
						<view class="row-icon"><uni-icons type="wallet" size="18" color="#10b981" /></view>
						<text class="row-label">当前油价</text>
						<view class="row-input-group">
							<input class="row-input" type="number" v-model="hybrid.fuelPrice" />
							<text class="row-suffix">元/升</text>
						</view>
					</view>

					<view class="slider-block">
						<view class="slider-head">
							<text class="row-label">纯电比例</text>
							<text class="slider-value">{{ hybrid.evRatio }}%</text>
						</view>
						<slider
							class="custom-slider"
							:min="0"
							:max="100"
							:step="1"
							:value="hybrid.evRatio"
							activeColor="#10b981"
							backgroundColor="#e5e7eb"
							block-color="#10b981"
							block-size="22"
							@change="onEvRatioChange"
						/>
						<view class="slider-labels">
							<text>纯油 0%</text>
							<text>纯电 100%</text>
						</view>
						<view class="slider-tip">
							<uni-icons type="info" size="12" color="#f59e0b" />
							<text>提示：请根据日常驾驶习惯自行调整</text>
						</view>
					</view>
				</view>
			</view>
		</view>

		<!-- 行程与人数 -->
		<view class="section-card">
			<view class="section-head">
				<view class="section-icon">
					<uni-icons type="calendar" size="20" color="#10b981" />
				</view>
				<text class="section-title">行程与人数</text>
			</view>

			<view class="section-body">
				<view class="row">
					<view class="row-icon"><uni-icons type="calendar" size="18" color="#10b981" /></view>
					<text class="row-label">行程周期</text>
					<view class="row-input-group">
						<input class="row-input" type="number" v-model="form.days" />
						<text class="row-suffix">天</text>
					</view>
				</view>
				<view class="row">
					<view class="row-icon"><uni-icons type="staff" size="18" color="#10b981" /></view>
					<text class="row-label">出行人数</text>
					<view class="row-input-group">
						<input class="row-input" type="number" v-model="form.people" />
						<text class="row-suffix">人</text>
					</view>
				</view>
			</view>
		</view>

		<!-- 住宿与餐饮 -->
		<view class="section-card">
			<view class="section-head">
				<view class="section-icon">
					<uni-icons type="home" size="20" color="#10b981" />
				</view>
				<text class="section-title">住宿与餐饮</text>
			</view>

			<view class="section-body">
				<view class="row">
					<view class="row-icon"><uni-icons type="home-filled" size="18" color="#10b981" /></view>
					<text class="row-label">日均住宿费</text>
					<view class="row-input-group">
						<input class="row-input" type="number" v-model="form.hotelPerDay" />
						<text class="row-suffix">元/天</text>
					</view>
				</view>
				<view class="row">
					<view class="row-icon"><uni-icons type="shop" size="18" color="#10b981" /></view>
					<text class="row-label">人日均餐费</text>
					<view class="row-input-group">
						<input class="row-input" type="number" v-model="form.mealPerDay" />
						<text class="row-suffix">元/人/天</text>
					</view>
				</view>
			</view>
		</view>

		<!-- 额外预算 -->
		<view class="section-card">
			<view class="section-head">
				<view class="section-icon">
					<uni-icons type="wallet" size="20" color="#10b981" />
				</view>
				<text class="section-title">额外预算</text>
			</view>

			<view class="section-body">
				<view
					v-for="(item, index) in extraItems"
					:key="index"
					class="extra-item"
				>
					<view class="row">
						<view class="row-icon"><uni-icons type="wallet" size="18" color="#10b981" /></view>
						<view class="row-input-group extra-name-input">
							<input class="row-input" v-model="item.name" placeholder="项目名称" placeholder-class="ph" />
						</view>
						<view class="row-input-group extra-amount-input">
							<input class="row-input" type="number" v-model="item.amount" />
							<text class="row-suffix">元</text>
						</view>
					</view>

					<view class="extra-meta">
						<view class="per-pill" :class="{ active: item.isPerPerson }" @tap="togglePerPerson(index)">
							<view class="check-circle small" :class="{ active: item.isPerPerson }">
								<uni-icons v-if="item.isPerPerson" type="checkmarkempty" size="12" color="#fff" />
							</view>
							<text>是否为人均</text>
						</view>
						<view class="del-btn" @tap="confirmDelete(index)">
							<uni-icons type="trash" size="16" color="#ef4444" />
							<text class="del-text">删除</text>
						</view>
					</view>
				</view>

				<view class="add-btn" @tap="addItem">
					<uni-icons type="plus" size="18" color="#10b981" />
					<text>新增项目</text>
				</view>
			</view>
		</view>

		<!-- 删除确认弹窗 -->
		<uni-popup ref="deletePopup" type="dialog">
			<uni-popup-dialog
				type="warn"
				title="确认删除"
				content="确定要删除该额外预算项目吗？"
				:before-close="true"
				@confirm="onConfirmDelete"
				@close="onCancelDelete"
			></uni-popup-dialog>
		</uni-popup>
	</view>
</template>

<script>
	export default {
		data() {
			return {
				// 基础行程
				form: {
					mileage: 500,
					isRoundTrip: false,
					useHighway: false,
					days: 5,
					people: 2,
					hotelPerDay: 300,
					mealPerDay: 100
				},
				// 车辆类型选项卡
				vehicleTab: 0,
				vehicleTabs: ['油车', '电车', '混动车'],
				vehicleIcons: ['fire', 'loop', 'paperplane'],
				// 油车
				gas: { consumption: 8, price: 8 },
				// 电车
				ev: { consumption: 15, price: 1.5 },
				// 混动车
				hybrid: {
					evConsumption: 15,
					evPrice: 1.5,
					fuelConsumption: 5,
					fuelPrice: 8,
					evRatio: 50
				},
				// 额外预算
				extraItems: [
					{ name: '停车费', amount: 100, isPerPerson: false },
					{ name: '景点门票', amount: 100, isPerPerson: true }
				],
				pendingDeleteIndex: null
			}
		},
		methods: {
			toggleRoundTrip() {
				this.form.isRoundTrip = !this.form.isRoundTrip
			},
			toggleHighway() {
				this.form.useHighway = !this.form.useHighway
			},
			togglePerPerson(index) {
				this.extraItems[index].isPerPerson = !this.extraItems[index].isPerPerson
			},
			onEvRatioChange(e) {
				this.hybrid.evRatio = e.detail.value
			},
			addItem() {
				this.extraItems.push({ name: '', amount: '', isPerPerson: false })
			},
			confirmDelete(index) {
				this.pendingDeleteIndex = index
				this.$refs.deletePopup.open()
			},
			onConfirmDelete() {
				if (this.pendingDeleteIndex !== null) {
					this.extraItems.splice(this.pendingDeleteIndex, 1)
					this.pendingDeleteIndex = null
				}
			},
			onCancelDelete() {
				this.pendingDeleteIndex = null
			}
		}
	}
</script>

<style>
	.trip-page {
		min-height: 100vh;
		padding: 24rpx 24rpx 60rpx;
		background-color: var(--bg-page);
		box-sizing: border-box;
		transition: background-color 0.25s ease;
	}

	/* 卡片 */
	.section-card {
		background-color: var(--bg-card);
		border-radius: 24rpx;
		box-shadow: var(--shadow-card);
		margin-bottom: 24rpx;
		overflow: hidden;
		transition: background-color 0.25s ease;
	}

	/* 卡片头 */
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

	/* 卡片体 */
	.section-body {
		padding: 8rpx 32rpx 28rpx;
	}

	/* 行 */
	.row {
		display: flex;
		align-items: center;
		padding: 24rpx 0;
		border-bottom: 1rpx solid var(--divider);
		gap: 16rpx;
	}
	.row:last-child {
		border-bottom: none;
	}
	.row-tap {
		transition: background-color 0.15s ease;
	}
	.row-tap:active {
		background-color: var(--bg-hover);
	}
	.row-icon {
		width: 48rpx;
		height: 48rpx;
		border-radius: 12rpx;
		background-color: var(--bg-card-soft);
		display: flex;
		align-items: center;
		justify-content: center;
		flex-shrink: 0;
	}
	.row-label {
		font-size: 28rpx;
		color: var(--text-primary);
		flex-shrink: 0;
	}
	.row-text {
		flex: 1;
		display: flex;
		flex-direction: column;
		gap: 4rpx;
	}
	.row-tip {
		font-size: 22rpx;
		color: var(--text-tertiary);
	}

	/* 输入框组 */
	.row-input-group {
		flex: 1;
		display: flex;
		align-items: center;
		justify-content: flex-end;
		gap: 10rpx;
	}
	.row-input {
		flex: 1;
		text-align: right;
		font-size: 28rpx;
		color: var(--text-primary);
		min-width: 0;
		background-color: var(--bg-input);
		border-radius: 12rpx;
		padding: 12rpx 20rpx;
	}
	.ph {
		color: var(--text-tertiary);
	}
	.row-suffix {
		font-size: 24rpx;
		color: var(--text-tertiary);
		flex-shrink: 0;
		min-width: 80rpx;
		text-align: left;
	}

	/* 自定义圆形勾选 */
	.check-circle {
		width: 40rpx;
		height: 40rpx;
		border-radius: 50%;
		border: 2rpx solid var(--border-color);
		background-color: var(--bg-input);
		display: flex;
		align-items: center;
		justify-content: center;
		transition: all 0.2s ease;
		flex-shrink: 0;
	}
	.check-circle.active {
		background-color: var(--primary);
		border-color: var(--primary);
	}
	.check-circle.small {
		width: 32rpx;
		height: 32rpx;
	}

	/* 车辆类型 pill tabs */
	.pill-tabs {
		display: flex;
		background-color: var(--bg-input);
		border-radius: 16rpx;
		padding: 8rpx;
		margin: 16rpx 0 8rpx;
		gap: 8rpx;
	}
	.pill-tab {
		flex: 1;
		display: flex;
		flex-direction: column;
		align-items: center;
		justify-content: center;
		gap: 8rpx;
		padding: 16rpx 0;
		font-size: 28rpx;
		color: var(--text-secondary);
		border-radius: 12rpx;
		transition: all 0.2s ease;
	}
	.pill-tab.active {
		background-color: var(--bg-card);
		color: var(--primary);
		font-weight: 600;
		box-shadow: 0 2rpx 8rpx rgba(15, 23, 42, 0.08);
	}

	.tab-panel {
		padding-top: 8rpx;
	}

	/* slider */
	.slider-block {
		padding: 24rpx 0 8rpx;
	}
	.slider-head {
		display: flex;
		justify-content: space-between;
		align-items: center;
		margin-bottom: 16rpx;
	}
	.slider-value {
		font-size: 32rpx;
		font-weight: 700;
		color: var(--primary);
	}
	.custom-slider {
		margin: 0 8rpx;
	}
	.slider-labels {
		display: flex;
		justify-content: space-between;
		font-size: 22rpx;
		color: var(--text-tertiary);
		margin-top: 8rpx;
		padding: 0 8rpx;
	}
	.slider-tip {
		display: flex;
		align-items: center;
		gap: 6rpx;
		margin-top: 16rpx;
		padding: 12rpx 16rpx;
		background-color: var(--warning-soft);
		border-radius: 12rpx;
	}
	.slider-tip text {
		font-size: 22rpx;
		color: var(--warning);
	}

	/* 额外预算项 */
	.extra-item {
		padding: 20rpx 0;
		border-bottom: 1rpx dashed var(--divider);
	}
	.extra-item:last-of-type {
		border-bottom: none;
	}
	.extra-name-input {
		flex: 1.4;
	}
	.extra-amount-input {
		flex: 1;
	}
	.extra-meta {
		display: flex;
		justify-content: space-between;
		align-items: center;
		margin-top: 16rpx;
		padding-left: 64rpx;
	}
	.per-pill {
		display: flex;
		align-items: center;
		gap: 12rpx;
		padding: 8rpx 16rpx;
		border-radius: 32rpx;
		background-color: var(--bg-input);
	}
	.per-pill.active {
		background-color: var(--primary-tint);
	}
	.per-pill text {
		font-size: 24rpx;
		color: var(--text-secondary);
	}
	.per-pill.active text {
		color: var(--primary);
		font-weight: 500;
	}
	.del-btn {
		display: flex;
		align-items: center;
		gap: 6rpx;
		padding: 8rpx 12rpx;
		border-radius: 12rpx;
	}
	.del-btn:active {
		background-color: var(--danger-soft);
	}
	.del-text {
		font-size: 24rpx;
		color: var(--danger);
	}

	/* 新增按钮 */
	.add-btn {
		display: flex;
		align-items: center;
		justify-content: center;
		gap: 8rpx;
		margin-top: 24rpx;
		padding: 20rpx;
		border-radius: 16rpx;
		border: 2rpx dashed var(--primary);
		background-color: var(--primary-tint);
	}
	.add-btn text {
		font-size: 28rpx;
		color: var(--primary);
		font-weight: 500;
	}
</style>
