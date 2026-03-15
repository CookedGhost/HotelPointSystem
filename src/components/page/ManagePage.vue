<script setup lang="ts">
import {Web3} from 'web3'
import fs from 'fs';
import { ElMessage } from 'element-plus';
import { onMounted, onUnmounted, ref } from 'vue';

import { CONTRACT_ADDR, WALLET_URL } from '@/core/settings'; 
import erc20abi from '@/core/contract_abi/erc20.json' with {type: 'json'}
import AllowanceTable from '@/components/AllowanceTable.vue';

const web3 = new Web3(WALLET_URL)
const erc20Contract = new web3.eth.Contract(erc20abi, CONTRACT_ADDR['erc20'])
erc20Contract.handleRevert = true

const coinOwner = ref('')
const totalSupply = ref(0)
const balance = ref(0)
const allowance = ref(0)

const mintTokenTo = ref('')
const mintTokenAmmount = ref(0)

const getCoinOwner = async () => {
    try{
        if(erc20Contract.methods.coinOwner)
            erc20Contract.methods.coinOwner().call({ value: '0x0' })
                .then((_coinOwner: any) => {
                    coinOwner.value = _coinOwner
                })
        else throw Error('ERC20合约中 coinOwner 变量不存在')
    } catch(error) {
        ElMessage.error(`error: ${error}`)
    }
}

const getTotalSupply = async () => {
    try{
        if(erc20Contract.methods.totalSupply)
            erc20Contract.methods.totalSupply().call({ value: '0x0' })
                .then((_totalSupply: any) => {
                    totalSupply.value = _totalSupply
                })
        else throw Error('ERC20合约中 totalSupply 变量不存在')
    } catch(error) {
        ElMessage.error(`${error}`)
    }
}

const getBalance = async () => {
    try{
        const accounts = await web3.eth.getAccounts()
        const defaultAccount = accounts[0]
        if(erc20Contract.methods.balanceOf){
            erc20Contract.methods.balanceOf(defaultAccount)
            .call({value: '0x0'})
            .then((_balance: any) => {
                balance.value = _balance
            })
        }
    } catch(error) {
        ElMessage.error(`${error}`)
    }
}

const mintToken = async () => {
    if(mintTokenTo.value.trim() == '') {
        ElMessage.error('铸币流向地址不能为空！')
        return
    }

    const waittingMsg = ElMessage({
        message: '铸币中...',
        type: 'info',
        duration: 0,
        showClose: false
    })
    try{
        const accounts = await web3.eth.getAccounts()
        const defaultAccount = accounts[0]

        if(erc20Contract.methods.mintToken){
            await erc20Contract.methods.mintToken(mintTokenTo.value, mintTokenAmmount.value)
                .send({
                    from: defaultAccount,
                    gas: '1000000',
                    gasPrice: '10000000000',
                    value: '0x0'
                })
            waittingMsg.close()
            ElMessage.success('铸币完成！')

            getTotalSupply()
            getBalance()
        }
    } catch(error) {
        waittingMsg.close()
        ElMessage.error(`${error}`)
    }
}

const listenContractEvent = async () => {
    try {
        if(erc20Contract.events.Transfer)
            erc20Contract.events.Transfer()
                .on('data', (event) => {
                    console.log('捕获到 Transfer 事件:', event.returnValues)
                })
        console.log('监听部署完毕')
    } catch(error){
        ElMessage.error(`${error}`)
    }
}

onMounted(() => {
    getCoinOwner()
    getTotalSupply()
    getBalance()
})

</script>

<template>
    <div>
        <p>
            积分管理员: {{ coinOwner }}
        </p>
        <p>
            现流动积分: {{ totalSupply }} <br/>
            持有积分：{{ balance }}
        </p>
        <p>
            自动扣款可用额度: {{ allowance }}
        </p>
        <p>
            铸币流向地址: <input v-model="mintTokenTo" placeholder="an address" /><br/>
            铸造数量: <input type="number" v-model="mintTokenAmmount" /><br/>
            <el-button type="primary" @click="mintToken()">铸造代币</el-button>
        </p>
    </div>
    <div>
        <allowance-table></allowance-table>      
    </div>

</template>