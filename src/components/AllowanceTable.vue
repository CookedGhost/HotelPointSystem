<template>
  <div>
    <el-button type="primary" @click="addRow">新增行</el-button>
    <el-table :data="tableData" border style="width: 100%; margin-top: 20px;">
      <!-- 交易地址列 -->
      <el-table-column prop="address" label="交易地址" min-width="250">
        <template #default="{ row, $index }">
          <div v-if="editCell.rowIndex === $index && editCell.field === 'address'" class="editable-cell">
            <el-input
              v-model="row.address"
              @blur="saveEdit"
              @keyup.enter="saveEdit"
              @change="() => row.dirty = true"
              size="small"
              placeholder="请输入地址"
            />
          </div>
          <div v-else @click="startEdit($index, 'address')" class="cell-text">
            {{ row.address || '—' }}
          </div>
        </template>
      </el-table-column>

      <!-- 交易额度列 -->
      <el-table-column prop="amount" label="交易额度" min-width="150">
        <template #default="{ row, $index }">
          <div v-if="editCell.rowIndex === $index && editCell.field === 'amount'" class="editable-cell">
            <el-input-number
              v-model="row.amount"
              @blur="saveEdit"
              @keyup.enter="saveEdit"
              @change="() => {row.dirty = true, row.amountChanged = true}"
              :controls="false"
              size="small"
              :min="0"
            />
          </div>
          <div v-else @click="startEdit($index, 'amount')" class="cell-text">
            {{ row.amount !== undefined ? row.amount : '0' }}
          </div>
        </template>
      </el-table-column>

      <!-- 备注列 -->
      <el-table-column prop="note" label="备注" min-width="200">
        <template #default="{ row, $index }">
          <div v-if="editCell.rowIndex === $index && editCell.field === 'note'" class="editable-cell">
            <el-input
              v-model="row.note"
              @blur="saveEdit"
              @keyup.enter="saveEdit"
              @change="() => row.dirty = true"
              size="small"
              placeholder="请输入备注"
            />
          </div>
          <div v-else @click="startEdit($index, 'note')" class="cell-text">
            {{ row.note || '—' }}
          </div>
        </template>
      </el-table-column>

      <!-- 操作列：新增保存按钮 -->
      <el-table-column label="操作" width="160" fixed="right">
        <template #default="{ row, $index }">
          <el-button type="primary" size="small" @click="handleSave(row)">保存</el-button>
          <el-button type="danger" size="small" @click="deleteRow($index)">删除</el-button>
        </template>
      </el-table-column>

      <!-- 状态列：显示未保存提示 -->
      <el-table-column label="状态" width="100">
        <template #default="{ row }">
          <span v-if="row.dirty" style="color: red;">未保存</span>
          <span v-if="!row.dirty" style="color: green;">已保存</span>
        </template>
      </el-table-column>
    </el-table>
  </div>
</template>

<script setup>
import { ref, nextTick, onMounted, onUnmounted } from 'vue'
import { ElMessage } from 'element-plus'
import {Web3} from 'web3'

import { CONTRACT_ADDR, WALLET_URL } from '@/core/settings'; 
import erc20abi from '@/core/contract_abi/erc20.json' with {type: 'json'}

const web3 = new Web3(WALLET_URL)
const erc20Contract = new web3.eth.Contract(erc20abi, CONTRACT_ADDR['erc20'])
erc20Contract.handleRevert = true

// 表格数据
const tableData = ref([])

const getAllowance = async (spender) => {
    try{
        if(erc20Contract.methods.allowanceOf){
            erc20Contract.methods.allowanceOf(walletAccount, spender)
            .call({value: '0x0'})
        }
    } catch(error){
        ElMessage.error(`${error}`)
    }
}

// 当前正在编辑的单元格（行索引和字段名）
const editCell = ref({ rowIndex: -1, field: '' })

// 开始编辑
const startEdit = (rowIndex, field) => {
  editCell.value = { rowIndex, field }
  nextTick(() => {
    const inputEl = document.querySelector('.editable-cell input, .editable-cell .el-input__inner')
    if (inputEl) inputEl.focus()
  })
}

// 退出编辑状态
const saveEdit = () => {
  editCell.value = { rowIndex: -1, field: '' }
}

// 新增行
const addRow = () => {
  tableData.value.push({ address: '', amount: 0, note: '', dirty: false, amountChanged: false })
  nextTick(() => {
    const table = document.querySelector('.el-table__body-wrapper')
    if (table) table.scrollTop = table.scrollHeight
  })
}

// 删除行
const deleteRow = (index) => {
  tableData.value.splice(index, 1)
  ElMessage.success('已删除')
}

// 保存按钮逻辑：可在此调用智能合约或后端 API
const handleSave = async (row) => {
  // 模拟保存操作
  console.log('保存行数据：', row)

  const waittingMsg = ElMessage({
        message: '数据保存中',
        type: 'info',
        duration: 0,
        showClose: false
  })
  try {
    const accounts = await web3.eth.getAccounts()
    const defaultAccount = accounts[0]
    // 修改配额
    if(row.amountChanged){
        if(erc20Contract.methods.approve){
            await erc20Contract.methods.approve(row.address, row.amount)
            .send({
                from: defaultAccount,
                gas: 1000000,
                gasPrice: '10000000000',
                value: '0x0'
            })
            row.dirty = false
            row.amountChanged = false
            ElMessage.success('数据已保存')
        } else{
            ElMessage.error(`error: Contract Method 'allowanceOf' isn't exist!`)
        }
    }else { // 获取配额
        if(erc20Contract.methods.allowanceOf){
            await erc20Contract.methods.allowanceOf(defaultAccount, row.address)
            .call({value: '0x0'})
            .then((_allowance) => {
                row.amount = _allowance
            })
            row.dirty = false
            row.amountChanged = false
            ElMessage.success('数据已保存')
        } else{
            ElMessage.error(`error: Contract Method 'allowanceOf' isn't exist!`)
        }
    }
  } catch(error) {
    ElMessage.error(`${error}`)
  }
  waittingMsg.close()
}

</script>

<style scoped>
.cell-text {
  width: 100%;
  height: 100%;
  padding: 5px 0;
  cursor: pointer;
  min-height: 32px;
  line-height: 22px;
}
.editable-cell {
  width: 100%;
}
</style>