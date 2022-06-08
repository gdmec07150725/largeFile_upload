<template>
  <div>
    <div style="margin-bottom: 20px">
      <input type="file" @change="handleFileChange" />
      <el-button @click="handleUpload">upload</el-button>
      <el-button @click="handlePause">pause</el-button>
      <el-button @click="handleResume">resume</el-button>
    </div>
    <div>
      <div>
        <div>calculate chunk hash</div>
        <el-progress :percentage="hashPercentage"></el-progress>
      </div>
      <div>
        <div>percentage</div>
        <el-progress :percentage="fakeUploadPercentage"></el-progress>
      </div>
    </div>
    <el-table :data="data">
      <el-table-column
        prop="hash"
        label="chunk hash"
        align="center"
      ></el-table-column>
      <el-table-column label="size(KB)" align="center" width="120">
        <template v-slot="{ row }">
          {{ row.size }}
        </template>
      </el-table-column>
      <el-table-column label="percentage" align="center">
        <template v-slot="{ row }">
          <el-progress
            :percentage="parseInt(row.percentage)"
            color="#909399"
          ></el-progress>
        </template>
      </el-table-column>
    </el-table>
  </div>
</template>

<script>
// 切片大小
// the chunk size
const SIZE = 50 * 1024 * 1024 // 10M;

export default {
  filters: {
    transformByte(val) {
      return Number((val / 1024).toFixed(0));
    }
  },
  computed: {
    uploadPercentage() {
      if (!this.container.file || !this.data.length) return 0;
      const loaded = this.data.map(item => item.size * item.percentage).reduce((acc, cur) => acc + cur);
      return parseInt(loaded / this.container.file.size);
    }
  },
  watch: {
    uploadPercentage(now) {
      if (now > this.fakeUploadPercentage) {
        this.fakeUploadPercentage = now;
      }
    }
  },
  data: () => ({
    container: {
      file: null
    },
    data: [],
    hashPercentage: 0,
    requestList: [],
    fakeUploadPercentage: 0
  }),
  methods: {
    handleFileChange(e) {
      const [file] = e.target.files;
      if (!file) return;
      Object.assign(this.$data, this.$options.data());
      this.container.file = file;
    },
    // 生成文件切片
    createFileChunk(file, size = SIZE) {
      const fileChunkList = [];
      let cur = 0;
      while (cur < file.size) {
        fileChunkList.push({ file: file.slice(cur, cur + size)});
        cur += size;
      }
      return fileChunkList;
    },
    // 上传切片, 同时过滤已上传的切片
    async uploadChunks(uploadedList = []) {
      const requestList = this.data
        .filter(({ hash }) => !uploadedList.includes(hash))
        .map(({ chunk, hash, fileHash, index }) => {
          const formData = new FormData();
          formData.append("chunk", chunk);
          formData.append("hash", hash);
          formData.append("fileHash", fileHash);
          formData.append("filename", this.container.file.name);
          return { formData, index };
        })
        .map(({ formData, index }) => this.request({
          url: "http://localhost:8081/upload",
          data: formData,
          onProgress: this.createProgressHandler(this.data[index]),
          requestList: this.requestList
        }));
      // 并发请求
      await Promise.all(requestList);
      // 之前上传切片数量 + 本次上传的切片数量 = 所有切片数量时合并切片
      if (uploadedList.length + requestList.length === this.data.length) {
        await this.mergeRequest();
      }
    },
    async mergeRequest() {
      await this.request({
        url: "http://localhost:8081/merge",
        headers: {
          "content-type": "application/json"
        },
        data: JSON.stringify({
          size: SIZE,
          filename: this.container.file.name,
          fileHash: this.container.hash,
        })
      });
      this.$message.success("upload success");
    },
    // 生成文件的hash （web-worker）
    calculateHash(fileChunkList) {
      return new Promise(resolve => {
        // 添加worker属性
        this.container.worker = new Worker("/hash.js");
        this.container.worker.postMessage({ fileChunkList });
        this.container.worker.onmessage = e => {
          const { percentage, hash } = e.data;
          this.hashPercentage = parseInt(percentage.toFixed(2));
          if (hash) {
            resolve(hash);
          }
        }
      })
    },
    // 校验文件是否已经上传
    async verifyUpload(filename, fileHash) {
      const { data } = await this.request({
        url: "http://localhost:8081/verify",
        headers: {
          "content-type": "application/json"
        },
        data: JSON.stringify({
          filename,
          fileHash
        })
      });
      return JSON.parse(data);
    },
    async handleUpload() {
      if (!this.container.file) return;
      const fileChunkList = this.createFileChunk(this.container.file);
      this.container.hash = await this.calculateHash(fileChunkList);
      const { shouldUpload, uploadedList } = await this.verifyUpload(
        this.container.file.name,
        this.container.hash
      );

      if (!shouldUpload) {
        this.$message.success("skip upload: file had uploaded");
        return;
      }
      this.data = fileChunkList.map(({ file }, index) => ({
        fileHash: this.container.hash,
        chunk: file,
        index,
        size: file.size,
        // 文件名 + 数组下标
        hash: this.container.hash + "-" + index,
        percentage: uploadedList.includes(this.container.hash + "-" + index) ? 100 : 0
      }));
      await this.uploadChunks(uploadedList);
    },
    createProgressHandler(item) {
      return e => {
        item.percentage = parseInt(String((e.loaded / e.total) * 100));
        console.log("percentage", item.percentage);
      }
    },
    async handleResume() {
      const { uploadedList } = await this.verifyUpload(
        this.container.file.name,
        this.container.hash
      );
      await this.uploadChunks(uploadedList);
    },
    handlePause() {
      this.requestList.forEach(xhr => xhr?.abort());
      this.requestList = [];
      if (this.container.worker) {
        this.container.worker.onmessage = null;
      }
    },
    request({
      url,
      method = "post",
      data,
      headers = {},
      onProgress = e => e,
      requestList
    }) {
      return new Promise(resolve => {
        const xhr = new XMLHttpRequest();
        xhr.upload.onprogress = onProgress; // 获取进度
        xhr.open(method, url);
        Object.keys(headers).forEach(key => xhr.setRequestHeader(key, headers[key]));
        xhr.send(data);
        xhr.onload = e => {
          // 将请求成功的xhr从列表中删除
          if (requestList) {
            const xhrIndex = requestList.findIndex(item => item === xhr);
            requestList.splice(xhrIndex, 1);
          }
          resolve({
            data: e.target.response
          });
        };
        // 暴露当前xhr给外部
        requestList?.push(xhr);
      })
    }
  }
}
</script>

<style></style>
