<template>
  <div id="app">
    <el-form ref="form" :rules="rules" :model="form">
      <el-form-item label="添加规格" prop="sku_arr">
        <div style="display: flex;">
          <el-popover placement="top" width="240" v-model="add_popover_bool">
            <div style="display: flex; grid-gap: 10px;">
              <el-input
                ref="addValueInput"
                v-model.trim="add_value"
                @keyup.enter.native="confirm()"
              />
              <el-button type="primary" @click="confirm()">确定</el-button>
            </div>

            <el-button
              slot="reference"
              size="small"
              type="primary"
              :disabled="attrBtnDisabled"
              style="margin-left: 40px;"
              >自定义</el-button
            >
          </el-popover>

          <el-button
            type="primary"
            @click="onSubmit()"
            style="margin-left: 100px;"
            >提交</el-button
          >
        </div>
      </el-form-item>

      <!-- 规格列表 和 表格 -->
      <section style="margin: 0 0 20px 50px;">
        <!-- 展示已经选择的 -->
        <div
          v-for="(item, index) in form.sku_arr"
          :key="index"
          style="margin-top: 10px;"
        >
          <!-- 属性 -->
          <div>
            {{ item.attr }}
            <el-button
              type="danger"
              size="mini"
              icon="el-icon-delete"
              circle
              @click="deleteAttr(index)"
            >
            </el-button>
          </div>

          <!-- 属性值 -->
          <div style="display: flex; margin-top: 10px;">
            <div
              v-for="(ktem, kndex) in item.valueList"
              :key="kndex"
              style="margin-right: 20px;"
            >
              <el-input
                size="small"
                ref="attrValueInput"
                v-model.trim="item.valueList[kndex].name"
                placeholder="值"
                style="width: 100px;"
                @focus="attrValueFocus(item.valueList[kndex].name)"
                @blur="newAttrValueBlur(item.attr, item.valueList[kndex])"
              ></el-input>

              <el-button
                v-if="item.valueList.length > 1"
                type="danger"
                size="mini"
                icon="el-icon-delete"
                circle
                @click="
                  deleteSmall(index, kndex, item.attr, item.valueList[kndex])
                "
              />
            </div>
            <el-button
              type="primary"
              size="mini"
              :disabled="item.valueList.includes('')"
              icon="el-icon-plus"
              @click="addAttributeValue(index)"
              >添加值</el-button
            >
          </div>
        </div>

        <el-table
          :data="form.table_data"
          :span-method="spanMethod"
          style="margin-top: 20px;"
          border
        >
          <template v-for="item_column in table_column">
            <el-table-column
              v-if="item_column == '图片'"
              :key="item_column"
              min-width="150"
              width="170"
              align="center"
              :resizable="false"
              label="图片"
            >
              <template>
                <!-- <el-form-item :prop="`table_data.${ $index }.图片`" :rules="rules.sku_img" label-width="0"> -->
                图片组件
                <!-- </el-form-item> -->
              </template>
            </el-table-column>

            <!-- 销售价格 使用表单验证 和 自定义表头 -->
            <el-table-column
              v-else-if="item_column == '销售价格'"
              :key="item_column"
              align="center"
              :resizable="false"
            >
              <!-- 自定义表头 -->
              <template slot="header">
                <div><span style="color: #ff5353;">*</span>销售价格</div>
              </template>

              <template slot-scope="{ row, $index }">
                <el-form-item
                  :prop="`table_data.${$index}.销售价格`"
                  label-width="0"
                  label=" "
                >
                  <el-input
                    v-model="row[item_column]"
                    :placeholder="item_column"
                  />
                </el-form-item>
              </template>
            </el-table-column>

            <!-- 市场价格 -->
            <el-table-column
              v-else-if="item_column == '市场价格'"
              :key="item_column"
              align="center"
              :resizable="false"
              :label="item_column"
            >
              <template slot-scope="{ row }">
                <div style="height: 62px;">
                  <el-input
                    v-model="row[item_column]"
                    :placeholder="item_column"
                  />
                </div>
              </template>
            </el-table-column>

            <!-- 库存 -->
            <el-table-column
              v-else-if="item_column == '库存'"
              :key="item_column"
              align="center"
              :resizable="false"
              :label="item_column"
            >
              <template slot-scope="{ row }">
                <div style="height: 62px;">
                  <el-input
                    v-model="row[item_column]"
                    :placeholder="item_column"
                  />
                </div>
              </template>
            </el-table-column>

            <!-- 其他属性列 -->
            <el-table-column
              v-else
              align="center"
              :resizable="false"
              :key="item_column"
              :prop="item_column"
              :label="item_column"
            >
            </el-table-column>
          </template>
        </el-table>
      </section>
    </el-form>
  </div>
</template>

<script>
let base_column = ["销售价格", "市场价格", "库存"]; // 基本的列

let first_column_rule = []; // 第一列与第二列使用相同的合并规则 (不能存在data中)
let old_attr = ""; // 每次当属性获得焦点时都会获取输入框内的值，保存于此
let old_attr_value = ""; // 每次当属性值获得焦点时都会获取输入框内的值，保存于此
export default {
  data() {
    return {
      maxGroupId: 0,
      default_attr: ["颜色", "尺寸", "重量"], // 默认规格
      table_column: base_column, // 表格列
      add_popover_bool: false, // 添加属性的小弹窗
      add_item_popover_bool: false,
      add_value: "", // 添加属性的
      add_item: "", // 添加属性子项的
      // 上边的数据是录入sku相关

      // 表单
      form: {
        sku_arr: [],
        table_data: [], // 表格中的数据
      },

      // 验证规则
      rules: {
        // sku 相关验证
        sku_arr: {
          validator: (rule, value, callback) => {
            if (value.length === 0) return callback(new Error("请添加规格"));
            else return callback();
          },
          trigger: "blur",
        },
        sku_img: [
          {
            required: true,
            message: "图片不能为空",
            trigger: "blur",
          },
          {
            type: "string",
            message: "请等待图片上传完毕",
            trigger: "blur",
          },
        ],
        sku_sale_price: {
          required: true,
          message: "价格不能为空",
          trigger: "blur",
        },
      },
    };
  },
  methods: {
    // 点击自定义里的确定 添加新的规格
    confirm() {
      if (!this.add_value) return;
      this.form.sku_arr.push({
        attr: this.add_value,
        valueList: [{ itemId: 0 }],
        groupId: this.maxGroupId,
      });
      this.maxGroupId += 1;
      this.generateTableColumn();
      this.traverseSku();

      this.add_popover_bool = false;
      this.add_value = "";
    },
    // 添加子项
    confirmItem(idx) {
      if (!this.add_item) return;
      if (this.form.sku_arr[idx].valueList.length < 1) {
        this.form.sku_arr[idx].valueList.push({
          name: this.add_item,
          itemId: 0,
        });
      } else {
        this.form.sku_arr[idx].valueList.push({
          name: this.add_item,
          itemId: this.form.sku_arr[idx].valueList.slice(-1)[0].itemId + 1,
        });
      }

      this.generateTableColumn();
      this.traverseSku();
      this.add_item_popover_bool = false;
      this.add_item = "";
    },
    // 属性获得焦点时 得到旧的值 存一下
    attrFocus(oldVal) {
      old_attr = oldVal;
    },
    // 属性失去焦点时
    attrBlur(newVal) {
      // 如果 '新值等于旧值' 或者 '空' 什么也不做
      if (newVal === old_attr || newVal === "") return;
      // 生成处理表头数据和表格数据
      this.generateTableColumn();
      this.traverseSku();
    },
    // 删除属性
    deleteAttr(idx) {
      // this.form.sku_arr.splice(idx, 1);
      this.$delete(this.form.sku_arr, idx);
      // 生成处理表头数据和表格数据
      this.generateTableColumn();
      this.traverseSku();
    },

    // 添加属性值
    async addAttributeValue(idx) {
      const length = this.form.sku_arr[idx].valueList[
        this.form.sku_arr[idx].valueList.length - 1
      ].itemId;
      this.form.sku_arr[idx].valueList.push({
        itemId: length + 1,
      });
      // 让新增的输入框获得焦点
      await this.$nextTick();
      this.$refs.attrValueInput[this.$refs.attrValueInput.length - 1].focus();
    },
    // 属性值获得焦点时 得到旧的值 在输入框失去焦点的时候, 如果值没有变化, 则什么也不做
    attrValueFocus(oldVal) {
      console.log(oldVal);
      old_attr_value = oldVal;
    },
    // 属性值失去焦点时, 操作表格数据 (新版本 可以实现无限个规格)
    newAttrValueBlur(curr_attr, newVal) {
      console.log("失去焦点");
      if (newVal === old_attr_value) return;
      //  这里根据规格生成的笛卡尔积计算出table中需要改变的行索引 ( 包含新增和修改 )
      let cartesian_arr = this.generateBaseData(this.form.sku_arr);
      console.log(cartesian_arr);
      let change_idx_arr = []; // 需要被改变的行的索引
      for (let i in cartesian_arr) {
        if (cartesian_arr[i][curr_attr] === newVal.name)
          change_idx_arr.push(Number(i));
      }
      // 新的表格应该有的长度与现有的表格长度比较, 区分新增还是修改
      let length_arr = this.form.sku_arr.map((x) => x.valueList.length);
      let new_table_length = length_arr.reduce(
        (acc, curr_item) => acc * curr_item
      ); // 新的表格数据长度 求乘积
      let old_table_length = this.form.table_data.length; // 旧的表格数据长度
      // 如果是修改
      if (new_table_length === old_table_length) {
        this.form.table_data.forEach((item, index) => {
          if (change_idx_arr.includes(index))
            this.form.table_data[index][curr_attr] = newVal.name;
        });
        return;
      }
      // 如果是新增
      if (new_table_length > old_table_length) {
        // 得到当前属性的当前值和其他规格的 sku_arr, 构造新的表格数据
        let other_sku_arr = this.form.sku_arr.map((item) => {
          if (item.attr !== curr_attr) return item;
          else
            return {
              attr: item.attr,
              valueList: [newVal],
            };
        });
        // 得到新增的表格数据
        let ready_map = this.generateBaseData(other_sku_arr);
        let new_table_data = this.mergeTableData(ready_map);
        change_idx_arr.forEach((item_idx, index) => {
          this.form.table_data.splice(item_idx, 0, new_table_data[index]);
        });
      }
    },
    // 删除属性值 四个参数：'一级数组索引', '二级索引', '属性名字', '属性值'  后两个参数用来删除行
    deleteSmall(idx, kdx, attr_name, attr_val) {
      this.form.sku_arr[idx].valueList.splice(kdx, 1);
      let data_length = this.form.table_data.length;
      for (let i = 0; i < data_length; i++) {
        console.log(this.form.table_data[i][attr_name]);
        console.log(attr_val);
        if (this.form.table_data[i][attr_name] == attr_val.name) {
          this.form.table_data.splice(i, 1);
          i = i - 1;
          data_length = data_length - 1;
        }
      }
      console.log(this.form.table_data);
    },

    // 根据 `this.form.sku_arr` 生成表格列, `table_column` 用于 el-table-column 的 v-for
    async generateTableColumn() {
      this.table_column = this.form.sku_arr
        .map((x) => x.attr)
        .concat(base_column);
      /*
                        	不写 `$nextTick`会有bug, 没想明白为啥, 大概是vue懒得更新dom吧
                        	bug复现方式: 删除`$nextTick`后，勾选颜色，再勾选尺寸，再取消勾选颜色，观察el-table
                        */
      await this.$nextTick();
      if (this.form.sku_arr.length != 0) this.table_column.splice(1, 0, "图片");
    },

    // 合并行
    spanMethod({ row, column, rowIndex, columnIndex }) {
      if (columnIndex == 0) {
        let key_0 = column.label;
        let first_idx = this.form.table_data.findIndex(
          (x) => x[key_0] == row[key_0]
        );
        const calcSameLength = () =>
          this.form.table_data.filter((x) => x[key_0] == row[key_0]).length;
        first_column_rule =
          rowIndex == first_idx ? [calcSameLength(), 1] : [0, 0];
        return first_column_rule;

        // 第二列的图片与第一列主规格使用相同合并规则 ( 恰好el-table的合并方法是横着走的 )
      } else if (columnIndex == 1) {
        return first_column_rule;

        // 其他列
      } else {
        // 表格数据的每一项,
        const callBacks = (table_item, start_idx = 0) => {
          if (columnIndex < start_idx) return true;
          let curr_key = this.table_column[start_idx];
          return (
            table_item[curr_key] === row[curr_key] &&
            callBacks(table_item, ++start_idx)
          );
        };
        let first_idx = this.form.table_data.findIndex((x) => callBacks(x));
        const calcSameLength = () =>
          this.form.table_data.filter((x) => callBacks(x)).length;
        return rowIndex == first_idx ? [calcSameLength(), 1] : [0, 0];
      }
    },
    // 合并 sku 与 '图片', '销售价格', '库存', '市场价格' , 返回整个表格数据数组
    mergeTableData(arr) {
      return arr.map((item) => ({
        ...item,
        销售价格: "",
        市场价格: "",
        库存: "",
        图片: "",
      }));
    },
    // 遍历 `sku_arr` 生成表格数据
    traverseSku() {
      let ready_map = this.generateBaseData(this.form.sku_arr);
      this.form.table_data = this.mergeTableData(ready_map);
    },
    // 重新实现笛卡尔积  入参是: this.form.sku_arr 传入的数组 '为空', '长度为1', '长度大于1' 三种情况 分别处理
    generateBaseData(arr) {
      if (arr.length === 0) return [];
      if (arr.length === 1) {
        let [item_spec] = arr;
        return item_spec.valueList.map((x) => {
          return {
            [item_spec.attr]: x.name,
          };
        });
      }
      if (arr.length >= 1) {
        return arr.reduce((accumulator, spec_item) => {
          let acc_value_list = Array.isArray(accumulator.valueList)
            ? accumulator.valueList.map((item) => {
                return item.name;
              })
            : accumulator;
          let item_value_list = spec_item.valueList;
          let result = [];
          for (let i in acc_value_list) {
            for (let j in item_value_list) {
              let temp_data = {};
              // 如果是对象
              if (acc_value_list[i]?.constructor === Object) {
                temp_data = {
                  ...acc_value_list[i],
                  [spec_item.attr]: item_value_list[j].name,
                };
                // 否则如果是字符串
              } else {
                temp_data[accumulator.attr] = acc_value_list[i];
                temp_data[spec_item.attr] = item_value_list[j].name;
              }
              result.push(temp_data);
            }
          }
          return result;
        });
      }
    },

    onSubmit() {
      console.log(this.form.sku_arr);
      console.log(this.form.table_data);
      this.$refs.form.validate(async (valid, object) => {
        if (!valid) {
          // 获取元素距离body顶部的距离
          let getTop = (dom) => {
            let top = dom.offsetTop;
            // while括号里的值是 dom.offsetParent
            while ((dom = dom.offsetParent)) top = top + dom.offsetTop;
            return top;
          };
          let [first_prop] = Object.keys(object);
          let top = getTop(
            document.querySelector(`label[for='${first_prop}']`)
          );
          window.scrollTo({
            top: top - 70,
            behavior: "smooth",
          });
          return;
        }
        this.btn_loading = true;
      });
    },
  },
  mounted() {
    this.form.sku_arr.push({
      attr: "颜色",
      groupId: 0,
      valueList: [
        { itemId: 0, name: "白色" },
        { itemId: 1, name: "绿色" },
        { itemId: 2, name: "蓝色" },
      ],
    });
  },
  computed: {
    // 已添加的属性(字符串数组)
    selectedAttr() {
      return this.form.sku_arr.map((x) => x.attr);
    },
    // 是否可以添加属性 最多两个属性
    attrBtnDisabled() {
      return false;
      // return this.form.sku_arr.length >= 2;
    },
  },
};
</script>
<style scoped>
label[for*="table_data"] {
  visibility: hidden;
  margin-top: -40px;
}
</style>
