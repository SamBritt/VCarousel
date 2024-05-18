<template>
  <div class="carousel">
    <div :style="{ position: 'relative' }">
      <div
        ref="scrollerParentRef"
        class="carousel-container">
        <div
          ref="scrollerRef"
          :class="['carousel-scroll', { 'is-scrolling': isScrolling }]"
          :style="{ transform: `translateX(${translateX}px)` }"
          @keyup.tab="tabChildren"
          @pointerenter="handlePointerEnter"
          @pointerdown="handlePointerDown"
          @pointermove="handlePointerMove"
          @pointerup="handlePointerUp"
          @pointerleave="handlePointerLeave">
          <slot />
        </div>
      </div>

      <button
        v-if="showLeftButton"
        class="carousel-arrow-buttons button-left"
        @click="() => selectPage(currentPage - 1)"
        aria-label="Scroll Left Button">
        <img
          src=""
          role="presentation"
          class="icon-left"
          alt="left button" />
      </button>

      <button
        v-if="showRightButton"
        class="carousel-arrow-buttons button-right"
        @click="() => selectPage(currentPage + 1)"
        aria-label="Scroll Right Button">
        <img
          src=""
          role="presentation"
          class="icon-right"
          alt="right button" />
      </button>
    </div>

    <div
      v-if="dots && !hideDots"
      class="carousel-dots">
      <button
        v-for="(dot, idx) in dots"
        :key="`dot-${idx}`"
        :class="['carousel-dot', { 'is-active': currentPage === idx }]"
        :aria-label="`Carousel page ${dot} of ${dots}`"
        :aria-current="currentPage === idx"
        tabindex="0"
        @click="() => selectPage(idx)"></button>
    </div>
  </div>
</template>

<script lang="ts" setup>
import { computed, onBeforeMount, onMounted, onUnmounted, ref, watch, type Ref } from 'vue'

const { hideArrows = false, hideDots = false } = defineProps<{
  hideArrows?: Boolean
  hideDots?: Boolean
  previewMode?: 'infinite' | 'switch' | 'page'
}>()

const scrollerRef: Ref<HTMLElement | null> = ref(null)
const scrollerParentRef: Ref<HTMLElement | null> = ref(null)
const children: Ref<HTMLElement[]> = ref([])
const focusedChild: Ref<HTMLElement | undefined> = ref(undefined)

const scrollByPositions: Ref<number[]> = ref([])
const resizing = ref(false)
const draggingEnabled = ref(true)
const isDown = ref(false)
const mouseMoving = ref(false)
const startX = ref(0)
const startValue = ref(0)
const endValue = ref(0)
const showRightButton = ref(false)
const showLeftButton = ref(false)
const debounceResizeId = ref(0)
const parentRightPos = ref(0)
const endX = ref(0)
const distanceTraveled = ref(0)
const translateX = ref(0)
const endTranslateX = ref(0)
const dots = ref(0)
const currentPage = ref(0)
const windowWidth = ref(0)

onBeforeMount(() => window.addEventListener('resize', handleResize))

onMounted(() => init())

onUnmounted(() => {
  window.removeEventListener('resize', handleResize)
  resizing.value = false
})

watch(
  () => [mouseMoving.value],
  () => {
    if (mouseMoving.value) {
      children.value.forEach((element) => {
        element.style.pointerEvents = 'none'
        element.style.cursor = 'none'
        element.style.userSelect = 'none'
      })
    } else {
      children.value.forEach((element) => {
        element.style.pointerEvents = 'auto'
        element.style.cursor = 'pointer'
        element.style.userSelect = 'auto'
      })
    }
  }
)

const isScrolling = computed(() => isDown.value && mouseMoving.value)

/**
 * Initializes core values.
 * No scrolling functionality when there aren't enough children
 */
const init = () => {
  if (scrollerRef.value) {
    children.value = Array.from(scrollerRef.value.children) as HTMLElement[]
  }

  draggingEnabled.value = scrollerRef.value!.offsetWidth > scrollerParentRef.value!.offsetWidth

  windowWidth.value = window.innerWidth

  if (!draggingEnabled.value) {
    selectPage(0)
    dots.value = 0
    currentPage.value = 0
    showLeftButton.value = false
    showRightButton.value = false
    return
  }

  if (draggingEnabled.value) {
    showRightButton.value = true
    setEndPosition()
    createScrollValues()
    applyFocusToChildren()
    calculatePages()

    if (currentPage.value >= dots.value) {
      selectPage(dots.value - 1)
    }
  }

  const { right } = scrollerParentRef.value!.getBoundingClientRect()

  parentRightPos.value = right
}

/**
 * Hides or shows arrows depending on position
 */
const checkArrowValues = () => {
  if (translateX.value >= endTranslateX.value) {
    showRightButton.value = true
    showLeftButton.value = true
  }

  if (translateX.value <= endTranslateX.value) {
    showRightButton.value = false
  }

  if (translateX.value >= 0) {
    showLeftButton.value = false
  }
}

/**
 * Sets total pages
 */
const calculatePages = () => {
  if (scrollByPositions.value.length < 1) {
    dots.value = 0
    return
  }

  dots.value = scrollByPositions.value.length
}

/**
 * Creates values to scroll by
 */
const createScrollValues = () => {
  const childrenInView = Array.from(children.value).filter(
    (item: HTMLElement) => item.offsetLeft < scrollerParentRef.value!.offsetWidth
  )

  let isSoloChild = childrenInView.length <= 1
  let lastChild = childrenInView[childrenInView.length - 1]
  let border = parseInt(window.getComputedStyle(lastChild).getPropertyValue('border-width'))
  let scrollByValue = !isSoloChild ? lastChild.offsetLeft : children.value[1].offsetLeft

  let pages = [] as Array<number>
  let idx = 0

  for (let i = 0; i >= endTranslateX.value; i -= scrollByValue - border) {
    pages[idx] = i
    idx += 1
  }

  if (pages[pages.length - 1] > endTranslateX.value) {
    pages.push(endTranslateX.value)
  }

  scrollByPositions.value = pages
}

const handlePointerEnter = () => {
  isDown.value = false
  mouseMoving.value = false
}

const handlePointerDown = (e: PointerEvent) => {
  if (!draggingEnabled.value) return

  e.preventDefault()

  startX.value = e.clientX
  isDown.value = true
  startValue.value = translateX.value
  focusedChild.value && focusedChild.value.blur()
  focusedChild.value = undefined
}

const handlePointerLeave = () => {
  if (!draggingEnabled.value) return
  if (!isDown.value) return

  isDown.value = false
  checkArrowValues()

  if (translateX.value >= 0) {
    translateX.value = 0
    return
  }

  if (translateX.value <= endTranslateX.value) {
    translateX.value = endTranslateX.value
    return
  }

  if (translateX.value > scrollByPositions.value[currentPage.value]) {
    selectPage(currentPage.value - 1)
  } else {
    selectPage(currentPage.value + 1)
    return
  }
}

const handlePointerUp = () => {
  if (!draggingEnabled.value) return

  isDown.value = false
  mouseMoving.value = false
  distanceTraveled.value = 0
  endValue.value = translateX.value

  snapToPage()
  checkArrowValues()
}

const handlePointerMove = (e: PointerEvent) => {
  if (!draggingEnabled.value) return
  if (!isDown.value) return

  mouseMoving.value = true

  if (mouseMoving.value && !isDown.value) return

  e.preventDefault()

  endX.value = e.clientX
  distanceTraveled.value = endX.value - startX.value
  translateX.value = translateX.value + distanceTraveled.value
  startX.value = e.clientX
}

/**
 * Reinitializes when resized
 */
const handleResize = () => {
  if (window.innerWidth !== windowWidth.value) {
    resizing.value = false

    clearTimeout(debounceResizeId.value)

    debounceResizeId.value = setTimeout(() => {
      init()
    }, 1000)

    resizing.value = false
  }

  windowWidth.value = window.innerWidth
}

/**
 * Snaps to page depending on direction dragged
 */
const snapToPage = () => {
  if (!draggingEnabled.value) return

  const { right } = scrollerRef.value!.getBoundingClientRect()

  if (translateX.value > 0) {
    selectPage(0)
    return
  }

  if (Math.round(right) <= parentRightPos.value) {
    translateX.value = endTranslateX.value
    currentPage.value = dots.value - 1
    return
  }

  let scrolledBy = endValue.value - startValue.value

  if (endValue.value < startValue.value) {
    scrolledBy < -10 ? selectPage(currentPage.value + 1) : selectPage(currentPage.value)
    return
  }

  scrolledBy > 10 ? selectPage(currentPage.value - 1) : selectPage(currentPage.value)
}

/**
 * Selects page
 * @param idx page to select
 */
const selectPage = (idx: number) => {
  if (idx < dots.value && idx > -1) {
    currentPage.value = idx
    translateX.value = scrollByPositions.value[idx]
    checkArrowValues()
  }
}

/**
 * Sets end value
 */
const setEndPosition = () => {
  endTranslateX.value = -(
    scrollerRef.value!.getBoundingClientRect().width - scrollerParentRef.value!.offsetWidth
  )
}

/**
 * Ensures children are tabbable
 */
const applyFocusToChildren = () => {
  Array.from(children.value).forEach((element) => {
    return (element.tabIndex = 0)
  })
}
/**
 * Handles paging when tabbing
 * @param e event
 */
const tabChildren = (e: KeyboardEvent) => {
  const target = e.target as HTMLElement

  if (e !== undefined) {
    let childrenInView = children.value.filter(
      (el) => el.offsetLeft < scrollerParentRef.value!.offsetWidth
    )
    const previousChild = focusedChild.value ? focusedChild.value : target

    if (focusedChild.value === undefined) {
      children.value[0].focus()
      selectPage(0)
    }

    if (!children.value.includes(target)) return

    focusedChild.value = target

    const firstChild = childrenInView[0]
    let indexOfPrevious = children.value.indexOf(previousChild)
    let indexOfCurrent = children.value.indexOf(focusedChild.value)

    const parentWidth = scrollerParentRef.value!.offsetWidth
    const elementLeft = focusedChild.value.offsetLeft
    const elementWidth = focusedChild.value.offsetWidth
    console.log(currentPage.value)
    console.log((elementLeft + elementWidth) * currentPage.value > parentWidth)

    if (indexOfPrevious < indexOfCurrent) {
      if (
        (elementLeft + elementWidth) * (currentPage.value + 1) >
        parentWidth * (currentPage.value + 1)
      ) {
        selectPage(currentPage.value + 1)
      }
    }

    if (indexOfPrevious > indexOfCurrent) {
      if (children.value.includes(focusedChild.value)) {
        selectPage(currentPage.value - 1)
      }
    }

    if (firstChild === target && currentPage.value !== 0) {
      selectPage(0)
    }

    if (focusedChild.value === children.value[children.value.length - 1]) {
      selectPage(dots.value - 1)
    }
  }
}
</script>

<style scoped lang="scss">
.carousel {
  position: relative;

  &-container {
    position: relative;
    width: 100%;
    display: flex;
    overflow-x: clip;
    touch-action: pan-y;
  }

  &-scroll {
    display: flex;
    transition: transform 0.5s ease;

    &::-webkit-scrollbar {
      display: none;
    }

    &.is-scrolling {
      cursor: -webkit-grabbing;
      cursor: -moz-grabbing;
      cursor: grabbing;
      transition: none;
    }
  }

  &-arrow-buttons {
    position: absolute;
    padding: 1rem;
    display: flex;
    align-items: center;
    justify-content: center;
    top: 50%;
    transform: translateY(-50%);
    width: 2rem;
    height: 2rem;
    background: rgba(246, 246, 246, 0.13);
    border: 2px solid rgba(255, 255, 255, 0.8);
    border-radius: 100%;
    z-index: 40;

    &.button-left {
      left: 10px;
    }

    &.button-right {
      right: 10px;
    }

    .icon-left {
      transform: rotate(180deg);
    }
  }

  &-dots {
    display: flex;
    justify-content: center;
    width: 100%;
    padding: 1rem 0;
    cursor: pointer;
  }

  &-dot {
    width: calc(1rem / 2.5);
    height: calc(1rem / 2.5);
    padding: 0;
    border: none;
    border-radius: 100%;
    background: lightgray;
    margin: 0 0.25rem;

    &.is-active {
      transition: all 200ms ease;
      background: #7c878e;
      width: 0.5rem;
      height: 0.5rem;
    }
  }
}
</style>
