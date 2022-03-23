<script>
import _ from 'lodash'
import dayjs from 'dayjs'
import WordButton from './WordButton.vue'
import Loading from './Loading.vue'
import { onMounted, ref, computed } from 'vue'

export default {
	components: {
		WordButton,
		Loading
	},
	setup() {
		const list = ref([])
		const ansList = ref([])
		const ansInput = ref(null)
		const searchList = ref([])
		const searchInput = ref(null)
		const inputList = ref([])
		const inputErr = ref(false)
		const loadingAns = ref(false)
		const loadingSearch = ref(false)

		const getData = () => {
			const url = 'https://plum-chloride.jp/kotonoha-tango/public/data/A_data_new.csv?ver=' + dayjs().format('YYYYMMDD')
			fetch(url)
				.then(res => {
					return res.text()
				})
				.then(data => {
					list.value = _.uniq(data.split(/\r\n|\n/))
				})
		}

		const h2k = (str) => {
			return str.replace(/[\u3041-\u3096]/g, function(match) {
				var chr = match.charCodeAt(0) + 0x60;
				return String.fromCharCode(chr);
			});
		}

		const showInputErr = () => {
			inputErr.value = true
			setTimeout(() => {inputErr.value = false}, 3000)
		}

		const registItem = () => {
			// console.log(ansInput.value)
			if(typeof ansInput.value.value === 'string' && ansInput.value.value.length === 5){
				const registStr = h2k(ansInput.value.value)
				if(_.indexOf(list.value, registStr) > -1){
					inputList.value.push([registStr, [0,0,0,0,0]])
					ansInput.value.value = ''
					return
				}
			}

			showInputErr()
		}

		const deleteItem = (index) => {
			inputList.value.splice(index, 1)
		}

		const changeItem = (index, num) => {
			if(_.isArray(inputList.value[index]) && _.isArray(inputList.value[index][1])){
				inputList.value[index][1][num] = (inputList.value[index][1][num] + 1) % 3
			}
		}

		const indexList = computed(() => {
			const indexList = _.transform(list.value, (res, val, key) => {
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

			return indexList
		})

		const inputObj = computed(() => {
			return _.transform(inputList.value, (result, value) => {
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
		})

		const displayAns = computed(() => {
			return ansList.value.slice(0, 87)
		})
		const displaySearch = computed(() => {
			return searchList.value.slice(0, 99)
		})

		// 候補絞り込み
		const refined = () => {
			loadingAns.value = true
			ansList.value = []
			setTimeout(() => {
				const selectedObj = inputObj.value
				const [allPosList, allDelList] = _.transform(selectedObj, (result, value, key) => {
					if(value[0] == 1){
						if(value[2][0] > 1){
							console.log(indexList.value[6][key.repeat(value[2][0])])
							result[0].push(indexList.value[6][key.repeat(value[2][0])])
						}
						if(value[2][1] > 0){
							const reject = indexList.value[6][key.repeat(value[2][1] + 1)]
							console.log(reject)
							if(reject){
								result[1].push(reject)
							}
						}
						const pending = _.indexOf(value[1], 1) > -1
						console.time('for')
						_.forEach(value[1], (v, k) => {
							if(v === 2){
								result[0].push(indexList.value[k][key])
								result[1].push(indexList.value[5].filter(o => indexList.value[k][key].indexOf(o) === -1))
							}
						})
						console.timeEnd('for')
						const tmpList = _.union(..._.transform(value[1], (res, v, k) => {
							if(v === 1){
								result[1].push(indexList.value[k][key])
							}else if(pending){
								res.push(indexList.value[k][key])
							}
						}, []))
						if(tmpList.length){
							result[0].push(tmpList)
						}
					}else{
						result[1].push(indexList.value[6][key])
					}
				}, [[], []])
				console.log(allPosList, allDelList)
				const posList = allPosList.length ? _.intersection(...allPosList): indexList.value[5]
				const delList = _.union(...allDelList)

				console.time('filter')
				const filList = posList.filter(i => delList.indexOf(i) === -1)
				console.timeEnd('filter')
				const num = filList.length
				ansList.value = _.map(filList, (val) => {
					return list.value[val]
				})
				loadingAns.value = false
				// console.log(ansList.value.length, ansList.value)
			}, 0)
		}

		// 検索処理
		const search = () => {
			loadingSearch.value = true
			searchList.value = []
			setTimeout(() => {
				const str = searchInput.value.value
				const searchStrList = _.groupBy(h2k(str).split(''))

				console.time('create mergeIndex')
				const mergeIndex = _.union(..._.transform(searchStrList, (res, val, key) => {
					if(typeof indexList.value[6][key.repeat(val.length)] !== 'undefined'){
						res.push(indexList.value[6][key.repeat(val.length)])
					}
				}, []))
				console.timeEnd('create mergeIndex')

				console.time('create countList')
				const countList = _.transform(mergeIndex, (res, val) => {
					const count = _.reduce(searchStrList, (sum, v, k) => {
						return sum + (indexList.value[6][k.repeat(v.length)].indexOf(val) !== -1 ? v.length: 0)
					}, 0)
					res[val] = {'id': val, 'str': list.value[val], 'count': count, 'len': count}
				}, {})
				console.timeEnd('create countList')

				const sortedList = _.sortBy(countList, (obj) => {return -obj.count})
				searchList.value = sortedList
				loadingSearch.value = false
			}, 0)
			// console.log(sortedList)
		}

		const copyToClipboard = (text) => {
			navigator.clipboard.writeText(text)
				.then(() => {
					console.log(`copied: ${text}`)
				})
				.catch(e => {
					console.error(e)
				})
		}

		onMounted(() => {
			getData()
			ansInput.value.onkeypress = (e) => {
				const key = e.keyCode || e.charCode || 0
				if(key == 13){
					registItem()
				}
			}
			searchInput.value.onkeypress = (e) => {
				const key = e.keyCode || e.charCode || 0
				if(key == 13){
					search()
				}
			}
		})

		return {
			list,
			ansInput,
			ansList,
			searchInput,
			registItem,
			inputList,
			deleteItem,
			changeItem,
			refined,
			search,
			copyToClipboard,
			loadingAns,
			loadingSearch,
			displaySearch,
			displayAns,
			inputErr
		}
	}
}
</script>

<template>
	<div class="main">
		<div class="answerArea">
			<div class="inputs">
				<input class="input_text" placeholder="回答入力" type="text" maxlength="5" ref="ansInput">
				<button class="input_button" @click="registItem">追加</button>
			</div>
			<div class="btn">
				<WordButton :inputList="inputList" :delFunc="deleteItem" :changeFunc="changeItem" :refinedFunc="refined" />
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
	<div :class="[inputErr ? 'alert': 'alert non_visi']">
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
