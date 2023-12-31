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
          :data-source="categorys"
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
                @confirm="handleDelete(record.id)"
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
      title="分类表单"
      v-model:visible="modalVisible"
      :confirm-loading="modalLoading"
      @ok="handleModalOk"
  >
    <a-form :model="category" :label-col="{span: 6}" :wrapper-col="{span: 18}">
      <a-form-item label="名称">
        <a-input v-model:value="category.name"/>
      </a-form-item>
      <a-form-item label="父分类">
        <a-select
            v-model:value="category.parent"
            ref="select"
        >
          <a-select-option :value="0">无</a-select-option>
          <a-select-option v-for="val in categorys" :key="val.id" :value="val.id" :disabled="val.id === category.id">{{val.name}}</a-select-option>
        </a-select>
      </a-form-item>
      <a-form-item label="顺序">
        <a-input v-model:value="category.sort"/>
      </a-form-item>
    </a-form>
  </a-modal>
</template>
<script lang="ts">
import {defineComponent, onMounted, ref} from "vue";
import axios from "axios";
import {message} from "ant-design-vue";
import {Tool} from "@/utils/tool";

export default defineComponent({
  name: 'AdminCategory',
  setup() {
    const categorys = ref();

    const loading = ref(false);
    const columns = [
      {
        title: '名称',
        dataIndex: 'name',
        align: 'center'
      },
      {
        title: '父分类',
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
      axios.get("/category/list").then(response => {
        loading.value = false;
        const data = response.data;
        if (data.code == 200) {
          categorys.value = data.data;
        } else {
          message.error(data.message)
        }
      });
    };


    //------表单-----
    const category = ref()
    const modalVisible = ref(false)
    const modalLoading = ref(false)
    const handleModalOk = () => {
      modalLoading.value = true;
      axios.post("/category/edit", category.value).then(response => {
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
     * 编辑
     */
    const edit = (record: any) => {
      modalVisible.value = true;
      category.value = Tool.copy(record)
    }

    /**
     * 新增
     */
    const add = () => {
      modalVisible.value = true;
      category.value = {}
    }

    /**
     * 删除
     */
    const handleDelete = (id: any) => {
      axios.delete("/category/delete/" + id).then(response => {
        const data = response.data;
        if (data.code == 200) {
          message.success(data.message)
          handleQuery()
        } else {
          message.error(data.message)
        }
      })
    }

    const cancel = () => {
      message.info("取消删除")
    };

    onMounted(()=>{
      handleQuery();
    })
    /**
     * 名称
     */
    const param = ref();
    param.value = {};

    return {
      categorys,
      columns,
      loading,

      edit,
      modalVisible,
      modalLoading,
      handleModalOk,
      category,

      add,
      handleDelete,
      cancel,
      handleQuery,
      param
    }
  }
})
</script>