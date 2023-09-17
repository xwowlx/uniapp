<template>
	<view class="word_vote">
		<view class="cover_img">
			<view class="title_tip">
				<view class="cover">
					封面图（可以不上传）
				</view>
				<view class="tip">
					（ 宽高比：650 × 300 ）
				</view>
			</view>
			<view class="upload_img">
				<uni-file-picker 
				@select="selectCoverFileFunc($event)"
				:auto-upload="false" 
				limit="1"
				:del-icon="false" 
				disable-preview 
				file-mediatype="image" 
				:imageStyles="coverImageStyles">
					<view class="upload">
						<text class="uploadImg">&#xe727;</text>
					</view>
				</uni-file-picker>
			</view>
		</view>
		
		<view class="basic_settings">
			<view class="title_tip">
				<view class="title">
					基础设置
				</view>
			</view>
			<view class="settings">
				<view class="title">
					<input type="text"  v-model="title" placeholder="填写投票标题"  placeholder-style="color:#bababa;font-size:16px"/>
				</view>
				<view class="explanation">
					<textarea v-model="explanation" placeholder="投票说明 (非必填)" placeholder-style="color:#bababa;font-size:14px"></textarea>
				</view>
			</view>
		</view>
		
		<view class="vote_options_settings">
			<view class="title_tip">
				<view class="title">
					投票选项设置
				</view>
			</view>
			<view class="option_list">
				<view class="option_item" v-for="(item,index) in options" :key="item.id">
					<view class="option_input">
						<text class="removeOption" @click="removeOption(item.id)">&#xe618;</text><input type="text" v-model="item.name" placeholder="输入选项名称" placeholder-style="color:#bababa;font-size:14px">
					</view>				
					<view class="option_upload">
						<uni-file-picker
						@select="selectVoteItemFileFunc($event,index)"
						:auto-upload="false" 
						limit="1"
						:del-icon="false" 
						disable-preview 
						file-mediatype="image" 
						:imageStyles="voteItemImageStyles">
							<view class="upload">
								<text class="smallUploadImg">&#xe727;</text>
							</view>
						</uni-file-picker>
					</view>
				</view>
			</view>
			<view class="option_add" @click="addOption()">
				<text class="addOption">&#xe660;</text><text>添加选项</text>
			</view>
		</view>
		
		<view class="vote_rules_settings">
			<view class="title_tip">
				<view class="title">
					投票规则设置
				</view>
			</view>	
			<view class="rule_list">
				<view class="rule_item">
					<text>投票截止时间</text>
					<view >
						<uni-datetime-picker 
							:border="false" 
							:clear-icon="false" 
							v-model="voteEndTime"
							:start="startDate"
							:end="endDate"
							></uni-datetime-picker>
					</view>
				</view>
			</view>
		</view>
	</view>
	
	<view class="vote_btn" >
		<button type="primary" @click="submitVote">发起投票</button>
	</view>
</template>

<script>
	import {getBaseUrl, requestUtil} from "../../util/requestUtil.js"
	import {isEmpty} from "../../util/stringUtil.js"
	import {timeFormat} from "../../util/dateUtil.js"
	export default{
		data(){
			const curDate=new Date();
			const vv=new Date(curDate.getTime()+24*60*60*1000);
			return{
				title:'',
				explanation:'',
				coverImageFileName:'',
				coverImageStyles: {
					width:"700rpx",
					height:"400rpx",
					border:false
				},
				voteItemImageStyles:{
					width:"150rpx",
					height:"120rpx",
					border:false
				},
				voteEndTime:timeFormat(vv),
				options:[
					{
						id:1,
						name:'',
						image:""
					},
					{
						id:2,
						name:'',
						image:""
					}
				]
			}
		},
		computed:{
			startDate(){
				return new Date();
			},
			endDate(){
				const curDate=new Date();
				const vv=new Date(curDate.getTime()+24*60*60*1000*365);
				return vv;
			}
		},
		methods:{
			addOption:function(){
				var option={
					id:this.options[this.options.length-1].id+1,
					name:'',
					image:""
				}
				this.options.push(option);
			},
			removeOption:function(id){
				const index=this.options.findIndex(v=>v.id===id)
				this.options.splice(index,1);
			},
			selectCoverFileFunc:function(e){
				console.log(e.tempFilePaths[0])
				uni.uploadFile({
					header:{token:uni.getStorageSync("token")},
					url:getBaseUrl()+"/vote/uploadCoverImage",
					filePath:e.tempFilePaths[0],
					name:"coverImage",
					success: (res) => {
						let result=JSON.parse(res.data);
						if(result.code==0){
							this.coverImageFileName=result.coverImageFileName;
						}
					}
				})
			},
			selectVoteItemFileFunc:function(e,index){
				console.log("index="+index)
				console.log(e.tempFilePaths[0])
				uni.uploadFile({
					header:{token:uni.getStorageSync("token")},
					url:getBaseUrl()+"/vote/uploadVoteItemImage",
					filePath:e.tempFilePaths[0],
					name:"voteItemImage",
					success: (res) => {
						let result=JSON.parse(res.data);
						if(result.code==0){
							this.options[index].image=result.voteItemImageFileName;
						}
					}
				})
			},
			submitVote:async function(e){
				// 验证
				if(isEmpty(this.title)){
					uni.showToast({
						icon:"error",
						title:"请填写投票标题"
					})
					return;
				}
				// 验证投票选项，如果有名称的，必须要上传图片
				for(var i=0;i<this.options.length;i++){
					var option=this.options[i];
					if(!isEmpty(option.name)){
						if(isEmpty(option.image)){
							console.log("请上传第"+(i+1)+"个投票选项图片")
							uni.showToast({
								icon:"error",
								title:"请上传第"+(i+1)+"个投票选项图片"
							})
							return;
						}
					}
				}
				// 投票选项，至少2个选项
				let resultOptions=this.options.filter(function(value,index,self){
					return !isEmpty(value.name)
				})
				if(resultOptions.length<2){
					uni.showToast({
						icon:"error",
						title:"请至少填写两个投票选项"
					})
					return;
				}
				let form={
					title:this.title,
					coverImage:this.coverImageFileName,
					explanation:this.explanation,
					voteEndTime:this.voteEndTime,
					voteItemList:resultOptions,
					type:2
				}
				let result=await requestUtil({url:"/vote/add",data:form,method:"post"});
				if(result.code==0){
					uni.showToast({
						icon:"success",
						title:"投票发起成功！"
					})
				}
			}
		}
	}
</script>

<style lang="scss">
	@import "/common/css/iconfont.css";
	.word_vote{
		padding: 20px;
		padding-bottom: 70px;
		.cover_img{
			.title_tip{
				margin-left: 10rpx;
				font-size: 26rpx;
				color: gray;
				display: flex;
				justify-content: space-between;
			}
			.upload_img{
				border-radius: 5px;
				margin-top: 20rpx;
				width:100%;
				height: 360rpx;
				background-color: white;
				display: flex;
				align-items: center;
				justify-content: center;
				.upload{
					margin: 10rpx;
					background-color: #f4f5f7;
					width:90%;
					height: 80%;
					display: flex;
					align-items: center;
					justify-content: center;
				}
			}
		}
		
		.basic_settings{
			margin-top: 20px;
			.title_tip{
				margin-left: 10rpx;
				font-size: 26rpx;
				color: gray;
				margin-bottom: 10px;
				.title{
					
				}
			}
			.settings{
				
				border-radius: 5px;
				background-color: white;
				.title{
					padding: 10px;
					input{
						font-size: 1.3rem;
						border-bottom: 1px solid #e4e4e4;
						padding-bottom: 15px;
					}
				}
				.explanation{
					padding: 10px;
					textarea{
						height: 100px;
					}
				}
			}
			
		}
		
		.vote_options_settings{
			margin-top: 20px;
			.title_tip{
				margin-left: 10rpx;
				font-size: 26rpx;
				color: gray;
				margin-bottom: 10px;
				.title{
					
				}
			}
			.option_list{
				.option_item{
					margin-top: 10px;
					border-radius: 5px;
					background-color: white;
					padding: 10px;
					.option_input{
						display: flex;
					}
					.option_upload{
						margin-top: 20rpx;
						.upload{
							margin: 10rpx;
							background-color: #f4f5f7;
							width:90rpx;
							height: 90rpx;
							display: flex;
							align-items: center;
							justify-content: center;
						}
					}
				}
			}
			.option_add{
				margin-top: 10px;
				border-radius: 5px;
				background-color: white;
				padding: 10px;
				display: flex;
				color:blue;
				font-size:14px
			}
		}
		
		.vote_rules_settings{
			margin-top: 20px;
			.title_tip{
				margin-left: 10rpx;
				font-size: 26rpx;
				color: gray;
				margin-bottom: 10px;
				.title{
					
				}
			}
			.rule_list{
				border-radius: 5px;
				background-color: white;
				.rule_item{
					display: flex;
					justify-content: space-between;
					padding: 12px;
					border-bottom: 1px solid #e4e4e4;
					font-size: 28rpx;
					align-items: center;
					height: 45rpx;
				}
			}
		}
		
	}
	
	.vote_btn{
		height: 120rpx;
		width: 100%;
		background-color: white;
		position: fixed;
		bottom: 0;
		border-top: 1px solid #e4e4e4;
		button{
			margin: 10px;
		}
	}
</style>