<script setup lang="ts">
import { Address, fromNano } from '@ton/core'
import { BclSDK, simpleTonapiProvider } from 'ton-bcl-sdk'
import { Api, HttpClient } from 'tonapi-sdk-js'
import { Decimal } from 'decimal.js'
import { computed, ref, watch } from 'vue';

interface JettonInfo {
  totalTon: Decimal;
  tonCollected: Decimal;
  tonCollectedPercent: Decimal;
  allSupply: Decimal;
  totalSupply: Decimal;
  supplyPercent: Decimal;
}

const TON_API_TOKEN = '' // optional
const TON_FUN_API_ENDPOINT = '' // optional;
const TON_FUN_MASTER_ADDRESS = 'EQCCR3ChGA_4qY9s9HjASX7OmBaAFHZi8_lkqlkDl6R8FPvW'

const tonApiClient = new Api(
  new HttpClient({
    baseUrl: 'https://tonapi.io',
    baseApiParams: {
      headers: {
        'Content-type': 'application/json',
        ...(TON_API_TOKEN ? {
          Authorization: `Bearer ${TON_API_TOKEN}`,
        } : {}),
      },
    },
  }),
)

const jettonAddress = ref('EQBMIIYlgAI5HtWc2GTNWh5bDKY0GwHW6i3CqGxORqLDXNuE')

const jettonAddressInfo = computed<{ valid: false } | { valid: true; address: Address }>(() => {
  const isValid = Address.isFriendly(jettonAddress.value)

  if (isValid) {
    return {
      valid: true,
      address: Address.parse(jettonAddress.value),
    }
  } else {
    return {
      valid: false,
    }
  }
})

const jettonInfo = ref<{
  jettonAddress: string
  info: JettonInfo
}>()

let abortController: AbortController = new AbortController()

const reset = () => {
  abortController.abort()
  abortController = new AbortController()
}

watch(jettonAddressInfo, async (addressInfo) => {
  if (!addressInfo.valid) return
  reset()

  const info = await getMemepadJettonInfo(addressInfo.address.toString())

  jettonInfo.value = {
    jettonAddress: addressInfo.address.toString(),
    info,
  }
}, { immediate: true })

const jettonInfoList = computed(() => jettonInfo.value ? {
  jettonAddress: jettonInfo.value.jettonAddress,
  list: [
    { title: 'Total TON', value: jettonInfo.value.info.totalTon.toFixed(2) },
    { title: 'TON Collected', value: jettonInfo.value.info.tonCollected.toFixed(2) },
    { title: 'TON Collected %', value: jettonInfo.value.info.tonCollectedPercent.toFixed(2) },
    { title: 'All Supply', value: jettonInfo.value.info.allSupply.toFixed(2) },
    { title: 'Total Supply', value: jettonInfo.value.info.totalSupply.toFixed(2) },
    { title: 'Supply %', value: jettonInfo.value.info.supplyPercent.toFixed(2) },
  ],
} : undefined)

function fromNanoToDecimal(value: string | bigint | number) {
  if (typeof value === 'bigint') return new Decimal(fromNano(value))
  return new Decimal(value).div(1e9)
}

async function getMemepadJettonInfo(jettonAddress: string): Promise<JettonInfo> {
  const address = Address.parse(jettonAddress)
  const sdk = BclSDK.create({
    apiProvider: simpleTonapiProvider(tonApiClient),
    clientOptions: {
      endpoint: TON_FUN_API_ENDPOINT,
    },
    masterAddress: Address.parse(TON_FUN_MASTER_ADDRESS),
  })

  const bclData = await sdk.openCoin(address).getBclData()

  const tonCollected = fromNanoToDecimal(bclData.tonLiqCollected)
  const fullPriceTonFees = fromNanoToDecimal(bclData.fullPriceTonFees)
  const fullPriceTon = fromNanoToDecimal(bclData.fullPriceTon)

  const totalTon = fullPriceTon.minus(fullPriceTonFees)
  const tonCollectedPercent = tonCollected.div(totalTon).times(100)

  const bclSupply = fromNanoToDecimal(bclData.bclSupply)
  const liqSupply = fromNanoToDecimal(bclData.liqSupply)
  const totalSupply = fromNanoToDecimal(bclData.totalSupply)

  const allSupply = bclSupply.plus(liqSupply)
  const supplyPercent = totalSupply.div(allSupply).times(100)

  return {
    totalTon,
    tonCollected,
    tonCollectedPercent,

    allSupply,
    totalSupply,
    supplyPercent,
  }
}
</script>

<template>
  <div class="p-10 max-w-[600px] mx-auto">
    <label class="flex flex-col gap-2" :class="{ 'text-red-600': !jettonAddressInfo.valid }">
      <span class="text-lg">
        {{ `jetton address${!jettonAddressInfo.valid ? '(invalid)' : ''}:` }}
      </span>

      <input v-model="jettonAddress" class="w-full ring-1 ring-green-500 text-xl rounded-lg p-2" />
    </label>

    <div v-if="jettonInfoList" class="flex flex-col gap-4 mt-10">
      <div class="text-md font-bold">{{ `jetton info for address: ${jettonInfoList.jettonAddress}` }}</div>

      <ul class="flex flex-col gap-2">
        <li v-for="item in jettonInfoList.list" :key="item.title" class="flex gap-2 items-center justify-between">
          <span class="text-lg">{{ item.title }}:</span>
          <span class="text-xl">{{ item.value }}</span>
        </li>
      </ul>
    </div>
  </div>
</template>
