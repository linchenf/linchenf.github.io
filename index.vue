<template>
  <div class="app-container">
    <el-form :model="queryParams" ref="queryForm" :inline="true" v-show="showSearch" label-width="68px">
      <el-form-item label="填表日期" prop="submitDate">
        <el-date-picker clearable size="small"
          v-model="queryParams.submitDate"
          type="date"
          value-format="yyyy-MM-dd"
          placeholder="选择填表日期">
        </el-date-picker>
      </el-form-item>
      <el-form-item label="填写人" prop="createBy">
        <el-input
          v-model="queryParams.createBy"
          placeholder="请输入填写人"
          clearable
          size="small"
          @keyup.enter.native="handleQuery"
        />
      </el-form-item>
      <el-form-item>
        <el-button type="primary" icon="el-icon-search" size="mini" @click="handleQuery">搜索</el-button>
        <el-button icon="el-icon-refresh" size="mini" @click="resetQuery">重置</el-button>
      </el-form-item>
    </el-form>

    <el-row :gutter="10" class="mb8">
      <el-col :span="1.5">
        <el-button
          type="primary"
          plain
          icon="el-icon-plus"
          size="mini"
          @click="handleAdd"
          v-hasPermi="['finance:weeklyReport:add']"
        >新增</el-button>
      </el-col>
      <el-col :span="1.5">
        <el-button
          type="success"
          plain
          icon="el-icon-edit"
          size="mini"
          :disabled="single"
          @click="handleUpdate"
          v-hasPermi="['finance:weeklyReport:edit']"
        >修改</el-button>
      </el-col>
      <right-toolbar :showSearch.sync="showSearch" @queryTable="getList"></right-toolbar>
    </el-row>

    <el-table v-loading="loading" :data="weeklyReportList" @selection-change="handleSelectionChange">
      <el-table-column type="selection" width="55" align="center" />
      <el-table-column label="主键" align="center" prop="id" v-if="false"/>
      <el-table-column label="填表日期" align="center" prop="submitDate" width="180">
        <template slot-scope="scope">
          <span>{{ parseTime(scope.row.submitDate, '{y}-{m}-{d}') }}</span>
        </template>
      </el-table-column>
      <el-table-column label="填写人" align="center" prop="createBy" />
      <el-table-column label="状态" align="center" prop="statusType">
        <template slot-scope="scope">
          <span v-if="scope.row.statusType === 0">未提交</span>
          <span v-else-if="scope.row.statusType === 1">已提交</span>
        </template>
      </el-table-column>
      <el-table-column label="操作" align="center" class-name="small-padding fixed-width">
        <template slot-scope="scope">
          <el-button
            size="mini"
            type="text"
            icon="el-icon-edit"
            @click="handleUpdate(scope.row)"
            v-hasPermi="['finance:weeklyReport:edit']"
          >修改</el-button>
        </template>
      </el-table-column>
    </el-table>

    <pagination
      v-show="total>0"
      :total="total"
      :page.sync="queryParams.pageNum"
      :limit.sync="queryParams.pageSize"
      @pagination="getList"
    />

    <!-- 添加或修改业务周报主表的增删查改对话框 -->
    <el-dialog :title="title" :visible.sync="open" :close-on-click-modal="false" width="1000px" append-to-body>
      <div class="form-content">
        <el-form ref="tableData" :rules="rules" label-width="80px">
          <el-row>
            <el-col>
              <table class="table">
                <tr>
                  <th>细分类型</th>
                  <th>指标</th>
                  <th>本月实际数</th>
                  <th>本月达成率</th>
                  <th>累计实际数</th>
                  <th>累计达成率</th>
                  <th>全年达成率</th>
                </tr>
                <tr v-for="(row, index) in tableData.slice(0,-1)" :key="index">
                  <th v-if="row.rowspan !== 1" width="100" :rowspan="row.rowspan>0 ? row.rowspan : null">{{ row.subType }}</th>
                  <th width="80">{{ row.target }}</th>
                  <td align="center">
                    <el-form-item :prop="`monthlyActualNum${index}`">
                      <el-input
                          type="number"
                          v-model="row.monthlyActualNum"
                      >
                      </el-input>
                    </el-form-item>
                  </td>
                  <td>
                    <el-form-item :prop="`monthlyAchieveRate${index}`">
                      <el-input
                          type="number"
                          v-model="row.monthlyAchieveRate"
                      >
                      </el-input>
                    </el-form-item>
                  </td>
                  <td>
                    <el-form-item :prop="`accumulateActualNum${index}`">
                      <el-input
                          type="number"
                          v-model="row.accumulateActualNum"
                      >
                      </el-input>
                    </el-form-item>
                  </td>
                  <td>
                    <el-form-item :prop="`accumulateAchieveRate${index}`">
                      <el-input
                          type="number"
                          v-model="row.accumulateAchieveRate"
                          clearable
                      >
                      </el-input>
                    </el-form-item>
                  </td>
                  <td>
                    <el-form-item :prop="`annualAchieveRate${index}`">
                      <el-input
                          type="number"
                          v-model="row.annualAchieveRate"
                      >
                      </el-input>
                    </el-form-item>
                  </td>
                </tr>
              </table>
            </el-col>
          </el-row>
          <el-form-item label="说明">
            <el-input v-model="remark" type="textarea" :rows="3"
                      maxlength="500"
                      show-word-limit/>
          </el-form-item>
        </el-form>


        <div slot="footer" class="dialog-footer">
          <el-button :loading="buttonLoading" type="primary" @click="submitForm">提交</el-button>
          <el-button :loading="buttonLoading" type="primary" @click="submitForm1">保存为草稿</el-button>
          <el-button @click="cancel">取 消</el-button>
        </div>
      </div>
    </el-dialog>

  </div>
</template>

<script>
import { listWeeklyReport, getWeeklyReport, delWeeklyReport, addWeeklyReport, updateWeeklyReport, exportWeeklyReport } from "@/api/finance/weeklyReport";
import { listDetail, getDetail, getEchoData, delDetail, addDetail, addBatchDetail, updateBatchDetail, updateDetail, exportDetail } from "@/api/finance/weeklyReportDetail";
export default {
  name: "WeeklyReport",
  components: {
  },
  data() {
    return {
	  //按钮loading
	  buttonLoading: false,
      // 遮罩层
      loading: true,
      // 导出遮罩层
      exportLoading: false,
      // 选中数组
      ids: [],
      // 非单个禁用
      single: true,
      // 非多个禁用
      multiple: true,
      // 显示搜索条件
      showSearch: true,
      // 总条数
      total: 0,
      // 业务周报主表的增删查改表格数据
      weeklyReportList: [],
      // 业务周报详情的增删查改表格数据
      detailList: [],
      // 弹出层标题
      title: "",
      // 是否显示弹出层
      open: false,
      // 查询参数
      queryParams: {
        pageNum: 1,
        pageSize: 10,
        department: undefined,
        submitDate: undefined,
        statusType: undefined,
        createBy: undefined,
        createTime: undefined,
        updateBy: undefined,
        updateTime: undefined
      },
      // 表单参数
      form: {},
      //表单详情
      detailForm: {},
      tableData: [
        {
          id:0,
          subType: '立白供应商',
          target: '日均规模',
          monthlyActualNum: null,
          monthlyAchieveRate: null,
          accumulateActualNum: null,
          accumulateAchieveRate: null,
          annualAchieveRate: null,
          rowspan: 3
        },
        {
          id:0,
          subType: '立白供应商',
          target: '收入',
          monthlyActualNum: null,
          monthlyAchieveRate: null,
          accumulateActualNum: null,
          accumulateAchieveRate: null,
          annualAchieveRate: null,
          rowspan: 1
        },
        {
          id:0,
          subType: '立白供应商',
          target: '息差',
          monthlyActualNum: null,
          monthlyAchieveRate: null,
          accumulateActualNum: null,
          accumulateAchieveRate: null,
          annualAchieveRate: null,
          rowspan: 1
        },
        {
          id:0,
          subType: '医药港',
          target: '日均规模',
          monthlyActualNum: null,
          monthlyAchieveRate: null,
          accumulateActualNum: null,
          accumulateAchieveRate: null,
          annualAchieveRate: null,
          rowspan: 3
        },
        {
          id:0,
          subType: '医药港',
          target: '收入',
          monthlyActualNum: null,
          monthlyAchieveRate: null,
          accumulateActualNum: null,
          accumulateAchieveRate: null,
          annualAchieveRate: null,
          rowspan: 1
        },
        {
          id:0,
          subType: '医药港',
          target: '息差',
          monthlyActualNum: null,
          monthlyAchieveRate: null,
          accumulateActualNum: null,
          accumulateAchieveRate: null,
          annualAchieveRate: null,
          rowspan: 1
        },
        {
          id:0,
          subType: '非助贷业务',
          target: '日均规模',
          monthlyActualNum: null,
          monthlyAchieveRate: null,
          accumulateActualNum: null,
          accumulateAchieveRate: null,
          annualAchieveRate: null,
          rowspan: 3
        },
        {
          id:0,
          subType: '非助贷业务',
          target: '收入',
          monthlyActualNum: null,
          monthlyAchieveRate: null,
          accumulateActualNum: null,
          accumulateAchieveRate: null,
          annualAchieveRate: null,
          rowspan: 1
        },
        {
          id:0,
          subType: '非助贷业务',
          target: '息差',
          monthlyActualNum: null,
          monthlyAchieveRate: null,
          accumulateActualNum: null,
          accumulateAchieveRate: null,
          annualAchieveRate: null,
          rowspan: 1
        },
        {
          id:0,
          subType: '担保助贷',
          target: '日均规模',
          monthlyActualNum: null,
          monthlyAchieveRate: null,
          accumulateActualNum: null,
          accumulateAchieveRate: null,
          annualAchieveRate: null,
          rowspan: 3
        },
        {
          id:0,
          subType: '担保助贷',
          target: '收入',
          monthlyActualNum: null,
          monthlyAchieveRate: null,
          accumulateActualNum: null,
          accumulateAchieveRate: null,
          annualAchieveRate: null,
          rowspan: 1
        },
        {
          id:0,
          subType: '担保助贷',
          target: '息差',
          monthlyActualNum: null,
          monthlyAchieveRate: null,
          accumulateActualNum: null,
          accumulateAchieveRate: null,
          annualAchieveRate: null,
          rowspan: 1
        },
        {
          id:0,
          subType: '合计',
          target: '日均规模',
          monthlyActualNum: null,
          monthlyAchieveRate: null,
          accumulateActualNum: null,
          accumulateAchieveRate: null,
          annualAchieveRate: null,
          rowspan: 3
        },
        {
          id:0,
          subType: '合计',
          target: '收入',
          monthlyActualNum: null,
          monthlyAchieveRate: null,
          accumulateActualNum: null,
          accumulateAchieveRate: null,
          annualAchieveRate: null,
          rowspan: 1
        },
        {
          id:0,
          subType: '合计',
          target: '息差',
          monthlyActualNum: null,
          monthlyAchieveRate: null,
          accumulateActualNum: null,
          accumulateAchieveRate: null,
          annualAchieveRate: null,
          rowspan: 1
        },
        // 其它行数据...
      ],
      //说明
      remark :undefined,
      remarkId:undefined,
      statusType:undefined,
      // 表单校验
      // rules: {
      //   monthlyActualNum: [{ required: true, message: '表格不能为空', trigger: 'blur' }],
      //   monthlyAchieveRate: [{ required: true, message: '表格不能为空', trigger: 'blur' }],
      //   accumulateActualNum: [{ required: true, message: '表格不能为空', trigger: 'blur' }],
      //   annualAchieveRate: [{ required: true, message: '表格不能为空', trigger: 'blur' }],
      //   accumulateAchieveRate: [{ required: true, message: '表格不能为空', trigger: 'blur' }]
      // }
      rules: {
        monthlyActualNum0: [
          { required: true, message: '本月实际数不能为空', trigger: 'blur' },
        ],
        monthlyAchieveRate0: [
          { required: true, message: '本月达成率不能为空', trigger: 'blur' },
        ],
        accumulateActualNum0: [
          { required: true, message: '累计实际数不能为空', trigger: 'blur' },
        ],
        accumulateAchieveRate0: [
          { required: true, message: '累计达成率不能为空', trigger: 'blur' },
        ],
        annualAchieveRate0: [
          { required: true, message: '全年达成率不能为空', trigger: 'blur' },
        ],
      }

    };
  },
  created() {
    this.getList();
    // 遍历表格数据，生成校验规则
    // this.tableData.forEach((row, index) => {
    //   this.$set(this.rules, `monthlyActualNum${index}`, [
    //     { required: true, message: '本月实际数不能为空', trigger: 'blur' },
    //   ]);
    //   this.$set(this.rules, `monthlyAchieveRate${index}`, [
    //     { required: true, message: '本月达成率不能为空', trigger: 'blur' },
    //   ]);
    //   this.$set(this.rules, `accumulateActualNum${index}`, [
    //     { required: true, message: '累计实际数不能为空', trigger: 'blur' },
    //   ]);
    //   this.$set(this.rules, `accumulateAchieveRate${index}`, [
    //     { required: true, message: '累计达成率不能为空', trigger: 'blur' },
    //   ]);
    //   this.$set(this.rules, `annualAchieveRate${index}`, [
    //     { required: true, message: '全年达成率不能为空', trigger: 'blur' },
    //   ]);
    // });
  },
  methods: {
    /** 查询业务周报主表的增删查改列表 */
    getList() {
      this.loading = true;
      listWeeklyReport(this.queryParams).then(response => {
        this.weeklyReportList = response.rows;
        this.total = response.total;
        this.loading = false;
      });
    },
    // 取消按钮
    cancel() {
      this.open = false;
      this.reset();
    },
    // 表单重置
    reset() {
      this.tableData = [
        {
          id:0,
          subType: '立白供应商',
          target: '日均规模',
          monthlyActualNum: null,
          monthlyAchieveRate: null,
          accumulateActualNum: null,
          accumulateAchieveRate: null,
          annualAchieveRate: null,
          rowspan: 3
        },
        {
          id:0,
          subType: '立白供应商',
          target: '收入',
          monthlyActualNum: null,
          monthlyAchieveRate: null,
          accumulateActualNum: null,
          accumulateAchieveRate: null,
          annualAchieveRate: null,
          rowspan: 1
        },
        {
          id:0,
          subType: '立白供应商',
          target: '息差',
          monthlyActualNum: null,
          monthlyAchieveRate: null,
          accumulateActualNum: null,
          accumulateAchieveRate: null,
          annualAchieveRate: null,
          rowspan: 1
        },
        {
          id:0,
          subType: '医药港',
          target: '日均规模',
          monthlyActualNum: null,
          monthlyAchieveRate: null,
          accumulateActualNum: null,
          accumulateAchieveRate: null,
          annualAchieveRate: null,
          rowspan: 3
        },
        {
          id:0,
          subType: '医药港',
          target: '收入',
          monthlyActualNum: null,
          monthlyAchieveRate: null,
          accumulateActualNum: null,
          accumulateAchieveRate: null,
          annualAchieveRate: null,
          rowspan: 1
        },
        {
          id:0,
          subType: '医药港',
          target: '息差',
          monthlyActualNum: null,
          monthlyAchieveRate: null,
          accumulateActualNum: null,
          accumulateAchieveRate: null,
          annualAchieveRate: null,
          rowspan: 1
        },
        {
          id:0,
          subType: '非助贷业务',
          target: '日均规模',
          monthlyActualNum: null,
          monthlyAchieveRate: null,
          accumulateActualNum: null,
          accumulateAchieveRate: null,
          annualAchieveRate: null,
          rowspan: 3
        },
        {
          id:0,
          subType: '非助贷业务',
          target: '收入',
          monthlyActualNum: null,
          monthlyAchieveRate: null,
          accumulateActualNum: null,
          accumulateAchieveRate: null,
          annualAchieveRate: null,
          rowspan: 1
        },
        {
          id:0,
          subType: '非助贷业务',
          target: '息差',
          monthlyActualNum: null,
          monthlyAchieveRate: null,
          accumulateActualNum: null,
          accumulateAchieveRate: null,
          annualAchieveRate: null,
          rowspan: 1
        },
        {
          id:0,
          subType: '担保助贷',
          target: '日均规模',
          monthlyActualNum: null,
          monthlyAchieveRate: null,
          accumulateActualNum: null,
          accumulateAchieveRate: null,
          annualAchieveRate: null,
          rowspan: 3
        },
        {
          id:0,
          subType: '担保助贷',
          target: '收入',
          monthlyActualNum: null,
          monthlyAchieveRate: null,
          accumulateActualNum: null,
          accumulateAchieveRate: null,
          annualAchieveRate: null,
          rowspan: 1
        },
        {
          id:0,
          subType: '担保助贷',
          target: '息差',
          monthlyActualNum: null,
          monthlyAchieveRate: null,
          accumulateActualNum: null,
          accumulateAchieveRate: null,
          annualAchieveRate: null,
          rowspan: 1
        },
        {
          id:0,
          subType: '合计',
          target: '日均规模',
          monthlyActualNum: null,
          monthlyAchieveRate: null,
          accumulateActualNum: null,
          accumulateAchieveRate: null,
          annualAchieveRate: null,
          rowspan: 3
        },
        {
          id:0,
          subType: '合计',
          target: '收入',
          monthlyActualNum: null,
          monthlyAchieveRate: null,
          accumulateActualNum: null,
          accumulateAchieveRate: null,
          annualAchieveRate: null,
          rowspan: 1
        },
        {
          id:0,
          subType: '合计',
          target: '息差',
          monthlyActualNum: null,
          monthlyAchieveRate: null,
          accumulateActualNum: null,
          accumulateAchieveRate: null,
          annualAchieveRate: null,
          rowspan: 1
        },
        // 其它行数据...
      ];
      this.detailForm = {
        id: undefined,
        fkId: undefined,
        department: undefined,
        subType: undefined,
        target: undefined,
        monthlyActualNum: undefined,
        monthlyAchieveRate: undefined,
        accumulateActualNum: undefined,
        accumulateAchieveRate: undefined,
        annualAchieveRate: undefined,
        remark: undefined,
        createBy: undefined,
        createTime: undefined,
        updateBy: undefined,
        updateTime: undefined
      };
      this.remark=undefined;
      this.remarkId=undefined;
      this.statusType=undefined;
      this.resetForm("form");
      this.resetForm("detailForm");
    },
    /** 搜索按钮操作 */
    handleQuery() {
      this.queryParams.pageNum = 1;
      this.getList();
    },
    /** 重置按钮操作 */
    resetQuery() {
      this.resetForm("queryForm");
      this.handleQuery();
    },
    // 多选框选中数据
    handleSelectionChange(selection) {
      this.ids = selection.map(item => item.id)
      this.single = selection.length!==1
      this.multiple = !selection.length
    },
    /** 新增按钮操作 */
    handleAdd() {
      this.reset();
      this.tableData[0].id=null;
      this.open = true;
      this.title = "添加业务周报";
    },
    /** 修改按钮操作 */
    handleUpdate(row) {
      this.loading = true;
      this.reset();
      this.statusType=row.statusType
      const id = row.id || this.ids
      getEchoData(id).then(response => {
        this.loading = false;
        this.tableData = response.data;
        this.remark = response.data[response.data.length-1].remark;
        this.remarkId = response.data[response.data.length-1].id;
        this.open = true;
        this.title = "修改业务周报";
      });
    },
    /** 提交按钮 */
    submitForm() {
      if (this.statusType == 1) {
        this.msgError('已提交，无法更改')
      } else{
        this.$refs['tableData'].validate(valid => {
          if(valid){
            this.detailForm=this.tableData[0]
            if (this.detailForm.id != null) {
              this.buttonLoading = true;
              // this.detailForm=this.tableData[0]
              this.form.id=this.detailForm.fkId;
              this.form.statusType=1;
              updateWeeklyReport(this.form).then(response => {
                let detailFormList = [];
                for(let item of this.tableData){
                  this.detailForm.id=item.id
                  this.detailForm.department='供应链金融战略业务';
                  this.detailForm.subType=item.subType;
                  this.detailForm.target=item.target;
                  this.detailForm.monthlyActualNum=item.monthlyActualNum;
                  this.detailForm.monthlyAchieveRate=item.monthlyAchieveRate;
                  this.detailForm.accumulateActualNum=item.accumulateActualNum;
                  this.detailForm.accumulateAchieveRate=item.accumulateAchieveRate;
                  this.detailForm.annualAchieveRate=item.annualAchieveRate;
                  let tmp = { ...this.detailForm };
                  detailFormList.push(tmp);
                  // addDetail(this.detailForm);
                }
                //添加说明
                this.detailForm.department='供应链金融战略业务';
                this.detailForm.subType='说明';
                this.detailForm.target='说明';
                this.detailForm.remark=this.remark;
                this.detailForm.id=this.remarkId;
                this.detailForm.monthlyActualNum=null;
                this.detailForm.monthlyAchieveRate=null;
                this.detailForm.accumulateActualNum=null;
                this.detailForm.accumulateAchieveRate=null;
                this.detailForm.annualAchieveRate=null;
                //addDetail(this.detailForm);
                detailFormList.push(this.detailForm);
                updateBatchDetail(detailFormList);

                this.buttonLoading = false;
                this.msgSuccess("修改成功");
                this.open = false;
                this.getList();
              }).finally(() => {
                this.buttonLoading = false
              });
            } else {
              this.buttonLoading = true;
              // this.detailForm=this.tableData[0]
              this.form.department='供应链金融战略业务';
              this.form.statusType=1;
              let fkId=1;
              addWeeklyReport(this.form)
                  .then(response => {
                    fkId=response.data;
                    let detailFormList = [];
                    for(let item of this.tableData){
                      this.detailForm.fkId=fkId;
                      this.detailForm.department='供应链金融战略业务';
                      this.detailForm.subType=item.subType;
                      this.detailForm.target=item.target;
                      this.detailForm.monthlyActualNum=item.monthlyActualNum;
                      this.detailForm.monthlyAchieveRate=item.monthlyAchieveRate;
                      this.detailForm.accumulateActualNum=item.accumulateActualNum;
                      this.detailForm.accumulateAchieveRate=item.accumulateAchieveRate;
                      this.detailForm.annualAchieveRate=item.annualAchieveRate;
                      let tmp = { ...this.detailForm };
                      detailFormList.push(tmp);
                      // addDetail(this.detailForm);
                    }
                    //添加说明
                    this.detailForm.fkId=fkId;
                    this.detailForm.department='供应链金融战略业务';
                    this.detailForm.subType='说明';
                    this.detailForm.target='说明';
                    this.detailForm.remark=this.remark;
                    this.detailForm.monthlyActualNum=null;
                    this.detailForm.monthlyAchieveRate=null;
                    this.detailForm.accumulateActualNum=null;
                    this.detailForm.accumulateAchieveRate=null;
                    this.detailForm.annualAchieveRate=null;
                    //addDetail(this.detailForm);
                    detailFormList.push(this.detailForm);
                    addBatchDetail(detailFormList);
                    this.buttonLoading = false;
                    this.open = false;
                    this.getList();
                    this.msgSuccess("新增成功");
                    this.buttonLoading = false;
                  });

            }
          }
        })
      }
    },
    //保存草稿按钮
    submitForm1() {
      if (this.statusType == 1) {
        this.msgError('已提交，无法更改')
      } else{
        this.detailForm=this.tableData[0]
        if (this.detailForm.id != null) {
          this.buttonLoading = true;
          // this.detailForm=this.tableData[0]
          this.form.id=this.detailForm.fkId;
          this.form.statusType=0;
          updateWeeklyReport(this.form).then(response => {
            let detailFormList = [];
            for(let item of this.tableData){
              this.detailForm.id=item.id
              this.detailForm.department='供应链金融战略业务';
              this.detailForm.subType=item.subType;
              this.detailForm.target=item.target;
              this.detailForm.monthlyActualNum=item.monthlyActualNum;
              this.detailForm.monthlyAchieveRate=item.monthlyAchieveRate;
              this.detailForm.accumulateActualNum=item.accumulateActualNum;
              this.detailForm.accumulateAchieveRate=item.accumulateAchieveRate;
              this.detailForm.annualAchieveRate=item.annualAchieveRate;
              let tmp = { ...this.detailForm };
              detailFormList.push(tmp);
              // addDetail(this.detailForm);
            }
            //添加说明
            this.detailForm.department='供应链金融战略业务';
            this.detailForm.subType='说明';
            this.detailForm.target='说明';
            this.detailForm.remark=this.remark;
            this.detailForm.id=this.remarkId;
            this.detailForm.monthlyActualNum=null;
            this.detailForm.monthlyAchieveRate=null;
            this.detailForm.accumulateActualNum=null;
            this.detailForm.accumulateAchieveRate=null;
            this.detailForm.annualAchieveRate=null;
            //addDetail(this.detailForm);
            detailFormList.push(this.detailForm);
            updateBatchDetail(detailFormList);

            this.buttonLoading = false;
            this.msgSuccess("修改成功");
            this.open = false;
            this.getList();
          }).finally(() => {
            this.buttonLoading = false
          });
        } else {
          this.buttonLoading = true;
          // this.detailForm=this.tableData[0]
          this.form.department='供应链金融战略业务';
          this.form.statusType=0;
          let fkId=1;
          addWeeklyReport(this.form)
              .then(response => {
                fkId=response.data;
                let detailFormList = [];
                for(let item of this.tableData){
                  this.detailForm.fkId=fkId;
                  this.detailForm.department='供应链金融战略业务';
                  this.detailForm.subType=item.subType;
                  this.detailForm.target=item.target;
                  this.detailForm.monthlyActualNum=item.monthlyActualNum;
                  this.detailForm.monthlyAchieveRate=item.monthlyAchieveRate;
                  this.detailForm.accumulateActualNum=item.accumulateActualNum;
                  this.detailForm.accumulateAchieveRate=item.accumulateAchieveRate;
                  this.detailForm.annualAchieveRate=item.annualAchieveRate;
                  let tmp = { ...this.detailForm };
                  detailFormList.push(tmp);
                  // addDetail(this.detailForm);
                }
                //添加说明
                this.detailForm.fkId=fkId;
                this.detailForm.department='供应链金融战略业务';
                this.detailForm.subType='说明';
                this.detailForm.target='说明';
                this.detailForm.remark=this.remark;
                this.detailForm.monthlyActualNum=null;
                this.detailForm.monthlyAchieveRate=null;
                this.detailForm.accumulateActualNum=null;
                this.detailForm.accumulateAchieveRate=null;
                this.detailForm.annualAchieveRate=null;
                //addDetail(this.detailForm);
                detailFormList.push(this.detailForm);
                addBatchDetail(detailFormList);
                this.buttonLoading = false;
                this.open = false;
                this.getList();
                this.msgSuccess("新增成功");
                this.buttonLoading = false;
              });

        }
      }
    }
  }
};
</script>

<style lang="scss" scoped>
.table {
  width: 100%;
  text-align: center;
}

.table > tr > th {
  background-color: #ccc;
}

th, td {
  line-height: 50px;
  border: 1px solid #ccc;
}

.form-content {
  width: 90%;
  margin: 0 auto;
}

.table {
  .el-form-item {
    ::v-deep .el-form-item__content {
      margin-left: 0 !important;
    }
  }
}
</style>
