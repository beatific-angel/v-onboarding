<template>
  <div v-if="!isFinished" data-v-onboarding-wrapper>
    <slot :key="index" :step="activeStep" :next="next" :previous="previous" :exit="exit" :is-first="isFirstStep" :is-last="isLastStep" :index="index">
      <VOnboardingStep :key="index" />
    </slot>
  </div>
</template>
<script lang="ts">
import VOnboardingStep from '@/components/VOnboardingStep.vue';
import useGetElement from '@/composables/useGetElement';
import { OnboardingState, STATE_INJECT_KEY } from '@/types/index';
import type { StepEntity } from '@/types/StepEntity';
import { defaultVOnboardingWrapperOptions, VOnboardingWrapperOptions } from '@/types/VOnboardingWrapper';
import merge from 'lodash.merge';
import { computed, defineComponent, PropType, provide, ref, watch } from 'vue';
export default defineComponent({
  name: 'VOnboardingWrapper',
  components: {
    VOnboardingStep
  },
  props: {
    steps: {
      type: Array as PropType<StepEntity[]>,
      default: () => []
    },
    options: {
      type: Object as PropType<VOnboardingWrapperOptions>,
      default: () => ({})
    }
  },
  emits: ['finish', 'exit'],
  setup(props, { expose, emit }) {
    const index = ref(OnboardingState.IDLE)
    const privateIndex = ref(index.value)
    const setIndex = (value: number | ((_: number) => number)) => {
      if (typeof value === 'function') {
        index.value = value(index.value);
      } else {
        index.value = value;
      }
    }
    const { beforeHook, afterHook } = useStepHooks()
    const activeStep = computed(() => props.steps?.[privateIndex.value])
    watch(index, async (newIndex, oldIndex) => {
      const oldStep = props.steps?.[oldIndex]
      if (oldStep) {
        await afterHook(oldStep)
      }
      const newStep = props.steps?.[newIndex]
      if (newStep) {
        await beforeHook(newStep)
      }
      privateIndex.value = newIndex
    })
    const isFinished = computed(() => {
      return privateIndex.value === OnboardingState.FINISHED
    })
    const start = () => setIndex(0)
    const finish = () => {
      setIndex(OnboardingState.FINISHED)
      emit('finish')
    }
    const exit = () => emit('exit')
    expose({
      start,
      finish,
      goToStep: setIndex
    })
    const previous = () => {
      setIndex(current => current - 1)
    }
    const next = () => {
      const next = privateIndex.value + 1
      if (next === props.steps.length) {
        finish()
        return
      }
      setIndex(next)
    }
    const state = ref<OnboardingState>({
      step: activeStep,
      options: computed(() => merge({}, defaultVOnboardingWrapperOptions, props.options)),
      next,
      previous,
      finish,
      exit,
      isFirstStep: computed(() => privateIndex.value === 0),
      isLastStep: computed(() => privateIndex.value === props.steps.length - 1)
    } as OnboardingState)
    provide(STATE_INJECT_KEY, state)

    const isFirstStep = computed(() => privateIndex.value === 0)
    const isLastStep = computed(() => privateIndex.value === props.steps.length - 1)
    return {
      index,
      activeStep,
      next,
      previous,
      isFinished,
      setIndex,
      isFirstStep,
      isLastStep,
      finish,
      exit
    }
  }
})
function useSetElementClassName() {
  const setClassName = ({ element, classList = [] }: { element: Element | null; classList?: string[] }) => {
    if (!element) return;
    element.classList.add(...classList)
  }
  const unsetClassName = ({ element, classList = [] }: { element: Element | null; classList?: string[] }) => {
    if (!element) return;
    element.classList.remove(...classList)
  }
  return { setClassName, unsetClassName }
}
function useStepHooks() {
  const { setClassName, unsetClassName } = useSetElementClassName()
  const beforeHook = (step: StepEntity) => {
    unsetClassName({ element: useGetElement(step.attachTo.element), classList: step.attachTo.classList });
    return step.on?.beforeStep?.()
  }

  const afterHook = (step: StepEntity) => {
    setClassName({ element: useGetElement(step.attachTo.element), classList: step.attachTo.classList });
    return step.on?.afterStep?.()
  }

  return { beforeHook, afterHook }
}
</script>
