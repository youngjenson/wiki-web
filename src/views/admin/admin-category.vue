<template>
  <a-layout>
    <a-layout-content
        :style="{ background: '#fff', padding: '24px', margin: 0, minHeight: '280px' }"
    >
      <p>
        <a-form layout="inline" :model="param">
          <a-form-item>
            <a-input v-model:value="param.name" placeholder="名称"/>
          </a-form-item>
          <a-form-item>
            <a-button type="primary" @click="handleQuery({page:1,size:pagination.pageSize})">
              查询
            </a-button>
          </a-form-item>
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
          :pagination="pagination"
          :loading="loading"
          @change="handleTableChange"
      >
        <template #cover="{ text: cover }">
          <img v-if="cover" :src="cover" alt="avatar">
        </template>
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
        <a-input v-model:value="category.parent"/>
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
    const pagination = ref({
      current: 1,
      pageSize: 7,
      total: 0
    });
    const loading = ref(false);
    const columns = [
      {
        title: '名称',
        dataIndex: 'name'
      },
      {
        title: '父分类',
        key: 'parent',
        dataIndex: 'parent'
      },
      {
        title: '顺序',
        dataIndex: 'sort'
      },
      {
        title: '操作',
        key: 'action',
        slots: {customRender: 'action'}
      }
    ];

    /**
     * 数据查询
     */
    const handleQuery = (params: any) => {

      console.log(params.page, params.size)
      loading.value = true;
      axios.get("/category/list", {
        params: {
          page: params.page,
          size: params.size,
          name: param.value.name
        }
      }).then(response => {
        loading.value = false;
        const data = response.data;
        if (data.code == 200) {
          categorys.value = data.data.list;
          //重置分页按钮
          pagination.value.current = params.page;
          pagination.value.total = data.data.total
        } else {
          message.error(data.message)
        }
      });
    };

    /**
     * 表格点击页码时触发
     */
    const handleTableChange = (pagination: any) => {
      handleQuery({
        page: pagination.current,
        size: pagination.pageSize
      })
    };
    onMounted(() => {
      handleQuery({
        page: 1,
        size: pagination.value.pageSize
      });
    });

    //------表单-----
    const category = ref({})
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
          handleQuery({
            page: pagination.value.current,
            size: pagination.value.pageSize
          })
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
          handleQuery({
            page: pagination.value.current,
            size: pagination.value.pageSize
          })
        } else {
          message.error(data.message)
        }
      })
    }

    const cancel = () => {
      message.info("取消删除")
    };

    /**
     * 名称
     */
    const param = ref();
    param.value = {};

    return {
      categorys,
      pagination,
      columns,
      loading,
      handleTableChange,

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