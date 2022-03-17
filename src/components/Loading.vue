<script>
export default ({
	props: {
		show: {
			type: Boolean,
			default: false
		},
		text: {
			type: String,
			default: '集計中…'
		}
	},
	setup() {
		return {}
	}
})
</script>

<template>
	<div v-if="show" class="loading">{{ text }}</div>
</template>

<style lang="scss">
</style>
