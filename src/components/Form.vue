<template>
  <div style="width: 100%;padding-left: 10px;border-left: 5px solid #2598f8;margin-bottom: 20px;padding-top: 5px;">CDN 插件</div>
  <el-form style="margin-top: 30px;" ref="form" :model="formData" label-width="60px">
    <el-alert  title="Tip：请填写如下阿里云oss配置信息" type="info" />
    <!-- oss 配置部分 -->
    <el-form-item label="OSS" size="large" required>
      accessKeyId<el-input type="password" v-model="ossConfig.accessKeyId" placeholder="请填写 accessKeyId" />
      accessKeySecret<el-input type="password" v-model="ossConfig.accessKeySecret" placeholder="请填写 accessKeySecret" />
      region<el-input v-model="ossConfig.region" placeholder="例如 oss-cn-beijing" />
      bucket<el-input type="password" v-model="ossConfig.bucket" placeholder="请填写 bucket 名称" />
    </el-form-item>

    <el-alert style="margin: 20px 0 20px 0;" title="Tip：请选择附件字段和链接字段" type="info" />
    <el-form-item label="附件" size="large" required>
      <el-select v-model="attchImgFieldId" placeholder="请选择附件所在字段" style="width: 100%">
        <el-option v-for="meta in fieldListSeView" :key="meta.id" :label="meta.name" :value="meta.id" />
      </el-select>
    </el-form-item>
    <el-form-item label="链接" size="large" required>
      <el-select v-model="linkFieldId" placeholder="请选择链接所在字段" style="width: 100%">
        <el-option v-for="meta in fieldListSeView" :key="meta.id" :label="meta.name" :value="meta.id" />
      </el-select>
    </el-form-item>

    <el-button @click="imgConvertLink">图床转换</el-button>

  </el-form>
</template>


<script setup>
import { bitable } from '@lark-base-open/js-sdk'
import { ref, onMounted, onUnmounted, computed, h } from 'vue';
import OSS from 'ali-oss';

// 数据 -- start
const attchImgFieldId = ref('')
const linkFieldId = ref('')
const fieldListSeView = ref([])

const ossConfig = ref({
  region: '',
  accessKeyId: '',
  accessKeySecret: '',
  bucket: '',
  secure: true
})
// 数据 -- end

const fileUrl = ref('');



// -- 核心算法区域
// --001== 上传文件file至阿里云oss
const uploadFile = async (file, recordId) => {
  const client = new OSS({...ossConfig.value});
  
  try {
    const result = await client.put(`${formatDate()}/${recordId}.jpg`, file);
    console.log(result)
    fileUrl.value = result.url; // 获取上传后的文件URL
    console.log('文件上传成功，URL:', fileUrl.value);

    return fileUrl.value
  } catch (e) {
    console.error('文件上传失败:', e);

    return error
  }
};


// -- 辅助算法区域

// --001== 获取附件单元格的图片，上传到oss，并写回对应的链接字段
const imgConvertLink = async() => {

  // 非空判断
  if (ossConfig.value.region && ossConfig.value.accessKeyId && ossConfig.value.accessKeySecret && ossConfig.value.bucket && attchImgFieldId.value && linkFieldId.value) {
    console.log('必填信息已填写')
  } else {
    await bitable.ui.showToast({
      toastType: 'warning',
      message: '请填写必要信息'
    })

    return
  }

  // 存入到缓存中
  localStorage.setItem('ossConfig', JSON.stringify(ossConfig.value))
  localStorage.setItem('attchImgFieldId', attchImgFieldId.value)
  localStorage.setItem('linkFieldId', linkFieldId.value)
  
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
    message: '正在转换中...'
  })

  for (let recordId of RecordList) {
    const attchImgUrl = await attachmentField.getAttachmentUrls(recordId)
    console.log("recordId:", recordId)

    // 步骤1: 从URL获取Blob对象
    const response = await fetch(attchImgUrl);
    const blob = await response.blob();

    // 步骤2: 将Blob转换为File对象
    const file = new File([blob], "image.jpg", { type: blob.type });
    console.log("file: ", file)
    // 步骤3：上传File对象到OSS
    const CDNLink = await uploadFile(file, recordId)
    console.log('CDNLink', CDNLink)
    // 步骤4：将链接写回对应单元格
    await linkField.setValue(recordId, CDNLink)  
  }

  await bitable.ui.showToast({
    toastType: 'success',
    message: '转换完成'
  })

}

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
  const randomString = '';
  const characters = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789';
  
  for (let i = 0; i < 4; i++) {
    let randomIndex = Math.floor(Math.random() * characters.length);
    randomString += characters.charAt(randomIndex);
  }
  
  // 输出结果
  return randomString;
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
  // 从缓存中获取数据 -- end

});

onUnmounted( () => {
  localStorage.setItem('ossConfig', JSON.stringify(ossConfig.value))
  localStorage.setItem('attchImgFieldId', attchImgFieldId.value)
  localStorage.setItem('linkFieldId', linkFieldId.value)
})

</script>


<style scoped>

</style>
