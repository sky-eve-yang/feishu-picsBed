<template>
  <div style="width: 100%;padding-left: 10px;border-left: 5px solid #2598f8;margin-bottom: 20px;padding-top: 5px;">{{ $t('title') }}</div>
  <div style="line-height: 1.5;font-size: 14px;color: #909399;" v-html="$t('desc')"></div>
  <el-form style="margin-top: 30px;" ref="form" :model="formData" label-width="60px">
    <el-alert style="background-color: #e1eaff;color: #606266;"  :title="$t('ossTip')" type="info" />
    <el-link style="color: #3e75f5;margin: 20px 0;" type="primary" href="https://jfsq6znqku.feishu.cn/wiki/B09wwgeQsinKtykyT68cBs2Dndg?fromScene=spaceOverview"
  target="_blank">👉  {{ $t('labels.apiDocument') }}</el-link>
    
    <!-- oss 配置部分 -->
    <el-form-item label="OSS" size="large" required>
      accessKeyId<el-input v-model="ossConfig.accessKeyId" :placeholder="$t('placeholder.accessKeyId')" />
      accessKeySecret<el-input type="password" v-model="ossConfig.accessKeySecret" :placeholder="$t('placeholder.accessKeySecret')" />
      region<el-input v-model="ossConfig.region" :placeholder="$t('placeholder.region')" />
      bucket<el-input v-model="ossConfig.bucket" :placeholder="$t('placeholder.bucket')" />
    </el-form-item>
    

    <el-alert style="margin: 20px 0 20px 0;background-color: #e1eaff;color: #606266;" :title="$t('fieldSelectedTip')" type="info" />
    <el-form-item :label="$t('labels.attchment')" size="large" required>
      <el-select v-model="attchImgFieldId" :placeholder="$t('placeholder.attchment')" style="width: 100%">
        <el-option v-for="meta in fieldListSeView" :key="meta.id" :label="meta.name" :value="meta.id" />
      </el-select>
    </el-form-item>
    <el-form-item :label="$t('labels.link')" size="large" required>
      <el-select v-model="linkFieldId" :placeholder="$t('placeholder.link')" style="width: 100%">
        <el-option v-for="meta in fieldListSeView" :key="meta.id" :label="meta.name" :value="meta.id" />
      </el-select>
    </el-form-item>


    <div></div>

    <el-form-item :label="$t('labels.reshape')" size="large">
      <!-- <h3>图像处理</h3> -->
      <div>
        <el-checkbox v-model="isImageReshaped" :label="$t('reshape.isUsed')" size="large" />
      </div>

      <div v-if="isImageReshaped" style="display: flex;flex-direction: row;align-items: center;">
        <div style="width: 40%;">
          <el-input width="200"  type="number" v-model="imageSize.width" :placeholder="$t('placeholder.width')">
            <template #prepend>{{ $t('reshape.width') }}</template>
          </el-input>
        </div>
        
        <span style="margin: 0 20px;">×</span>  
        <div style="width: 40%;">
          <el-input  type="number" v-model="imageSize.height" :placeholder="$t('placeholder.height')">
            <template #prepend>{{ $t('reshape.height') }}</template>
          </el-input>
        </div>
      </div>

      <div v-if="isImageReshaped" style="margin-top: 20px;">
        <el-radio-group v-model="reshapeMode" class="ml-4">
          <el-tooltip
            class="box-item"
            effect="dark"
            :content="$t('tooltip.fill')"
            placement="top-start"
          >
            <el-radio label="fill" size="large">Fill Mode</el-radio>
          </el-tooltip>

          <el-tooltip
            class="box-item"
            effect="dark"
            :content="$t('tooltip.pad')"
            placement="top-start"
          >
            <el-radio label="pad" size="large">Pad Mode</el-radio>
          </el-tooltip>
          
        </el-radio-group>
      </div>
    </el-form-item>
    

    
    

    <el-button @click="imgConvertLink">{{ $t('tranfer') }}</el-button>

  </el-form>
</template>


<script setup>
import { bitable } from '@lark-base-open/js-sdk'
import { ref, onMounted, onUnmounted, computed, h } from 'vue';
import OSS from 'ali-oss';
import { useI18n } from 'vue-i18n';

// 数据 -- start
const attchImgFieldId = ref('')
const linkFieldId = ref('')
const fieldListSeView = ref([])

const ossConfig = ref({
  region: '',
  accessKeyId: '',
  accessKeySecret: '',
  bucket: '',
  secure: true,
})
// 数据 -- end


const isImageReshaped = ref(false)
const reshapeMode = ref('fill')
const imageSize = ref({
  width: 800,
  height: 800
})


const fileUrl = ref('');
const { t } = useI18n();


// -- 核心算法区域
// --001== 获取附件单元格的图片，上传到oss，并写回对应的链接字段
const imgConvertLink = async() => {
  // 非空判断
  if (ossConfig.value.region && ossConfig.value.accessKeyId && ossConfig.value.accessKeySecret && ossConfig.value.bucket && attchImgFieldId.value && linkFieldId.value) {
    console.log('必填信息已填写')
  } else {
    await bitable.ui.showToast({
      toastType: 'warning',
      message: t('toast.lackRequiredInfo')
    })
    return
  }

  // 存入到缓存中
  localStorage.setItem('ossConfig', JSON.stringify(ossConfig.value))  // 阿里云oss配置
  localStorage.setItem('attchImgFieldId', attchImgFieldId.value)   // 附件字段ID
  localStorage.setItem('linkFieldId', linkFieldId.value)  // 链接字段ID
  localStorage.setItem('isImageReshaped', isImageReshaped.value)  // 是否启动图像处理参数
  localStorage.setItem('imageSize', JSON.stringify(imageSize.value))  // 图像宽高设置
  localStorage.setItem('reshapeMode', reshapeMode.value)  // 图像处理模式：fill、pad

  // 加载bitable实例
  const { tableId, viewId } = await bitable.base.getSelection();
  const table = await bitable.base.getActiveTable();
  const view = await table.getViewById(viewId);

  // ## mode1: 全部记录
  // const RecordList = await view.getVisibleRecordIdList()

  // ## model2: 交互式选择记录 
  const RecordList = await bitable.ui.selectRecordIdList(tableId, viewId);

  const attachmentField = await table.getFieldById(attchImgFieldId.value)
  const linkField = await table.getFieldById(linkFieldId.value)
  console.log('linkFieldId.value', linkFieldId.value)

  await bitable.ui.showToast({
    toastType: 'loading',
    message: t('toast.isTransfering')
  })

  for (let recordId of RecordList) {
    let attchImgUrl = null
    try {
      attchImgUrl = await attachmentField.getAttachmentUrls(recordId)
    } catch(e) {
      continue
    }
    console.log("attchImgUrl:", attchImgUrl)
    console.log("recordId:", recordId)
    // attchImgUrl 为列表时
    if (attchImgUrl.length > 1) {
      let totalCDNLink = ''
      for (let tempUrl of attchImgUrl) {
        const CDNLink = await getCDNLinkByTempUrl(tempUrl, recordId)
        totalCDNLink += ( CDNLink+ ';' )
      }
      totalCDNLink = totalCDNLink.slice(0, -1)
      await linkField.setValue(recordId, totalCDNLink)  
      

    } else { // attchImgUrl 为单值时
      const CDNLink = await getCDNLinkByTempUrl(attchImgUrl, recordId)
      await linkField.setValue(recordId, CDNLink)  
    }

    // 步骤4：将链接写回对应单元格
  }

  await bitable.ui.showToast({
    toastType: 'success',
    message: t('toast.finished')
  })

}



// -- 辅助算法区域
// --001== 上传文件file至阿里云oss
const uploadFile = async (file, recordId) => {
  const client = new OSS({...ossConfig.value});
  console.log("文件上传开始", client)
  try {
    const result = await client.put(`${formatDate()}/${recordId}${generate4RandomChars()}.jpg`, file);
    console.log(result)
    fileUrl.value = result.url; // 获取上传后的文件URL
    return fileUrl.value
  } catch (e) {
    console.error('文件上传失败:', e);
    return error
  }
};

// --002== 格式化输出当前时间 yyyy/mm/dd
const formatDate = () => {
  const currentDate = new Date();

  // 获取年份、月份和日期
  const year = currentDate.getFullYear();
  const month = String(currentDate.getMonth() + 1).padStart(2, '0');
  const day = String(currentDate.getDate()).padStart(2, '0');

  // 生成yyyy/mm/dd格式的字符串
  return `${year}/${month}/${day}`;

}

// --003== 生成四位随机字符
const generate4RandomChars = () => {
  // 生成四位随机字符
  let randomString = '';
  let characters = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789';
  
  for (let i = 0; i < 4; i++) {
    let randomIndex = Math.floor(Math.random() * characters.length);
    randomString += characters.charAt(randomIndex);
  }
  
  // 输出结果
  return randomString;
}

// --004 core== 依据附件的临时链接，上传至CDN，获得CDNLink
const getCDNLinkByTempUrl = async (attchImgUrl, recordId) => {
  // 步骤1: 从URL获取Blob对象
  const response = await fetch(attchImgUrl);
  const blob = await response.blob();

  // 步骤2: 将Blob转换为File对象
  const file = new File([blob], "image.jpg", { type: blob.type });
  console.log("file: ", file)
  // 步骤3：上传File对象到OSS
  const CDNLink = await uploadFile(file, recordId)
  console.log("CDNLink", CDNLink)
  let res;

  // if (ossConfig.value.accessKeyId == 'LTAI5tRWXbosiUxSbZ3LCX2p')
  //   res = CDNLink.replace('https://lanyansanqi-silu-erp-bucket.oss-cn-beijing.aliyuncs.com', 'http://sl.siluerp.com')
  // else 
  //   res = CDNLink

  res = CDNLink.replace('https://lanyansanqi-silu-erp-bucket.oss-cn-beijing.aliyuncs.com', 'http://sl.siluerp.com')
  
  if (isImageReshaped.value) {
    res += `?x-oss-process=image/resize,m_${reshapeMode.value},h_${imageSize.value.height},w_${imageSize.value.width},limit_0`
  }
  console.log('getCDNLinkByTempUrl() >> CDNLink', res)


  return res
}


onMounted(async () => {
  // 获取字段列表 -- start
  const selection = await bitable.base.getSelection()
  const table = await bitable.base.getTableById(selection.tableId)
  const view = await table.getViewById(selection.viewId)
  fieldListSeView.value = await view.getFieldMetaList()
  // 获取字段列表 -- end
  
  // 从缓存中获取数据 -- start
  if (localStorage.getItem('ossConfig') !== null) {
    ossConfig.value = JSON.parse(localStorage.getItem('ossConfig')) 
  }
  if (localStorage.getItem('attchImgFieldId') !== null) {
    attchImgFieldId.value = localStorage.getItem('attchImgFieldId')
  }
  if (localStorage.getItem('linkFieldId') !== null) {
    linkFieldId.value = localStorage.getItem('linkFieldId')
  }
  // if (localStorage.getItem('isImageReshaped') !== null) {
  //   isImageReshaped.value = localStorage.getItem('isImageReshaped')
  // }
  if (localStorage.getItem('reshapeMode') !== null) {
    reshapeMode.value = localStorage.getItem('reshapeMode')
  }
  if (localStorage.getItem('imageSize') !== null) {
    imageSize.value = JSON.parse(localStorage.getItem('imageSize')) 
  }
  // 从缓存中获取数据 -- end

});


</script>


<style scoped>

</style>
