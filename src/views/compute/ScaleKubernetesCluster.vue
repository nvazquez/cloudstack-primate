// Licensed to the Apache Software Foundation (ASF) under one
// or more contributor license agreements.  See the NOTICE file
// distributed with this work for additional information
// regarding copyright ownership.  The ASF licenses this file
// to you under the Apache License, Version 2.0 (the
// "License"); you may not use this file except in compliance
// with the License.  You may obtain a copy of the License at
//
//   http://www.apache.org/licenses/LICENSE-2.0
//
// Unless required by applicable law or agreed to in writing,
// software distributed under the License is distributed on an
// "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
// KIND, either express or implied.  See the License for the
// specific language governing permissions and limitations
// under the License.

<template>
  <div class="form-layout">
    <a-spin :spinning="loading">
      <a-form
        :form="form"
        @submit="handleSubmit"
        layout="vertical">
        <a-form-item :label="$t('cks.cluster.size')">
          <a-input
            v-decorator="['size', {
              initialValue: '1',
              rules: [{
                validator: (rule, value, callback) => {
                  if (value && (isNaN(value) || value <= 0)) {
                    callback('Please enter a valid number')
                  }
                  callback()
                }
              }]
            }]"
            :placeholder="apiParams.size.description"/>
        </a-form-item>
        <a-form-item :label="$t('serviceofferingid')">
          <a-select
            id="offering-selection"
            v-decorator="['serviceofferingid', {}]"
            showSearch
            optionFilterProp="children"
            :filterOption="(input, option) => {
              return option.componentOptions.children[0].text.toLowerCase().indexOf(input.toLowerCase()) >= 0
            }"
            :loading="serviceOfferingLoading"
            :placeholder="apiParams.serviceofferingid.description">
            <a-select-option v-for="(opt, optIndex) in this.serviceOfferings" :key="optIndex">
              {{ opt.name || opt.description }}
            </a-select-option>
          </a-select>
        </a-form-item>

        <div :span="24" class="action-button">
          <a-button @click="closeAction">{{ this.$t('Cancel') }}</a-button>
          <a-button :loading="loading" type="primary" @click="handleSubmit">{{ this.$t('OK') }}</a-button>
        </div>
      </a-form>
    </a-spin>
  </div>
</template>

<script>
import { api } from '@/api'

export default {
  name: 'ScaleKubernetesCluster',
  props: {
    resource: {
      type: Object,
      required: true
    }
  },
  data () {
    return {
      originalSize: 1,
      serviceOfferings: [],
      serviceOfferingLoading: false,
      minCpu: 2,
      minMemory: 2048,
      loading: false
    }
  },
  beforeCreate () {
    this.form = this.$form.createForm(this)
    this.apiConfig = this.$store.getters.apis.scaleKubernetesCluster || {}
    this.apiParams = {}
    this.apiConfig.params.forEach(param => {
      this.apiParams[param.name] = param
    })
  },
  created () {
  },
  mounted () {
    this.fetchData()
  },
  methods: {
    fetchData () {
      this.originalSize = !this.isObjectEmpty(this.resource) ? 1 : this.resource.size
      this.fetchKubernetesVersionData()
    },
    isValidValueForKey (obj, key) {
      return key in obj && obj[key] != null
    },
    arrayHasItems (array) {
      return array !== null && array !== undefined && Array.isArray(array) && array.length > 0
    },
    isObjectEmpty (obj) {
      return !(obj !== null && obj !== undefined && Object.keys(obj).length > 0 && obj.constructor === Object)
    },
    fetchKubernetesVersionData () {
      const params = {}
      if (!this.isObjectEmpty(this.resource)) {
        params.id = this.resource.kubernetesversionid
      }
      api('listKubernetesSupportedVersions', params).then(json => {
        const versionObjs = json.listkubernetessupportedversionsresponse.kubernetessupportedversion
        if (this.arrayHasItems(versionObjs) && !this.isObjectEmpty(versionObjs[0])) {
          this.minCpu = versionObjs[0].mincpunumber
          this.minMemory = versionObjs[0].minmemory
        }
      }).finally(() => {
        this.fetchServiceOfferingData()
      })
    },
    fetchServiceOfferingData () {
      this.serviceOfferings = []
      const params = {}
      this.serviceOfferingLoading = true
      api('listServiceOfferings', params).then(json => {
        var items = json.listserviceofferingsresponse.serviceoffering
        if (items != null) {
          for (var i = 0; i < items.length; i++) {
            if (items[i].iscustomized === false &&
                items[i].cpunumber >= this.minCpu && items[i].memory >= this.minMemory) {
              this.serviceOfferings.push(items[i])
            }
          }
        }
      }).finally(() => {
        this.serviceOfferingLoading = false
        if (this.arrayHasItems(this.serviceOfferings)) {
          for (var i = 0; i < this.serviceOfferings.length; i++) {
            if (this.serviceOfferings[i].id === this.resource.serviceofferingid) {
              this.form.setFieldsValue({
                serviceofferingid: i
              })
              break
            }
          }
        }
      })
    },
    handleSubmit (e) {
      e.preventDefault()
      this.form.validateFields((err, values) => {
        if (err) {
          return
        }
        this.loading = true
        const params = {
          id: this.resource.id
        }
        if (this.isValidValueForKey(values, 'size') && values.size > 0) {
          params.kubernetesversionid = this.kubernetesVersions[values.kubernetesversionid].id
        }
        if (this.isValidValueForKey(values, 'serviceofferingid') && this.arrayHasItems(this.serviceOfferings)) {
          params.serviceofferingid = this.serviceOfferings[values.serviceofferingid].id
        }
        api('scaleKubernetesCluster', params).then(json => {
          this.$message.success('Successfully scaled Kubernetes cluster: ' + this.resource.name)
        }).catch(error => {
          this.$notification.error({
            message: 'Request Failed',
            description: (error.response && error.response.headers && error.response.headers['x-description']) || error.message
          })
        }).finally(() => {
          this.$emit('refresh-data')
          this.loading = false
          this.closeAction()
        })
      })
    },
    closeAction () {
      this.$emit('close-action')
    }
  }
}
</script>

<style scoped lang="less">
  .form-layout {
    width: 60vw;

    @media (min-width: 500px) {
      width: 450px;
    }
  }

  .action-button {
    text-align: right;

    button {
      margin-right: 5px;
    }
  }
</style>
