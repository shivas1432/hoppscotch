<template>
  <div>
    <div
      v-if="
        testResults &&
        (testResults.expectResults.length ||
          testResults.tests.length ||
          haveEnvVariables)
      "
    >
      <div class="divide-y-4 divide-dividerLight border-b border-dividerLight">
        <div v-if="haveEnvVariables" class="flex flex-col">
          <details class="flex flex-col divide-y divide-dividerLight" open>
            <summary
              class="group flex min-w-0 flex-1 cursor-pointer items-center justify-between text-tiny text-secondaryLight transition focus:outline-none"
            >
              <span
                class="inline-flex items-center justify-center truncate px-4 py-2 transition group-hover:text-secondary"
              >
                <icon-lucide-chevron-right
                  class="indicator mr-2 flex flex-shrink-0"
                />
                <span class="capitalize-first truncate">
                  {{ t("environment.title") }}
                </span>
              </span>
            </summary>
            <div class="divide-y divide-dividerLight">
              <div
                v-if="noEnvSelected && !globalHasAdditions"
                class="flex bg-bannerInfo p-4 text-secondaryDark"
                role="alert"
              >
                <icon-lucide-alert-triangle class="svg-icons mr-4" />
                <div class="flex flex-col">
                  <p>
                    {{ t("environment.no_environment_description") }}
                  </p>
                  <p class="mt-3 flex space-x-2">
                    <HoppButtonSecondary
                      :label="t('environment.add_to_global')"
                      class="!bg-primary text-tiny"
                      filled
                      @click="addEnvToGlobal()"
                    />
                    <HoppButtonSecondary
                      :label="t('environment.create_new')"
                      class="!bg-primary text-tiny"
                      filled
                      @click="displayModalAdd(true)"
                    />
                  </p>
                </div>
              </div>
              <HttpTestResultEnv
                v-for="(env, index) in testResults.envDiff.global.additions"
                :key="`env-${env.key}-${index}`"
                :env="env"
                status="additions"
                global
              />
              <HttpTestResultEnv
                v-for="(env, index) in testResults.envDiff.global.updations"
                :key="`env-${env.key}-${index}`"
                :env="env"
                status="updations"
                global
              />
              <HttpTestResultEnv
                v-for="(env, index) in testResults.envDiff.global.deletions"
                :key="`env-${env.key}-${index}`"
                :env="env"
                status="deletions"
                global
              />
              <HttpTestResultEnv
                v-for="(env, index) in testResults.envDiff.selected.additions"
                :key="`env-${env.key}-${index}`"
                :env="env"
                status="additions"
              />
              <HttpTestResultEnv
                v-for="(env, index) in testResults.envDiff.selected.updations"
                :key="`env-${env.key}-${index}`"
                :env="env"
                status="updations"
              />
              <HttpTestResultEnv
                v-for="(env, index) in testResults.envDiff.selected.deletions"
                :key="`env-${env.key}-${index}`"
                :env="env"
                status="deletions"
              />
            </div>
          </details>
        </div>
        <div
          v-if="testResults.tests"
          class="divide-y-4 divide-dividerLight test-results-entry"
        >
          <template
            v-for="(result, index) in testResults.tests"
            :key="`result-${index}`"
          >
            <HttpTestResultEntry
              v-if="shouldShowEntry(result)"
              :test-results="result"
              :show-test-type="props.showTestType"
            />
          </template>
        </div>
        <div
          v-if="testResults.expectResults"
          class="divide-y divide-dividerLight"
        >
          <HttpTestResultReport
            v-if="testResults.expectResults.length"
            :test-results="testResults"
          />
          <div
            v-for="(result, index) in testResults.expectResults"
            :key="`result-${index}`"
            class="flex items-center px-4 py-2"
          >
            <div class="flex flex-shrink-0 items-center overflow-x-auto">
              <component
                :is="result.status === 'pass' ? IconCheck : IconClose"
                class="svg-icons mr-4"
                :class="
                  result.status === 'pass' ? 'text-green-500' : 'text-red-500'
                "
              />
              <div
                class="flex flex-shrink-0 items-center space-x-2 overflow-x-auto"
              >
                <span
                  v-if="result.message"
                  class="inline-flex text-secondaryDark"
                >
                  {{ result.message }}
                </span>
                <span class="inline-flex text-secondaryLight">
                  <icon-lucide-minus class="svg-icons mr-2" />
                  {{
                    result.status === "pass"
                      ? t("test.passed")
                      : t("test.failed")
                  }}
                </span>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>
    <HoppSmartPlaceholder
      v-else-if="testResults && testResults.scriptError"
      :src="`/images/states/${colorMode.value}/youre_lost.svg`"
      :alt="`${t('error.post_request_script_fail')}`"
      :heading="t('error.post_request_script_fail')"
      :text="t('helpers.post_request_script_fail')"
    >
    </HoppSmartPlaceholder>
    <template v-else>
      <div
        class="py-2 pl-4 ml-4 mb-2 text-secondaryLight border-secondaryLight border-l"
      >
        {{ t("empty.tests") }}
      </div>
    </template>
    <EnvironmentsMyDetails
      :show="showMyEnvironmentDetailsModal"
      action="new"
      :env-vars="getAdditionVars"
      @hide-modal="displayModalAdd(false)"
    />
    <EnvironmentsTeamsDetails
      :show="showTeamEnvironmentDetailsModal"
      action="new"
      :env-vars="getAdditionVars"
      :editing-team-id="
        workspace.type === 'team' ? workspace.teamID : undefined
      "
      @hide-modal="displayModalAdd(false)"
    />
  </div>
</template>

<script setup lang="ts">
import { useI18n } from "@composables/i18n"
import { useReadonlyStream, useStream } from "@composables/stream"
import { isEqual } from "lodash-es"
import { computed, ref } from "vue"
import { HoppTestData, HoppTestResult } from "~/helpers/types/HoppTestResult"
import {
  globalEnv$,
  selectedEnvironmentIndex$,
  setSelectedEnvironmentIndex,
} from "~/newstore/environments"

import IconCheck from "~icons/lucide/check"
import IconClose from "~icons/lucide/x"

import { GlobalEnvironment } from "@hoppscotch/data"
import { useVModel } from "@vueuse/core"
import { useService } from "dioc/vue"
import { useColorMode } from "~/composables/theming"
import { invokeAction } from "~/helpers/actions"
import { WorkspaceService } from "~/services/workspace.service"

const props = withDefaults(
  defineProps<{
    modelValue: HoppTestResult | null | undefined
    showTestType: "all" | "passed" | "failed"
    showEmptyMessage?: boolean
  }>(),
  {
    showEmptyMessage: true,
  }
)

const emit = defineEmits<{
  (e: "update:modelValue", val: HoppTestResult | null | undefined): void
}>()

const testResults = useVModel(props, "modelValue", emit)

const shouldShowEntry = (result: HoppTestData) => {
  if (props.showTestType === "all") return true
  if (props.showTestType === "passed")
    return result.expectResults.some((x) => x.status === "pass")
  if (props.showTestType === "failed")
    return result.expectResults.some((x) => x.status === "fail")
  return false
}

const t = useI18n()
const colorMode = useColorMode()

const workspaceService = useService(WorkspaceService)
const workspace = workspaceService.currentWorkspace

const showMyEnvironmentDetailsModal = ref(false)
const showTeamEnvironmentDetailsModal = ref(false)

const displayModalAdd = (shouldDisplay: boolean) => {
  if (workspace.value.type === "personal")
    showMyEnvironmentDetailsModal.value = shouldDisplay
  else showTeamEnvironmentDetailsModal.value = shouldDisplay
}

/**
 * Get the "addition" environment variables
 * @returns Array of objects with key-value pairs of arguments
 */
const getAdditionVars = () =>
  testResults?.value?.envDiff?.selected?.additions
    ? testResults.value.envDiff.selected.additions
    : []

const haveEnvVariables = computed(() => {
  if (!testResults.value) return false
  return (
    testResults.value.envDiff.global.additions.length ||
    testResults.value.envDiff.global.updations.length ||
    testResults.value.envDiff.global.deletions.length ||
    testResults.value.envDiff.selected.additions.length ||
    testResults.value.envDiff.selected.updations.length ||
    testResults.value.envDiff.selected.deletions.length
  )
})

const selectedEnvironmentIndex = useStream(
  selectedEnvironmentIndex$,
  { type: "NO_ENV_SELECTED" },
  setSelectedEnvironmentIndex
)

const globalEnvVars = useReadonlyStream(globalEnv$, {} as GlobalEnvironment)

const noEnvSelected = computed(
  () => selectedEnvironmentIndex.value.type === "NO_ENV_SELECTED"
)

const globalHasAdditions = computed(() => {
  if (!testResults.value?.envDiff.selected.additions) return false
  return (
    testResults.value.envDiff.selected.additions.every(
      (x) =>
        globalEnvVars.value.variables.findIndex((y) => isEqual(x, y)) !== -1
    ) ?? false
  )
})

const addEnvToGlobal = () => {
  if (!testResults.value?.envDiff.selected.additions) return

  invokeAction("modals.global.environment.update", {
    variables: testResults.value.envDiff.selected.additions,
    isSecret: false,
  })
}
</script>
