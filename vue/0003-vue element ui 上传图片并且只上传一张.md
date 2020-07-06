### 照片墙方式上传图片，且固定上传几张
```
<template>
  <div>
    <el-row :gutter="24" style="margin-top: 25px;">
      <el-col :span="14" :md="14" :sm="24" :xs="24">
        <el-form ref="updateForm" :model="record" label-width="140px">
          <el-form-item label="照片" prop="imgURL" :rules="[{ required: true, message: '请上传照片', triggle: 'blur' }]">
            <el-upload
              :class="{ hide: hideUpload }"
              action="void"
              list-type="picture-card"
              :http-request="handleUpload"
              :before-upload="handleBeforeUpload"
              :on-preview="handlePictureCardPreview"
              :on-remove="handleRemove"
              :on-change="handleUploadChange"
              :on-progress="handleUploadProgress"
              accept="image/jpeg,image/jpg,image/png,image/gif"
              :file-list="fileList"
            >
              <el-progress v-if="uploadFlag" type="circle" :percentage="uploadPercent" style="margin-top:30px;"></el-progress>
              <i class="el-icon-plus"></i>
              <li slot="tip" class="el-upload__tip" style="line-height: 12px; color: #a9b0b4;">
                将照片上传，文件支持5M以内的PNG、JPG、GIF格式
              </li>
            </el-upload>
            <el-dialog :visible.sync="dialogVisible">
              <img width="100%" :src="dialogImageUrl" alt />
            </el-dialog>
          </el-form-item>

          <el-form-item>
            <el-button type="primary" size="small" :loading="loading" @click="submitForm">提交</el-button>
          </el-form-item>
        </el-form>
      </el-col>
    </el-row>
  </div>
</template>

<script>
export default {
  name: 'Upload',
  data() {
    return {
      loading: false,
      dialogImageUrl: '',
      fileList: [],
      hideUpload: true,
      limitCount: 1,
      uploadFlag: false,
      uploadPercent: 0,
    };
  },
  methods: {
    handleBeforeUpload(file) {
      this.loading = true;
      const isJPG = ['image/jpeg', 'image/jpg', 'image/png', 'image/gif'].includes(file.type);
      const isLt2M = file.size / 1024 / 1024 < 2;

      if (!isJPG) {
        this.$message.error('图片格式必须为 jpeg/jpg/png/gif 中的一种');
      }
      if (!isLt2M) {
        this.$message.error('文件大小不可以超过2MB');
      }
      return isJPG && isLt2M;
    },
    handleRemove(file, fileList) {
      this.loading = true;
      this.uploadFlag = false;
      this.uploadPercent = 0;
      for (let i = 0; i < fileList.length; i++) {
        if (fileList[i] === file) {
          fileList.splice(i, 1);
        }
      }
      this.record.imgURL = '';
      this.hideUpload = fileList.length >= this.limitCount;
      this.loading = false;
    },
    handlePictureCardPreview(file) {
      this.dialogImageUrl = file.url;
      this.dialogVisible = true;
    },
    handleUpload(file) {
      this.uploadFlag = false;
      this.uploadPercent = 0;
      const formData = new FormData();
      formData.append('file', file.file);
      formData.append('fileType', 1);
      this.$store
        .dispatch('upload', formData)
        .then(res => {
          file.onSuccess();
          this.record.imgURL = res.data;
          this.loading = false;
        })
        .catch(err => {
          console.log(err);
          this.loading = false;
        });
    },
    handleUploadChange(file, fileList) {
      this.hideUpload = fileList.length >= this.limitCount;
    },
    handleUploadProgress(event, file, fileList) {
      this.uploadFlag = true;
      this.uploadPercent = Math.abs(file.percentage.toFixed(0));
    },
  }
};
</script>

<style lang="scss" scoped>
.hide >>> .el-upload--picture-card {
  display: none;
}
</style>
```
