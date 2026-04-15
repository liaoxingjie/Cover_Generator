<template>
  <div class="app-container">
    <el-container>
      <!-- 左侧模板选择区 -->
      <el-aside width="280px">
        <div class="sidebar">
          <h2>模板库</h2>
          <el-button @click="refreshTemplates" size="small" icon="Refresh">刷新</el-button>
          <div class="template-list">
            <div 
              v-for="template in templates" 
              :key="template.id"
              :class="['template-item', { active: currentTemplate?.id === template.id }]"
              @click="selectTemplate(template)"
            >
              <img v-if="template.preview" :src="template.preview" :alt="template.name" />
              <div v-else class="no-preview">无预览图</div>
              <span class="template-name">{{ template.name }}</span>
            </div>
          </div>
        </div>
      </el-aside>

      <!-- 中间画布编辑区 -->
      <el-main>
        <div class="canvas-area">
          <div class="toolbar">
            <span>画布比例：</span>
            <el-radio-group v-model="currentRatio" @change="onRatioChange">
              <el-radio-button value="3:4">小红书竖 (3:4)</el-radio-button>
              <el-radio-button value="16:9">小红书横 (16:9)</el-radio-button>
              <el-radio-button value="9:16">抖音竖 (9:16)</el-radio-button>
              <el-radio-button value="16:9">抖音横 (16:9)</el-radio-button>
            </el-radio-group>
          </div>
          <div class="canvas-wrapper">
            <canvas ref="canvasRef" id="main-canvas"></canvas>
          </div>
        </div>
      </el-main>

      <!-- 右侧属性编辑区 -->
      <el-aside width="320px">
        <div class="properties-panel">
          <h2>属性编辑</h2>
          
          <div v-if="selectedObject" class="property-section">
            <h3>选中对象</h3>
            <el-form label-position="top" size="small">
              <el-form-item label="文本内容">
                <el-input 
                  v-if="selectedObject.type === 'i-text'"
                  v-model="selectedObject.text"
                  @input="updateObject"
                />
              </el-form-item>
              <el-form-item label="字号">
                <el-input-number 
                  v-model="selectedObject.fontSize"
                  :min="8"
                  :max="200"
                  @change="updateObject"
                />
              </el-form-item>
              <el-form-item label="颜色">
                <el-color-picker v-model="selectedObject.fill" @change="updateObject" />
              </el-form-item>
              <el-form-item label="字重">
                <el-select v-model="selectedObject.fontWeight" @change="updateObject">
                  <el-option label="常规" value="normal" />
                  <el-option label="粗体" value="bold" />
                </el-select>
              </el-form-item>
              <el-form-item label="对齐方式">
                <el-select v-model="selectedObject.textAlign" @change="updateObject">
                  <el-option label="左对齐" value="left" />
                  <el-option label="居中" value="center" />
                  <el-option label="右对齐" value="right" />
                </el-select>
              </el-form-item>
              <el-button @click="deleteObject" type="danger" size="small">删除</el-button>
            </el-form>
          </div>

          <div class="property-section">
            <h3>文本字段</h3>
            <el-form label-position="top" size="small">
              <el-form-item 
                v-for="(field, index) in templateFields" 
                :key="index"
                :label="field.label"
              >
                <el-input v-model="field.value" @input="updateField(field)" />
              </el-form-item>
            </el-form>
            <el-button @click="addText" type="primary" size="small" style="width: 100%">添加文本</el-button>
          </div>

          <div class="property-section">
            <h3>导出设置</h3>
            <el-form label-position="top" size="small">
              <el-form-item label="格式">
                <el-select v-model="exportFormat">
                  <el-option label="PNG" value="png" />
                  <el-option label="JPG" value="jpeg" />
                </el-select>
              </el-form-item>
              <el-form-item label="质量 (JPG)">
                <el-slider v-model="exportQuality" :min="0.1" :max="1" :step="0.1" />
              </el-form-item>
            </el-form>
            <el-button @click="exportAll" type="success" size="large" style="width: 100%">一键导出四尺寸</el-button>
          </div>
        </div>
      </el-aside>
    </el-container>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, watch } from 'vue'
import { ElMessage } from 'element-plus'
import { fabric } from 'fabric'

// 类型定义
interface Template {
  id: string
  name: string
  preview?: string
  config: TemplateConfig
}

interface TemplateConfig {
  fields: FieldConfig[]
  background?: string
}

interface FieldConfig {
  key: string
  label: string
  value: string
  defaultStyle: {
    fontSize: number
    fill: string
    fontWeight: string
    textAlign: string
    left: number
    top: number
  }
}

// 状态
const canvasRef = ref<HTMLCanvasElement | null>(null)
let canvas: fabric.Canvas | null = null
const templates = ref<Template[]>([])
const currentTemplate = ref<Template | null>(null)
const currentRatio = ref<string>('3:4')
const selectedObject = ref<fabric.Object | null>(null)
const templateFields = ref<FieldConfig[]>([])
const exportFormat = ref<string>('png')
const exportQuality = ref<number>(0.9)

// 画布尺寸配置
const ratioConfig: Record<string, { width: number; height: number }> = {
  '3:4': { width: 900, height: 1200 },
  '16:9': { width: 1280, height: 720 },
  '9:16': { width: 720, height: 1280 }
}

// 初始化
onMounted(() => {
  initCanvas()
  loadTemplates()
})

// 初始化画布
function initCanvas() {
  if (!canvasRef.value) return
  
  canvas = new fabric.Canvas('main-canvas', {
    width: 900,
    height: 1200,
    backgroundColor: '#ffffff'
  })

  canvas.on('selection:created', handleSelection)
  canvas.on('selection:updated', handleSelection)
  canvas.on('selection:cleared', () => {
    selectedObject.value = null
  })
}

// 加载模板
async function loadTemplates() {
  try {
    // 模拟模板数据（实际应从本地目录读取）
    const mockTemplates: Template[] = [
      {
        id: 'template-1',
        name: '简约风格',
        config: {
          fields: [
            {
              key: 'title',
              label: '主标题',
              value: '我的封面标题',
              defaultStyle: {
                fontSize: 72,
                fill: '#333333',
                fontWeight: 'bold',
                textAlign: 'center',
                left: 450,
                top: 400
              }
            },
            {
              key: 'subtitle',
              label: '副标题',
              value: '副标题内容',
              defaultStyle: {
                fontSize: 36,
                fill: '#666666',
                fontWeight: 'normal',
                textAlign: 'center',
                left: 450,
                top: 500
              }
            }
          ]
        }
      },
      {
        id: 'template-2',
        name: '商务风格',
        config: {
          fields: [
            {
              key: 'title',
              label: '主标题',
              value: '专业封面',
              defaultStyle: {
                fontSize: 64,
                fill: '#1a1a1a',
                fontWeight: 'bold',
                textAlign: 'left',
                left: 100,
                top: 300
              }
            }
          ]
        }
      }
    ]
    
    templates.value = mockTemplates
    if (mockTemplates.length > 0) {
      selectTemplate(mockTemplates[0])
    }
  } catch (error) {
    ElMessage.error('加载模板失败')
    console.error(error)
  }
}

// 刷新模板
function refreshTemplates() {
  loadTemplates()
  ElMessage.success('模板已刷新')
}

// 选择模板
function selectTemplate(template: Template) {
  currentTemplate.value = template
  templateFields.value = JSON.parse(JSON.stringify(template.config.fields))
  
  if (canvas) {
    canvas.clear()
    canvas.setBackgroundColor('#ffffff', () => {})
    
    // 添加背景色块示例
    const bgRect = new fabric.Rect({
      left: 0,
      top: 0,
      width: canvas.width,
      height: 100,
      fill: '#f0f0f0',
      selectable: false
    })
    canvas.add(bgRect)
    
    // 添加文本字段
    template.config.fields.forEach(field => {
      const text = new fabric.IText(field.value, {
        left: field.defaultStyle.left,
        top: field.defaultStyle.top,
        fontSize: field.defaultStyle.fontSize,
        fill: field.defaultStyle.fill,
        fontWeight: field.defaultStyle.fontWeight,
        textAlign: field.defaultStyle.textAlign as any,
        fontFamily: 'Arial'
      })
      canvas!.add(text)
    })
    
    canvas.requestRenderAll()
  }
}

// 切换比例
function onRatioChange() {
  if (!canvas) return
  
  const size = ratioConfig[currentRatio.value] || ratioConfig['3:4']
  canvas.setDimensions({ width: size.width, height: size.height })
  
  // 按比例缩放所有对象
  const objects = canvas.getObjects()
  objects.forEach(obj => {
    if (obj.left !== undefined) {
      obj.set('left', (obj.left / 900) * size.width)
    }
    if (obj.top !== undefined) {
      obj.set('top', (obj.top / 1200) * size.height)
    }
  })
  
  canvas.requestRenderAll()
}

// 处理选择
function handleSelection(e: fabric.IEvent) {
  const target = e.selected?.[0]
  if (target) {
    selectedObject.value = target
  }
}

// 更新对象
function updateObject() {
  if (canvas && selectedObject.value) {
    selectedObject.value.setCoords()
    canvas.requestRenderAll()
  }
}

// 删除对象
function deleteObject() {
  if (canvas && selectedObject.value) {
    canvas.remove(selectedObject.value)
    selectedObject.value = null
  }
}

// 更新字段
function updateField(field: FieldConfig) {
  if (!canvas) return
  
  const objects = canvas.getObjects()
  const textObj = objects.find(obj => 
    obj.type === 'i-text' && 
    (obj as fabric.IText).text?.includes(field.key)
  )
  
  if (textObj && textObj.type === 'i-text') {
    ;(textObj as fabric.IText).set('text', field.value)
    canvas.requestRenderAll()
  }
}

// 添加文本
function addText() {
  if (!canvas) return
  
  const newText = new fabric.IText('新文本', {
    left: 100,
    top: 100,
    fontSize: 48,
    fill: '#000000',
    fontWeight: 'normal'
  })
  
  canvas.add(newText)
  canvas.setActiveObject(newText)
  handleSelection({ selected: [newText] } as any)
}

// 导出所有尺寸
function exportAll() {
  if (!canvas) return
  
  const sizes = [
    { name: 'xiaohongshu-vertical', ratio: '3:4' },
    { name: 'xiaohongshu-horizontal', ratio: '16:9' },
    { name: 'douyin-vertical', ratio: '9:16' },
    { name: 'douyin-horizontal', ratio: '16:9' }
  ]
  
  sizes.forEach(size => {
    const config = ratioConfig[size.ratio]
    if (!config) return
    
    // 临时调整画布尺寸
    const originalWidth = canvas.width
    const originalHeight = canvas.height
    
    canvas.setDimensions({ width: config.width, height: config.height })
    
    // 缩放对象
    const objects = canvas.getObjects()
    objects.forEach(obj => {
      if (obj.left !== undefined && originalWidth) {
        obj.set('left', (obj.left / originalWidth) * config.width)
      }
      if (obj.top !== undefined && originalHeight) {
        obj.set('top', (obj.top / originalHeight) * config.height)
      }
    })
    
    canvas.requestRenderAll()
    
    // 导出
    const dataURL = canvas.toDataURL({
      format: exportFormat.value as any,
      quality: exportQuality.value
    })
    
    // 下载
    const link = document.createElement('a')
    link.download = `${size.name}.${exportFormat.value === 'jpeg' ? 'jpg' : 'png'}`
    link.href = dataURL
    link.click()
    
    // 恢复原始尺寸
    canvas.setDimensions({ width: originalWidth, height: originalHeight })
    objects.forEach(obj => {
      if (obj.left !== undefined && config.width) {
        obj.set('left', (obj.left / config.width) * originalWidth)
      }
      if (obj.top !== undefined && config.height) {
        obj.set('top', (obj.top / config.height) * originalHeight)
      }
    })
    canvas.requestRenderAll()
  })
  
  ElMessage.success('四尺寸封面已导出！')
}
</script>

<style scoped lang="scss">
.app-container {
  height: 100vh;
  overflow: hidden;
}

.el-container {
  height: 100%;
}

.sidebar {
  padding: 20px;
  background: #f5f7fa;
  height: 100%;
  overflow-y: auto;
  
  h2 {
    margin-bottom: 15px;
    font-size: 18px;
  }
}

.template-list {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 10px;
  margin-top: 15px;
}

.template-item {
  cursor: pointer;
  border: 2px solid #e4e7ed;
  border-radius: 8px;
  overflow: hidden;
  transition: all 0.3s;
  
  &:hover {
    border-color: #409eff;
  }
  
  &.active {
    border-color: #409eff;
    box-shadow: 0 0 0 2px rgba(64, 158, 255, 0.2);
  }
  
  img {
    width: 100%;
    height: 80px;
    object-fit: cover;
  }
  
  .no-preview {
    width: 100%;
    height: 80px;
    background: #e4e7ed;
    display: flex;
    align-items: center;
    justify-content: center;
    color: #909399;
    font-size: 12px;
  }
  
  .template-name {
    display: block;
    padding: 8px;
    text-align: center;
    font-size: 12px;
    background: #fff;
  }
}

.canvas-area {
  display: flex;
  flex-direction: column;
  height: 100%;
  background: #e4e7ed;
  padding: 20px;
}

.toolbar {
  margin-bottom: 15px;
  display: flex;
  align-items: center;
  gap: 10px;
}

.canvas-wrapper {
  flex: 1;
  display: flex;
  align-items: center;
  justify-content: center;
  background: #fff;
  border-radius: 8px;
  overflow: auto;
  
  canvas {
    box-shadow: 0 2px 12px rgba(0, 0, 0, 0.1);
  }
}

.properties-panel {
  padding: 20px;
  background: #f5f7fa;
  height: 100%;
  overflow-y: auto;
  
  h2 {
    margin-bottom: 15px;
    font-size: 18px;
  }
}

.property-section {
  margin-bottom: 25px;
  padding-bottom: 20px;
  border-bottom: 1px solid #e4e7ed;
  
  &:last-child {
    border-bottom: none;
  }
  
  h3 {
    margin-bottom: 12px;
    font-size: 14px;
    color: #606266;
  }
}
</style>
