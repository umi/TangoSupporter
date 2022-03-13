<script>
import _ from 'lodash'

export default {
	props: {
		inputList: {
			type: Array,
			required: false,
			default: () => ([])
		},
		delFunc: Function,
		changeFunc: Function,
		refinedFunc: Function
	},
	methods: {
	},
	mounted() {
	},
	computed: {
	}
}
</script>

<template>
	<div v-if="inputList.length">
		<div v-for="(item, index) in inputList" class="row">
			<div v-for="(str, i) in item[0].split('')" :class="`col state${item[1][i]} round${~~(index/5)%2}`" @click="changeFunc(index, i)">{{ str }}</div>
			<div @click="delFunc(index)" :class="`col round${~~(index/5)%2}`">❌</div>
		</div>
		<button @click="refinedFunc" class="input_button">集計</button>
	</div>
</template>

<style lang="scss">
:root {
	--hit: rgb(167,210,141);
	--brow: rgb(252, 201, 72);
}
.row {
    display: flex;
    justify-content: center;
    padding-top: 0.5ex;
    padding-bottom: 0.5ex;
	.col {
		width: 13%;
		height: auto;
		aspect-ratio: 1 / 1;
		font-size: 1.3em;
		border-radius: 0;
		display: flex;
		flex-direction: column;
		justify-content: center;
		align-items: center;
		background-color: rgb(228,237,241);
		border: solid 0.15em rgb(167,210,141);
		text-align: center;
		font-weight: bold;
		vertical-align: middle;
		user-select: none;
		margin: 0 0.3ex;
		cursor: pointer;

		&.state0 {
			background-color: gray;
			border: solid 0.15em gray;
			color: white;
		}
		&.state1 {
			background-color: var(--brow);
			border: solid 0.15em var(--brow);
			color: black;
		}
		&.state2 {
			background-color: var(--hit);
			border: solid 0.15em var(--hit);
			color: black;
		}
		&.round1 {
			border-radius: 50%!important;
		}

	}
}

.input_button {
	font-size: 2em;
	margin: 5px;
	border: solid #555 0px;
	background-color: rgb(211,147,93);
	color: black;
	cursor: pointer;
}
</style>
