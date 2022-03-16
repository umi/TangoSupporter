<script>
import _ from 'lodash'
import dayjs from 'dayjs'
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
			loadingAns: false,
			loadingSearch: false
		}
	},
	props: {
	},
	methods: {
		getData() {
			// const url = 'https://raw.githubusercontent.com/plumchloride/tango/main/kotonoha-tango/public/data/A_data_new.csv'
			const url = 'https://plum-chloride.jp/kotonoha-tango/public/data/A_data_new.csv?ver=' + dayjs().format('YYYYMMDD')
			fetch(url)
				.then(res => {
					return res.text()
				})
				.then(data => {
					this.list = _.uniq(data.split(/\r\n|\n/))
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
			this.loadingAns = true
			setTimeout(() => {
				const selectedObj = this.inputObj
				const intersection = _.spread(_.intersection)
				const union = _.spread(_.union)
				const [allPosList, allDelList] = _.transform(selectedObj, (result, value, key) => {
					if(value[0] == 1){
						if(value[2][0] > 1){
							console.log(this.listIndex[6][key.repeat(value[2][0])])
							result[0].push(this.listIndex[6][key.repeat(value[2][0])])
						}
						if(value[2][1] > 0){
							const reject = this.listIndex[6][key.repeat(value[2][1] + 1)]
							console.log(reject)
							if(reject){
								result[1].push(reject)
							}
						}
						const pending = _.indexOf(value[1], 1) > -1
						_.forEach(value[1], (v, k) => {
							if(v === 2){
								result[0].push(this.listIndex[k][key])
								result[1].push(_.filter(this.listIndex[5], o => _.indexOf(this.listIndex[k][key], o) === -1))
							}
						})
						const tmpList = union(_.transform(value[1], (res, v, k) => {
							if(v === 1){
								result[1].push(this.listIndex[k][key])
							}else if(pending){
								res.push(this.listIndex[k][key])
							}
						}, []))
						if(tmpList.length){
							result[0].push(tmpList)
						}
					}else{
						result[1].push(this.listIndex[6][key])
					}
				}, [[], []])
				console.log(allPosList, allDelList)
				const posList = allPosList.length ? intersection(allPosList): this.listIndex[5]
				const delList = union(allDelList)

				_.pullAll(posList, delList)
				const num = posList.length
				this.ansList = _.map(posList, (val) => {
					return this.list[val]
				})
				this.loadingAns = false
				// console.log(this.ansList.length, this.ansList)
			}, 0)
		},
		search() {
			this.loadingSearch = true
			setTimeout(() => {
				const str = this.$refs.searchInput.value
				const searchStrList = _.groupBy(this.h2k(str).split(''))

				const mergeIndex = _.union(..._.transform(searchStrList, (res, val, key) => {
					if(typeof this.listIndex[6][key.repeat(val.length)] !== 'undefined'){
						res.push(this.listIndex[6][key.repeat(val.length)])
					}
				}, []))

				const countList = _.transform(mergeIndex, (res, val) => {
					const count = _.reduce(searchStrList, (sum, v, k) => {
						return sum + (_.indexOf(this.listIndex[6][k.repeat(v.length)], val) > -1 ? v.length: 0)
					}, 0)
					res[val] = {'id': val, 'str': this.list[val], 'count': count, 'len': count}
				}, {})

				const sortedList = _.sortBy(countList, (obj) => {return -obj.count})
				this.searchList = sortedList
				this.loadingSearch = false
			}, 0)
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
				const words = val.split('')
				_.forEach(words, (v, k) => {
					if(typeof res[k][v] === 'undefined'){ res[k][v] = [] }
					res[k][v].push(key)
					if(_.indexOf(words, v) === k){
						const picks = _.filter(words, o => o === v)
						_.forEach(picks, (char, index) => {
							const word = char.repeat(index + 1)
							if(typeof res[6][word] === 'undefined'){ res[6][word] = [] }
							res[6][word].push(key)
						})
					}
				})
			}, [{}, {}, {}, {}, {}, [], {}])
			// console.log(list)

			return list
		},
		inputObj() {
			return _.transform(this.inputList, (result, value) => {
				const words = value[0].split('')
				_.forEach(words, (val, key) => {
					if(_.indexOf(words, val) === key){
						let picks = _.map(words, (v, k) => {
							return v === val ? value[1][k]: null
						})
						const groups = _.groupBy(picks)
						const len = [1, 0]
						const hit = (groups[1] ? groups[1].length: 0) + (groups[2] ? groups[2].length: 0)
						if(!groups[null] || words.length - groups[null].length > 0){
							if(hit > 0){
								len[0] = hit
								if(!!groups[0]){
									len[1] = hit
								}
								if(!!groups[1]){
									picks = _.map(picks, v => v === 0 ? 1: v)
								}
							}
						}

						if(!result[val] || result[val][0] === 0){
							result[val] = hit > 0 ? [1, _.unzipWith([[0,0,0,0,0], picks], Math.max), len] : [0]
						}else{
							result[val] = [1, _.unzipWith([result[val][1], picks], Math.max), _.unzipWith([result[val][2], len], Math.max)]
						}
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
			<div class="btn">
				<WordButton :inputList="inputList" :delFunc="delete" :changeFunc="change" :refinedFunc="refined" />
			</div>
			<Loading :show="loadingAns" />
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
			<Loading :show="loadingSearch" text="検索中…" />
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

<style lang="scss" scoped>
.main {
    display: flex;
    justify-content: center;
	gap: 30px;
	padding: 10px 0;
	background: #e4edf1;

	@media screen and (max-width: 700px) {
		flex-direction: column;
	}
}
.answerArea,
.searchArea {
	min-width: 340px;
}
.answerArea {
	.btn {
		max-width: 340px;
		margin: 0 auto;
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
    max-width: 340px;
    font-size: 1.1em;
    margin: 10px auto 0;

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
