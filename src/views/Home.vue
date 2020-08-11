<template>
	<div class="app-wrapper">
		<div class="sidebar-container">
			<file-search
				v-model="searchTitle"
				@create="fileCreate"
				@keyup.enter.native="handleSearch"
				@search="handleSearch"
				@clear="getFileList"
				@change="handleChange"
				clearable
			></file-search>
			<!-- <el-button type="primary" @click="createTest">测-增</el-button>
			<el-button type="danger" @click="deleteTest">测-删</el-button>
			<el-button type="warning" @click="updateTest">测-改</el-button>
			<el-button type="success" @click="queryTest">测-查</el-button> -->
			<file-list :fileList="fileList" :active.sync="activeIndex" :selectedFile.sync="selectedFile">
				<template v-slot:menu>
					<li>置顶</li>
					<li @click="fileDelete">删除</li>
				</template>
			</file-list>
		</div>
		<div class="main-container">
			<file-edit v-model="fileItem.content" :title.sync="fileItem.title" :boxShadow="false" :subfield="false" :shortCut="false" @titleBlur="updateTitle" @change="updateContent"/>
		</div>
	</div>
</template>

<script>
import FileSearch from '@/components/FileSearch';
import FileList from '@/components/FileList';
import FileEdit from '@/components/FileEdit';
import dayjs from 'dayjs';
export default {
	name: 'Home',
	components: {
		FileSearch,
		FileList,
		FileEdit
	},
	data() {
		return {
			searchTitle: '',
			fileList: [],
			fileItem: {
				_id: '',
				title: '手摸手Electron + Vue实战教程（三）',
				content: ''
			},
			activeIndex: 0,
			selectedFile: {}
		};
	},
	watch:{
		activeIndex(newValue){
			this.fileItem = this.fileList[newValue]
		}
	},
	mounted() {
		this.init();
	},
	methods: {
		async init() {
			await this.getFileList();
			if (this.fileList.length === 0){ 
				return 
				};
			const [firstFileItem] = this.fileList;
			this.fileItem = firstFileItem;
		},
		async getFileList(query = {}) {
			const list = await this.$db.markdown.find(query).sort({ isTop: -1, updatedAt: -1 });
			// 格式化时间，添加内容备份字段
			this.fileList = list.map(item => {
				item.originalContent = item.content;
				item.createdAt = dayjs(item.createdAt).format('YYYY-MM-DD HH:mm:ss');
				item.updatedAt = dayjs(item.updatedAt).format('YYYY-MM-DD HH:mm:ss');
				return item;
			});
		},
		// // 增
		// createTest() {
		// 	const fileNew = { title: '无标题笔记', content: '',isTop:false };
		// 	this.$db.markdown.insert(fileNew);
		// },
		// // 删
		// async deleteTest() {
		// 	const list = await this.$db.markdown.find().sort({ updatedAt: -1 });
		// 	if (list.length === 0) return;
		// 	this.$db.markdown.remove({ _id: list[0]._id }).then(() => {
		// 		this.$message.warning('删除成功');
		// 	});
		// },
		// // 改
		// async updateTest() {
		// 	const list = await this.$db.markdown.find().sort({ updatedAt: -1 });
		// 	if (list.length === 0) return;
		// 	this.$db.markdown.update({ _id: list[0]._id }, { $set: { title: '修改过的标题' } }).then(() => {
		// 		this.$message.success('修改成功');
		// 	});
		// },
		// // 查
		// async queryTest() {
		// 	const list = await this.$db.markdown.find().sort({ updatedAt: -1 });
		// 	console.log(list);
		// },
		fileCreate() {
			console.log('dafaf')
			const defaultFile = { title: '无标题的笔记', content: '', isTop: false };
			this.$db.markdown.insert(defaultFile).then(async () => {
				await this.getFileList();
				const [firstFileItem] = this.fileList;
				this.fileItem = firstFileItem;
				this.activeIndex = 0;
			});
		},
		fileDelete() {
			this.$confirm('此操作将永久删除该笔记, 是否继续?', '提示', {
				confirmButtonText: '确定',
				cancelButtonText: '取消',
				type: 'warning'
			})
				.then(() => {
					const { _id } = this.selectedFile;
					this.$db.markdown
						.remove({ _id })
						.then(num => {
							this.$message.success(`删除了${num}个项目`);
							this.init();
						})
						.catch(() => {
							this.$message.error('删除失败');
						});
				})
				.catch(() => {});
		},
		onSubmit(value) {
			console.log(value);
			console.log(this.fileItem);
		},
		handleSearch() {
			const reg = new RegExp(`${this.searchTitle}`, 'i');
			const query = { title: reg };
			this.getFileList(query);
		},
		handleChange(val) {
			if (!val) {
				this.getFileList();
			}
		},
		handleTop() {
			this.$db.markdown.update({ _id }, { $set: { isTop: !isTop } }).then(() => {
				this.init();
			});
		},
		updateTitle(title) {
			const { _id } = this.fileItem;
			this.$db.markdown.update({ _id, title: { $ne: title } }, { $set: { title } }).then(async () => {
				this.refreshList();
			});
		},
		updateContent(content) {
			const { _id, originalContent } = this.fileItem;
			if (originalContent === content) return;
			if (this.timerId) clearTimeout(this.timerId);
			this.timerId = setTimeout(() => {
				this.$db.markdown.update({ _id, content: { $ne: content } }, { $set: { content } }).then(async () => {
					await this.refreshList();
				});
			}, 1000);
		},
		async refreshList() {
			if (this.activeIndex === 0) return;
			await this.getFileList();
			const [firstFileItem] = this.fileList;
			this.fileItem = firstFileItem;
			this.activeIndex = 0;
		}
	}
};
</script>

<style lang="stylus" scoped>
.app-wrapper
	display flex
	.sidebar-container
		width 300px
		height 100vh
		border-right 1px solid #eaeefb
	.main-container
		flex 1
		overflow hidden
</style>
