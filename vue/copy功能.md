

```vue
1.安装依赖
npm install --save v-clipboard
 
2.在main.js中引入
import Vue from 'vue'
import Clipboard from 'v-clipboard'
 
Vue.use(Clipboard)
 
3.使用
<template>
    <button v-clipboard="value"
        v-clipboard:success="clipboardSuccessHandler" 
        v-clipboard:error="clipboardErrorHandler">    
        Copy to clipboard
   </button> 
</template>
 
 
export default {
   data() {
      return {
         value:'2659100297@qq.com'
      }
   },
   methods: {
      // Success event handler 
      clipboardSuccessHandler ({ value, event }){
         console.log('success', value);
         this.$message.success("已复制");
      },
      // Error event handler
      clipboardErrorHandler ({ value, event }) {
         console.log('error', value)
      }
    }
}
```

```vue
// demo
   <el-table-column prop="hotspot_addr" label="Onboarding Key" :show-overflow-tooltip="true" min-width="120">
        <template slot-scope="scope">
          <span>{{ scope.row.hotspot_addr || '-' }}</span>
          <i
            v-clipboard:success="clipboardSuccessHandler"
            v-clipboard:error="clipboardErrorHandler"
            v-clipboard="() => scope.row.hotspot_addr"
            class="el-icon-document-copy copy-icon"
            title="复制"
          />
        </template>
      </el-table-column>
      
      

 	clipboardSuccessHandler() {
      this.$message({
        type: 'success',
        message: '复制成功'
      })
    },
    clipboardErrorHandler() {
      this.$message({
        type: 'error',
        message: '复制失败'
      })
    }
      
      
```

