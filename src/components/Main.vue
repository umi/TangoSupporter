<script>
import _ from 'lodash'
import WordButton from './WordButton.vue'
import Loading from './Loading.vue'

export default {
	expose: [
		'getData'
	],
	components: {
		WordButton,
		Loading
	},
	data() {
		return {
			list: [],
			ansList: [],
			searchList: [],
			inputList: [],
			inputErr: 0,
			loading: false
		}
	},
	props: {
	},
	methods: {
		getData() {
			fetch('https://raw.githubusercontent.com/plumchloride/tango/main/kotonoha-tango/public/data/A_data_new.csv')
				.then(res => {
					return res.text()
				})
				.then(data => {
					this.list = _.uniq(data.split("\n"))
				})
		},
		regist() {
			// console.log(this.$refs.ansInput.value)
			if(typeof this.$refs.ansInput.value === 'string' && this.$refs.ansInput.value.length === 5){
				const registStr = this.h2k(this.$refs.ansInput.value)
				if(_.indexOf(this.list, registStr) > -1){
					this.inputList.push([registStr, [0,0,0,0,0]])
					this.$refs.ansInput.value = ''
					return
				}
			}

			this.showInputErr()
		},
		delete(index) {
			this.inputList.splice(index, 1)
		},
		change(index, num) {
			if(_.isArray(this.inputList[index]) && _.isArray(this.inputList[index][1])){
				this.inputList[index][1][num] = (this.inputList[index][1][num] + 1) % 3
			}
		},
		h2k(str) {
			return str.replace(/[\u3041-\u3096]/g, function(match) {
				var chr = match.charCodeAt(0) + 0x60;
				return String.fromCharCode(chr);
			});
		},
		refined() {
			this.loading = true
			setTimeout(() => {
				const selectedObj = this.inputObj
				const intersection = _.spread(_.intersection)
				const union = _.spread(_.union)
				const listIndex = _.cloneDeep(this.listIndex[5])
				const [allPosList, allDelList] = _.transform(selectedObj, (result, value, key) => {
					if(value[0] == 1){
						if(_.indexOf(value[1], 2) > -1){
							_.forEach(value[1], (v, k) => {
								if(v === 2){
									result[0].push(this.listIndex[k][key])
									result[1].push(_.pullAll(listIndex, this.listIndex[k][key]))
								}
							})
						}else{
							const tmpList = _.transform(value[1], (res, v, k) => {
								if(v === 1){
									result[1].push(this.listIndex[k][key])
								}else{
									res.push(this.listIndex[k][key])
								}
							}, [])
							result[0].push(union(tmpList))
						}
					}else{
						result[1].push(this.listIndex[6][key])
					}
				}, [[], []])
				const posList = allPosList.length ? intersection(allPosList): listIndex
				const delList = union(allDelList)

				_.pullAll(posList, delList)
				const num = posList.length
				this.ansList = _.map(posList, (val) => {
					return this.list[val]
				})
				this.loading = false
				// console.log(this.ansList.length, this.ansList)
			}, 0)
		},
		search() {
			const str = this.$refs.searchInput.value
			const searchStr = _.uniq(this.h2k(str).split('')).join('')
			const mergeIndex = _.transform(searchStr.split(''), (res, val) => {
				if(typeof this.listIndex[6][val] !== 'undefined'){
					res[0] = _.concat((res[0]||[]), this.listIndex[6][val])
				}
			}, [])[0]

			const countList = _.transform(mergeIndex, (res, val) => {
				res[val] = {'id': val, 'str': this.list[val], 'count': (res[val]?res[val]['count']:0) + 1, 'len': _.uniq(this.list[val].split('')).length}
			}, {})

			const sortedList = _.sortBy(countList, (obj) => {return -(obj.count + obj.len)})
			this.searchList = sortedList
			// console.log(sortedList)
		},
		mergeStatus(objValue, srcValue) {
			if(_.isArray(objValue)){
				if(objValue[0] === 0){
					return srcValue
				}else if(srcValue[0] === 1){
					return [1, _.map(objValue[1], (v, k) => {
						return Math.max(v, srcValue[1][k])
					})]
				}
			}
		},
		showInputErr() {
			this.inputErr = 1
			setTimeout(() => {this.inputErr = 0}, 3000)
		},
		copyToClipboard(text) {
			navigator.clipboard.writeText(text)
				.then(() => {
					console.log(`copied: ${text}`)
				})
				.catch(e => {
					console.error(e)
				})
		}
	},
	mounted() {
		this.getData()
		this.$refs.ansInput.onkeypress = (e) => {
			const key = e.keyCode || e.charCode || 0
			if(key == 13){
				this.regist()
			}
		}
		this.$refs.searchInput.onkeypress = (e) => {
			const key = e.keyCode || e.charCode || 0
			if(key == 13){
				this.search()
			}
		}
	},
	computed: {
		listIndex() {
			const list = _.transform(this.list, (res, val, key) => {
				res[5].push(key)
				_.forEach(val.split(''), (v, k) => {
					if(typeof res[k][v] === 'undefined'){ res[k][v] = [] }
					res[k][v].push(key)
					if(typeof res[6][v] === 'undefined'){ res[6][v] = [] }
					res[6][v].push(key)
				})
			}, [{}, {}, {}, {}, {}, [], {}])
			// console.log(list)

			return list
		},
		inputObj() {
			return _.transform(this.inputList, (result, value) => {
				_.forEach(value[0].split(''), (val, key) => {
					switch(value[1][key]){
						case 0:
							if(!result[val]){
								result[val] = [0]
							}
							break
						case 1:
						case 2:
							if(!result[val] || result[val][0] === 0){
								result[val] = [1, _.fill([0,0,0,0,0], value[1][key], key, key+1)]
							}else{
								result[val] = [1, _.fill(result[val][1], Math.max(result[val][1][key], value[1][key]), key, key+1)]
							}
							break
						default:
							break
					}
				})
			}, {})
		},
		displayAns() {
			return this.ansList.slice(0, 87)
		},
		displaySearch() {
			return this.searchList.slice(0, 99)
		}
	}
}
</script>

<template>
	<div class="main">
		<div class="answerArea">
			<div class="inputs">
				<input class="input_text" placeholder="回答入力" type="text" maxlength="5" ref="ansInput">
				<button class="input_button" @click="regist">追加</button>
			</div>
			<WordButton :inputList="inputList" :delFunc="delete" :changeFunc="change" :refinedFunc="refined" />
			<Loading :show="loading" />
			<p>
				候補:{{ ansList.length }}
				/ 全単語:{{ list.length }}
			</p>
			<div class="words">
				<div v-for="item in displayAns" @click="copyToClipboard(item)">{{ item }}</div>
			</div>
		</div>
		<div class="searchArea">
			<div class="inputs">
				<input class="input_text" placeholder="検索文字入力" type="text" ref="searchInput">
				<button class="input_button" @click="search">検索</button>
			</div>
			<div class="words">
				<div v-for="item in displaySearch" @click="copyToClipboard(item.str)" :class="`count${item.count} len${item.len}`">{{ item.str }}</div>
			</div>
		</div>
	</div>
	<div :class="[inputErr === 1 ? 'alert': 'alert non_visi']">
		<div>
			<p id="alert_text">注意<br>ことのはアプリの辞書内の単語を記入して下さい</p>
		</div>
	</div>
</template>

<style lang="scss">
.main {
    display: flex;
    justify-content: center;
	gap: 30px;
	padding: 10px 0;
	background: #e4edf1;
}
.ansArea,
.searchArea {
	min-width: 340px;
	max-width: 340px;
}
.searchArea {
	.inputs {
		margin-bottom: 32px;
	}
}
.inputs {
    display: flex;
    justify-content: center;

	.input_text {
		font-size: 1.5em;
		width: 9em;
		text-align: center;
		border: solid rgb(167,210,141) 2px;
		background-color: rgb(228,237,241);
		border-radius: 1em;
		padding: 5px;
		/* margin-bottom: 10px; */
		margin: 3px;
	}
	.input_button {
		font-size: 2em;
		margin: 5px;
		border: solid #555 0px;
		background-color: rgb(211,147,93);
		color: black;
		cursor: pointer;
	}
}
.words {
    display: flex;
    flex-wrap: wrap;
    justify-content: space-between;
    gap: 5px 2px;
	width: 340px;
    font-size: 1.1em;
	margin-top: 10px;
	>div {
		cursor: pointer;
		background: #d1e3c6;
		padding: 2px 10px;
		&.count5 {
			background: #f59494;
		}
		&.count4 {
			background: #fdb36f;
		}
		&.count3 {
			background: #edd164;
		}
		&.count2 {
			background: #c3d373;
		}
	}
}
.alert {
    width: 100vw;
    position: absolute;
    display: flex;
    justify-content: center;

	& div {
		width: 50vw;
		background-color: rgb(211,147,93);
		text-align: center;
		font-weight: bold;
		border-radius: 0.5em;
		margin: 1em;
		padding: 0.4em;
		position: fixed;
		top: 1vw;
	}
}
.non_visi {
	display: none;
}
</style>
