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
          :data-source="ebooks"
          :row-key="record => record.id"
          :pagination="pagination"
          :loading="loading"
          @change="handleTableChange"
      >
        <template #cover="{ text: cover }">
          <a-image v-if="cover" :width="80" :src="cover" alt="avatar"></a-image>
        </template>
        <template v-slot:category="{text:record}">
          <!--          {{record}}-->
          <span>{{ getCategoryName(record.category1Id) }} / {{ getCategoryName(record.category2Id) }}</span>
        </template>
        <template v-slot:action="{ text, record }">
          <a-space size="small">
            <router-link :to="'/admin/doc?ebookId='+ record.id">
              <a-button type="primary">
                文档管理
              </a-button>
            </router-link>
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
      title="电子书表单"
      v-model:visible="modalVisible"
      :confirm-loading="modalLoading"
      @ok="handleModalOk"
  >
    <a-form :model="ebook" :label-col="{span: 6}" :wrapper-col="{ span : 18}">
      <a-form-item label="封面">
        <a-input v-model:value="ebook.cover"/>
      </a-form-item>
      <a-form-item label="名称">
        <a-input v-model:value="ebook.name"/>
      </a-form-item>
      <a-form-item label="分类">
        <a-cascader
            v-model:value="categoryIds"
            :field-names="{ label:'name',value:'id',children:'children' }"
            :options="categoriesTree"
            :load-data="handleQueryCategory"
        />
      </a-form-item>
      <a-form-item label="描述">
        <a-input v-model:value="ebook.description" type="text"/>
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
  name: 'AdminEbook',
  setup() {
    const ebooks = ref();
    const pagination = ref({
      current: 1,
      pageSize: 7,
      total: 0
    });
    const loading = ref(false);
    const columns = [
      {
        title: '封面',
        dataIndex: 'cover',
        align: 'center',
        slots: {customRender: 'cover'}
      },
      {
        title: '名称',
        dataIndex: 'name',
        align: 'center',
      },
      {
        title: '分类',
        slots: {customRender: 'category'},
        align: 'center',
      },
      {
        title: '文档数',
        dataIndex: 'docCount',
        align: 'center',
      },
      {
        title: '阅读数',
        dataIndex: 'viewCount',
        align: 'center',
      },
      {
        title: '点赞数',
        dataIndex: 'voteCount',
        align: 'center',
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
    const handleQuery = (params: any) => {

      console.log(params.page, params.size)
      loading.value = true;
      axios.get("/ebook/list", {
        params: {
          page: params.page,
          size: params.size,
          name: param.value.name
        }
      }).then(response => {
        loading.value = false;
        const data = response.data;
        if (data.code == 200) {
          ebooks.value = data.data.list;
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
      handleQueryCategory();
    });

    //------表单-----
    const categoryIds = ref();
    const ebook = ref()
    const modalVisible = ref(false)
    const modalLoading = ref(false)
    const handleModalOk = () => {
      modalLoading.value = true;
      ebook.value.category1Id = categoryIds.value[0]
      ebook.value.category2Id = categoryIds.value[1]
      axios.post("/ebook/edit", ebook.value).then(response => {
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
      console.log(ebook)
      ebook.value = Tool.copy(record)
      categoryIds.value = [ebook.value.category1Id, ebook.value.category2Id]
    }

    /**
     * 新增
     */
    const add = () => {
      modalVisible.value = true;
      ebook.value = {}
    }

    /**
     * 删除
     */
    const handleDelete = (id: any) => {
      axios.delete("/ebook/delete/" + id).then(response => {
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
    };

    /**
     * 查询分类的信息
     */
    const categoriesTree = ref();
    let categoryTemp: any
    const handleQueryCategory = () => {
      loading.value = true;
      axios.get("/category/list").then(response => {
        loading.value = false;
        const data = response.data;
        if (data.code == 200) {
          console.log("11")
          categoriesTree.value = data.data;
          categoryTemp = data.data;
          //加载完分类数据再查询电子书
          handleQuery({
            page: 1,
            size: pagination.value.pageSize
          });
        } else {
          message.error(data.message)
        }
      });
    };
    const cancel = () => {
      message.info("取消删除")
    };

    /**
     * 名称
     */
    const param = ref();
    param.value = {};

    /**
     * 获取分类名称
     */
    const getCategoryName = (cid: any) => {
      let result = "";
      categoryTemp.forEach((item: any) => {
        if (item.id == cid) {
          result = item.name;
          return result
        }
        item.children.forEach((initem: any) => {
          if (initem.id == cid) {
            result = initem.name
            return result
          }
        })
      })
      return result;
    }

    return {
      ebooks,
      pagination,
      columns,
      loading,
      handleTableChange,

      edit,
      modalVisible,
      modalLoading,
      handleModalOk,
      ebook,

      add,
      handleDelete,
      cancel,
      handleQuery,
      param,
      categoryIds,
      categoriesTree,
      handleQueryCategory,
      getCategoryName
    }
  }
})
</script>