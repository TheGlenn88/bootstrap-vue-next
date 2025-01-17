<template>
  <div :class="computedClasses" class="btn-group">
    <b-button
      :id="computedId"
      ref="splitButton"
      :variant="splitVariant || variant"
      :size="size"
      :class="buttonClasses"
      :disabled="splitDisabledBoolean || disabled"
      :type="splitButtonType"
      v-bind="buttonAttr"
      @click="onSplitClick"
      @keydown.esc="emit('update:modelValue', !modelValueBoolean)"
    >
      <slot name="button-content">
        {{ text }}
      </slot>
    </b-button>
    <b-button
      v-if="splitBoolean"
      ref="button"
      :variant="variant"
      :size="size"
      :disabled="disabled"
      :class="toggleClass"
      class="dropdown-toggle-split dropdown-toggle"
      aria-expanded="false"
      @click="onButtonClick"
    >
      <span class="visually-hidden">
        <slot name="toggle-text">
          {{ toggleText }}
        </slot>
      </span>
    </b-button>
    <ul
      v-if="!lazyBoolean || modelValueBoolean"
      v-show="lazyBoolean || modelValueBoolean"
      ref="floating"
      :style="{
        position: strategy,
        top: `${y ?? 0}px`,
        left: `${x ?? 0}px`,
        width: 'max-content',
      }"
      class="dropdown-menu show"
      :class="dropdownMenuClasses"
      :aria-labelledby="computedId"
      :role="role"
      @click="onClickInside"
    >
      <slot />
    </ul>
  </div>
</template>

<script setup lang="ts">
// import type {BDropdownEmits, BDropdownProps} from '../types/components'
import {flip, type Middleware, offset, shift, type Strategy, useFloating} from '@floating-ui/vue'
import {onClickOutside} from '@vueuse/core'
import {computed, ref, toRef, watch} from 'vue'
import {useBooleanish, useId} from '../../composables'
import type {Booleanish, ButtonType, ButtonVariant, ClassValue, Size} from '../../types'
import {BvEvent, resolveFloatingPlacement, stringToInteger} from '../../utils'
import BButton from '../BButton/BButton.vue'

// TODO add navigation through keyboard events
// TODO standardize keydown vs keyup events globally

interface BDropdownProps {
  id?: string
  menuClass?: ClassValue
  size?: Size
  splitClass?: ClassValue
  splitVariant?: ButtonVariant
  text?: string
  toggleClass?: ClassValue
  autoClose?: boolean | 'inside' | 'outside'
  block?: Booleanish
  dark?: Booleanish
  disabled?: Booleanish
  isNav?: Booleanish
  dropup?: Booleanish
  dropend?: Booleanish
  dropstart?: Booleanish
  alignStart?: Booleanish
  alignEnd?: Booleanish
  noFlip?: Booleanish
  noShift?: Booleanish
  offset?: number | string | {mainAxis?: number; crossAxis?: number; alignmentAxis?: number | null}
  role?: string
  split?: Booleanish
  splitButtonType?: ButtonType
  splitHref?: string
  splitDisabled?: Booleanish
  noCaret?: Booleanish
  toggleText?: string
  variant?: ButtonVariant
  modelValue?: Booleanish
  lazy?: Booleanish
  strategy?: Strategy
  floatingMiddleware?: Middleware[]
}

const props = withDefaults(defineProps<BDropdownProps>(), {
  autoClose: true,
  block: false,
  dark: false,
  disabled: false,
  dropup: false,
  isNav: false,
  dropend: false,
  dropstart: false,
  alignEnd: false,
  alignStart: false,
  lazy: false,
  noFlip: false,
  noShift: false,
  offset: 0,
  role: 'menu',
  split: false,
  splitButtonType: 'button',
  splitHref: undefined,
  noCaret: false,
  toggleText: 'Toggle dropdown',
  variant: 'secondary',
  modelValue: false,
  strategy: 'absolute',
})

interface BDropdownEmits {
  (e: 'show', value: BvEvent): void
  (e: 'shown'): void
  (e: 'hide', value: BvEvent): void
  (e: 'hidden'): void
  (e: 'click', event: MouseEvent): void
  (e: 'toggle'): void
  (e: 'update:modelValue', value: boolean): void
}

const emit = defineEmits<BDropdownEmits>()

const computedId = useId(toRef(props, 'id'), 'dropdown')

const modelValueBoolean = useBooleanish(toRef(props, 'modelValue'))
const blockBoolean = useBooleanish(toRef(props, 'block'))
const darkBoolean = useBooleanish(toRef(props, 'dark'))
const dropupBoolean = useBooleanish(toRef(props, 'dropup'))
const dropendBoolean = useBooleanish(toRef(props, 'dropend'))
const isNavBoolean = useBooleanish(toRef(props, 'isNav'))
const dropstartBoolean = useBooleanish(toRef(props, 'dropstart'))
const alignStartBoolean = useBooleanish(toRef(props, 'alignStart'))
const alignEndBoolean = useBooleanish(toRef(props, 'alignEnd'))
const splitBoolean = useBooleanish(toRef(props, 'split'))
const noCaretBoolean = useBooleanish(toRef(props, 'noCaret'))
const noFlipBoolean = useBooleanish(toRef(props, 'noFlip'))
const noShiftBoolean = useBooleanish(toRef(props, 'noShift'))
const lazyBoolean = useBooleanish(toRef(props, 'lazy'))
const splitDisabledBoolean = useBooleanish(toRef(props, 'splitDisabled'))

const floating = ref<HTMLElement | null>(null)
const button = ref<HTMLElement | null>(null)
const splitButton = ref<HTMLElement | null>(null)

const referencePlacement = computed(() => (!splitBoolean.value ? splitButton.value : button.value))
const floatingPlacement = computed(() =>
  resolveFloatingPlacement({
    top: dropupBoolean.value,
    bottom: !dropupBoolean.value,
    start: dropstartBoolean.value,
    end: dropendBoolean.value,
    dropstart: alignStartBoolean.value,
    dropend: alignEndBoolean.value,
  })
)
const floatingMiddleware = computed<Middleware[]>(() => {
  if (props.floatingMiddleware !== undefined) {
    return props.floatingMiddleware
  }
  const off = typeof props.offset === 'string' ? stringToInteger(props.offset, 0) : props.offset
  const arr: Middleware[] = [offset(off)]
  if (noFlipBoolean.value === false) {
    arr.push(flip())
  }
  if (noShiftBoolean.value === false) {
    arr.push(shift())
  }
  return arr
})
const {x, y, strategy, update} = useFloating(referencePlacement, floating, {
  placement: floatingPlacement,
  middleware: floatingMiddleware,
  strategy: props.strategy,
})

const computedClasses = computed(() => ({
  'd-grid': blockBoolean.value,
  'd-flex': blockBoolean.value && splitBoolean.value,
}))

const buttonClasses = computed(() => [
  splitBoolean.value ? props.splitClass : props.toggleClass,
  {
    'nav-link': isNavBoolean.value,
    'dropdown-toggle': !splitBoolean.value,
    'dropdown-toggle-no-caret': noCaretBoolean.value && !splitBoolean.value,
    'w-100': splitBoolean.value && blockBoolean.value,
  },
])

const dropdownMenuClasses = computed(() => [
  props.menuClass,
  {
    'dropdown-menu-dark': darkBoolean.value,
  },
])

const buttonAttr = computed(() => ({
  'aria-expanded': splitBoolean.value ? undefined : false,
  'href': splitBoolean.value ? props.splitHref : undefined,
}))

const onButtonClick = () => {
  emit('toggle')
  const currentModelValue = modelValueBoolean.value
  const e = new BvEvent(currentModelValue ? 'hide' : 'show')
  currentModelValue ? emit('hide', e) : emit('show', e)
  if (e.defaultPrevented) return
  emit('update:modelValue', !currentModelValue)
  currentModelValue ? emit('hidden') : emit('shown')
}

const onSplitClick = (event: MouseEvent) => {
  splitBoolean.value ? emit('click', event) : onButtonClick()
}

onClickOutside(
  floating,
  () => {
    if (modelValueBoolean.value && (props.autoClose === true || props.autoClose === 'outside')) {
      emit('update:modelValue', !modelValueBoolean.value)
    }
  },
  {ignore: [button, splitButton]}
)
const onClickInside = () => {
  if (modelValueBoolean.value && (props.autoClose === true || props.autoClose === 'inside')) {
    // TODO consider using computed({get, set}) syntax (or vueuse/useVModel) for modifying update events in all components
    // For simplicity
    emit('update:modelValue', !modelValueBoolean.value)
  }
}

watch(modelValueBoolean, update)
</script>
