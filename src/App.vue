<script setup>
import { ref, onMounted } from 'vue';

// ⚠️ 确保这里填的是你 Worker 后端的正确地址
const API_BASE = 'https://band.kessoku.us.kg';

const challenge = ref(null);
const selectedIndexes = ref([]);
const message = ref('');
const isSuccess = ref(false);
const loading = ref(false);

// 获取验证码
const fetchCaptcha = async () => {
  loading.value = true;
  message.value = '';
  isSuccess.value = false;
  selectedIndexes.value = [];
  
  try {
    const res = await fetch(`${API_BASE}/api/captcha`);
    if (!res.ok) throw new Error('网络连接失败');
    const data = await res.json();
    
    // 图片路径处理
    if (data && data.images) {
      data.images = data.images.map(img => {
        return img.startsWith('http') ? img : `${API_BASE}${img}`;
      });
    }
    challenge.value = data;
  } catch (error) {
    console.error(error);
    message.value = '图片加载失败，请刷新重试';
  } finally {
    loading.value = false;
  }
};

// 选中/取消选中
const toggleSelect = (index) => {
  if (isSuccess.value) return; 
  if (selectedIndexes.value.includes(index)) {
    selectedIndexes.value = selectedIndexes.value.filter(i => i !== index);
  } else {
    selectedIndexes.value.push(index);
  }
};

// 提交验证
const verifyCaptcha = async () => {
  if (!challenge.value) return;
  loading.value = true;
  
  try {
    const res = await fetch(`${API_BASE}/api/verify`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
            id: challenge.value.id,
            selectedIndexes: selectedIndexes.value
        })
    });
    const data = await res.json();

    if (data.success) {
        isSuccess.value = true;
        message.value = ""; // 清空文字，直接显示成功动画
        
        // --- 核心逻辑：通知父窗口 ---
        console.log("验证成功，准备发送消息给父窗口...");
        
        // 延迟 1秒 发送消息，让用户看完成功动画
        setTimeout(() => {
            // 如果是在 iframe 中
            window.parent.postMessage({
                type: 'CAPTCHA_RESULT',
                payload: {
                    captchaId: challenge.value.id,
                    selectedIndexes: selectedIndexes.value
                }
            }, '*');

            // 如果你是直接在浏览器打开这个网页测试，弹个窗提示一下
            if (window.self === window.top) {
                alert('验证通过！(独立窗口模式)');
            }
        }, 1000);

    } else {
        message.value = "选错啦，再仔细看看~";
        selectedIndexes.value = []; // 清空选择
        // 失败后 1.5秒自动刷新
        setTimeout(() => { 
            if(!isSuccess.value) {
                message.value = '';
                fetchCaptcha(); 
            }
        }, 1500);
    }
  } catch (e) {
      message.value = "请求出错，请重试";
  } finally {
      loading.value = false;
  }
};

onMounted(() => {
  fetchCaptcha();
});
</script>

<template>
  <!-- 外层容器：负责居中 -->
  <div class="app-wrapper">
    <div class="captcha-card">
      
      <!-- Loading 状态 -->
      <div v-if="loading && !challenge" class="loading-state">
        <div class="spinner"></div>
        <p>正在召唤成员...</p>
      </div>

      <!-- 主要内容 -->
      <div v-else-if="challenge" class="content-box">
        <div class="header">
          请选出所有的 <span class="target-name">{{ challenge.targetName }}</span>
        </div>
        
        <div class="grid">
          <div 
            v-for="(img, index) in challenge.images" 
            :key="index"
            class="img-wrapper"
            :class="{ active: selectedIndexes.includes(index) }"
            @click="toggleSelect(index)"
          >
            <img :src="img" loading="lazy" />
            <!-- 选中时的遮罩和对勾 -->
            <transition name="scale">
              <div class="selection-overlay" v-if="selectedIndexes.includes(index)">
                <div class="checkmark-icon">✓</div>
              </div>
            </transition>
          </div>
        </div>

        <!-- 底部按钮区 -->
        <div class="footer">
          <button class="btn refresh" @click="fetchCaptcha" :disabled="isSuccess">
            <span class="icon">↻</span>
          </button>
          <button 
              class="btn verify" 
              @click="verifyCaptcha" 
              :disabled="selectedIndexes.length === 0 || isSuccess"
          >
              {{ isSuccess ? '验证通过' : '确认提交' }}
          </button>
        </div>

        <!-- 错误提示消息 -->
        <div v-if="message" class="message error">
          {{ message }}
        </div>

        <!-- 成功时的全屏遮罩动画 -->
        <transition name="fade">
            <div v-if="isSuccess" class="success-mask">
                <div class="success-content">
                    <div class="big-checkmark">✓</div>
                    <p>验证通过</p>
                </div>
            </div>
        </transition>

      </div>
      
      <!-- 网络错误状态 -->
      <div v-else class="error-state" @click="fetchCaptcha">
          <p>加载失败</p>
          <button class="btn refresh-link">点击重试</button>
      </div>
    </div>
  </div>
</template>

<style>
/* 1. 全局重置：确保根元素占满屏幕且无默认边距 */
html, body, #app {
    margin: 0;
    padding: 0;
    width: 100%;
    height: 100%;
    overflow: hidden; /* 禁止滚动条，防止抖动 */
    font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, "Helvetica Neue", Arial, sans-serif;
}

/* 2. 容器设置：使用 Flexbox 实现绝对居中 */
.app-wrapper {
    display: flex;
    justify-content: center; /* 水平居中 */
    align-items: center;     /* 垂直居中 */
    width: 100%;
    height: 100%; /* 占满父容器 */
    
    /* 背景色：建议使用轻微的遮罩色，这样嵌入 Iframe 时更自然 */
    background: rgba(255, 255, 255, 0.0); /* 透明背景，让父级背景透出来，或者保留你喜欢的渐变 */
    /* 如果你希望验证码页面有独立背景，可以用下面的代码： */
    /* background: linear-gradient(135deg, #fdfbfb 0%, #ebedee 100%); */
}

/* 3. 卡片样式优化 */
.captcha-card {
    width: 340px;
    max-width: 90vw; /* 移动端适配：最大宽度不超过屏幕90% */
    background: #fff;
    padding: 20px;
    border-radius: 16px;
    box-shadow: 0 10px 30px rgba(0,0,0,0.1); 
    position: relative;
    box-sizing: border-box;
    /* 增加 margin auto 确保在某些 flex 失效场景下也能居中 */
    margin: auto; 
}

/* --- 以下保持原有样式 --- */
.header { font-size: 16px; margin-bottom: 15px; color: #444; text-align: center; }
.target-name { color: #e91e63; font-weight: bold; border-bottom: 2px dashed #e91e63; padding-bottom: 2px; }

.grid {
    display: grid;
    grid-template-columns: repeat(3, 1fr);
    gap: 8px;
    margin-bottom: 20px;
}

.img-wrapper {
    position: relative;
    aspect-ratio: 1;
    cursor: pointer;
    border-radius: 8px;
    overflow: hidden;
    transition: transform 0.2s;
    background: #f0f0f0;
}
.img-wrapper:active { transform: scale(0.95); }
.img-wrapper.active { box-shadow: 0 0 0 3px #e91e63; transform: scale(0.96); }
.img-wrapper img { width: 100%; height: 100%; object-fit: cover; display: block; }

.selection-overlay {
    position: absolute; top: 0; left: 0; width: 100%; height: 100%;
    background: rgba(233, 30, 99, 0.2); display: flex; align-items: center; justify-content: center;
}
.checkmark-icon {
    background: #e91e63; color: white; width: 24px; height: 24px; border-radius: 50%;
    font-size: 14px; display: flex; align-items: center; justify-content: center;
}

.footer { display: flex; gap: 12px; height: 44px; }
.btn { border: none; border-radius: 8px; cursor: pointer; font-size: 15px; font-weight: 600; display: flex; align-items: center; justify-content: center; }
.btn.refresh { width: 44px; background: #f5f7fa; color: #666; font-size: 20px; }
.btn.verify { flex: 1; background: #e91e63; color: white; }
.btn:disabled { opacity: 0.6; cursor: not-allowed; }

.message { margin-top: 15px; text-align: center; font-size: 14px; padding: 8px; border-radius: 6px; }
.message.error { color: #ff4d4f; background: #fff2f0; }

.success-mask {
    position: absolute; top: 0; left: 0; width: 100%; height: 100%;
    background: rgba(255, 255, 255, 0.95); border-radius: 16px;
    display: flex; align-items: center; justify-content: center; z-index: 10; flex-direction: column;
}
.big-checkmark { font-size: 60px; color: #52c41a; animation: popIn 0.5s; }
.success-content p { color: #52c41a; font-weight: bold; margin-top: 10px; text-align: center; }

.loading-state { text-align: center; padding: 40px 0; color: #888; }
.spinner {
    width: 30px; height: 30px; border: 3px solid #f0f0f0; border-top: 3px solid #e91e63;
    border-radius: 50%; margin: 0 auto 15px; animation: spin 0.8s linear infinite;
}

@keyframes spin { 0% { transform: rotate(0deg); } 100% { transform: rotate(360deg); } }
@keyframes popIn { 0% { transform: scale(0); opacity: 0; } 80% { transform: scale(1.1); } 100% { transform: scale(1); opacity: 1; } }
.scale-enter-active, .scale-leave-active { transition: all 0.2s; }
.scale-enter-from, .scale-leave-to { opacity: 0; transform: scale(0.5); }
.fade-enter-active, .fade-leave-active { transition: opacity 0.3s; }
.fade-enter-from, .fade-leave-to { opacity: 0; }
</style>
