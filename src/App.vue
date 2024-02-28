<template>
  <a-config-provider
      :theme="{
      token: {
        colorPrimary: '#ffcd00',
        colorText: '#000000',

      },

    }"
  >

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
          <img src="../src/assets/tuji@3x.png" alt="Profile Picture">
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
        <div class="vertical-layout">
          <a-row type="flex" justify="center" class="row-item">
            <a-col :span="24">
              <a-dropdown>
                <a-button>
                  添加剧情
                  <DownOutlined />
                </a-button>
                <template #overlay>
                  <a-menu>
                    <a-menu-item v-for="story in stories" :key="story.id"
                                 :disabled="!story.enabled"
                                 @click="updateStoryStateAndSendMessage(story.id)">
                      {{ story.header }}
                    </a-menu-item>
                  </a-menu>
                </template>
              </a-dropdown>
            </a-col>
          </a-row>

          <a-row type="flex" justify="center" class="row-item" >
            <a-col :span="24">
              <a-switch class="switchCOT" v-model:checked="switchCOT" checked-children="显示思维链" un-checked-children="隐藏思维链"></a-switch>
            </a-col>
          </a-row>

          <a-row type="flex" justify="center" class="row-item">
            <a-col :span="24">
              <a-button type="primary" style="color:#161717" @click="showModal">日记</a-button>
              <a-modal v-model:open="open" title="Basic Modal" @ok="handleOk">
              </a-modal>
            </a-col>
          </a-row>
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
               :class="['message', message.sender === 'User' ? 'user-message' : message.sender === 'System' ? 'system-message' : 'ai-message']">

            <!-- 头像容器，如果系统消息不需要头像，则可以这里做条件渲染 -->
            <div v-if="message.sender !== 'System'" class="message-avatar"></div>
            <!-- 消息内容 -->
            <span :id="'message-' + message.request_id">{{ message.content }}</span>
          </div>
        </div>
        <!-- 输入容器 -->
        <div id="input-container">
          <input type="text" v-model="userInput" placeholder="在此处输入文本开始体验哦..." @keypress.enter="sendMessage" />
          <a-space direction="horizontal">
            <a-button type="primary" style="color:#161717" :loading="sendMessagesLoading" @click="sendMessage">
              {{ sendButtonStr }}
            </a-button>
            <!-- 触发下拉菜单的按钮 -->
            <a-button type="primary" style="color:#161717" @click="toggleDropdown">
              <span>功能</span>
            </a-button>
          </a-space>


        </div>
      </div>

    </div>

    <!-- 底部反馈区 -->
    <div id="feedback-section">
      <!-- 反馈输入区域 -->
    </div>

  </div>
  </a-config-provider>
</template>

<script setup>
import { ref, onMounted } from 'vue';
import { v4 as uuidv4 } from 'uuid';
import { DownOutlined } from '@ant-design/icons-vue';
import axios from "axios";

// 使用ref创建响应式引用
// const remoteUrl = ref("http://127.0.0.1:8000");
const remoteUrl = ref("http://182.254.242.30:8888");
const userInput = ref('');
const feedbackInput = ref('');
const messages = ref([]);
const lastMessageLength = ref(0);
const sessionId = ref(''); // 假设在onMounted或其他生命周期钩子中赋值
const typeWriterIndexes = ref({});
const situationData = ref(null);
const isExpanded = ref(false);
const showDropdown = ref(false);
const strength = ref(88);
const energy = ref(68);
const mood = ref("开心");
const characterLocation = ref("客厅");
const action = ref("站立");
const switchCOT = ref(false);
const sendMessagesLoading = ref(false);/*发送按钮的加载状态*/
const sendButtonStr = ref("发送");/*发送按钮的文字*/
const open = ref(false);
const generateDiary = () => {

};
const showModal = () => {
  open.value = true;
};
const handleOk = () => {
  open.value = false;
};

const stories = ref([
  {id: 1 ,header :"新手教程",content:'"看来我们要在新的时空开始冒险了！！"{user}一扭头就撞上了{char}充满期待的大眼神，"我该怎么称呼你呢？"\n' +
        '（{user}输入昵称，昵称={user}）\n' +
        '"你可以叫我哥哥。"{user}回答。\n' +
        '"哥哥！我喜欢这个称呼~"{char}开心的蹦了起来，"我我，我是从另一个故事的兔子洞来到这里的，但是我好像还没有名字..."兔子女孩尴尬的低下了头。\n' +
        '（给兔子女孩起个名字，名字={char}）\n' +
        '"不知道你喜不喜欢{char}这个名字呢？"{user}问。\n' +
        '"{char}！我爱这个名字！我也能拥有自己的名字了~会写童话故事的人类都好厉害！"\n' +
        '"emmm，我好像并不具备这种能力..."{user}回答。\n' +
        '"你们可是童话世界的创世主！呃，那个..."{char}突然扑闪着亮晶晶的大眼睛看着{user}，"虽然我之前都是一个小小的童话配角，但是这次，拜托拜托，"{char}充满期待的大眼睛着实让人难以拒绝。\n' +
        '"我可以陪你转转，但不能确保——"{user}回答。\n' +
        '"太棒了！我们该先做点什么呢？猪吉要是能说再点啥就好了..."\n' +
        '"我猜他是在装睡。"{user}回答。\n' +
        '（点击猪，猪打了个嗝，获得了金币）\n' +
        '"哇！哥哥能用猪吉变出魔法金币！"\n' +
        '"我是猪鳄！！“猪鳄气鼓鼓的满脸不情愿，“可恶的人类，还以为能多藏一会儿呢，好吧好吧，我确实能提供你们生存所需的一点经济能力”，“记住！只有一点点点点。”\n' +
        '（连续点击猪，连续获得金币）\n' +
        '“你你你你你！”猪鳄气地说不出话来，直接闭上了眼睛。\n' +
        '“哈哈！哥哥比猪吉厉害多了！略略略~”{char}得意的冲猪鳄做了个鬼脸。\n' +
        '“咕——”{char}害羞的低下了头，”肯...肯定是猪吉在叫！“\n' +
        '“哈哈，是我好饿呢，我们去找找房间里有没有吃的吧！”{user}回答。\n' +
        '（切换到厨房，开始烹饪）\n' +
        '”哇，肚子吃的圆鼓鼓了！哥哥做的食物真好吃！“{char}满足的摸了摸小肚子，“我们去有很多小花的地方看看吧！”\n' +
        '（切换到种植间，收露水）\n' +
        '“这里的露珠都在空中飘飘！！好神奇！”{char}好奇的看着种植间里漂浮的露珠。\n' +
        '“ 哥哥一下子就抓到露珠了！我都够不着！”{char}努力的伸着小手掂了掂脚，又嘟着嘴低下了头。\n' +
        '”啊，那是什么大宝贝？！“{char}突然发现了什么似的跑向了种植间的角落。\n' +
        '{user}用露水只做了一瓶香香汽水\n' +
        '”哥哥变出了香香汽水！哥哥是创世主大人！“{char}兴奋的喊道。\n' +
        '”上面写着沐浴时加入，可以消除一天的疲劳，变得元气满满...“，{char}念着念着就睁大了亮晶晶的眼睛看着{user}。\n' +
        '“今天经历了这么多，确实也该舒缓一下疲劳了呢。”{user}回答。',active:false,enabled:true},
  { id: 2, header: '遇到博物馆保安咕噜和呱呱', content: '关键地点：博物馆\n' +
        '关键人物：咕噜、呱呱\n' +
        '故事概要：咕噜和呱呱是博物馆的保安，他们的职责是维持博物馆的秩序，但由于天生的生理构造咕噜的眼睛只能往两边看，所以经常用余光瞥人，而呱呱的眼睛只能往头顶看，他努力往前看的时候总给人一种蔑视感；所以大家都觉得他们目中无人，很难相处。当然他们彼此之间由于无法对视，也并不认同对方。久而久之他们开始接受自己的人设，不再刻意解释。{user}和{char}不知道写个什么故事，想来到博物馆找一些灵感，遇到了博物馆盗窃案和咕噜和呱呱的重重阻挠，{char}决定写一个推理故事，自己成为大侦探，破案的同时解开了他俩的心结，发现他俩和好比抓到犯人更有意义，觉得自己不想继续当一个合格的侦探，开始想别的故事。（地点：博物馆前门）\n' +
        '“你好，鱼鱼先生~请问我们可以进去吗？~”{char}可爱又有礼貌的询问了一下。\n' +
        '“我叫咕噜，请。”咕噜面朝前方，但仿佛并没有看到{char}。\n' +
        '“等等！很抱歉您不能进去。”咕噜突然往后一步拦住了{char}，“您的着装不规范。”\n' +
        '“啊？可是你刚刚还说...”{char}不开心的嘟嘟嘴。\n' +
        '“抱歉，我只有在侧面才能看清您。”咕噜仿佛在机械的背台词。\n' +
        '“呜呜，可是我只有这一件衣服嘛...”{char}可怜巴巴的看着咕噜，但咕噜仿佛并没有看到{char}。\n' +
        '“你这么想去博物馆看看的话，不如我们去侧门试试吧~”{user}小声建议道。\n' +
        '“好！”{char}又重新打起了精神。\n' +
        '（地点：博物馆侧门）\n' +
        '“侧门是呱呱先生！它好像只能看到天花板！”{char}兴奋的说道。\n' +
        '“可是天花板是镜面做的呢。”{user}回答。\n' +
        '“或许，我们可以...”{char}转动着小眼睛，仿佛想到了什么好主意，躲进了{user}的大衣里。\n' +
        '“你好，我想进入博物馆参观。”{user}礼貌的说道。\n' +
        '“请，呱。”呱呱面无表情的说道。\n' +
        '“阿——”{char}小声的打了个喷嚏。\n' +
        '“阿嚏——”{user}赶紧接上了喷嚏，呱呱看了一眼天花板镜子里的{user}，并没有说什么。', active: false, enabled: true },
  { id: 3, header: '成为大侦探', content: '（地点：博物馆大厅）\n' +
        '“成功过关！”{char}从{user}的大衣里跳了出来，一脸得意的模样。\n' +
        '“不过，我这么做不会给咕噜和呱呱带来麻烦吧？他们好像对待工作很严肃的样子...”{char}苦恼的看向{user}。\n' +
        '“你要是不捣蛋应该没问题吧。”{user}话音刚落，博物馆的灯瞬间灭了，警报立刻响了起来。\n' +
        '“啊啊啊！不会是来抓我们的吧！”{char}赶紧又躲进了{user}的大衣里。\n' +
        '（博物馆的灯随即又恢复了供电）\n' +
        '“感觉不是很妙的样子...”{user}看着从正门冲进来的咕噜，正好用侧面死死的盯着他俩，且迅速转身向他俩冲了过来。\n' +
        '“我已经看到你的小尾巴了！是谁放你们进来的？！”咕噜突然在十步开外刹住了脚步，努力张大了嘴字正腔圆的质问道，“就是你们偷了博物馆的镇馆之宝吗？”\n' +
        '“啊！我们可没有偷东西呢！”{char}偷偷探出了半个小脑袋，着急的解释道。\n' +
        '“应该是有什么误会吧，博物馆是失窃了吗？”{user}礼貌的问道。\n' +
        '“对不起，请你们配合检查。”咕噜在十步开外严肃的叫道。\n' +
        '“他们，我放进来的。”呱呱慢悠悠的走了过来，漫不经心的说道。\n' +
        '“ 所以这就是你一个人看门的目的？！把不法分子放进来破坏博物馆？！”咕噜突然从刚刚的严肃变得激动起来。\n' +
        '“咕噜，喝水。”呱呱慢悠悠的说道，只见咕噜拿起手边的保温杯，喝了一口后便平静了下来。\n' +
        '“先找到馆宝吧。”呱呱看了一眼{user}和{char}，“小家伙藏得挺好，先接受一下检查吧。”\n' +
        '“对...对不起咕噜先生，偷偷溜进来是我不对。”{char}老老实实的站了出来，认真的道歉。\n' +
        '“啊，那个，下次不许这样了啊。”咕噜揣着保温杯突然有点脸红，语气也不自在起来。\n' +
        '“原来咕噜也没有这么凶巴巴嘛~”{char}偷偷跟{user}说。\n' +
        '“你们有看到什么异常情况吗？”呱呱问道。\n' +
        '“我们刚进来，还没来得及...”{user}话还没说完，{char}就赶紧拉住了{user}\n' +
        '“不如我们就写一个大侦探的故事吧！我想当大侦探！哥哥！”{char}兴奋的小声跟{user}说道。\n' +
        '“不过，我想我们可以协助调查，为你们提供更多的信息。”{user}清了清嗓子，认真的说道。\n' +
        '“博物馆现在确实需要帮助，那就拜托你们了。”呱呱悠悠的说道。\n' +
        '“这么关键的时候！你是不是又想偷懒！”咕噜突然又激动起来。\n' +
        '“多个帮手能早点找到馆宝，是好事。”，呱呱说道，“博物馆的馆宝是一颗稀有的矿石，刚才自动触发的警报就是因为它消失了。”\n' +
        '“那我们快去现场调查吧！”{char}挥舞着小手，干劲十足。\n' +
        '“那我们分头行动，我和呱呱先去调查一下博物馆的录像，你和咕噜去现场收集线索好吗？”{user}问{char}\n' +
        '“好哒，咕噜先生出发！”{char}蹦蹦跳跳的跑向咕噜，咕噜有些不知所措的跟在{char}身后。', active: false, enabled: false },
  { id: 4, header: '博物馆内的失窃案', content: '（地点：录像室）\n' +
        '“从录像带上看，离展柜最近的只有咕咕，但毛儿手脚灵活的很，不过他们都是博物馆的老熟人了，我并不怀疑他们。”呱呱慢悠悠的说道，“虽说我从没见过你们，你们一出现就出事了，但我也不怀疑你们。”\n' +
        '“？我不是很明白...”{user}回答道。\n' +
        '“直觉吧，好了，说说你的发现吧。”呱呱把画面转向{user}。\n' +
        '（打开录像带，在断电前后的画面上圈出可疑的点）\n' +
        '{user}：“咕咕小姐前后几乎没有移动过位置，连表情都很连贯，看上去像是吓坏了。”\n' +
        '  “毛儿有臂长的优势，但很难如此精巧的取到矿石...”\n' +
        '  “我猜测是一种个头远小于他们的家伙干的，当然还有一种可能，那就是矿石自己跑了..”\n' +
        '“哦？矿石自己跑了，听上去是个值得调查的方向哈哈哈。”呱呱开心的大笑道，“那我们去看看你的小伙伴有没有找到什么它自己跑了的线索。”', active: false, enabled: false, collapsible: 'disabled' },
  { id: 5, header: '真相只有一个！', content: '（地点：博物馆中心区域）\n' +
        '“哥哥！哥哥你快来看！”{char}看到{user}，兴奋的蹦跳着叫道，“我发现矿石自己逃跑的线索了！”\n' +
        '“你！你们简直就是胡闹！”咕噜皱紧了眉头，紧紧的攥着保温杯。\n' +
        '“矿石本来就是镇子上的天外来物，现在自己跑了也很合理嘛~”呱呱笑着拍了拍咕噜的肩膀。\n' +
        '“我在陈列台旁边薄薄的灰上找到了矿石的小脚印！咕噜给我看过矿石的模样，每个印记都能和矿石的棱角对上！” {char}认真的凑在台子前指着淡淡的痕迹说道。\n' +
        '“也有可能是某种飞行动物拖动矿石留下的痕迹！”咕噜喝了一口水，冷漠的说道。\n' +
        '“嗯，咕噜说的也很有道理呢！”{char}认真的点了点头。\n' +
        '“噗...那个，我就是随便猜测的。”咕噜一口水差点喷出来，完全没想到{char}会认同自己。\n' +
        '“对了，矿石是天外来物嘛？”{user}转向呱呱问道。\n' +
        '“是咕噜在森林里发现的，我们从没见过这样的矿石，而且它好像还砸出了一个大洞。”呱呱说道。\n' +
        '“他还打算在那个洞里种菜，太离谱了！！”咕噜一想起这事又激动起来。\n' +
        '“哈哈，哈哈，你不是前阵子叨叨着想吃嫩白菜。”呱呱憨憨的笑了起来。\n' +
        '“哇！我也喜欢吃嫩白菜！我也可以帮呱呱种菜！”{char}一脸馋样的举起了小手。\n' +
        '“不如我们现在就去那个洞附近看看吧~”{user}提议道。\n' +
        '“好耶！我要去看看我的白菜天堂！”{char}迫不及待地就要往外跑。', active: false, enabled: false, collapsible: 'disabled' },
]);
// 使用onMounted代替created生命周期钩子
onMounted(() => {
  const storedSessionId = localStorage.getItem('sessionId');
  if (storedSessionId) {
    sessionId.value = storedSessionId;
  } else {
    sessionId.value = uuidv4();
    localStorage.setItem('sessionId', sessionId.value);
  }

  validateSession();
  fetchSituationData();
  fetchStatus();
});
const replacePlaceholders = (content) => {
  return content.replace(/{user}/g, '哥哥').replace(/{char}/g, '兔叽');
};
const updateStoryStateAndSendMessage = (storyId) => {
  const storyIndex = stories.value.findIndex(story => story.id === storyId);
  if (storyIndex !== -1 && stories.value[storyIndex].enabled) {
    // 标记当前故事为已读并禁用
    stories.value[storyIndex].active = true;
    stories.value[storyIndex].enabled = false;

    // 解锁下一个故事（如果存在）
    if (storyIndex + 1 < stories.value.length) {
      stories.value[storyIndex + 1].enabled = true;
    }
    // 处理故事内容中的占位符
    const processedContent = replacePlaceholders(stories.value[storyIndex].content);
    const requestId = Date.now().toString(); // 生成唯一请求ID

    updateMessage(sessionId, 'system', processedContent);
    messages.value.push({ sender: 'System', content: '', request_id: requestId }); // 添加一个空消息
    typeWriterEffectSystem(processedContent, requestId, 10); // 启动打字机效果
  }
};

const updateMessage = async (sessionId, role, messageContent) => {
  try {
    const response = await fetch(`${remoteUrl.value}/update-message`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        sessionId: sessionId.value,
        role: role,
        message: messageContent
      })
    });

    const data = await response.json();

    if (data.success) {
      console.log('Message updated successfully');
    } else {
      console.log('Failed to update message');
    }
  } catch (error) {
    console.error('Error updating message:', error);
  }
};

const validateSession = async () => {
  try {
    const response = await fetch(remoteUrl.value + '/validate-session', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({ sessionId: sessionId.value })
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
};
const addOrUpdateMessage = (sender, content, requestId) => {
  // 方法内容不变
  const existingMessageIndex = messages.value.findIndex(m => m.request_id === requestId);
  if (existingMessageIndex !== -1) {
    // 使用打字机更新现有消息的内容而不使用vue响应式特性
    // this.messages[existingMessageIndex].content = content;
    typeWriterEffect(content, requestId, 100);
  } else {
    // 首次添加空白新消息
    content="";
    messages.value.push({ sender, content, request_id: requestId });
    // 初始化typeWriterIndex为0
    typeWriterIndexes[requestId] = 0;
  }
};
const addSystemMessage = (sender, content, requestId) => {
  const existingMessageIndex = messages.value.findIndex(m => m.request_id === requestId);
  if (existingMessageIndex === -1) {
    // 首次添加消息，初始化内容为空字符串
    messages.value.push({ sender, content: '', request_id: requestId });
    // 初始化该消息的打字机效果索引为0，并开始逐字符显示
    typeWriterIndexes.value[requestId] = 0;
    typeWriterEffectSystem(content, requestId, 100);
  }
  // 如果消息已存在，不做任何操作，因为系统消息仅传输一次
};

const typeWriterEffect = (text, requestId, speed = 100) => {
  const messageIndex = messages.value.findIndex(m => m.request_id === requestId);
  const typeWriterIndex = typeWriterIndexes[requestId] || 0;

  if (messageIndex !== -1 && typeWriterIndex < text.length) {
    messages.value[messageIndex].content += text.charAt(typeWriterIndex);
    typeWriterIndexes[requestId]++; // 更新typeWriterIndex
    setTimeout(() => {
      typeWriterEffect(text, requestId, speed); // 继续递归调用
    }, speed);
  } else {
    // 打字机效果完成后的延时请求
    checkAndUpdateTasks(messages.value[messageIndex].content);
  }
};
const typeWriterEffectSystem = (text, requestId, speed = 100) => {
// 确保每个消息的打字索引是独立管理的
  if (typeWriterIndexes.value[requestId] === undefined) {
    typeWriterIndexes.value[requestId] = 0;
  }

  const index = typeWriterIndexes.value[requestId];
  const messageIndex = messages.value.findIndex(m => m.request_id === requestId);

  // 当消息索引有效且未达到文本长度时继续打印
  if (messageIndex !== -1 && index < text.length) {
    messages.value[messageIndex].content += text.charAt(index);
    typeWriterIndexes.value[requestId] = index + 1;
    console.log(index)
    setTimeout(() => {
      typeWriterEffectSystem(text, requestId, speed);
    }, speed);
  }
};


const checkAndUpdateTasks = (finalText) => {
  // 检查文本是否包含特定的任务
  if (finalText.includes("情境更新任务")) {
    setTimeout(() => {
      // 请求情境更新的逻辑
      fetchSituationData()
    }, 5000);
  }
  if (finalText.includes("记忆更新任务")) {
    setTimeout(() => {
      // 请求记忆更新的逻辑
      // this.updateMemory();
    }, 5000);
  }
  if (finalText.includes("情绪更新任务")) {
    setTimeout(() => {
      // 请求情绪更新的逻辑
      // this.updateEmotion();
    }, 5000);
  }
};

const sendMessage = async () => {
  if (userInput.value.trim()) {
    messages.value.push({
      sender: 'User',
      content: userInput.value,
      request_id: Date.now().toString() // 生成一个临时的request_id
    });
    sendMessagesLoading.value = false;
    sendButtonStr.value = '发送';
    // 使用 fetch 发送请求

    try {
      sendMessagesLoading.value = true;
      sendButtonStr.value = '发送中...';
      const response = await fetch(remoteUrl.value+'/generate', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          // 其他需要的头部信息
        },
        body: JSON.stringify({
          data: userInput.value, // 用户输入的数据
          sessionId: sessionId.value, // 携带sessionId
          fullCOT: switchCOT.value // 携带COT开关状态
        })
      });
      // 清空输入框
      userInput.value = '';
      // 处理流式响应
      const reader = response.body.getReader();
      sendButtonStr.value = '回复中...';
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
          console.log(jsonStr)
          try {
            // 解析JSON数据
            const jsonData = JSON.parse(jsonStr); // 从"data:"之后开始解析
            const text = jsonData?.output?.text;
            const requestId = jsonData?.request_id;
            if (text && requestId) {
              addOrUpdateMessage('Ai', text,requestId);

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
      sendMessagesLoading.value = false;
      sendButtonStr.value = '发送';
    }
    sendMessagesLoading.value = false;
    sendButtonStr.value = '发送';

  }
};

const fetchSituationData = async () => {
  try {

    const response = await fetch(remoteUrl.value+'/situation-data/', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        // 其他需要的头部信息
      },
      body: JSON.stringify({
        sessionId: sessionId.value // 携带sessionId
      })
    });

    if (!response.ok) {
      // 如果响应状态不是2xx, 抛出错误
      throw new Error(`Error fetching situation data: ${response.status}`);
    }

    // 确保使用 await 调用 .json() 方法
    const data = await response.json();
    console.log(data.situation)
    // 更新组件的数据属性
    situationData.value = data.situation;

    // typeWriterEffectSystem(processedContent, requestId, 10); // 启动打字机效果
    // 例如，保存到数据属性中：
    // this.situation = situationData;
  } catch (error) {
    console.error('Error fetching situation data:', error);
  }
};

const fetchStatus = async () => {
  // 方法内容不变
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
  //height: 60%;
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
  content: url('../src/assets/energy_icon.png'); /* 元气值图标 */
}

.strength-icon:before {
  content: url('../src/assets/strength_icon.png'); /* 体力值图标 */
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

.vertical-layout .row-item {
  margin-top: 24px;
  //padding-top: 24px; /* 设置垂直间距为24px */
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
  max-height: 160px;
  overflow: hidden;
  padding: 10px;
  margin: 16px;
  background-color: #444444;
  border-radius: 10px;
  text-align: left;
  white-space: pre-wrap;
  transition: max-height 0.8s ease; /* 动画过渡效果 */
}
#chat-header:hover {
  max-height: none; /* 鼠标悬停时移除最大高度限制 */
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
  //margin-right: 32px;
  word-wrap: break-word;
  white-space: pre-wrap;
}


.system-message {
  background-image: linear-gradient(to right, rgba(59, 59, 59, 1), rgba(255, 205, 0, 0.6));
  color: #f0f0f0;
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
  background-image: url('../src/assets/ai_head.png'); /* 根据实际路径调整 */

}

/* 应用头像样式 */
.user-message .message-avatar {
  background-color: #ffcd00; /* 用户头像使用亮黄色背景 */
  background-image: url('../src/assets/user_head.jpg'); /* 根据实际路径调整 */
}

/* 输入框和按钮的容器样式 */
#input-container {
  display: flex;
  //justify-content: space-between; /* 使输入框和按钮分布在两侧 */
  align-items: center; /* 垂直居中对齐 */
  //height: 80px;
  position: relative; /* 设置相对定位用于下拉菜单的绝对定位 */
}



/* 输入框样式 */
input[type="text"] {
  flex-grow: 4; /* 输入框占据剩余空间 */
  margin-right: 10px; /* 与按钮之间的距离 */
  height: 36px; /* 输入框高度，您可以根据需要调整 */
  //padding: 10px; /* 增加内边距以便文本居中 */
  border-radius: 20px;
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





/* 按钮文本样式，确保在动画之上 */

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
