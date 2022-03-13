<script>
import {defineComponent, ref} from 'vue'

export default defineComponent({
	props: {
		show: Boolean
	},
	setup(props) {
		return {
			props
		}
	}
})
</script>

<template>
	<div v-if="show" class="loading">集計中…</div>
</template>

<style lang="scss">
</style>
