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
                <a-button type="primary" @click="add()">
                  新增
                </a-button>
              </a-form-item>
            </a-form>
          </p>
          <a-table
              v-if="docs"
              :columns="columns"
              :data-source="docs"
              :row-key="record => record.id"
              :loading="loading"
              :pagination="false"
              :defaultExpandAllRows = "true"
              size="small"
          >
            <template v-slot:action="{ text: record }">
              <a-space size="small">
                <a-button type="primary" @click="edit(record)" size="small">编辑</a-button>
                <a-popconfirm
                    title="确认要删除该条数据吗?"
                    ok-text="是"
                    cancel-text="否"
                    @confirm="showConfirm(record.id)"
                    @cancel="cancel"
                >
                  <a-button type="primary" danger size="small">删除</a-button>
                </a-popconfirm>
              </a-space>
            </template>
          </a-table>
        </a-col>
        <a-col :span="16" v-if="doc">
          <p>
            <a-form layout="inline" :model="param">
              <a-form-item>
                <a-button type="primary" @click="handleSave()">保存</a-button>
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
                  placeholder="请选择父文档"
                  tree-default-expand-all
                  :tree-data="treeSelectData"
                  :field-names="{label:'name',children: 'children', value: 'id'}"
              >
              </a-tree-select>
            </a-form-item>
            <a-form-item>
              <a-input v-model:value="doc.sort" placeholder="顺序"/>
            </a-form-item>
            <a-form-item>
              <a-button type="primary" @click="handlePreviewContent"><EyeOutlined/>内容预览</a-button>
            </a-form-item>
            <a-form-item>
              <div id="content"></div>
            </a-form-item>
          </a-form>

        </a-col>
      </a-row>
      <a-drawer width="900" placement="right" :closable="false" :visible="drawerVisible" @close="onDrawerClose">
        <div class="editor-content-view" :innerHTML="previewHtml"></div>
      </a-drawer>
    </a-layout-content>
  </a-layout>
  <!--  <a-modal-->
  <!--      title="文档表单"-->
  <!--      v-model:visible="modalVisible"-->
  <!--      :confirm-loading="modalLoading"-->
  <!--      @ok="handleModalOk">-->
  <!--  </a-modal>-->
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
        // align: 'center'
      },
      {
        title: '操作',
        key: 'action',
        slots: {customRender: 'action'}
      }
    ];
    const treeSelectData = ref()
    treeSelectData.value = [];

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
          treeSelectData.value = Tool.copy(docs.value);
          if (Tool.isEmpty(treeSelectData.value)) treeSelectData.value = [];
          treeSelectData.value.unshift({id: 0, name: '无'})
        } else {
          message.error(data.message)
        }
      });
    };

    const handleQueryContent = () => {
      axios.get("/doc/find-content/"+doc.value.id).then(response => {
        if(response.data.code === 200){
          editor.txt.html(response.data.data);
        }else {
          message.error(response.data.message);
        }
      })
    }


    //------表单-----
    const doc = ref()
    doc.value = {
      ebookId: route.query.ebookId
    }
    const modalVisible = ref(false)
    const modalLoading = ref(false)
    let editor: any;
    const createEditor = () => {
      if (editor != null) editor.destroy();
      editor = new E('#content');
      editor.config.zIndex = 0;
      editor.create();
    }

    const handleSave = () => {
      modalLoading.value = true;
      doc.value.content = editor.txt.html();
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
      handleQueryContent();
      treeSelectData.value = Tool.copy(docs.value);
      console.log(treeSelectData.value);
      setDisable(treeSelectData.value, record.id);
      treeSelectData.value.unshift({id: '0', name: '无'})
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

    // 富文本预览
    const drawerVisible = ref(false);
    const previewHtml = ref();
    const handlePreviewContent =() => {
      const html = editor.txt.html();
      previewHtml.value = html;
      drawerVisible.value = true;
    }
    const onDrawerClose = () => {
      drawerVisible.value = false;
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
      //初始化编辑器
      createEditor();
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
      handleSave,
      doc,

      add,
      handleDelete,
      cancel,
      handleQuery,
      param,
      treeSelectData,
      showConfirm,
      handlePreviewContent,
      onDrawerClose,
      drawerVisible,
      previewHtml
    }
  }
})
</script>