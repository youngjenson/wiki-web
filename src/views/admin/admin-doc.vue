<template>
  <a-layout>
    <a-layout-content
        :style="{ background: '#fff', padding: '24px', margin: 0, minHeight: '280px' }"
    >
      <p>
        <a-form layout="inline" :model="param">
          <a-form-item>
            <a-button type="primary" @click="add()">
              新增
            </a-button>
          </a-form-item>
        </a-form>
      </p>
      <a-table
          :columns="columns"
          :data-source="docs"
          :row-key="record => record.id"
          :loading="loading"
          :pagination="false"
          :expandRowByClick="true"
      >
        <template v-slot:action="{ text: record }">
          <a-space size="small">
            <a-button type="primary" @click="edit(record)">编辑</a-button>
            <a-popconfirm
                title="确认要删除该条数据吗?"
                ok-text="是"
                cancel-text="否"
                @confirm="showConfirm(record.id)"
                @cancel="cancel"
            >
              <a-button type="primary" danger>删除</a-button>
            </a-popconfirm>
          </a-space>
        </template>
      </a-table>
    </a-layout-content>
  </a-layout>

  <a-modal
      title="文档表单"
      v-model:visible="modalVisible"
      :confirm-loading="modalLoading"
      @ok="handleModalOk"
  >
    <a-form :model="doc" :label-col="{span: 6}" :wrapper-col="{span: 18}">
      <a-form-item label="名称">
        <a-input v-model:value="doc.name"/>
      </a-form-item>
      <a-form-item label="父文档">
        <a-tree-select
            v-model:value="doc.parent"
            style="width: 100%"
            :dropdown-style="{ maxHeight: '400px', overflow: 'auto' }"
            placeholder="请选择父文档"
            tree-default-expand-all
            :tree-data="treeSelectData"
            :field-names="{label:'name',children: 'children', value: 'id'}"
        >
        </a-tree-select>
      </a-form-item>
      <a-form-item label="顺序">
        <a-input v-model:value="doc.sort"/>
      </a-form-item>
      <a-form-item label="内容">
        <div id="content"></div>
      </a-form-item>
    </a-form>
  </a-modal>
</template>
<script lang="ts">
import {createVNode, defineComponent, onMounted, ref, toRaw} from "vue";
import axios from "axios";
import {message, Modal} from "ant-design-vue";
import {Tool} from "@/utils/tool";
import {useRoute} from "vue-router";
import {ExclamationCircleOutlined} from "@ant-design/icons-vue";
import E from 'wangeditor';

export default defineComponent({
  name: 'Admindoc',
  setup() {
    const route = useRoute()
    const docs = ref();

    const loading = ref(false);
    const columns = [
      {
        title: '名称',
        dataIndex: 'name',
        align: 'center'
      },
      {
        title: '父文档',
        key: 'parent',
        dataIndex: 'parent',
        align: 'center'
      },
      {
        title: '顺序',
        dataIndex: 'sort',
        align: 'center'
      },
      {
        title: '操作',
        key: 'action',
        align: 'center',
        slots: {customRender: 'action'}
      }
    ];

    /**
     * 数据查询
     */
    const handleQuery = () => {
      loading.value = true;
      axios.get("/doc/list?ebookId=" + route.query.ebookId).then(response => {
        loading.value = false;
        const data = response.data;
        if (data.code == 200) {
          docs.value = data.data;
        } else {
          message.error(data.message)
        }
      });
    };


    //------表单-----
    const treeSelectData = ref()
    treeSelectData.value = [];
    const doc = ref()
    const modalVisible = ref(false)
    const modalLoading = ref(false)
    let editor: any;
    const createEditor = () => {
      if(editor != null) editor.destroy();
      editor = new E('#content');
      editor.create();
    }

    const handleModalOk = () => {
      modalLoading.value = true;
      axios.post("/doc/edit", doc.value).then(response => {
        const data = response.data;
        if (data.code === 200) {
          modalVisible.value = false;
          modalLoading.value = false;
          message.success(data.message)
          handleQuery()
        } else {
          modalLoading.value = false;
          message.error(data.message)
        }
      })
    }
    /**
     * 将某个节点及其子孙节点全部置为disabled
     */
    const setDisable = (treeSelectData: any, id: any) => {
      for (let i = 0; i < treeSelectData.length; i++) {
        const node = treeSelectData[i]
        console.log(node)
        if (node.id === id) {
          console.log("disabled", node)
          node.disabled = true;
          const children = node.children
          if (Tool.isNotEmpty(children)) {
            for (let j = 0; j < children.length; j++) {
              setDisable(children, children[j].id)
            }
          }
        } else {
          const children = node.children
          if (Tool.isNotEmpty(children)) {
            setDisable(children, id)
          }
        }
      }
    };

    let ids: any[] = [];
    /**
     * 查找整根树枝
     */
    const getDeleteIds = (treeSelectData: any, id: any) => {
      for (let i = 0; i < treeSelectData.length; i++) {
        const node = treeSelectData[i]
        // console.log(node)
        if (node.id === id) {
          // console.log("disabled", node)
          // node.disabled = true;
          ids.push(id)
          const children = node.children
          if (Tool.isNotEmpty(children)) {
            for (let j = 0; j < children.length; j++) {
              getDeleteIds(children, children[j].id)
            }
          }
        } else {
          const children = node.children
          if (Tool.isNotEmpty(children)) {
            getDeleteIds(children, id)
          }
        }
      }
    };

    /**
     * 编辑
     */
    const edit = (record: any) => {
      modalVisible.value = true;
      doc.value = Tool.copy(record);

      treeSelectData.value = Tool.copy(docs.value);
      console.log(treeSelectData.value);
      setDisable(treeSelectData.value, record.id);
      treeSelectData.value.unshift({id: '0', name: '无'})
      setTimeout(function () {
        createEditor();
      }, 100)
    }

    /**
     * 新增
     */
    const add = () => {
      modalVisible.value = true;
      doc.value = {
        ebookId: route.query.ebookId
      }
      treeSelectData.value = Tool.copy(docs.value);
      if (Tool.isEmpty(treeSelectData.value)) treeSelectData.value = [];
      treeSelectData.value.unshift({id: 0, name: '无'})
      setTimeout(function () {
        createEditor()
      }, 100)
    }

    /**
     * 删除
     */
    const handleDelete = (id: any) => {
      getDeleteIds(docs.value, id);
      console.log(ids)
      axios({
        method: "delete",
        url: '/doc/delete',
        data: ids
      }).then(response => {
        const data = response.data;
        if (data.code == 200) {
          message.success(data.message)
          handleQuery()
        } else {
          message.error(data.message)
        }
      })
    }

    const showConfirm = (id: any) => {
      Modal.confirm({
        title: '你确定要删除该文档吗?',
        icon: createVNode(ExclamationCircleOutlined),
        content: '该操作将删除该文档下的所有子文档',
        onOk() {
          handleDelete(id);
        },
        // eslint-disable-next-line @typescript-eslint/no-empty-function
        onCancel() {
          message.info("取消删除")
        },
      });
    };

    const cancel = () => {
      message.info("取消删除")
    };

    onMounted(() => {
      handleQuery();
    })
    /**
     * 名称
     */
    const param = ref();
    param.value = {};

    return {
      docs,
      columns,
      loading,

      edit,
      modalVisible,
      modalLoading,
      handleModalOk,
      doc,

      add,
      handleDelete,
      cancel,
      handleQuery,
      param,
      treeSelectData,
      showConfirm
    }
  }
})
</script>