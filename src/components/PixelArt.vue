<template>
  <el-container>
    <el-main>
      <div class="container">
        <div id="canvas" ref="canvas" :class="currentTool"></div>
        <el-card class="controls">
          <template #header>
            <div class="card-header">
              <span>工具栏</span>
            </div>
          </template>
          
          <div class="button-group">
            <el-tooltip content="画笔工具" placement="right">
              <el-button 
                :type="currentTool === 'pen' ? 'primary' : ''" 
                @click="setTool('pen')"
                icon="Edit"
              >画笔</el-button>
            </el-tooltip>
            
            <el-tooltip content="橡皮擦工具" placement="right">
              <el-button 
                :type="currentTool === 'eraser' ? 'primary' : ''" 
                @click="setTool('eraser')"
                icon="Delete"
              >橡皮</el-button>
            </el-tooltip>
          </div>

          <div class="color-section">
            <input type="color" v-model="selectedColor" class="color-picker">
          </div>
          
          <el-divider />
          
          <div class="button-group">
            <el-tooltip content="清空画布" placement="right">
              <el-button type="danger" @click="clearCanvas" icon="Delete">清除</el-button>
            </el-tooltip>
            
            <el-tooltip content="上传图片" placement="right">
              <el-button type="success" @click="handleUpload" icon="Upload">上传</el-button>
            </el-tooltip>
            
            <el-divider>导出选项</el-divider>
            
            <el-tooltip content="导出为PNG" placement="right">
              <el-button type="primary" @click="exportAs('1')" icon="Picture">PNG</el-button>
            </el-tooltip>
            
            <el-tooltip content="导出为JPEG" placement="right">
              <el-button type="primary" @click="exportAs('2')" icon="Picture">JPEG</el-button>
            </el-tooltip>
            
            <el-tooltip content="导出为SVG" placement="right">
              <el-button type="primary" @click="exportAs('3')" icon="PictureFilled">SVG</el-button>
            </el-tooltip>
            
            <el-tooltip content="导出为ICO" placement="right">
              <el-button type="primary" @click="exportAs('4')" icon="Monitor">ICO</el-button>
            </el-tooltip>
          </div>
        </el-card>
      </div>
    </el-main>
    
    <input
      type="file"
      ref="fileInput"
      accept="image/*"
      style="display: none"
      @change="onFileSelected"
    >
  </el-container>
</template>

<script>
import { ElMessageBox, ElMessage } from 'element-plus'

export default {
  name: 'PixelArt',
  data() {
    return {
      isDrawing: false,
      currentTool: 'pen',
      selectedColor: '#000000',
      pixels: []
    }
  },
  mounted() {
    this.initCanvas()
    this.setupEventListeners()
    this.pixels.forEach(pixel => {
      pixel.style.backgroundColor = 'white'
    })
    ElMessage.success('画布已准备就绪')
  },
  methods: {
    initCanvas() {
      const canvas = this.$refs.canvas
      // 创建32x32的像素网格
      for (let i = 0; i < 32 * 32; i++) {
        const pixel = document.createElement('div')
        pixel.classList.add('pixel')
        canvas.appendChild(pixel)
        this.pixels.push(pixel)
      }
    },
    setupEventListeners() {
      const canvas = this.$refs.canvas
      canvas.addEventListener('mousedown', this.startDrawing)
      canvas.addEventListener('mousemove', this.draw)
      canvas.addEventListener('mouseup', this.stopDrawing)
      canvas.addEventListener('mouseleave', this.stopDrawing)
    },
    setTool(tool) {
      this.currentTool = tool
    },
    startDrawing(e) {
      this.isDrawing = true
      this.draw(e)
    },
    draw(e) {
      if (!this.isDrawing) return
      
      const pixel = e.target
      if (pixel.classList.contains('pixel')) {
        if (this.currentTool === 'pen') {
          pixel.style.backgroundColor = this.selectedColor
        } else if (this.currentTool === 'eraser') {
          if (pixel.style.backgroundColor && pixel.style.backgroundColor !== 'white') {
            pixel.style.backgroundColor = 'white'
          }
        }
      }
    },
    stopDrawing() {
      this.isDrawing = false
    },
    clearCanvas() {
      ElMessageBox.confirm('确定要清空画布吗？', '提示', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning'
      }).then(() => {
        this.pixels.forEach(pixel => {
          pixel.style.backgroundColor = 'white'
        })
        ElMessage.success('画布已清空')
      }).catch(() => {})
    },
    handleUpload() {
      this.$refs.fileInput.click()
    },
    onFileSelected(e) {
      const file = e.target.files[0]
      if (!file) return

      const reader = new FileReader()
      reader.onload = (event) => {
        const img = new Image()
        img.onload = () => {
          const tempCanvas = document.createElement('canvas')
          tempCanvas.width = 32
          tempCanvas.height = 32
          const ctx = tempCanvas.getContext('2d')

          ctx.drawImage(img, 0, 0, 32, 32)
          const imageData = ctx.getImageData(0, 0, 32, 32)

          if (confirm('上传的图片将会替换当前画布内容，是否继续？')) {
            for (let i = 0; i < imageData.data.length; i += 4) {
              const pixelIndex = Math.floor(i / 4)
              const r = imageData.data[i]
              const g = imageData.data[i + 1]
              const b = imageData.data[i + 2]
              const a = imageData.data[i + 3]

              if (a > 0) {
                const color = `rgba(${r}, ${g}, ${b}, ${a / 255})`
                this.pixels[pixelIndex].style.backgroundColor = color
              } else {
                this.pixels[pixelIndex].style.backgroundColor = 'white'
              }
            }
          }
        }
        img.src = event.target.result
      }
      reader.readAsDataURL(file)
      e.target.value = ''
    },
    exportAs(format) {
      const tempCanvas = document.createElement('canvas')
      const ctx = tempCanvas.getContext('2d')
      const pixelSize = 1000
      tempCanvas.width = pixelSize
      tempCanvas.height = pixelSize

      ctx.clearRect(0, 0, pixelSize, pixelSize)
      const cellSize = pixelSize / 32

      this.pixels.forEach((pixel, index) => {
        const row = Math.floor(index / 32)
        const col = index % 32
        const color = pixel.style.backgroundColor
        
        if (color && color !== 'white') {
          ctx.fillStyle = color
          ctx.fillRect(col * cellSize, row * cellSize, cellSize, cellSize)
          
          ctx.strokeStyle = '#d0d0d0'
          ctx.lineWidth = 2
          ctx.strokeRect(col * cellSize, row * cellSize, cellSize, cellSize)
        }
      })

      let fileName = 'pixel-art'
      let mimeType = 'image/png'
      
      switch (format) {
        case '1': // PNG
          fileName += '.png'
          mimeType = 'image/png'
          break
        case '2': // JPEG
          {
            const imageData = ctx.getImageData(0, 0, pixelSize, pixelSize)
            ctx.fillStyle = 'white'
            ctx.fillRect(0, 0, pixelSize, pixelSize)
            ctx.putImageData(imageData, 0, 0)
            fileName += '.jpg'
            mimeType = 'image/jpeg'
          }
          break
        case '3': // SVG
          this.exportAsSVG()
          return
        case '4': // ICO
          fileName += '.ico'
          mimeType = 'image/x-icon'
          break
        default:
          ElMessage.error('无效的格式选择')
          return
      }

      tempCanvas.toBlob((blob) => {
        const url = URL.createObjectURL(blob)
        const a = document.createElement('a')
        a.href = url
        a.download = fileName
        document.body.appendChild(a)
        a.click()
        document.body.removeChild(a)
        URL.revokeObjectURL(url)
        ElMessage.success(`已导出为 ${fileName}`)
      }, mimeType)
    },
    exportAsSVG() {
      const size = 1000
      const cellSize = size / 32
      
      let svgContent = `<svg width="${size}" height="${size}" xmlns="http://www.w3.org/2000/svg">`
      
      this.pixels.forEach((pixel, index) => {
        const row = Math.floor(index / 32)
        const col = index % 32
        const color = pixel.style.backgroundColor
        if (color && color !== 'white') {
          svgContent += `<rect x="${col * cellSize}" y="${row * cellSize}" 
            width="${cellSize}" height="${cellSize}" 
            fill="${color}"
            stroke="#d0d0d0"
            stroke-width="2"/>`
        }
      })
      
      svgContent += '</svg>'
      
      const blob = new Blob([svgContent], { type: 'image/svg+xml' })
      const url = URL.createObjectURL(blob)
      const a = document.createElement('a')
      a.href = url
      a.download = 'pixel-art.svg'
      document.body.appendChild(a)
      a.click()
      document.body.removeChild(a)
      URL.revokeObjectURL(url)
    }
  }
}
</script>

<style scoped>
.container {
  display: flex;
  flex-direction: row;
  justify-content: center;
  gap: 30px;
  padding: 20px;
}

#canvas {
  width: 826px;
  height: 826px;
  border: 2px solid #dcdfe6;
  border-radius: 4px;
  display: grid;
  grid-template-columns: repeat(32, 1fr);
  grid-template-rows: repeat(32, 1fr);
  background-color: #ccc;
  gap: 1px;
  padding: 1px;
  box-shadow: 0 2px 12px 0 rgba(0, 0, 0, 0.1);
}

.controls {
  width: 200px;
}

.button-group {
  display: flex;
  flex-direction: column;
  gap: 10px;
  padding: 0 10px;
}

.color-section {
  margin: 15px 0;
  padding: 0 10px;
}

.color-picker {
  width: 100%;
  height: 40px;
  padding: 0;
  border: 1px solid #dcdfe6;
  border-radius: 4px;
  cursor: pointer;
}

.controls .el-button {
  width: 100%;
  height: 40px;
  display: flex;
  align-items: center;
  justify-content: center;
}

.controls .el-color-picker {
  width: 100%;
}

.card-header {
  display: flex;
  justify-content: center;
  align-items: center;
  padding: 0 10px;
}

#canvas.pen {
  cursor: crosshair;
}

#canvas.eraser {
  cursor: cell;
}

.pixel {
  background-color: white;
  width: 100%;
  height: 100%;
  border: none;
  aspect-ratio: 1;
}

.pixel:hover {
  outline: 1px solid #666;
  outline-offset: -1px;
}

.el-divider {
  margin: 15px 10px;
}

/* 为导出按钮组添加样式 */
.button-group .el-divider {
  margin: 10px 0;
  font-size: 12px;
  color: #909399;
}

.button-group .el-button {
  margin-bottom: 8px;
}
</style> 