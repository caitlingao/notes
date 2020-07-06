```
<el-select
    placeholder="请选择审批人"
    value-key="name"
    v-model="approver">
    <el-option
        v-for="item in approvers"
        :key="item.id"
        :label="item.id"
        :value="item">
    </el-option>
</el-select>
```
```
[{id: 1, name: '张三'}, {id: 2, name: '李四'},{id: 3, name: '王五'}]
```
