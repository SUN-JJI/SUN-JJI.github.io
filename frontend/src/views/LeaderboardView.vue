<script setup lang="ts">
import { computed, watchEffect } from 'vue'
import { useRoute, useRouter } from 'vue-router'
import { useChallengeStore } from '../stores/challenge'
import AppIcon from '../components/ui/AppIcon.vue'
import LeaderboardTable from '../components/leaderboard/LeaderboardTable.vue'

const store = useChallengeStore()
const route = useRoute()
const router = useRouter()
const param = (value: string | string[] | undefined) => typeof value === 'string' ? value : ''
const activeScenario = computed(() => store.scenarioById(param(route.params.scenarioId)) ?? store.scenarios.find((scenario) => scenario.tasks.some((task) => task.status === 'active')))
const activeTask = computed(() => activeScenario.value?.tasks.find((task) => task.id === param(route.params.taskId) && task.status === 'active') ?? activeScenario.value?.tasks.find((task) => task.status === 'active'))
const rows = computed(() => activeScenario.value && activeTask.value ? store.ranked(activeScenario.value.id, activeTask.value.id) : [])
const updated = computed(() => store.updatedAt ? new Date(store.updatedAt).toLocaleString('ko-KR') : '미등록')

watchEffect(() => {
  if (activeScenario.value && activeTask.value && (!route.params.scenarioId || !route.params.taskId))
    router.replace(`/leaderboard/${activeScenario.value.id}/${activeTask.value.id}`)
})

function selectScenario(id: string) {
  const scenario = store.scenarioById(id)
  const task = scenario?.tasks.find((item) => item.status === 'active')
  if (task) router.push(`/leaderboard/${id}/${task.id}`)
}
</script>

<template>
  <div class="page-stack flex flex-col gap-6">
    <div class="page-heading">
      <div>
        <span class="eyebrow">RANKINGS</span>
        <h1><strong>Leaderboard</strong></h1>
        <p class="font-medium">팀별 최고 점수와 공식 순위를 확인하세요.</p>
      </div>
      <button class="data-page-action refresh-action" :disabled="store.loading" @click="store.load">
        새로고침
      </button>
    </div>
    <p v-if="store.loading" class="panel" role="status">리더보드 데이터를 불러오는 중입니다…</p>
    <section v-else-if="store.error" class="panel">
      <h2>데이터를 표시할 수 없습니다.</h2>
      <p class="form-error" role="alert">{{ store.error }}</p>
      <button class="primary-button compact" @click="store.load">다시 시도</button>
    </section>
    <template v-else>
      <div v-if="store.warnings.length" class="panel" role="status">
        <b>일부 데이터가 제외되었습니다.</b>
        <ul>
          <li v-for="warning in store.warnings" :key="warning">{{ warning }}</li>
        </ul>
      </div>
      <section v-if="store.scenarios.length" class="leader-tabs" aria-label="리더보드 분류">
        <div class="scenario-tabs" role="tablist" aria-label="시나리오 선택">
          <button v-for="scenario in store.scenarios" :key="scenario.id" type="button" role="tab" 
            :aria-selected="scenario.id === activeScenario?.id" :aria-controls="`scenario-panel-${scenario.id}`" 
            :disabled="!scenario.tasks.some((task) => task.status === 'active')" 
            :class="{ active: scenario.id === activeScenario?.id, disabled: !scenario.tasks.some((task) => task.status === 'active') }" 
            @click="selectScenario(scenario.id)"
          >
            {{ scenario.name }}
            <small v-if="!scenario.tasks.some((task) => task.status === 'active')">준비 중</small>
          </button>
        </div>
        <div v-if="activeScenario" :id="`scenario-panel-${activeScenario.id}`" class="task-segment-section" role="tabpanel">
          <div class="task-segments" aria-label="세부 태스크 선택">
            <RouterLink v-for="task in activeScenario.tasks" :key="task.id" 
              :to="task.status === 'active' ? `/leaderboard/${activeScenario.id}/${task.id}` : route.fullPath" 
              :aria-current="task.id === activeTask?.id ? 'page' : undefined" 
              :aria-disabled="task.status !== 'active'" :class="{ disabled: task.status !== 'active' }"
            >
              {{ task.name }}
              <small v-if="task.status !== 'active'">준비 중</small>
            </RouterLink>
          </div>
        </div>
      </section>
      <section v-if="activeScenario && activeTask" class="board-section">
        <div class="board-heading">
          <div>
            <div class="breadcrumb">
              Leaderboard 
              <AppIcon name="chevron" :size="13" /> 
              {{ activeScenario.name }} 
              <AppIcon name="chevron" :size="13" /> 
              {{ activeTask.name }}
            </div>
            <h2><strong>{{ activeTask.leaderboard.title }}</strong></h2>
            <p>{{ activeTask.leaderboard.description }}</p>
          </div>
          <div class="board-meta">
            <span><i class="live-dot"></i>{{ rows.length }} teams</span>
            <span>업데이트 {{ updated }}</span>
          </div>
        </div>
        <LeaderboardTable :config="activeTask.leaderboard" :rows="rows" />
      </section>
      <section v-else class="panel">
        <h2>등록된 과제가 없습니다.</h2>
        <p><code>public/data/tasks.json</code>에 시나리오와 태스크를 추가해 주세요.</p>
      </section>
    </template>
  </div>
</template>
