<template>
  <a-layout>
    <a-layout-content
        :style="{ background: '#fff', padding: '24px', margin: 0, minHeight: '280px' }"
    >
      <a-table
          :columns="columns"
          :data-source="ebooks"
          :row-key="record => record.id"
          :pagination="pagination"
          :loading="loading"
          @change="handleTableChange"
      >
        <template #cover="{ text: cover }">
          <img v-if="cover" :src="cover" alt="avatar">
        </template>
        <template v-slot:action="{ text , record }">
          <a-space size="small">
            <a-button type="primary" @click="edit(record)">编辑</a-button>
            <a-button type="primary" danger>删除</a-button>
          </a-space>
        </template>
      </a-table>
    </a-layout-content>
  </a-layout>
  <a-modal
      title="电子书表单"
      v-model:visible="modalVisible"
      :confirm-loading="modalLoading"
      @ok="handleModalOk"
  >
    <a-form :model="ebook" :label-col="{span: 6}" :wrapper-col="wrapperCol">
      <a-form-item label="封面">
        <a-input v-model:value="ebook.cover" />
      </a-form-item>
      <a-form-item label="名称">
        <a-input v-model:value="ebook.name" />
      </a-form-item>
      <a-form-item label="分类一">
        <a-input v-model:value="ebook.category1Id" />
      </a-form-item>
      <a-form-item label="分类二">
        <a-input v-model:value="ebook.category2Id" />
      </a-form-item>
      <a-form-item label="描述">
        <a-input v-model:value="ebook.desc" type="text"/>
      </a-form-item>
    </a-form>
  </a-modal>
</template>
<script lang="ts">
import {defineComponent, onMounted, ref} from "vue";
import axios from "axios";

export default defineComponent({
  name: 'AdminEbook',
  setup() {
    const ebooks = ref();
    const pagination = ref({
      current: 1,
      pageSize: 4,
      total: 0
    });
    const loading = ref(false);
    const columns = [
      {
        title: '封面',
        dataIndex: 'cover',
        slots: {customRender: 'cover'}
      },
      {
        title: '名称',
        dataIndex: 'name'
      },
      {
        title: '分类一',
        key: 'category1Id',
        dataIndex: 'category1Id'
      },
      {
        title: '分类二',
        dataIndex: 'category2Id'
      },
      {
        title: '文档数',
        dataIndex: 'docCount'
      },
      {
        title: '阅读数',
        dataIndex: 'viewCount'
      },
      {
        title: '点赞数',
        dataIndex: 'voteCount'
      },
      {
        title: '操作',
        key: 'action',
        slots: { customRender: 'action'}
      }
    ];

    /**
     * 数据查询
     */
    const handleQuery = (params: any) => {
      console.log(params.page,params.size)
      loading.value = true;
      axios.get("/ebook/list",{
        params: params
      }).then(response => {
        loading.value = false;
        const data = response.data;
        ebooks.value = data.data.list;
        //重置分页按钮
        pagination.value.current = params.page;
        pagination.value.total = data.data.total
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
    onMounted(()=>{
      handleQuery({
        page: 1,
        size: pagination.value.pageSize
      });
    });

    //------表单-----
    const ebook = ref({})
    const modalVisible = ref(false)
    const modalLoading = ref(false)
    const handleModalOk = () => {
      modalLoading.value = true;
      setTimeout(()=>{
        modalVisible.value = false;
        modalLoading.value = false;
      },2000)
      axios.post("/ebook/edit",ebook.value).then(response =>{
        const data = response.data;
        if(data.code === 200){
          modalVisible.value = false;
          modalLoading.value = false;

          handleQuery({
            page: pagination.value.current,
            size: pagination.value.pageSize
          })
        }
      })
    }
    /**
     * 编辑
     */
    const edit = (record: any) =>{
      modalVisible.value = true;
      ebook.value = record
    }
    return{
      ebooks,
      pagination,
      columns,
      loading,
      handleTableChange,
      edit,
      modalVisible,
      modalLoading,
      handleModalOk,
      ebook
    }
  }
})
</script>