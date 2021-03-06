<template>
  <div>
    <h1 class="text-xs-center">Select weather</h1>
    <v-container v-if="!robonomicsStatus" fluid fill-height class="px-3">
      <v-layout
        justify-center
        align-center
      >
        <v-flex text-xs-center>
          <h1>Load robonomics</h1>
          <v-progress-circular indeterminate color="primary"></v-progress-circular>
        </v-flex>
      </v-layout>
    </v-container>

    <v-container v-if="robonomicsStatus" grid-list-md class="px-3">
      <v-layout justify-center row wrap>
        <v-flex xs12 md10 lg6>
          <v-card>
            <v-card-text>
              <v-container grid-list-md class="px-3">
                <v-layout row wrap>
                  <v-flex md12 class="text-xs-center">
                    Lighthouse: <b>{{ lighthouse.name }}</b>
                    <div class="my-3"><b>{{model}}</b></div>
                    <div v-if="agents.list.length > 0" class="my-3">Agents: <a :href="agents.url" target="_blank">{{agents.list.length}}</a></div>
                    <v-alert
                      v-if="modelError!==''"
                      :value="true"
                      type="error"
                    >
                      {{modelError}}
                    </v-alert>
                    <v-btn
                      v-if="modelError===''"
                      :loading="loadingOrder"
                      :disabled="loadingOrder"
                      color="primary"
                      @click.native="order"
                    >
                      Order
                    </v-btn>
                  </v-flex>
                </v-layout>
              </v-container>
            </v-card-text>
          </v-card>
        </v-flex>
      </v-layout>

      <v-layout justify-center row wrap v-if="demand !== null">
        <v-flex xs12 md10 lg6>
          <v-card>
            <v-card-title primary-title>
              <div>
                <h3 class="headline mb-0">Received data</h3>
              </div>
            </v-card-title>
            <v-card-text>
              <v-progress-linear v-if="frees.length === 0" :indeterminate="true"></v-progress-linear>
              <div v-else v-for="(res, resIndex) in frees" :key="resIndex">
                <b>IPFS hash of data: </b>
                <a :href="`https://ipfs.io/ipfs/${res.hash}`" target="_blank">{{ res.hash }}</a>
                <br/>
                <v-progress-linear v-if="res.result.length === 0" :indeterminate="true"></v-progress-linear>
                <div v-else>
                  <div v-for="(item, i) in res.result" :key="i">
                    <p>{{ item }}</p>
                  </div>
                </div>
                <v-divider class="my-3" />
              </div>
            </v-card-text>
          </v-card>
        </v-flex>
      </v-layout>
    </v-container>
  </div>
</template>

<script>
import { Token } from 'robonomics-js'
import _findIndex from 'lodash/findIndex'
import _indexOf from 'lodash/indexOf'
import getRobonomics from '../utils/robonomics'
import getIpfs, { cat as ipfsCat } from '../utils/ipfs'
import rosBag from '../utils/rosBag'
import { getModel, getAgents } from '../utils/utils'
import * as config from '../config'

let robonomics

export default {
  data () {
    return {
      robonomicsStatus: false,
      token: null,
      loadingOrder: false,
      lighthouse: {
        name: '',
        address: ''
      },
      demand: null,
      frees: [],
      model: '',
      modelError: '',
      agents: {
        url: '',
        list: []
      }
    }
  },
  created () {
    getModel()
      .then((r) => {
        if (r === '' || r.length !== 46 || !/^Qm/.test(r)) {
          this.modelError = 'Error model'
        } else {
          this.model = r
        }
        return getAgents()
      })
      .then((r) => {
        this.agents = r
        return getIpfs()
      })
      .then(() => getRobonomics())
      .then((r) => {
        robonomics = r
        robonomics.ready().then(() => {
          if (config.TOKEN) {
            this.token = new Token(robonomics.web3, config.TOKEN)
          } else {
            this.token = robonomics.xrt
          }
          this.lighthouse.name = robonomics.lighthouse.name
          this.lighthouse.address = robonomics.lighthouse.address
          robonomics.getDemand(this.model, (msg) => {
            if (msg.account === robonomics.account) {
              this.demand = msg
            }
          })
          robonomics.getResult((msg) => {
            console.log('result unverified', msg)
            if (web3.toChecksumAddress(msg.liability) === web3.toChecksumAddress(robonomics.account)) {
              if (this.agents.list.length > 0 && _indexOf(this.agents.list, msg.account) < 0) {
                console.log('Skip result of', msg.account)
                return
              }
              const i = _findIndex(this.frees, (item) => {
                return item.hash === msg.result
              })
              if (i < 0) {
                this.frees.push({
                  hash: msg.result,
                  result: []
                })
                const k = this.frees.length - 1
                ipfsCat(msg.result)
                  .then((r) => {
                    rosBag(new Blob([r]), (bag) => {
                      this.frees[k].result.push(bag.message.data)
                    }, { topics: ['/data'] })
                  })
              }
            }
          })
          this.robonomicsStatus = true
        })
      })
  },
  methods: {
    order () {
      if (this.model.length === 46 && /^Qm/.test(this.model)) {
        this.loadingOrder = true
        web3.eth.getBlock('latest', (e, r) => {
          const demand = {
            objective: config.OBJECTIVE_TRADE,
            token: this.token.address,
            cost: config.PRICE,
            lighthouse: robonomics.lighthouse.address,
            validator: '0x0000000000000000000000000000000000000000',
            validatorFee: 0,
            deadline: r.number + 1000
          }
          robonomics.post('demand', this.model, demand)
            .then(() => {
              this.loadingOrder = false
            })
            .catch((e) => {
              this.loadingOrder = false
            })
        })
      }
    }
  }
}
</script>
