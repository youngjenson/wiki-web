<template>
  <a-layout>
    <a-layout-sider width="200" style="background: #fff">
      <a-menu
          :style="{ height: '100%', borderRight: 0 }"
          mode="inline"
          @click="handleClick"
      >
        <a-menu-item key="welcome">
            <MailOutlined/>
            <span>欢迎</span>
        </a-menu-item>
        <a-sub-menu v-for="item in categoriesTree" :key="item.id">
          <template v-slot:title>
            <span><user-outlined/>{{item.name}}</span>
          </template>
          <a-menu-item v-for="children in item.children" :key="children.id">
            <MailOutlined/><span>{{children.name}}</span>
          </a-menu-item>
        </a-sub-menu>
      </a-menu>
    </a-layout-sider>
    <a-layout-content
        :style="{ background: '#fff', padding: '24px', margin: 0, minHeight: '280px' }"
    >
      <div class="welcome" v-show="isShowWelcome">111</div>
      <a-list v-show="!isShowWelcome" item-layout="vertical" size="large" :grid="{gutter: 20,column: 3}" :data-source="ebooks">
        <template #renderItem="{ item }">
          <a-list-item key="item.name">
            <template #actions>
          <span v-for="{ type, text } in actions" :key="type">
            <component v-bind:is="type" style="margin-right: 8px" />
            {{ text }}
          </span>
            </template>
            <a-list-item-meta :description="item.description">
              <template #title>
                <router-link :to=" '/doc?ebookId=' + item.id">{{ item.name }}</router-link>
              </template>
              <template #avatar><a-avatar :src="item.cover" /></template>
            </a-list-item-meta>
            {{ item.content }}
          </a-list-item>
        </template>
      </a-list>
    </a-layout-content>
  </a-layout>
</template>
<script lang="ts">
import { defineComponent, onMounted ,ref} from 'vue';
import axios from "axios";
import {message} from "ant-design-vue";

export default defineComponent({
  name: 'HomeView',
  setup() {

    const actions: Record<string, string>[] = [
      { type: 'StarOutlined', text: '156' },
      { type: 'LikeOutlined', text: '156' },
      { type: 'MessageOutlined', text: '2' },
    ];

    const ebooks = ref()
    onMounted(()=>{
      handleQueryCategory()
    });


    /**
     * 查询电子书
     */
    const handleEbookQuery = (param : any) => {
      axios.get("/ebook/list",{
        params: {
          category2Id: param
        }
      }).then( resp =>{
        const data = resp.data
        if(data.code == 200){
          ebooks.value = data.data.list
        }else{
          message.error(data.message)
        }
      })
    }
    const isShowWelcome = ref(true)
    const handleClick = (value:any) => {
      if(value.key === 'welcome'){
        isShowWelcome.value = true;
      }else{
        handleEbookQuery(value.key)
        isShowWelcome.value = false
      }
    }


    /**
     * 加载分类菜单
     */
    const categoriesTree = ref();
    const handleQueryCategory = () => {
      axios.get("/category/list").then(response => {
        const data = response.data;
        if (data.code == 200) {
          console.log("11")
          categoriesTree.value = data.data;
        } else {
          message.error(data.message)
        }
      });
    };

    return{
      handleClick,
      actions,
      ebooks,
      categoriesTree,
      isShowWelcome,
      handleEbookQuery
    }
  }
});
</script>

<style scoped>
  .ant-avatar {
    width: 50px;
    height: 50px;
    line-height: 50px;
    border-radius: 8%;
    margin: 5px 0;
  }
</style>