<template>
  <div id="chat-container">
    <div id="chat-box">
      <div v-for="(message, index) in messages" :key="index"
           :class="['message', message.sender === 'User' ? 'user-message' : 'ai-message']">
        <span>{{ message.content }}</span>
      </div>
    </div>
    <div id="input-container">
      <input type="text" v-model="userInput" placeholder="发送消息..." @keypress.enter="sendMessage" />
      <button @click="sendMessage"> <span>发送</span></button>
    </div>
  </div>
    <div id="feedback-section">
      <textarea v-model="feedbackInput" placeholder="输入反馈..."></textarea>
      <button @click="submitFeedback">提交反馈</button>
    </div>

</template>

<script>
import axios from "axios";
const DASHSCOPE_API_URL = 'https://dashscope.aliyuncs.com/api/v1/services/aigc/text-generation/generation';
const headers = {
  'Authorization': 'Bearer sk-dc356b8ca42c41788717c007f49e134a',
  'Content-Type': 'application/json',
  'X-DashScope-SSE': 'enable',
};

const data = {
  "model": "qwen-max-1201",
  "input": {
    "messages": [
      {
        "role": "user",
        "content": "你好啊啊啊啊啊啊啊啊啊啊啊啊啊啊啊啊啊啊啊你是谁啊"
      }
    ]
  },
  "parameters": {}
};


export default {
  data() {
    return {
      userInput: '',
      feedbackInput: '',
      messages: [],
      // 假设 sessionId 已经在某处生成并存储
      sessionId: 'your-session-id'
    };
  },

  methods: {
    sendMessage() {
      if (this.userInput.trim()) {
        this.messages.push({
          sender: 'User',
          content: this.userInput,
          selected: false // 如果您需要消息选择功能
        });
        // 清空输入框
        this.userInput = '';
        // 这里添加发送消息到后端的代码
        // 准备要发送的数据，包括消息内容
        const dataToSend = {
          data: this.userInput, // 取最新的消息内容
        };

        axios.post(DASHSCOPE_API_URL, data, { headers })
            .then(response => {
              // 处理响应数据
              console.log(response.data);
            })
            .catch(error => {
              // 处理请求错误
              console.error(error);
            });
      }
    },
    submitFeedback() {
      const selectedMessages = this.messages.filter(m => m.selected);
      // 发送反馈到后端的逻辑...
      this.feedbackInput = '';
    },
    addMessage(sender, content) {
      this.messages.push({ sender, content, selected: false });
    }
  }
};
</script>

<style scoped>
html, body {
  height: 100%;
  width: 100%;
  margin: 0;
  display: flex;
  justify-content: center;
  align-items: center;
  background-color: #000; /* 或者任何您想要的背景颜色 */
}
#chat-container {
   display: flex;
   flex-direction: column;
   width: 80vw; /* 设置宽度为视窗宽度的 2/3 */
   height: 50vw;
   max-width: 1000px; /* 可以根据需要调整最大宽度 */
   margin: auto; /* 自动边距实现居中 */
   padding: 20px;
   box-shadow: 0 4px 8px rgba(0, 0, 0, 0.5);
   background-color: #1e1e1e;
   color: white;
   border-radius: 10px;
   position: relative;
 }

#chat-box {
  height: calc(100% - 120px); /* 减去按钮和间距的高度 */
  margin-bottom: 20px;
  overflow-y: auto; /* 超出部分滚动 */
  width: 100%; /* 聊天框宽度填满容器 */
  padding: 10px;
  background-color: #2a2a2a;
  border: 1px solid #444444;
  border-radius: 10px;
}
.message {
  padding: 10px;
  margin-bottom: 8px;
  border-radius: 8px;
  max-width: 80%;
  word-wrap: break-word;
}

.user-message {
  background-color: blue; /* 用户消息的背景颜色 */
  align-self: flex-end; /* 对齐到右边 */
  margin-left: auto;
}

.ai-message {
  background-color: gray; /* AI 消息的背景颜色 */
  align-self: flex-start; /* 对齐到左边 */
}
/* 输入框和按钮的容器样式 */
#input-container {
  display: flex;
  justify-content: space-between; /* 使输入框和按钮分布在两侧 */
  align-items: center; /* 垂直居中对齐 */
  height: 80px;
}

/* 输入框样式 */
input[type="text"] {
  flex-grow: 5; /* 输入框占据剩余空间 */
  margin-right: 10px; /* 与按钮之间的距离 */
  height: 50px; /* 输入框高度，您可以根据需要调整 */
  padding: 10px; /* 增加内边距以便文本居中 */
  border-radius: 5px;
  border: 2px solid transparent; /* 设置透明边框以保持布局稳定 */
  background-color: whitesmoke; /* 暗色背景 */
  transition: background-color 0.3s ease, border-color 0.3s ease, box-shadow 0.3s ease, background-image 0.3s ease;
  background-image: linear-gradient(45deg, transparent, transparent); /* 初始透明背景渐变 */
  color: black;
  outline: none; /* 移除焦点时的轮廓 */
  //transition: border-color 0.3s ease, box-shadow 0.3s ease;

}
/* 输入框悬停及焦点效果 */
input[type="text"]:hover, input[type="text"]:focus {
  border-color: #00dbde; /* 高亮边框颜色 */
  box-shadow: 0 0 8px #00dbde; /* 发光效果 */
  background-image: linear-gradient(to right, rgba(0, 128, 255, 0), rgba(0, 219, 222, 0.2)); /* 充能渐变效果 */
  caret-color: #fc00ff; /* 设置醒目的光标颜色 */
}

/* 输入框内的占位符样式 */
input[type="text"]::placeholder {
  color: gray; /* 浅色占位符 */
}

/* 按钮样式 */
button {
  background-image: linear-gradient(to left, #fc00ff, #00dbde); /* 初始渐变背景 */
  flex-grow: 1; /* 输入框占据剩余空间 */ height: 50px; /* 按钮高度与输入框一致 */
  padding: 10px; /* 增加内边距以便文本居中 */
  color: white;
  border: none;
  border-radius: 5px;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.2);
  cursor: pointer;
  transition: all 0.3s ease;
  z-index: 10;
  position: relative; /* 为了伪元素定位 */
  overflow: hidden; /* 确保伪元素不超出按钮边界 */
}

button::before {
  content: '';
  position: absolute;
  top: 0;
  left: -100%; /* 开始时将伪元素置于左侧之外 */
  width: 300%; /* 使伪元素宽度为按钮的三倍 */
  height: 100%;
  background-image: linear-gradient(to left, #fc00ff, #00dbde); /* 渐变背景 */
  transition: all 0.5s ease-out;
  z-index: -1;
}

button:hover::before {
  left: 0; /* 鼠标悬停时，移动伪元素以显示渐变效果 */
}

/* 按钮文本样式，确保在动画之上 */
button span {
  position: relative;
  z-index: 10;
}

/* 反馈区域样式 */
#feedback-section {
  position: absolute; /* 绝对定位 */
  bottom: 20px; /* 距离底部的位置 */
  left: 50%; /* 水平居中 */
  transform: translateX(-50%); /* 配合 left 使用，实现精确居中 */
  width: calc(100% - 40px); /* 宽度减去 padding */
  /* 其他样式保持不变 */
}

/* 适应移动设备 */
@media (max-width: 600px) {
  #chat-container {
    width: 90vw; /* 在小屏幕上，聊天框宽度更宽一些 */
  }
}
</style>
