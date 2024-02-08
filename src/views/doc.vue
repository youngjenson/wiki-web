<template>
  <a-layout>
    <a-layout-content :style="{background: '#ffff',padding: '24px', margin: 0, minHeight: '280px'}">
      <h3 v-if="docs === null">对不起，找不到相关文档！</h3>
      <a-row>
        <a-col :span="6">
          <a-tree
              v-if="docs"
              :tree-data="docs"
              @select="onSelect"
              :replaceFields="{title: 'name', key: 'id', value: 'id'}"
              :defaultExpandAll="true"
              :defaultSelectedKeys="defaultSelectedKeys"
          >
          </a-tree>
        </a-col>
        <a-col :span="18">
          <div class="editor-content-view" :innerHTML="html"></div>
        </a-col>
      </a-row>
    </a-layout-content>
  </a-layout>
</template>

<script lang="ts">
import {defineComponent, onMounted, ref, createVNode} from "vue";
import axios from 'axios';
import {message, Modal} from "ant-design-vue";
import {Tool} from "@/utils/tool";
import {useRoute} from "vue-router";
import E from "wangeditor";
import {ExclamationCircleOutlined} from "@ant-design/icons-vue";

export default defineComponent({
  name: 'Doc',
  setup() {
    const route = useRoute()
    const docs = ref();
    docs.value = []
    const loading = ref(false);
    const html = ref();
    const defaultSelectedKeys = ref();
    defaultSelectedKeys.value = [];


    const handleQueryContent = (id: number) => {
      axios.get("/doc/find-content/" + id).then(response => {
        if (response.data.code === 200) {
          html.value = response.data.data;
        } else {
          message.error(response.data.message);
        }
      })
    }
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
          console.log(docs.value)
          console.log("???"+Tool.isNotEmpty(docs.value));
          if(Tool.isNotEmpty(docs.value)) {
            defaultSelectedKeys.value = [docs.value[0].id];
            handleQueryContent(docs.value[0].id);
          }
        } else {
          message.error(data.message)
        }
      });
    };

    const onSelect = (selectedKeys: any, info: any) => {
      if (Tool.isNotEmpty(selectedKeys)) {
        handleQueryContent(selectedKeys[0]);
      }
    }

    onMounted(() => {
      handleQuery();
    })

    return {
      docs,
      html,
      onSelect,
      defaultSelectedKeys
    }
  }
})
</script>
<style>
.editor-content-view {
  padding: 0 10px;
  margin-top: 20px;
  overflow-x: auto;
}

.editor-content-view p,
.editor-content-view li {
  white-space: pre-wrap; /* 保留空格 */
}

.editor-content-view blockquote {
  border-left: 8px solid #d0e5f2;
  padding: 10px 10px;
  margin: 10px 0;
  background-color: #f1f1f1;
}

.editor-content-view code {
  font-family: monospace;
  background-color: #eee;
  padding: 3px;
  border-radius: 3px;
}

.editor-content-view pre > code {
  display: block;
  padding: 10px;
}

.editor-content-view table {
  border-collapse: collapse;
}

.editor-content-view td,
.editor-content-view th {
  border: 1px solid #ccc;
  min-width: 50px;
  height: 20px;
}

.editor-content-view th {
  background-color: #f1f1f1;
}

.editor-content-view ul,
.editor-content-view ol {
  padding-left: 20px;
}

.editor-content-view input[type="checkbox"] {
  margin-right: 5px;
}
</style>