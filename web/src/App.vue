<script setup lang="ts">
import dayjs from 'dayjs'
import { ClearOutlined, LoadingOutlined, RedoOutlined, KeyOutlined } from '@ant-design/icons-vue'
import { useStorage } from '@vueuse/core'
import { completion, creditSummary, completionTurbo } from '@/api'
import Message from './components/message.vue'

const createdAt = dayjs().format('YYYY-MM-DD HH:mm:ss')
const loadding = ref(false)
const visible = ref(false)
const summary = ref({} as any)
const models = ref(['text-davinci-003', 'gpt-3.5-turbo'])

const message = ref('')
const api_key = useStorage('api_key', '')
const chatModel = useStorage('model', 'gpt-3.5-turbo')
const continuously = useStorage('continuously', true)
const messages = useStorage('messages', [
  {
    username: "chatGPT",
    msg: "你好，有什么能帮助你的吗?",
    time: dayjs().format('HH:mm'),
    type: 0,
  },
])
const buildMessage = () => {
  let _messages = []
  for (let i = 0; i < messages.value.length; i++) {
    const element = messages.value[i]
    if (element.type === 0) {
      _messages.push("AI:\n" + element.msg)
    } else {
      _messages.push("User:\n" + element.msg)
    }
  }
  return _messages.join("\n\n") + "\n\AI:\n"
}


const sendMessage = async () => {
  loadding.value = true
  messages.value.push({
    username: "user",
    msg: message.value,
    time: dayjs().format('HH:mm'),
    type: 1,
  })
  let question = ""
  if (continuously.value) {
    question = buildMessage()
  } else {
    question = "User:\n" + message.value + "\n\nAI:\n"
  }
  message.value = ""
  const data: any = chatModel.value === 'gpt-3.5-turbo' ? await completionTurbo(question) : await completion(question)
  console.log(data)
  const replyMessage = data?.choices ? (data.choices[0]?.text ? data.choices[0].text : data.choices[0].message.content) : data?.error?.message
  messages.value.push({
    username: "chatGPT",
    msg: replyMessage,
    time: dayjs().format('HH:mm'),
    type: 0,
  })
  loadding.value = false
}

const clearMessages = () => {
  messages.value = []
}

const refushCredit = async () => {
  loadding.value = true
  summary.value = await creditSummary()
  loadding.value = false
}
const updateApiKey = () => {
  visible.value = false
  window.location.reload()
}

onMounted(async () => {
  await refushCredit()
})
</script>

<template>
  <div id="layout">
    <header id="header" class="bg-dark-50 text-white h-10 select-none">
      <LoadingOutlined v-if="loadding" class="pl-3 cursor-pointer" />
      <span class="text-size-5 pl-5">GPT</span>
      <a-tooltip>
        <template #title>清除聊天记录</template>
        <a-popconfirm title="确定清除本地所有聊天记录吗?" ok-text="是的" cancel-text="再想想" @confirm="clearMessages">
          <ClearOutlined class="pl-3 cursor-pointer" />
        </a-popconfirm>
      </a-tooltip>

      <a-tooltip>
        <template #title>自定义API_KEY</template>
        <KeyOutlined class="pl-3 cursor-pointer !text-red-400" @click="visible = true" />
      </a-tooltip>
      <a-checkbox v-model:checked="continuously" class="!text-white !pl-5">连续对话</a-checkbox>


      <a-select ref="select" v-model:value="chatModel" style="width: 180px" class="!pl-4">
        <a-select-option :value="model" v-for="model in models">{{ model }}</a-select-option>
      </a-select>

      <span class="float-right pr-3 pt-2">
        余额：{{ summary?.total_available }}
        <a-tooltip>
          <template #title>刷新余额</template>
          <RedoOutlined @click="refushCredit" />
        </a-tooltip>
      </span>
    </header>
    <div id="layout-body">
      <main id="main">
        <div class="flex-1 relative flex flex-col">
          <!-- header -->
          <!-- content -->
          <div class="flex-1 inset-0 overflow-hidden bg-transparent bg-bottom bg-cover flex flex-col">
            <!-- dialog -->
            <div class="flex-1 w-full self-center">
              <div class="relative px-3 py-1 m-auto flex flex-col">
                <div class="mx-0 my-1 self-center text-xs text-gray-400">
                  GPT已连接
                </div>
                <div class="mx-0 my-1 self-center text-xs text-gray-400">
                  {{ createdAt }}
                </div>
                <Message :message=message v-for="message in messages" :class="message.type ? 'send' : 'replay'" />
              </div>
            </div>
          </div>
        </div>
      </main>
      <footer id="footer">
        <div class="relative p-4 w-full overflow-hidden text-gray-600 focus-within:text-gray-400 flex items-center">
          <a-textarea v-model:value="message" :auto-size="{ minRows: 2, maxRows: 5 }" placeholder="请输入消息..."
            @pressEnter="sendMessage"
            class="appearance-none pl-10 py-2 w-full bg-white border border-gray-300 rounded-full text-sm placeholder-gray-800 focus:outline-none focus:border-blue-500 focus:border-blue-500 focus:shadow-outline-blue" />
          <span class="absolute inset-y-0 right-0 bottom-8 pr-6 flex items-end">
            <a-button shape="round" type="primary" @click="sendMessage">sent</a-button>
          </span>
        </div>

      </footer>
    </div>
    <a-modal v-model:visible="visible" title="更新API_KEY" @ok="updateApiKey" okText="更新" cancelText="关闭">
      <a-alert message="API_KEY往往是sk-开头的字符串" type="info" />
      <div class="mt-2"></div>
      <a-input v-model:value="api_key" placeholder="请输入API_KEY" />
    </a-modal>
  </div>
</template>

<style scoped>
body,
html {
  margin: 0;
  padding: 0;
}

#layout {
  display: flex;
  flex-direction: column;
  width: 100vw;
  height: 100vh;
  background-color: #f0f2f5;
}

#header {
  box-shadow: 2px 5px 5px 0px rgba(102, 102, 102, 0.5);
  flex-shrink: 0;
}

#layout-body {
  flex-grow: 2;
  overflow-y: auto;
  display: flex;
  flex-direction: column;
}

#footer {
  height: 100px;
  flex-shrink: 0;
}

#main {
  flex-grow: 2;
}

.replay {
  float: left;
  clear: both;
}

.send {
  float: right;
  clear: both;
}
</style>
