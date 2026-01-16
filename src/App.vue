<script setup lang="ts">
//Импортирую необходимые функции из вью
import { ref, computed, onUnmounted } from "vue";

//Определяем типы данных
interface Selection {
  x: number
  y: number
  width: number
  height: number
}
//Интерфейс для точки (координат мыши)
interface Point {
  x: number
  y: number
}

interface OriginalCoordinates {
  x: number
  y: number
  width: number
  height: number
}
//Реактивные переменные

const imageSrc = ref<string>('')
const imageElement = ref<HTMLImageElement>()
const fileInput = ref<HTMLInputElement>()
const isImageLoaded = ref(false)

const MAX_WIDTH = 800
const MAX_HEIGHT = 600

const originalImageSize = ref({ width: 0, height: 0 })
const displayImageSize = ref({ width: 0, height: 0 })
const scale = ref(1)
//шаг 2 рамка выделения - текущее состояние рамки
const selection = ref<Selection>({
  x: 0,
  y: 0,
  width: 0,
  height: 0
})

// Добавляем переменную для сохраненных координат
const savedCoordinates = ref<OriginalCoordinates | null>(null)

//шаг 3 - переменные для перетаскивания
const isCreatingSelection = ref(false)
const isDragging = ref(false)
const isResizing = ref(false)
const dragStart = ref<Point>({ x: 0, y: 0 })
const selectionStart = ref<Selection>({ x: 0, y: 0, width: 0, height: 0 })
const resizeCorner = ref<string>('')
const creationStart = ref<Point>({ x: 0, y: 0 })

//шаг 4 - вычисляемое свойство для стилей рамки
const selectionStyle = computed(() => ({
  left: `${selection.value.x}px`,
  top: `${selection.value.y}px`,
  width: `${selection.value.width}px`,
  height: `${selection.value.height}px`,
  cursor: isDragging.value ? 'grabbing' : 'move',
  display: selection.value.width > 0 && selection.value.height > 0 ? 'block' : 'none'
}))

const containerStyle = computed(() => ({
  maxWidth: `${MAX_WIDTH}px`,
  maxHeight: `${MAX_HEIGHT}px`,
  width: `${displayImageSize.value.width}px`,
  height: `${displayImageSize.value.height}px`
}))

const imageStyle = computed(() => ({
  width: `${displayImageSize.value.width}px`,
  height: `${displayImageSize.value.height}px`,
  objectFit: 'contain'
}))

const uploadImage = () => {
  fileInput.value?.click()
}

const handleFileSelect = (event: Event) => {
  const input = event.target as HTMLInputElement
  if (input.files && input.files[0]) {
    imageSrc.value = URL.createObjectURL(input.files[0])
    // Сбрасываем сохраненные координаты при загрузке нового изображения
    savedCoordinates.value = null
  }
}

const resetImage = () => {
  imageSrc.value = ''
  isImageLoaded.value = false
  originalImageSize.value = { width: 0, height: 0 }
  displayImageSize.value = { width: 0, height: 0 }
  scale.value = 1
  selection.value = { x: 0, y: 0, width: 0, height: 0 }
  savedCoordinates.value = null
}

const handleImageLoad = () => {
  if (!imageElement.value) return

  const naturalWidth = imageElement.value.naturalWidth
  const naturalHeight = imageElement.value.naturalHeight
  originalImageSize.value = { width: naturalWidth, height: naturalHeight }

  const needsScaling = naturalWidth > MAX_WIDTH || naturalHeight > MAX_HEIGHT

  if (needsScaling) {
    const widthScale = MAX_WIDTH / naturalWidth
    const heightScale = MAX_HEIGHT / naturalHeight

    scale.value = Math.min(widthScale, heightScale)

    displayImageSize.value = {
      width: Math.floor(naturalWidth * scale.value),
      height: Math.floor(naturalHeight * scale.value)
    }
  } else {
    scale.value = 1
    displayImageSize.value = { width: naturalWidth, height: naturalHeight }
  }

  isImageLoaded.value = true

  // Устанавливаем область выделения по умолчанию (25% от отображаемого изображения по центру)
  selection.value = {
    x: displayImageSize.value.width * 0.375,
    y: displayImageSize.value.height * 0.375,
    width: displayImageSize.value.width * 0.25,
    height: displayImageSize.value.height * 0.25
  }
}

// Функция для предотвращения перетаскивания изображения
const preventImageDrag = (event: DragEvent) => {
  event.preventDefault()
  return false
}

// Начало создания выделения
const startCreation = (event: MouseEvent) => {
  if (isDragging.value || isResizing.value) return

  const rect = (event.currentTarget as HTMLElement).getBoundingClientRect()
  const x = event.clientX - rect.left
  const y = event.clientY - rect.top

  // Проверяем, что клик был внутри области изображения
  if (x >= 0 && x <= displayImageSize.value.width &&
      y >= 0 && y <= displayImageSize.value.height) {

    isCreatingSelection.value = true
    creationStart.value = { x, y }

    // Устанавливаем начальную точку выделения
    selection.value = {
      x: x,
      y: y,
      width: 0,
      height: 0
    }

    document.addEventListener('mousemove', handleCreation)
    document.addEventListener('mouseup', stopCreation)
  }
}

// Создание выделения
const handleCreation = (event: MouseEvent) => {
  if (!isCreatingSelection.value) return

  const rect = imageElement.value?.parentElement?.getBoundingClientRect()
  if (!rect) return

  const currentX = event.clientX - rect.left
  const currentY = event.clientY - rect.top

  // Ограничиваем координаты областью изображения
  const boundedX = Math.max(0, Math.min(currentX, displayImageSize.value.width))
  const boundedY = Math.max(0, Math.min(currentY, displayImageSize.value.height))

  // Вычисляем новое выделение
  const newSelection = {
    x: Math.min(creationStart.value.x, boundedX),
    y: Math.min(creationStart.value.y, boundedY),
    width: Math.abs(boundedX - creationStart.value.x),
    height: Math.abs(boundedY - creationStart.value.y)
  }

  // Устанавливаем минимальный размер
  if (newSelection.width >= 5 && newSelection.height >= 5) {
    selection.value = newSelection
  }
}

// Завершение создания выделения
const stopCreation = () => {
  isCreatingSelection.value = false
  document.removeEventListener('mousemove', handleCreation)
  document.removeEventListener('mouseup', stopCreation)
}

// Начало перетаскивания
const startDrag = (event: MouseEvent) => {
  event.preventDefault()
  event.stopPropagation()

  isDragging.value = true
  dragStart.value = {
    x: event.clientX,
    y: event.clientY
  }

  selectionStart.value = { ...selection.value }

  document.addEventListener('mousemove', handleDrag)
  document.addEventListener('mouseup', stopDrag)
}

// Перетаскивание
const handleDrag = (event: MouseEvent) => {
  if (!isDragging.value) return

  const deltaX = event.clientX - dragStart.value.x
  const deltaY = event.clientY - dragStart.value.y

  const maxX = displayImageSize.value.width - selection.value.width
  const maxY = displayImageSize.value.height - selection.value.height

  selection.value.x = Math.max(0, Math.min(selectionStart.value.x + deltaX, maxX))
  selection.value.y = Math.max(0, Math.min(selectionStart.value.y + deltaY, maxY))
}

// Конец перетаскивания
const stopDrag = () => {
  isDragging.value = false
  document.removeEventListener('mousemove', handleDrag)
  document.removeEventListener('mouseup', stopDrag)
}

// Начало изменения размера
const startResize = (event: MouseEvent, corner: string) => {
  event.preventDefault()
  event.stopPropagation()

  isResizing.value = true
  resizeCorner.value = corner

  dragStart.value = { x: event.clientX, y: event.clientY }
  selectionStart.value = { ...selection.value }

  document.addEventListener('mousemove', handleResize)
  document.addEventListener('mouseup', stopResize)
}

// Изменение размера
const handleResize = (event: MouseEvent) => {
  if (!isResizing.value) return

  const deltaX = event.clientX - dragStart.value.x
  const deltaY = event.clientY - dragStart.value.y

  let newSelection = { ...selectionStart.value }

  switch (resizeCorner.value) {
    case 'tl':
      newSelection.x = Math.max(0, Math.min(selectionStart.value.x + deltaX, displayImageSize.value.width - 10))
      newSelection.y = Math.max(0, Math.min(selectionStart.value.y + deltaY, displayImageSize.value.height - 10))
      newSelection.width = selectionStart.value.width - (newSelection.x - selectionStart.value.x)
      newSelection.height = selectionStart.value.height - (newSelection.y - selectionStart.value.y)
      break

    case 'tr':
      newSelection.y = Math.max(0, Math.min(selectionStart.value.y + deltaY, displayImageSize.value.height - 10))
      newSelection.width = Math.max(10, selectionStart.value.width + deltaX)
      newSelection.height = selectionStart.value.height - (newSelection.y - selectionStart.value.y)
      break

    case 'bl':
      newSelection.x = Math.max(0, Math.min(selectionStart.value.x + deltaX, displayImageSize.value.width - 10))
      newSelection.width = selectionStart.value.width - (newSelection.x - selectionStart.value.x)
      newSelection.height = Math.max(10, selectionStart.value.height + deltaY)
      break

    case 'br':
      newSelection.width = Math.max(10, selectionStart.value.width + deltaX)
      newSelection.height = Math.max(10, selectionStart.value.height + deltaY)
      break
  }

  // Проверяем границы
  newSelection.width = Math.min(newSelection.width, displayImageSize.value.width - newSelection.x)
  newSelection.height = Math.min(newSelection.height, displayImageSize.value.height - newSelection.y)

  // Проверяем минимальный размер
  if (newSelection.width >= 5 && newSelection.height >= 5) {
    selection.value = newSelection
  }
}

// Конец изменения размера
const stopResize = () => {
  isResizing.value = false
  document.removeEventListener('mousemove', handleResize)
  document.removeEventListener('mouseup', stopResize)
}

// Функция для вычисления координат изображения
const calculateOriginalCoordinates = (): OriginalCoordinates | null => {
  if (scale.value === 0 || selection.value.width === 0 || selection.value.height === 0) {
    console.error('Выделение не создано или масштаб равен нулю')
    return null
  }

  const originalCoordinates = {
    x: Math.round(selection.value.x / scale.value),
    y: Math.round(selection.value.y / scale.value),
    width: Math.round(selection.value.width / scale.value),
    height: Math.round(selection.value.height / scale.value)
  }

  originalCoordinates.x = Math.min(originalCoordinates.x, originalImageSize.value.width - 1)
  originalCoordinates.y = Math.min(originalCoordinates.y, originalImageSize.value.height - 1)
  originalCoordinates.width = Math.min(originalCoordinates.width, originalImageSize.value.width - originalCoordinates.x)
  originalCoordinates.height = Math.min(originalCoordinates.height, originalImageSize.value.height - originalCoordinates.y)

  return originalCoordinates
}

// Сохранение результата - теперь вычисляет координаты только при нажатии
const saveSelection = () => {
  const coordinates = calculateOriginalCoordinates()

  if (!coordinates) {
    console.error('Не удалось вычислить координаты')
    return null
  }

  // Сохраняем координаты
  savedCoordinates.value = coordinates

  console.log('=== СОХРАНЕННЫЕ КООРДИНАТЫ ВЫДЕЛЕННОЙ ОБЛАСТИ ===')
  console.log('На отображаемом изображении:', selection.value)
  console.log('На оригинальном изображении:', coordinates)
  console.log('Масштаб:', scale.value)
  console.log('===============================')

  // Создаем кастомное событие с координатами
  const event = new CustomEvent('crop-selected', {
    detail: coordinates
  })

  // Эмитим событие
  document.dispatchEvent(event)

  return coordinates
}

// Функция для получения сохраненных координат (если нужно использовать где-то еще)
const getSavedCoordinates = () => {
  return savedCoordinates.value
}

// Очистка при удалении компонента
onUnmounted(() => {
  if (imageSrc.value) {
    URL.revokeObjectURL(imageSrc.value)
  }

  // Удаляем все обработчики событий на случай, если они остались
  document.removeEventListener('mousemove', handleCreation)
  document.removeEventListener('mouseup', stopCreation)
  document.removeEventListener('mousemove', handleDrag)
  document.removeEventListener('mouseup', stopDrag)
  document.removeEventListener('mousemove', handleResize)
  document.removeEventListener('mouseup', stopResize)
})

// Экспортируем функции для использования в родительском компоненте
defineExpose({
  saveSelection,
  getSavedCoordinates,
  resetImage,
  uploadImage
})
</script>

<template>
  <div class="image-cropper">
    <!-- Область загрузки изображения -->
    <div v-if="!imageSrc" class="upload-area" @click="uploadImage">
      <input
          ref="fileInput"
          type="file"
          accept="image/*"
          style="display: none"
          @change="handleFileSelect"
      />
      <p>Загрузите изображение для обрезки</p>
      <p class="upload-hint">Максимальный размер области: {{ MAX_WIDTH }}×{{ MAX_HEIGHT }}px</p>
    </div>

    <!-- Область работы с изображением -->
    <div v-else class="crop-container">
      <h3>Выделите область для обрезки</h3>
      <p>Кликните и перетащите на изображении, чтобы создать выделение</p>
      <p>Затем перетаскивайте рамку или изменяйте размер с помощью уголков</p>

      <!-- Информация о масштабировании -->
      <div v-if="isImageLoaded && scale.value < 1" class="scale-info">
        <p>Изображение масштабировано: {{ Math.round(scale.value * 100) }}% от оригинала</p>
        <p>Оригинал: {{ originalImageSize.width }}×{{ originalImageSize.height }}px →
          Отображение: {{ displayImageSize.width }}×{{ displayImageSize.height }}px</p>
      </div>

      <!-- Контейнер для изображения с ограничениями по размеру -->
      <div
          class="image-container"
          :style="containerStyle"
          @mousedown="startCreation"
      >
        <img
            ref="imageElement"
            :src="imageSrc"
            alt="Изображение для обрезки"
            @load="handleImageLoad"
            :style="imageStyle"
            class="crop-image"
            draggable="false"
            @dragstart.prevent="preventImageDrag"
            @mousedown.prevent
        />

        <!-- Рамка выделения -->
        <div
            v-if="isImageLoaded && selection.width > 0 && selection.height > 0"
            class="selection-box"
            :style="selectionStyle"
            @mousedown="startDrag"
        >
          <!-- Уголки для изменения размера -->
          <div class="corner top-left" @mousedown="startResize($event, 'tl')"></div>
          <div class="corner top-right" @mousedown="startResize($event, 'tr')"></div>
          <div class="corner bottom-left" @mousedown="startResize($event, 'bl')"></div>
          <div class="corner bottom-right" @mousedown="startResize($event, 'br')"></div>
        </div>
      </div>

      <!-- Панель управления -->
      <div class="controls">
        <button @click="resetImage" class="btn btn-secondary">
          Загрузить другое изображение
        </button>
        <button
            @click="saveSelection"
            class="btn btn-primary"
            :disabled="!selection.width || !selection.height"
        >
          Сохранить выделенную область
        </button>
      </div>

      <!-- Информация о сохраненных координатах -->
      <div v-if="savedCoordinates" class="coordinates-info">
        <h4> Координаты сохраненной области:</h4>
        <div class="coordinate-grid">
          <div class="coordinate-item">
            <span class="label">X:</span>
            <span class="value">{{ savedCoordinates.x }}px</span>
          </div>
          <div class="coordinate-item">
            <span class="label">Y:</span>
            <span class="value">{{ savedCoordinates.y }}px</span>
          </div>
          <div class="coordinate-item">
            <span class="label">Ширина:</span>
            <span class="value">{{ savedCoordinates.width }}px</span>
          </div>
          <div class="coordinate-item">
            <span class="label">Высота:</span>
            <span class="value">{{ savedCoordinates.height }}px</span>
          </div>
        </div>
      </div>
      <div v-else class="no-saved-coordinates">
        <p>Нажмите "Сохранить выделенную область" для получения координат фрагмента изображения.</p>
      </div>
    </div>
  </div>
</template>

<style scoped>
.image-cropper {
  font-family: Arial, sans-serif;
  max-width: 800px;
  margin: 0 auto;
  padding: 20px;
}

/* Область загрузки */
.upload-area {
  border: 2px dashed #ccc;
  border-radius: 8px;
  padding: 60px 20px;
  text-align: center;
  cursor: pointer;
  background-color: #f9f9f9;
  transition: all 0.3s;
}

.upload-area:hover {
  border-color: #007bff;
  background-color: #f0f8ff;
}

.upload-area p {
  color: #666;
  font-size: 16px;
  margin: 0 0 10px 0;
}

.upload-hint {
  font-size: 14px !important;
  color: #888 !important;
}

/* Контейнер для кропа */
.crop-container {
  text-align: center;
}

.crop-container h3 {
  color: #333;
  margin-bottom: 8px;
}

.crop-container p {
  color: #666;
  margin-bottom: 10px;
  font-size: 14px;
}

/* Информация о масштабировании */
.scale-info {
  background-color: #e7f3ff;
  border: 1px solid #b3d7ff;
  border-radius: 4px;
  padding: 10px;
  margin-bottom: 15px;
  font-size: 14px;
  color: #0066cc;
}

.scale-info p {
  margin: 5px 0;
  color: #0066cc !important;
}

/* Контейнер изображения */
.image-container {
  position: relative;
  display: inline-block;
  margin: 0 auto;
  overflow: hidden;
  border: 1px solid #ddd;
  background-color: #f5f5f5;
  box-sizing: content-box;
  cursor: crosshair;
  user-select: none;
  -webkit-user-select: none;
  -moz-user-select: none;
  -ms-user-select: none;
}

.crop-image {
  display: block;
  max-width: 100%;
  max-height: 100%;
  user-select: none;
  -webkit-user-select: none;
  -moz-user-select: none;
  -ms-user-select: none;
  pointer-events: none;
  -webkit-user-drag: none;
  -khtml-user-drag: none;
  -moz-user-drag: none;
  -o-user-drag: none;
}

.selection-box {
  position: absolute;
  border: 2px solid #007bff;
  background-color: rgba(0, 123, 255, 0.1);
  box-sizing: border-box;
  pointer-events: auto;
  user-select: none;
  -webkit-user-select: none;
  -moz-user-select: none;
  -ms-user-select: none;
}

.corner {
  position: absolute;
  width: 12px;
  height: 12px;
  background-color: #007bff;
  border: 2px solid white;
  border-radius: 2px;
  box-shadow: 0 0 2px rgba(0, 0, 0, 0.3);
  pointer-events: auto;
}

.top-left {
  top: -6px;
  left: -6px;
  cursor: nw-resize;
}

.top-right {
  top: -6px;
  right: -6px;
  cursor: ne-resize;
}

.bottom-left {
  bottom: -6px;
  left: -6px;
  cursor: sw-resize;
}

.bottom-right {
  bottom: -6px;
  right: -6px;
  cursor: se-resize;
}

.controls {
  margin: 20px 0;
  display: flex;
  gap: 10px;
  justify-content: center;
}

.btn {
  padding: 10px 20px;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 14px;
  transition: background-color 0.3s;
  min-width: 200px;
  user-select: none;
  -webkit-user-select: none;
  -moz-user-select: none;
  -ms-user-select: none;
}

.btn-secondary {
  background-color: #6c757d;
  color: white;
}

.btn-secondary:hover {
  background-color: #5a6268;
}

.btn-primary {
  background-color: #007bff;
  color: white;
}

.btn-primary:hover {
  background-color: #0069d9;
}

.btn-primary:disabled {
  background-color: #cccccc;
  cursor: not-allowed;
}
.coordinates-info {
  background-color: #e7f6ec;
  border: 1px solid #b8e1c7;
  border-radius: 4px;
  padding: 15px;
  margin-top: 20px;
  text-align: left;
}

.coordinates-info h4 {
  color: #0f5132;
  margin: 0 0 15px 0;
  font-size: 16px;
  display: flex;
  align-items: center;
  gap: 8px;
}

.coordinate-grid {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 10px;
  margin-bottom: 10px;
}

.coordinate-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 8px;
  background: white;
  border: 1px solid #d1e7dd;
  border-radius: 4px;
}

.coordinate-item .label {
  color: #0f5132;
  font-size: 14px;
  font-weight: bold;
}

.coordinate-item .value {
  color: #198754;
  font-weight: bold;
  font-family: monospace;
  font-size: 14px;
}

.no-saved-coordinates {
  background-color: #fff3cd;
  border: 1px solid #ffecb5;
  border-radius: 4px;
  padding: 15px;
  margin-top: 20px;
}

.no-saved-coordinates p {
  color: #856404;
  margin: 0;
  font-size: 14px;
  text-align: center;
}
</style>