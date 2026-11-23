<script setup lang="ts">
import { computed, ref, watchEffect } from 'vue'
import { useRoute, useRouter } from 'vue-router'
import { useChallengeStore } from '../stores/challenge'
import AppIcon from '../components/ui/AppIcon.vue'

const store = useChallengeStore()
const route = useRoute()
const router = useRouter()
const expandedTeamId = ref<string | null>(null)
const param = (value: string | string[] | undefined) => typeof value === 'string' ? value : ''
const activeScenario = computed(
  () => store.scenarioById(param(route.params.scenarioId)) ?? store.scenarios.find(
    (scenario) => scenario.tasks.some((task) => task.status === 'active')
  )
)
const activeTask = computed(
  () => activeScenario.value?.tasks.find(
    (task) => task.id === param(route.params.taskId) && task.status === 'active'
  ) ?? activeScenario.value?.tasks.find((task) => task.status === 'active')
)
const teamRows = computed(
  () => activeScenario.value && activeTask.value ? store.attemptsByTask(activeScenario.value.id, activeTask.value.id) : []
)
const selectedTeam = computed(() => teamRows.value.find((team) => team.teamId === expandedTeamId.value))

watchEffect(() => {
  if (activeScenario.value && activeTask.value && (!route.params.scenarioId || !route.params.taskId))
    router.replace(`/attempts/${activeScenario.value.id}/${activeTask.value.id}`)
})

function selectScenario(id: string) {
  const scenario = store.scenarioById(id)
  const task = scenario?.tasks.find((item) => item.status === 'active')
  if (task) router.push(`/attempts/${id}/${task.id}`)
}
function toggleTeam(teamId: string) {
  expandedTeamId.value = expandedTeamId.value === teamId ? null : teamId
}
function formatDate(value?: string) {
  return value ? new Date(value).toLocaleString(
    'ko-KR', { month: '2-digit', day: '2-digit', hour: '2-digit', minute: '2-digit' }
  ) : '없음'
}
function formatScore(value: number | null) {
  return value === null ? '—' : value.toFixed(value >= 10 ? 1 : 3)
}
</script>

<template>
  <div class="page-stack flex flex-col gap-6">
    <div class="page-heading">
      <div>
        <span class="eyebrow">ATTEMPT DETAILS</span>
        <h1><strong>응시 상세</strong></h1>
        <p class="font-medium">팀별 응시 현황과 회차별 측정 결과를 확인하세요.</p>
      </div>
      <button class="data-page-action refresh-action" :disabled="store.loading" @click="store.load">
        새로고침
      </button>
    </div>
    <p v-if="store.loading" class="panel" role="status">응시 데이터를 불러오는 중입니다…</p>
    <section v-else-if="store.error" class="panel">
      <h2>데이터를 표시할 수 없습니다.</h2>
      <p class="form-error" role="alert">
        {{ store.error }}
      </p>
      <button class="primary-button compact" @click="store.load">
        다시 시도
      </button>
    </section>
    <template v-else>
      <div v-if="store.warnings.length" class="panel" role="status">
        <b>일부 데이터가 제외되었습니다.</b>
        <ul>
          <li v-for="warning in store.warnings" :key="warning">
            {{ warning }}
          </li>
        </ul>
      </div>
      <section v-if="store.scenarios.length" class="leader-tabs" aria-label="응시 상세 분류">
        <div class="scenario-tabs" role="tablist" aria-label="시나리오 선택">
          <button v-for="scenario in store.scenarios" 
            :key="scenario.id" type="button" role="tab" 
            :aria-selected="scenario.id === activeScenario?.id" 
            :disabled="!scenario.tasks.some((task) => task.status === 'active')" 
            :class="{ 
              active: scenario.id === activeScenario?.id, 
              disabled: !scenario.tasks.some((task) => task.status === 'active') 
            }" 
            @click="selectScenario(scenario.id)"
          >
            {{ scenario.name }}
            <small v-if="!scenario.tasks.some((task) => task.status === 'active')">준비 중</small>
          </button>
        </div>
        <div v-if="activeScenario" class="task-segment-section" role="tabpanel">
          <div class="task-segments" aria-label="세부 태스크 선택">
          <RouterLink v-for="task in activeScenario.tasks" 
            :key="task.id" 
            :to="task.status === 'active' ? `/attempts/${activeScenario.id}/${task.id}` : route.fullPath" 
            :aria-current="task.id === activeTask?.id ? 'page' 
            : undefined" :aria-disabled="task.status !== 'active'" 
            :class="{ disabled: task.status !== 'active' }"
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
              응시 상세 
              <AppIcon name="chevron" :size="13" /> 
              {{ activeScenario.name }} 
              <AppIcon name="chevron" :size="13" /> 
              {{ activeTask.name }}
            </div>
            <h2><strong>{{ activeScenario.name }} {{ activeTask.name }}</strong></h2>
            <p>팀 행을 선택하면 해당 팀의 전체 회차별 측정 결과가 펼쳐집니다.</p>
          </div>
          <div class="board-meta">
            <span>
              <i class="live-dot"></i>
              {{ teamRows.length }} teams
            </span>
          </div>
        </div>
        <div class="table-shell">
          <div class="table-scroll">
            <table class="leader-table attempts-summary-table">
              <thead>
                <tr>
                  <th class="cell-team">팀</th>
                  <th>대표 회차</th>
                  <th>완주</th>
                  <th>제출 시각</th>
                  <th>최종 점수</th>
                  <th>측정 상태</th>
                  <th>전체 회차</th>
                </tr>
              </thead>
              <tbody>
                <template v-for="team in teamRows" :key="team.teamId">
                  <tr class="attempt-team-row" 
                    :class="{ expanded: expandedTeamId === team.teamId }" tabindex="0" 
                    :aria-expanded="expandedTeamId === team.teamId" 
                    @click="toggleTeam(team.teamId)" 
                    @keydown.enter="toggleTeam(team.teamId)"
                  >
                    <td class="team-cell">
                      <span class="team-logo">{{ team.teamName.charAt(0) }}</span>
                      <b>{{ team.teamName }}</b>
                    </td>
                    <td>{{ team.attempts[0]?.round }}회차</td>
                    <td>{{ team.attempts[0]?.completed ? '예' : '아니요' }}</td>
                    <td>{{ formatDate(team.attempts[0]?.submittedAt) }}</td>
                    <td class="cell-score">{{ formatScore(team.attempts[0]?.finalScore ?? null) }}</td>
                    <td>
                      <span class="status-pill" :class="String(team.attempts[0]?.measurementStatus ?? '').replaceAll(' ', '-')">
                        {{ team.attempts[0]?.measurementStatus }}
                      </span>
                    </td>
                    <td>{{ team.attempts.length }}회 
                      <span class="attempt-chevron">{{ expandedTeamId === team.teamId ? '▲' : '▼' }}</span>
                    </td>
                  </tr>
                </template>
                <tr v-if="teamRows.length === 0">
                  <td colspan="7">등록된 응시 결과가 없습니다.</td>
                </tr>
              </tbody>
            </table>
          </div>
          <div class="table-note">
            <span>i</span>
            상단에는 팀별 1회차만 표시됩니다. 팀 행을 클릭하면 전체 회차를 확인할 수 있습니다.
          </div>
        </div>
      </section>

      <section v-if="selectedTeam && activeScenario && activeTask" class="board-section attempt-detail-section">
        <div class="board-heading">
          <div>
            <div class="breadcrumb">회차별 측정 결과 
              <AppIcon name="chevron" :size="13" /> 
              {{ selectedTeam.teamName }}
            </div>
            <h2><strong>{{ selectedTeam.teamName }}</strong></h2>
            <p>{{ activeScenario.name }} · {{ activeTask.name }} · 총 {{ selectedTeam.attempts.length }}회</p>
          </div>
          <button class="detail-close-button" type="button" @click="expandedTeamId = null">
            상세 닫기
          </button>
        </div>
        <div class="table-scroll">
            <table class="leader-table attempt-detail-table">
            <thead>
              <tr>
                <th>회차</th>
                <th>완주</th>
                <th>제출 시각</th>
                <th>탐사율</th>
                <th>일치율</th>
                <th>접촉</th>
                <th>과제 점수</th>
                <th>통신 절약</th>
                <th>최종 점수</th>
                <th>측정 상태</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="attempt in selectedTeam.attempts" :key="attempt.attemptId">
                <td>{{ attempt.round }}</td>
                <td>{{ attempt.completed ? '예' : '아니요' }}</td>
                <td>{{ formatDate(attempt.submittedAt) }}</td>
                <td>{{ attempt.explorationRate === null ? '—' : `${attempt.explorationRate}%` }}</td>
                <td>{{ attempt.consistencyRate === null ? '—' : `${attempt.consistencyRate}%` }}</td>
                <td>{{ attempt.contacts ?? '—' }}</td><td>{{ formatScore(attempt.taskScore) }}</td>
                <td>{{ attempt.communicationSavings === null ? '—' : `${attempt.communicationSavings}%` }}</td>
                <td class="cell-score">{{ formatScore(attempt.finalScore) }}</td>
                <td>
                  <span class="status-pill" :class="attempt.measurementStatus.replaceAll(' ', '-')">
                    {{ attempt.measurementStatus }}
                  </span>
                </td>
              </tr>
            </tbody>
          </table>
        </div>
      </section>
    </template>
  </div>
</template>

<style scoped>
  .attempt-team-row { 
    cursor: pointer; 
  }
  .attempt-team-row:focus-visible { 
    outline: 2px solid var(--blue); 
    outline-offset: -2px; 
  }
  .attempt-team-row.expanded { 
    background: #f1f6ff; 
  }
  .attempt-chevron { 
    margin-left: 8px; 
    color: var(--blue); 
    font-size: 9px; 
  }
  .attempt-detail-section { 
    animation: detail-in 160ms ease-out; 
  }
  .attempt-detail-table th, 
  .attempt-detail-table td { 
    padding-block: 14px; 
  }
  .detail-close-button { 
    border: 1px solid var(--line); 
    border-radius: 8px; 
    background: #fff; 
    padding: 8px 12px; 
    color: var(--muted); 
    font-size: 11px; 
    font-weight: 700; 
    cursor: pointer; 
  }
  .detail-close-button:hover { 
    border-color: var(--blue); 
    color: var(--blue); 
  }
  @keyframes detail-in { 
    from { opacity: 0; transform: translateY(-5px); } to { opacity: 1; transform: translateY(0); } 
  }
</style>
