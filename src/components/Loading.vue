<script>
import {defineComponent, ref} from 'vue'

export default defineComponent({
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
	setup(props) {
		return {
			props
		}
	}
})
</script>

<template>
	<div v-if="show" class="loading">{{ text }}</div>
</template>

<style lang="scss">
</style>
