```
<el-table-column label="产品余额">
  <template slot-scope="{ row }">
    <span v-html="row.balances"></span>
  </template>
</el-table-column>
```
```
balances: "图片&nbsp;106939张<br />图书&nbsp;927本<br />电脑&nbsp;2台",
```
