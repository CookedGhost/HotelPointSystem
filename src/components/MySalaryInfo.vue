<template>
    姓名：<input v-model="salaryInfo.name" /> <br />
    薪水：<input v-model="salaryInfo.salary" /> <br />

    {{ salaryInfo }}
</template>

<script lang="ts">
 export default {
    name: "SalaryInfo"
 }
</script>

<script setup lang="ts">
import type {SalaryInfo} from "@/type/SalaryInfo"

defineProps<{salaryInfo:SalaryInfo}>()

</script>