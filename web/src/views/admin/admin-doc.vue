<template>
  <a-layout>
    <a-layout-content
        :style="{ background: '#fff', padding: '24px', margin: 0, minHeight: '280px' }"
    >
      <a-row :gutter="24">
        <a-col :span="8">
          <p>
            <a-form layout="inline" :model="param">
              <a-form-item>
                <a-button type="primary" @click="handleQuery">
                  查询
                </a-button>
              </a-form-item>
              <a-form-item>
                <a-button type="primary" @click="add">
                  新增
                </a-button>
              </a-form-item>
            </a-form>
          </p>
          <a-table
              v-if="level1.length > 0"
              :columns="columns"
              :row-key="record => record.id"
              :data-source="level1"
              :loading="loading"
              :pagination="false"
              size="small"
              :defaultExpandAllRows="true"
          >
            <template #name="{ text, record }">
              {{record.sort}} {{text}}
            </template>
            <template v-slot:action="{ text, record }">
              <a-space size="small">
                <a-button type="primary" @click="edit(record)" size="small">
                  编辑
                </a-button>
                <a-popconfirm
                    title="删除后不可恢复，确认删除？"
                    ok-text="是"
                    cancel-text="否"
                    @confirm="handleDelete(record.id)"
                >
                  <a-button type="danger" size="small">
                    删除
                  </a-button>
                </a-popconfirm>

              </a-space>
            </template>
          </a-table>
        </a-col>
        <a-col :span="16">
          <p>
            <a-form layout="inline" :model="param">
              <a-form-item>
                <a-button type="primary" @click="handleSave()">
                  保存
                </a-button>
              </a-form-item>
            </a-form>
          </p>
          <a-form :model="doc" layout="vertical">
            <a-form-item>
              <a-input v-model:value="doc.name" placeholder="名称"/>
            </a-form-item>
            <a-form-item>
              <a-tree-select
                  v-model:value="doc.parent"
                  style="width: 100%"
                  :dropdown-style="{ maxHeight: '400px', overflow: 'auto' }"
                  :tree-data="treeSelectData"
                  placeholder="请选择父文档"
                  tree-default-expand-all
                  :replaceFields="{title: 'name', key: 'id', value: 'id'}"
              >

              </a-tree-select>
            </a-form-item>
            <a-form-item>
              <a-input v-model:value="doc.sort" placeholder="顺序" />
            </a-form-item>
            <a-form-item>
              <div id="div1"></div>
            </a-form-item>
          </a-form>
        </a-col>
      </a-row>

    </a-layout-content>
  </a-layout>
<!--  <a-modal v-model:visible="modalVisible" title="文档表单" @ok="handleModalOk" :confirm-loading="modalLoading">-->
<!--  </a-modal>-->
</template>

<script lang="ts">
import { defineComponent, onMounted, ref, createVNode } from 'vue';
import axios from 'axios';
import { message, Modal } from 'ant-design-vue'
import {Tool} from "@/util/tool";
import {useRoute} from "vue-router";
import ExclamationCircleOutlined from "@ant-design/icons-vue/ExclamationCircleOutlined";
import E from 'wangeditor'


export default defineComponent({
  name: 'AdminDoc',
  setup() {
    const route = useRoute()
    const param = ref()
    param.value = {}
    const docs = ref();

    const loading = ref(false);

    const columns = [
      {
        title: '名称',
        dataIndex: 'name',
        slots: {customRender: 'name'}
      },
      {
        title: 'Action',
        key: 'action',
        slots: { customRender: 'action' }
      }
    ];
    const level1 = ref()
    level1.value = []

    /**
     * 数据查询
     **/
    const handleQuery = () => {
      loading.value = true;

      level1.value = [];

      axios.get("/doc/all").then((response) => {
        loading.value = false;
        const data = response.data;
        if (data.success) {
          docs.value = data.content
          level1.value = [];
          level1.value = Tool.array2Tree(docs.value, 0)
        } else {
          message.error(data.message)
        }
      });
    };


    //表单
    //因为树选择组件的树形状态，会随当前编辑的节点而变化，所以单独声明一个响应式变量
    const treeSelectData = ref()
    treeSelectData.value = []
    const doc = ref({})
    const modalVisible = ref(false)
    const modalLoading = ref(false)

    const handleSave = () => {
      modalLoading.value = true;
      axios.post("/doc/save", doc.value).then((response) => {
        const data = response.data;
        modalLoading.value = false;
        if (data.success) {
          modalVisible.value = false;

          //重新加载列表
          handleQuery();
        } else {
          message.error(data.message)
        }
      });
    }

    /**
     * 将某节点及其子节点全部置为disable
     */
    const setDisable = (treeSelectData: any, id: any) => {
      for (let i = 0; i < treeSelectData.length; i++) {
        const node = treeSelectData[i];
        if (node.id === id) {
          node.disabled = true;

          const children = node.children;
          if (Tool.isNotEmpty(children)) {
            for (let j = 0; j < children.length; j++) {
              setDisable(children, children[j].id)
            }
          }
        } else {
          const children = node.children;
          if (Tool.isNotEmpty(children)) {
            setDisable(children, id);
          }
        }
      }
    }

    /**
     * 查找整根树枝
     */
    const deleteIds: Array<string> = [];
    const deleteNames: Array<string> = [];

    const getDeleteIds = (treeSelectData: any, id: any) => {
      for (let i = 0; i < treeSelectData.length; i++) {
        const node = treeSelectData[i];
        if (node.id === id) {
          deleteIds.push(id)
          deleteNames.push(node.name);

          const children = node.children;
          if (Tool.isNotEmpty(children)) {
            for (let j = 0; j < children.length; j++) {
              getDeleteIds(children, children[j].id)
            }
          }
        } else {
          const children = node.children;
          if (Tool.isNotEmpty(children)) {
            getDeleteIds(children, id);
          }
        }
      }
    }
     /**
     * 编辑
     */
    const edit = (record: any) => {
      modalVisible.value = true;
      doc.value = Tool.copy(record);

      //不能选择当前节点及其所有子孙节点，作为父节点，会使树断开
      treeSelectData.value = Tool.copy(level1.value);
      setDisable(treeSelectData.value, record.id);

       // 为选择树添加一个"无"
      treeSelectData.value.unshift({id: 0, name: '无'});
    }
    /**
     * 新增
     */
    const add = () => {
      modalVisible.value = true;
      doc.value = {
        ebookId: route.query.ebookId
      }

      treeSelectData.value = Tool.copy(level1.value);

      treeSelectData.value.unshift({id: 0, name: '无'});
    }
    /**
     * 删除
     */
    const handleDelete = (id: number) => {
      deleteIds.length = 0;
      deleteNames.length = 0;

      getDeleteIds(level1.value, id)

      Modal.confirm({
        title: '重要提醒',
        icon: createVNode(ExclamationCircleOutlined),
        content: '将删除：【' + deleteNames.join("，") + "】删除后不可恢复，确认删除？",
        onOk() {
          // console.log(ids)
          axios.delete("/doc/delete/" + deleteIds.join(",")).then((response) => {
            const data = response.data; // data = commonResp
            if (data.success) {
              // 重新加载列表
              handleQuery();
            } else {
              message.error(data.message);
            }
          });
        },
      });
    }


    onMounted(() => {
      handleQuery();
      const editor = new E('#div1')
      editor.config.zIndex = 0;
      editor.create();
    });

    return {
      level1,
      columns,
      loading,
      edit,
      modalVisible,
      modalLoading,
      handleSave,
      doc,
      add,
      handleDelete,
      handleQuery,
      param,
      treeSelectData
    }
  }
});
</script>

<style scoped>
img {
  width: 50px;
  height: 50px;
}
</style>
