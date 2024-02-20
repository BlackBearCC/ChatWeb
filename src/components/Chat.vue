<template>
  <div id="page-container">

    <!-- 顶部标题栏 -->
    <div id="header">
      <!-- 标题内容 -->

    </div>

    <!-- 主体内容区域 -->
    <div id="main-content">

      <!-- 侧边栏 -->
      <div id="sidebar">
        <div class="profile-card">
          <img src="../assets/tuji@3x.png" alt="Profile Picture">
        </div>
<!--        <div class="status-label">在线</div>-->
        <!-- 体力值进度条 -->
        <div class="status-item strength-icon">
          <div class="progress-bar-container">
            <div class="progress-bar" :style="{ width: strength + '%' }"></div>
            <div class="progress-bar-text">{{ strength }}/100</div> <!-- 居中的文本容器 -->
          </div>
        </div>
        <!-- 元气值进度条 -->
        <div class="status-item energy-icon">
          <div class="progress-bar-container">
            <div class="progress-bar" :style="{ width: energy + '%' }"></div>
            <div class="progress-bar-text">{{ energy }}/100</div> <!-- 居中的文本容器 -->
          </div>
        </div>
        <!-- 新增的状态容器 -->
        <div class="status-container">
          <div class="status-tag">情绪：{{ mood || 'null' }}</div>
          <div class="status-tag">位置：{{ characterLocation || 'null' }}</div>
          <div class="status-tag">动作：{{ action || 'null' }}</div>
        </div>
      </div>

      <!-- 聊天容器 -->
      <div id="chat-container">
        <div id="chat-header">
          {{ situationData }}

        </div>

        <!-- 聊天框 -->
        <div id="chat-box">
          <div v-for="(message, index) in messages" :key="message.request_id"
               :class="['message', message.sender === 'User' ? 'user-message' : 'ai-message']">
            <!-- 头像容器 -->
            <div class="message-avatar"></div>
            <!-- 消息内容 -->
            <span :id="'message-' + message.request_id">{{ message.content }}</span>
          </div>
        </div>
        <!-- 输入容器 -->
        <div id="input-container">
          <input type="text" v-model="userInput" placeholder="在此处输入文本开始体验哦..." @keypress.enter="sendMessage" />
          <button @click="sendMessage"><span>发送</span></button>
        </div>
      </div>

    </div>

    <!-- 底部反馈区 -->
    <div id="feedback-section">
      <!-- 反馈输入区域 -->
    </div>

  </div>
</template>

<script>
import { v4 as uuidv4 } from 'uuid';
import axios from "axios";
export default {
  data() {
    return {
      userInput: '',
      feedbackInput: '',
      messages: [],
      lastMessageLength: 0, // 用来存储上一次消息文本的长度
      // 假设 sessionId 已经在某处生成并存储
      sessionId: '', // Initializing sessionId
      typeWriterIndexes: {}, // 字典用于存储每个消息的typeWriterIndex
      situationData: null, // 初始时为空
      isExpanded: false, // 控制展开状态的变量

      strength :88,
      energy:68,

      mood: "开心",
      characterLocation: "客厅",
      action: "站立"
    };
  },
  mounted() {
    // 检查本地存储中是否有 UUID
    const storedSessionId = localStorage.getItem('sessionId');

    if (storedSessionId) {
      // 如果存在，使用本地存储中的 UUID
      this.sessionId = storedSessionId;
    } else {
      // 否则，生成新的 UUID，并保存到本地存储
      this.sessionId = uuidv4();
      localStorage.setItem('sessionId', this.sessionId);
    }

    // 发送会话验证请求
    this.validateSession();
    this.fetchSituationData();
    this.fetchStatus();
  },

  methods: {
    async validateSession() {
      try {
        //http://182.254.242.30
        const response = await fetch('http://127.0.0.1:8000/validate-session', {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
            // other headers
          },
          body: JSON.stringify({ sessionId: this.sessionId })
        });

        const data = await response.json();

        if (data.valid) {
          console.log('Session is valid');
        } else {
          console.log('Session is invalid');
        }
      } catch (error) {
        console.error('Error validating session:', error);
      }
    },

    async sendMessage() {


      if (this.userInput.trim()) {
        this.messages.push({
          sender: 'User',
          content: this.userInput,
          request_id: Date.now().toString() // 生成一个临时的request_id
        });
        // 使用 fetch 发送请求

        try {
          const response = await fetch('http://127.0.0.1:8000/generate', {
            method: 'POST',
            headers: {
              'Content-Type': 'application/json',
              // 其他需要的头部信息
            },
            body: JSON.stringify({
              data: this.userInput, // 用户输入的数据
              sessionId: this.sessionId // 携带sessionId
            })
          });
          // 清空输入框
          this.userInput = '';
          // 处理流式响应
          const reader = response.body.getReader();
          while (true) {
            const { done, value } = await reader.read();
            if (done) break;
            const responseData = new TextDecoder("utf-8").decode(value);
            // console.log(responseData)
            // 查找 'data:' 部分
            let dataIndex = responseData.indexOf('data:');
            if (dataIndex !== -1) {
              // 提取 JSON 字符串
              let jsonStr = responseData.substring(dataIndex + 5);
              try {
                // 解析JSON数据
                const jsonData = JSON.parse(jsonStr); // 从"data:"之后开始解析
                const text = jsonData?.output?.text;
                const requestId = jsonData?.request_id;
                if (text && requestId) {
                  this.addOrUpdateMessage('Ai', text,requestId);
                  console.log(text)

                }
              } catch (e) {
                console.error('JSON解析错误:', e);
              }
            } else {
              console.log("没有找到 'data:' 部分");
            }

          }
        } catch (error) {
          console.error(error);
        }

      }
    },
    async fetchSituationData() {
      try {

        const response = await fetch('http://127.0.0.1:8000/situation-data/', {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
            // 其他需要的头部信息
          },
          body: JSON.stringify({
            sessionId: this.sessionId // 携带sessionId
          })
        });

        if (!response.ok) {
          // 如果响应状态不是2xx, 抛出错误
          throw new Error(`HTTP error! status: ${response.status}`);
        }

        // 确保使用 await 调用 .json() 方法
        const data = await response.json();
        console.log(data.situation)
        // 更新组件的数据属性
        this.situationData = data.situation;
        // 例如，保存到数据属性中：
        // this.situation = situationData;
      } catch (error) {
        console.error('Error fetching situation data:', error);
      }
    },

    async fetchStatus() {
      try {
        const response = await fetch('http://127.0.0.1:8000/fetch-status', {
          method: 'GET',
          headers: {
            'Content-Type': 'application/json',
            // other headers
          }
        });
        const data = await response.json();
        if (response.ok) {
          this.strength = data.strength|| 0;
          this.energy = data.energy|| 0;
          this.mood = data.mood;
          this.location = data.location;
          this.action = data.action;

        } else {
          console.error('Failed to fetch status');
        }
      } catch (error) {
        console.error('Error fetching status:', error);
      }
    }
  },


    submitFeedback() {
      const selectedMessages = this.messages.filter(m => m.selected);
      // 发送反馈到后端的逻辑...
      this.feedbackInput = '';
    },
    addOrUpdateMessage(sender, content,requestId) {
      const existingMessageIndex = this.messages.findIndex(m => m.request_id === requestId);
      if (existingMessageIndex !== -1) {
        // 使用打字机更新现有消息的内容而不使用vue响应式特性
        // this.messages[existingMessageIndex].content = content;
        this.typeWriterEffect(content, requestId, 100);
      } else {
        // 首次添加空白新消息
        content="";
        this.messages.push({ sender, content, request_id: requestId });
        // 初始化typeWriterIndex为0
        this.typeWriterIndexes[requestId] = 0;
      }
    },

    typeWriterEffect(text, requestId, speed = 100) {
      const messageIndex = this.messages.findIndex(m => m.request_id === requestId);
      const typeWriterIndex = this.typeWriterIndexes[requestId] || 0;

      if (messageIndex !== -1 && typeWriterIndex < text.length) {
        this.messages[messageIndex].content += text.charAt(typeWriterIndex);
        this.typeWriterIndexes[requestId]++; // 更新typeWriterIndex
        setTimeout(() => {
          this.typeWriterEffect(text, requestId, speed); // 继续递归调用
        }, speed);
      } else {
        // 打字机完成
      }
    },
    structuring(){

    }

};
</script>

<style scoped>

/* 设置 html 和 body 以确保背景覆盖整个屏幕 */
html, body {
  height: 100vh;
  width: 100vw;
  margin: 0;
  padding: 0;
  overflow: hidden; /* 阻止页面滚动 */
  background-color: #000; /* 或者任何您想要的背景颜色 */
  position: relative; /* 为了z-index起作用，需要设置position属性 */
}

/* 主体部分样式，包含侧边栏和主内容区 */
#main-content {
  flex: 1; /* 拉伸以填充剩余空间 */
  display: flex;
  //background-color: #fff;
}

/* 侧边栏样式 */
#sidebar {
  flex: 0 0 20%; /* 不拉伸，不收缩，宽度为总宽度的20% */
  background-color: #444444;
  margin-right: 16px;
  //padding: 16px;
  display: flex;
  flex-direction: column;
  align-items: center; /* 新增：确保内容居中 */
  border-radius: 10px;
}
/* 卡片样式 */
.profile-card {
  width: 100%; /* 卡片宽度相对于侧边栏宽度 */
  height: auto; /* 高度自动调整以保持图片比例 */
  //margin-bottom: 20px; /* 与下面的元素间隔 */
  border-radius: 10px; /* 圆角效果 */
  overflow: hidden; /* 隐藏溢出的图片部分 */
  //box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2); /* 添加阴影以提升层次感 */
}

/* 图片样式 */
.profile-card img {
  width: 100%; /* 使图片填满卡片 */
  height: auto; /* 自动调整高度以保持原有比例 */
  display: block; /* 避免底部出现空隙 */
}

/* 进度条和情绪状态的共通样式 */
.status-item {
  width: 80%; /* 全宽度以适应侧边栏 */
  display: flex; /* 使用 flex 布局来排列图标和文本 */
  align-items: center; /* 垂直居中对齐内容 */
  margin-bottom: 10px; /* 项目间的间距 */
}

.status-icon {
  width: 24px; /* 图标宽度 */
  height: 24px; /* 图标高度 */
  margin-right: 10px; /* 与文本间距 */
  background-size: cover; /* 图标覆盖满整个容器 */
}

/* 进度条样式 */
.progress-bar-container {
  flex-grow: 1; /* 允许进度条容器填充剩余空间 */
  margin-left: 10px;
  height: 14px; /* 进度条高度 */
  background-color: #ddd; /* 进度条背景颜色 */
  border-radius: 10px; /* 圆角效果 */
  overflow: hidden; /* 隐藏超出边界的进度条部分 */
  position: relative; /* 添加相对定位 */
}

.progress-bar {
  height: 100%; /* 进度条填满容器高度 */
  background-color: #ffcd00; /* 进度条颜色 */
  width: 0;
  text-align: center; /* 文本居中 */
  line-height: 10px; /* 行高与高度相同以垂直居中文本 */
  color: #1e1e1e;
  border-radius: 10px; /* 圆角效果 */
  transition: width 0.5s ease; /* 过渡动画 */
}
.progress-bar-text {
  position: absolute; /* 绝对定位 */
  width: 100%; /* 文本容器宽度填满父容器 */
  text-align: center; /* 文本水平居中 */
  line-height: 14px; /* 设置行高等于进度条高度以垂直居中文本 */
  color: black; /* 文本颜色 */
  top: 0; /* 顶部对齐 */
}

/* 使用 :before 伪元素为每个状态项添加图标 */
.energy-icon:before {
  content: url('../assets/energy_icon.png'); /* 元气值图标 */
}

.strength-icon:before {
  content: url('../assets/strength_icon.png'); /* 体力值图标 */
}

/* 状态容器样式，用于包裹情绪、位置、动作 */
.status-container {
  display: flex; /* 使用 flex 布局进行水平排列 */
  justify-content: space-around; /* 在项目之间添加平均空间 */
  width: 100%; /* 容器宽度 */
  margin-top: 20px; /* 与上方元素的间隔 */
}

/* 状态标签样式 */
.status-tag {
  font-size: 12px; /* 字体大小 */
  color: #161717; /* 字体颜色 */
  background-color: #ffcd00; /* 背景颜色 */
  padding: 2px 10px; /* 内边距 */
  border-radius: 20px; /* 圆角效果 */
}



/* 确保聊天容器填充整个屏幕，并且是最顶层的 */
#chat-container {
  flex: 1; /* 拉伸以填充剩余空间 */
  display: flex;
  flex-direction: column;
  border: 2px solid #444444;
  width: 50vw; /* 设置宽度为视窗宽度的 1/2 */
  height: 50vw;
  max-width: 1400px; /* 可以根据需要调整最大宽度 */
  margin: auto; /* 自动边距实现居中 */
  padding: 20px;
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.5);
  background-color: #1e1e1e;
  color: white;
  border-radius: 10px;
  position: relative;
  overflow: hidden; /* 隐藏溢出内容 */
}
/* 顶部标题栏样式 */
#chat-header {
  flex: 0.1 1 auto; /* 不拉伸，不收缩，自动基于内容设置大小 */
  max-height: 100px;
  overflow: hidden;
  padding: 10px;
  margin: 16px;
  background-color: #444444;
  border-radius: 10px;
  text-align: left;
  white-space: pre-wrap;
  transition: max-height 0.3s ease; /* 动画过渡效果 */
}

/* 设置聊天框以允许在内部滚动 */
#chat-box {
  flex: 1; /* 聊天框占据除输入框之外的所有可用空间 */
  overflow-y: auto; /* 允许在聊天框内部滚动 */
  padding: 10px; /* 内边距 */
}

input[type="text"] {
  box-sizing: border-box; /* 边框和填充包括在宽度内 */
  width: calc(100% - 70px); /* 减去按钮宽度和一些间隙 */
  max-width: calc(100% - 70px); /* 同上，确保最大宽度不超过屏幕宽度 */
  height: 40px; /* 设置一个固定高度 */
  max-height: 40px; /* 同上，确保最大高度不变 */
  margin: 0; /* 移除外边距 */
  padding: 0 10px; /* 设置内边距 */
  border: 1px solid #ccc; /* 根据需要设置边框样式 */
  outline: none; /* 移除聚焦时的外轮廓 */
  /* 其他样式... */
}

input[type="text"]:focus {
  border: 1px solid #fcd535; /* 聚焦时边框颜色，确保边框厚度不变 */
  /* 如果有额外的聚焦样式，确保它们不会影响尺寸 */
}
#chat-box::-webkit-scrollbar {
  width: 10px; /* 修改滚动条宽度 */
}

#chat-box::-webkit-scrollbar-track {
  background-color: #f0f0f0; /* 修改滚动条背景颜色 */
  border-radius: 5px; /* 修改滚动条滑块圆角 */
}

#chat-box::-webkit-scrollbar-thumb {
  background-color: #3b3b3b; /* 修改滚动条滑块颜色 */
  border-radius: 5px; /* 修改滚动条滑块圆角 */
  width: 8px; /* 修改滚动条滑块宽度 */
  height: 10px; /* 修改滚动条滑块高度 */
}
#chat-box::-webkit-scrollbar-thumb:hover {
  background-color: #ffcd00; /* 修改滚动条滑块悬停颜色 */
}

.message {
  display: flex;
  align-items: flex-start;
  padding: 10px;
  margin-bottom: 16px;
  border-radius: 8px;
  max-width: 100%;
  margin-right: 32px;
  word-wrap: break-word;
  white-space: pre-wrap;
}

.user-message {
  align-self: flex-start;
  background-color: transparent;
}

.ai-message {
  align-self: flex-start;
  background-color: #3b3b3b;
}
/* 这里是您的CSS样式，包括头像样式 */
.message-avatar {
  min-width: 50px;
  width: 50px;
  height: 50px;
  border-radius: 5px;
  margin-right: 10px;
  background-color: #ffcd00;
  background-size: cover;
}

.ai-message .message-avatar {
  background-image: url('../assets/ai_head.png'); /* 根据实际路径调整 */

}

/* 应用头像样式 */
.user-message .message-avatar {
  background-color: #ffcd00; /* 用户头像使用亮黄色背景 */
  background-image: url('../assets/user_head.jpg'); /* 根据实际路径调整 */
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
  flex-grow: 4; /* 输入框占据剩余空间 */
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
  border-color: #ffcd00; /* 高亮边框颜色 */
  box-shadow: 0 0 8px #ffcd00; /* 发光效果 */
  background-image: linear-gradient(to right, rgba(0, 128, 255, 0), rgba(255, 205, 0, 0.2)); /* 充能渐变效果 */
  caret-color: black; /* 设置醒目的光标颜色 */
}

/* 输入框内的占位符样式 */
input[type="text"]::placeholder {
  color: gray; /* 浅色占位符 */
}

/* 按钮样式 */
button {
  background-image: linear-gradient(to left, #ffcd00, #ffcd00); /* 初始渐变背景 */
  flex-grow: 1; /* 输入框占据剩余空间 */ height: 50px; /* 按钮高度与输入框一致 */
  min-width: 200px;
  padding: 10px; /* 增加内边距以便文本居中 */
  color: black;
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
  //background-image: transparent; /* 初始时渐变背景为透明 */
  transition: all 0.5s ease-out;
  z-index: -1;
}

button:hover::before {
  left: 0; /* 鼠标悬停时，移动伪元素以显示渐变效果 */
  background-image: linear-gradient(to left, #fc8f35, #ffcd00); /* 鼠标悬停时的渐变背景 */

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

@media (max-width: 600px) {
  html, body {
    height: 100%;
    max-width: 100vw;
    margin: 0;
    padding: 0;
    overflow: hidden; /* 阻止页面滚动 */
  }


  #chat-container {
    position: absolute; /* 绝对定位确保是最顶层 */
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    margin: auto; /* 自动边距实现居中 */
    max-width: 100vw; /* 最大宽度为视口宽度 */
    max-height: 100vh; /* 最大高度为视口高度 */
    min-width: 100vw;
    min-height: 100vh;
    border: none; /* 移除边框 */
    border-radius: 0; /* 移除圆角 */
    z-index: 1000; /* 高z-index值确保它在其他内容之上 */
    box-sizing: border-box; /* 确保边框和内边距包含在宽度和高度内 */
    display: flex;
    flex-direction: column; /* 设置聊天容器为列布局 */
    justify-content: space-between; /* 子元素间隔均匀分布 */
  }

  #chat-box {
    flex-grow: 1;
    overflow-y: auto; /* 允许在聊天框内部滚动 */
    padding: 10px; /* 为聊天框内容添加上下填充 */
    margin-bottom: 60px; /* 与固定输入容器的高度相同 */
    max-width: 100vw;
  }

  #input-container {
    position: fixed; /* 固定在屏幕底部 */
    bottom: 0;
    left: 0;
    right: 0;
    height: 60px; /* 输入容器的固定高度 */
    display: flex; /* 保持输入框和按钮水平排列 */
    align-items: center; /* 垂直居中 */
    max-width: 100vw;
    min-width: 100vw;
    padding: 0 10px; /* 为输入框添加左右填充 */
    background: #1e1e1e; /* 与聊天容器背景相同 */
  }

  input[type="text"] {
    flex-grow: 1; /* 输入框占据除按钮外的所有可用空间 */
    height: 40px; /* 减小输入框的高度以适应固定容器 */
    margin-right: 10px; /* 与按钮之间的距离 */

    width: 50vw; /* 减去按钮宽度和一些间隙 */
  }
  input[type="text"]:focus {
    flex-grow: 1; /* 输入框占据除按钮外的所有可用空间 */
    height: 40px; /* 减小输入框的高度以适应固定容器 */
    margin-right: 10px; /* 与按钮之间的距离 */
    max-width: 100vw;
    /* 保持宽高不变，不添加任何可能会增加尺寸的样式 */
  }

  button {
    width: 50px; /* 按钮宽度 */
    height: 40px; /* 减小按钮的高度以适应固定容器 */
  }
}

</style>
