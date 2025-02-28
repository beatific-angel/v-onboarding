<template>
  <div v-show="show">
    <svg style="width: 100%; height: 100%; position: fixed; top: 0; left: 0; opacity: 0.5; z-index: var(--v-onboarding-overlay-z, 10); pointer-events: none;">
      <path :d="path" />
    </svg>
    <div ref="stepElement" style="position: relative; z-index: var(--v-onboarding-step-z, 20);">
      <slot v-if="step">
        <div class="v-onboarding-item">
          <div class="v-onboarding-item__header">
            <span
              v-if="step.content.title"
              class="v-onboarding-item__header-title"
            >{{ step.content.title }}</span>
            <button @click="exit" class="v-onboarding-item__header-close">
              <svg
                xmlns="http://www.w3.org/2000/svg"
                class="h-4 w-4"
                fill="none"
                viewBox="0 0 24 24"
                stroke="currentColor"
              >
                <path
                  stroke-linecap="round"
                  stroke-linejoin="round"
                  stroke-width="2"
                  d="M6 18L18 6M6 6l12 12"
                />
              </svg>
            </button>
          </div>
          <p
            v-if="step.content.description"
            class="v-onboarding-item__description"
          >{{ step.content.description }}</p>
          <div class="v-onboarding-item__actions">
            <button
              v-if="!isFirstStep && isButtonVisible.previous"
              type="button"
              @click="previous"
              class="v-onboarding-btn-secondary"
            >{{ buttonLabels.previous }}</button>
            <button v-if="isButtonVisible.next"
              @click="() => isLastStep ? finish() : next()"
              type="button"
              class="v-onboarding-btn-primary"
            >{{ isLastStep ? buttonLabels.finish : buttonLabels.next }}</button>
          </div>
        </div>
      </slot>
      <div data-popper-arrow />
    </div>
  </div>
</template>
<script lang="ts">
import { OnboardingState, STATE_INJECT_KEY } from '@/types/index';
import { createPopper } from '@popperjs/core';
import merge from 'lodash.merge';
import { computed, defineComponent, inject, onMounted, Ref, ref, toRefs } from 'vue';
import useGetElement from '../composables/useGetElement';
import useSvgOverlay from '../composables/useSvgOverlay';
export default defineComponent({
  name: "VOnboardingStep",
  setup() {
    const show = ref(false)

    const state = inject(STATE_INJECT_KEY, {} as Ref<OnboardingState>)
    const { step, isFirstStep, isLastStep, options, next, previous, exit, finish } = toRefs(state.value)

    const mergedOptions = computed(() => merge({}, options?.value, step.value.options))

    const isButtonVisible = computed(() => {
      return {
        previous: !mergedOptions.value.hideButtons?.previous,
        next: !mergedOptions.value.hideButtons?.next
      }
    })

    const buttonLabels = computed(() => {
      return {
        previous: mergedOptions.value?.labels?.previousButton,
        next: mergedOptions.value?.labels?.nextButton,
        finish: mergedOptions.value?.labels?.finishButton,
      }
    })

    const { updatePath, path } = useSvgOverlay();

    const stepElement = ref<HTMLElement>();
    const attachElement = () => {
      const element = useGetElement(step?.value?.attachTo?.element);
      if (element && stepElement.value) {
        show.value = true
        if (mergedOptions.value?.scrollToStep?.enabled) {
          element.scrollIntoView(mergedOptions.value?.scrollToStep?.options)
        }
        createPopper(element, stepElement.value, mergedOptions.value.popper);
        if (mergedOptions.value?.overlay?.enabled) {
          updatePath(element, {
            padding: mergedOptions.value?.overlay?.padding,
            borderRadius: mergedOptions.value?.overlay?.borderRadius,
          });
        }
      }
    };

    onMounted(attachElement)
    return {
      stepElement,
      next,
      previous,
      path,
      show,
      step,
      isFirstStep,
      isLastStep,
      exit,
      finish,
      isButtonVisible,
      buttonLabels
    };
  },
})
</script>
