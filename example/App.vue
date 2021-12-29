<template>
  <div>
    <p>md5校验进度 ：{{ status }}{{ md5Progress }}</p>
    <uploader
      :options="options"
      :file-status-text="statusText"
      :autoStart="false"
      class="uploader-example"
      ref="uploader"
      @file-added="onFileAdded"
      
      @file-success="onFileSuccess"
      @file-complete="fileComplete"
      @complete="complete"
    ></uploader>
  </div>
</template>

<script>
import SparkMD5 from 'spark-md5'
import axios from 'axios'
export default {
  data () {
    return {
      status: 'wait',
      md5Progress: 0,
      options: {
        target: '//10.202.0.216:8081/boot/file/',
        testChunks: false,
        chunkSize: 100 * 1000,
        forceChunkSize: true,
        // 服务器分片校验函数，秒传及断点续传基础
        checkChunkUploadedByResponse: function (chunk, message) {
          console.log('checkChunkUploadedByResponse', chunk, message)
          let objMessage = JSON.parse(message)
          if (objMessage.skipUpload) {
            // 跳过分块上传
            return true
          }
          return (objMessage.uploaded || []).indexOf(chunk.offset + 1) >= 0
        }
      },
      attrs: {
        accept: 'image/*'
      },
      statusText: {
        success: '成功了',
        error: '出错了',
        uploading: '上传中',
        paused: '暂停中',
        waiting: '等待中'
      }
    }
  },
  methods: {
    complete () {
      console.log('complete', arguments)
    },
    fileComplete () {
      console.log('file complete', arguments)
    },
    statusSet (file, status) {
      this.status = status
    },
    onFileAdded (file) {
      console.log('onFileAdded', file)
      // this.panelShow = true

      // 计算MD5，下文会提到
      this.computeMD5(file)
    },
    onFileSuccess (rootFile, file, response, chunk) {
      console.log('onFileSuccess', rootFile, file, response, chunk)
      axios.post('/register', {
        ...file
      }).then(() => {
      // 继续上传
        file.resume()

        this.statusSet(file.id, 'upload')
      })
    },
    /**
     * 计算md5，实现断点续传及秒传
     * @param file
     */
    computeMD5 (file) {
      let fileReader = new FileReader()
      let time = new Date().getTime()
      let blobSlice = File.prototype.slice || File.prototype.mozSlice || File.prototype.webkitSlice
      let currentChunk = 0
      const chunkSize = this.options.chunkSize

      console.log('chunkSize', chunkSize)
      let chunks = Math.ceil(file.size / chunkSize)
      let spark = new SparkMD5.ArrayBuffer()
      // let partSpark = new SparkMD5.ArrayBuffer()

      // 文件状态设为"计算MD5"
      this.statusSet(file.id, 'md5')

      file.pause()

      loadNext()

      fileReader.onload = (e) => {
        spark.append(e.target.result)

        if (currentChunk < chunks) {
          currentChunk++
          loadNext()
          // 实时展示MD5的计算进度
          this.$nextTick(() => {
            this.md5Progress = ((currentChunk / chunks) * 100).toFixed(0)
          })
        } else {
          let md5 = spark.end()
          this.computeMD5Success(md5, file)
          console.log(`MD5计算完毕：${file.name} \nMD5：${md5} \n分片：${chunks} 大小:${file.size} 用时：${new Date().getTime() - time} ms`)
        }
      }
      fileReader.onerror = function () {
        this.error(`文件${file.name}读取出错，请检查该文件`)
        file.cancel()
      }
      function loadNext () {
        let start = currentChunk * chunkSize
        let end = start + chunkSize >= file.size ? file.size : start + chunkSize

        // console.log('loadNext', start, end)
        fileReader.readAsArrayBuffer(blobSlice.call(file.file, start, end))
      }
    },
    /** md5计算成功 */
    computeMD5Success (md5, file) {
      file.uniqueIdentifier = md5

      // 创建任务

      axios.post('/register', {
        fielMd5: md5,
        ...file
      }).then(() => {
      // 继续上传
        file.resume()

        this.statusSet(file.id, 'upload')
      }).catch((err) => {
        console.error(err)
        this.statusSet(file.id, 'error')
      })
    }
  },
  mounted () {
    this.$nextTick(() => {
      window.uploader = this.$refs.uploader.uploader
    })
  }
}
</script>

<style>
.uploader-example {
  width: 880px;
  padding: 15px;
  margin: 40px auto 0;
  font-size: 12px;
  box-shadow: 0 0 10px rgba(0, 0, 0, 0.4);
}
.uploader-example .uploader-btn {
  margin-right: 4px;
}
.uploader-example .uploader-list {
  max-height: 440px;
  overflow: auto;
  overflow-x: hidden;
  overflow-y: auto;
}
</style>
