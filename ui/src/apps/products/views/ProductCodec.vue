<template>
  <div class="details-view product-codec-view">
    <emq-details-page-head>
      <el-breadcrumb slot="breadcrumb">
        <el-breadcrumb-item :to="{ path: `/products` }">产品</el-breadcrumb-item>
        <el-breadcrumb-item>
          <product-breadcrumb
            :currentProduct="currentProduct || {}">
          </product-breadcrumb>
        </el-breadcrumb-item>
        <el-breadcrumb-item>编解码插件</el-breadcrumb-item>
      </el-breadcrumb>
    </emq-details-page-head>
    <div v-if="currentProduct" class="detail-tabs">
      <product-detail-tabs v-if="currentProduct"></product-detail-tabs>
    </div>

    <el-row :gutter="20">
      <el-col :span="14">
        <el-card class="code-card code-ide">
          <template v-slot:header class="clearfix">
            <span>编辑脚本</span>
            <div v-if="codeStatus !== null" class="review-wrap">
              <el-popover
                ref="popover"
                placement="right"
                width="280"
                trigger="hover"
                :disabled="codeStatus !== 3"
                :content="reviewOpinion">
              </el-popover>
              <span
                v-popover:popover
                :class="['running-status',
                  codeStatus === 3 ? 'failed' : codeStatus === 1 ? 'wait' : 'success']">
                <i v-if="codeStatus === 3" class="el-icon-question"></i>
                {{ codeStatusLabel }}
              </span>
            </div>
            <el-button
              class="code-run__btn"
              type="text"
              @click="run">
              运行
            </el-button>
            <el-button
              class="code-submit__btn"
              type="text"
              :disabled="!isSuccess"
              @click="save">
              提交
            </el-button>
          </template>
          <code-editor
            height="560px"
            lang="python"
            v-model="record.code"
            @changed="isSuccess = null">
          </code-editor>
        </el-card>
      </el-col>

      <el-col :span="10">
        <el-card class="code-card code-input">
          <template v-slot:header class="clearfix">
            <div class="input-title">
              <span>模拟输入</span>
              <el-popover
                placement="left"
                width="280"
                trigger="hover">
                <p>模拟数据可以是：二进制、字符串、JSON</p>
                <i slot="reference" class="el-icon-question"></i>
              </el-popover>
            </div>
            <div class="analog-type">
              <label>模拟类型：</label>
              <emq-select
                size="mini"
                v-model="record.analogType"
                :field="{options: [
                  { label: '数据上报', value: 1 },
                  { label: '数据下发', value: 2 },
                ]}">
              </emq-select>
            </div>
          </template>
          <el-form>
            <el-form-item label="主题：">
              <el-input v-model="record.topic">
              </el-input>
            </el-form-item>
            <el-form-item label="消息：">
              <el-input type="textarea" rows="3" v-model="record.input">
              </el-input>
            </el-form-item>
          </el-form>
        </el-card>

        <el-card class="code-card code-output">
          <template v-slot:header class="clearfix">
            <span>运行结果</span>
            <span
              v-if="isSuccess !== null"
              :class="['header-right', 'running-status', isSuccess ? 'success' : 'failed']">
              运行{{ isSuccess ? '成功' : '错误' }}
            </span>
          </template>
          <code-editor
            class="code-output__success"
            height="240px"
            v-model="output"
            lang="application/json"
            :disabled="true">
          </code-editor>
        </el-card>
      </el-col>
    </el-row>
  </div>
</template>


<script>
import { currentProductsMixin } from '@/mixins/currentProducts'
import { httpGet, httpPost, httpPut } from '@/utils/api'
import CodeEditor from '@/components/CodeEditor'
import EmqDetailsPageHead from '@/components/EmqDetailsPageHead'
import ProductDetailTabs from '@/apps/products/components/ProductDetailTabs'
import ProductBreadcrumb from '../components/ProductBreadcrumb'

export default {
  name: 'product-codec-view',

  mixins: [currentProductsMixin],

  components: {
    CodeEditor,
    EmqDetailsPageHead,
    ProductDetailTabs,
    ProductBreadcrumb,
  },

  data() {
    return {
      codeStatusLabel: '',
      isFirstCode: true,
      codeStatus: null,
      isSuccess: null,
      profileID: null,
      reviewOpinion: null,
      record: {
        analogType: 1,
        topic: '',
        code: '',
        input: '',
      },
      output: JSON.stringify({ output: 'result' }, null, 2),
    }
  },

  methods: {
    loadData() {
      this.isSuccess = false
      const currentCodec = JSON.parse(
        sessionStorage.getItem(this.currentProduct.productID),
      )
      if (currentCodec) {
        this.record = currentCodec
      }
      httpGet(`/codec?productID=${this.currentProduct.productID}`)
        .then((res) => {
          if (res.data !== '') {
            this.isFirstCode = false
            this.record.code = res.data.code
            this.codeStatus = res.data.codeStatus
            this.codeStatusLabel = res.data.codeStatusLabel
            this.profileID = res.data.id
            this.reviewOpinion = res.data.reviewOpinion
          } else {
            this.isFirstCode = true
          }
        })
    },
    run() {
      if (!this.record.code) {
        this.$message.warning('请输入脚本')
        return
      }
      if (!this.record.topic) {
        this.$message.warning('请输入主题')
        return
      }
      if (!this.record.input) {
        this.$message.warning('请输入消息')
        return
      }
      const data = {}
      Object.assign(data, this.record)
      data.productID = this.currentProduct.productID
      this.cacheCode(data)
      httpPost('/run_code', data).then((res) => {
        if (res.data.error) {
          this.isSuccess = false
          this.output = JSON.stringify(res.data.error, null, 2)
        } else {
          this.isSuccess = true
          this.output = JSON.stringify(res.data.output, null, 2)
        }
      })
    },
    cacheCode(data) {
      if (!data) {
        return
      }
      sessionStorage.setItem(
        data.productID,
        JSON.stringify(data),
      )
    },
    save() {
      const url = '/codec'
      const data = {
        code: this.record.code,
        productID: this.currentProduct.productID,
      }
      if (this.isFirstCode) {
        httpPost(url, data).then(() => {
          this.$message.success('提交成功')
          this.loadData()
        })
      } else if (this.profileID && !this.isFirstCode) {
        httpPut(`${url}/${this.profileID}`, data).then(() => {
          this.$message.success('更新成功')
          this.loadData()
        })
      }
    },
  },

  created() {
    this.loadData()
  },
}
</script>


<style lang="scss">
.product-codec-view {

  .el-card.code-card {
    .el-card__header {
      height: 61px;
      padding: 20px;
      background-color: var(--color-bg-table-head);
      border-bottom: 1px solid var(--color-line-card);
      .el-button {
        float: right;
        padding: 4px 0;
      }
      .header-right {
        float: right;
      }
    }
    .el-card__body {
      padding: 0px;
      height: 100%;
    }
  }

  .code-ide {
    height: 623px;
    .el-card__header {
      padding: 17px 20px !important;
      .running-status.failed::before {
        display: none;
      }
    }
    .review-wrap {
      display: inline-block;
      margin-left: 20px;
      width: 96px;
      padding: 3px 0;
      text-align: center;
      border-radius: 12px;
      background: var(--color-bg-gray);
    }
    .el-button.code-run__btn {
      margin-left: 40px;
    }
  }

  .code-input {
    height: 300px;
    .el-card__header {
      display: flex;
      .input-title {
        flex: 1;
      }
      .analog-type {
        flex: 1;
        .emq-select {
          width: 66%;
          margin-top: -3px;
        }
      }
    }
    .el-icon-question {
      padding-left: 5px;
    }
    .el-form-item {
      margin-bottom: 0px;
      position: relative;
      bottom: 8px;
    }
    .el-card__body {
      margin: 15px;
      .el-form-item__content {
        margin-bottom: 10px;
      }
    }
  }

  .code-output {
    height: 300px;
  }

  .CodeMirror {
    color: var(--color-text-lighter);
  }
  .CodeMirror-gutters {
    background-color: var(--color-bg-hover);
    border-right: 1px solid var(--color-line-card);
  }
  // python
  .cm-s-dracula.CodeMirror {
    background-color: var(--color-bg-card) !important;
  }
  .cm-s-dracula .CodeMirror-gutters {
    background-color: var(--color-bg-hover) !important;
    border-right: 1px solid var(--color-line-card) !important;
  }
  .cm-s-dracula .CodeMirror-linenumber {
    color: var(--color-text-lighter) !important;
  }
  // JSON
  .cm-s-lesser-dark.CodeMirror {
    background-color: var(--color-bg-card);
  }
  .cm-s-lesser-dark .CodeMirror-linenumber {
    color: var(--color-text-lighter) !important;
  }
  .cm-s-lesser-dark.CodeMirror {
    text-shadow: none;
  }
}
</style>