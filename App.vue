<script>
	export default {
		onLaunch: function() {
			console.log('App Launch')
		},
		onShow: function() {
			console.log('App Show')
		},
		onHide: function() {
			console.log('App Hide')
		}
	}
</script>

<style>
	/* ============ 浅色模式（默认） ============ */
	page {
		--bg-page: #f5f7fa;
		--bg-card: #ffffff;
		--bg-card-soft: #f9fafb;
		--bg-input: #f3f4f6;
		--bg-active: #ecfdf5;
		--bg-hover: #f3f4f6;

		--text-primary: #1f2937;
		--text-secondary: #6b7280;
		--text-tertiary: #9ca3af;
		--text-on-primary: #ffffff;

		--border-color: #e5e7eb;
		--border-soft: #f0f1f3;
		--divider: #f0f1f3;

		--primary: #10b981;
		--primary-hover: #059669;
		--primary-soft: #d1fae5;
		--primary-tint: #ecfdf5;

		--warning: #f59e0b;
		--warning-soft: #fef3c7;
		--danger: #ef4444;
		--danger-soft: #fee2e2;

		--shadow-card: 0 2rpx 16rpx rgba(15, 23, 42, 0.04);
		--shadow-pop: 0 8rpx 32rpx rgba(15, 23, 42, 0.08);

		background-color: #f5f7fa;
		color: #1f2937;
	}

	/* ============ 系统深色模式自动适配（官方 prefers-color-scheme 方式） ============ */
	@media (prefers-color-scheme: dark) {
		page {
			--bg-page: #0f172a;
			--bg-card: #1e293b;
			--bg-card-soft: #243349;
			--bg-input: #334155;
			--bg-active: #064e3b;
			--bg-hover: #334155;

			--text-primary: #f1f5f9;
			--text-secondary: #cbd5e1;
			--text-tertiary: #94a3b8;
			--text-on-primary: #ffffff;

			--border-color: #334155;
			--border-soft: #2c3a4f;
			--divider: #2c3a4f;

			--primary: #34d399;
			--primary-hover: #6ee7b7;
			--primary-soft: #065f46;
			--primary-tint: #064e3b;

			--warning: #fbbf24;
			--warning-soft: #78350f;
			--danger: #f87171;
			--danger-soft: #7f1d1d;

			--shadow-card: 0 2rpx 16rpx rgba(0, 0, 0, 0.3);
			--shadow-pop: 0 8rpx 32rpx rgba(0, 0, 0, 0.5);

			background-color: #0f172a;
			color: #f1f5f9;
		}

		/* uni-popup 暗色模式适配 */
		.uni-popup-dialog {
			background-color: #1e293b !important;
		}
		.uni-dialog-title-text {
			color: #f1f5f9 !important;
		}
		.uni-dialog-content-text {
			color: #cbd5e1 !important;
		}
		.uni-dialog-button {
			background-color: #1e293b !important;
		}
		.uni-dialog-button-text {
			color: #cbd5e1 !important;
		}
		.uni-dialog-button-text--default {
			color: #34d399 !important;
		}
		.uni-border-left {
			border-left-color: #334155 !important;
		}
	}
</style>
