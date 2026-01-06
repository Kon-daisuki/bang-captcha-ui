<script setup>
import { ref, onMounted } from 'vue';

// ⚠️ 这里填你已经部署好的 Worker 后端域名 (不带 /api)
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
    challenge.value = data;
    
    // 修复图片路径：因为后端返回的是相对路径 /api/img/xxx，我们需要拼接上前缀
    if (challenge.value && challenge.value.images) {
      challenge.value.images = challenge.value.images.map(img => {
        return img.startsWith('http') ? img : `${API_BASE}${img}`;
      });
    }
  } catch (error) {
    console.error(error);
    message.value = '加载验证码失败';
  } finally {
    loading.value = false;
  }
};

// 选中/取消选中图片
const toggleSelect = (index) => {
  if (isSuccess.value) return; // 成功后禁止修改
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
    // 1. 发送给后端验证 (UI层面的预验证)
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
        message.value = "验证通过！阿里嘎多~";
        
        // 2. 核心逻辑：验证通过后，通知父窗口(注册页)
        // 延迟 800ms 关闭，让用户看到成功的提示
        setTimeout(() => {
            window.parent.postMessage({
                type: 'CAPTCHA_RESULT',
                payload: {
                    captchaId: challenge.value.id,
                    selectedIndexes: selectedIndexes.value
                }
            }, '*');
        }, 800);
    } else {
        message.value = "选错了哦，再试一次吧";
        // 失败后 1秒自动刷新
        setTimeout(() => { 
            if(!isSuccess.value) fetchCaptcha(); 
        }, 1000);
    }
  } catch (e) {
      message.value = "请求出错";
  } finally {
      loading.value = false;
  }
};

onMounted(() => {
  fetchCaptcha();
});
</script>

<template>
  <div class="captcha-container">
    <div v-if="loading && !challenge" class="loading-state">
      <div class="spinner"></div>
      <p>正在召唤乐队成员...</p>
    </div>

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
          <div class="checkmark" v-if="selectedIndexes.includes(index)">✓</div>
        </div>
      </div>

      <div class="footer">
        <button class="btn refresh" @click="fetchCaptcha">刷新</button>
        <button 
            class="btn verify" 
            @click="verifyCaptcha" 
            :disabled="selectedIndexes.length === 0 || isSuccess"
        >
            {{ isSuccess ? '完成' : '确认' }}
        </button>
      </div>

      <div v-if="message" class="message" :class="{ success: isSuccess }">
        {{ message }}
      </div>
    </div>
    
    <div v-else class="error-state" @click="fetchCaptcha">
        加载失败，点击重试
    </div>
  </div>
</template>

<style>
/* 简单的重置 */
body { margin: 0; padding: 0; font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif; background-color: transparent; }

.captcha-container {
  width: 320px;
  background: #fff;
  padding: 16px;
  border-radius: 12px;
  /* 因为是在弹窗里，我们去掉外阴影，或者给一点点 */
  /* box-shadow: 0 4px 20px rgba(0,0,0,0.1); */
  margin: 0 auto;
  box-sizing: border-box;
}

.header { font-size: 15px; margin-bottom: 12px; color: #333; }
.target-name { color: #e91e63; font-weight: bold; font-size: 16px; }

.grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 6px;
  margin-bottom: 16px;
}

.img-wrapper {
  position: relative;
  aspect-ratio: 1;
  cursor: pointer;
  border-radius: 6px;
  overflow: hidden;
  border: 2px solid transparent;
  transition: all 0.2s;
}

.img-wrapper img { width: 100%; height: 100%; object-fit: cover; display: block; }

.img-wrapper.active {
  border-color: #e91e63;
  transform: scale(0.96);
}

.checkmark {
  position: absolute; top: 4px; right: 4px;
  background: #e91e63; color: white;
  width: 18px; height: 18px;
  border-radius: 50%;
  font-size: 12px; line-height: 18px; text-align: center;
  box-shadow: 0 2px 4px rgba(0,0,0,0.2);
}

.footer { display: flex; gap: 10px; }
.btn {
  flex: 1; border: none; padding: 8px 0; border-radius: 6px;
  font-size: 14px; cursor: pointer; font-weight: 500;
  transition: opacity 0.2s;
}
.btn:disabled { opacity: 0.5; cursor: not-allowed; }
.btn.refresh { background: #f5f5f5; color: #666; }
.btn.verify { background: #e91e63; color: white; }

.message {
  margin-top: 12px; text-align: center; font-size: 13px; color: #ff4d4f;
  background: #fff1f0; padding: 6px; border-radius: 4px;
}
.message.success { color: #52c41a; background: #f6ffed; }

/* Loading 状态 */
.loading-state { text-align: center; padding: 40px 0; color: #999; font-size: 14px; }
.spinner {
  width: 24px; height: 24px; border: 3px solid #f3f3f3;
  border-top: 3px solid #e91e63; border-radius: 50%;
  margin: 0 auto 10px; animation: spin 1s linear infinite;
}
@keyframes spin { 0% { transform: rotate(0deg); } 100% { transform: rotate(360deg); } }
</style>
