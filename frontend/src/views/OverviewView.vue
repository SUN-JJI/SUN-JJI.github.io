<script setup lang="ts">
import { useChallengeStore } from '../stores/challenge'
import AppIcon from '../components/ui/AppIcon.vue'
import MetricCard from '../components/overview/MetricCard.vue'
const store = useChallengeStore()
const scenarioIcon = (index: number) => index === 0 ? 'grid' as const : index === 1 ? 'spark' as const : 'database' as const
</script>
<template>
  <div class="page-stack flex flex-col gap-6">
    <section class="hero-panel">
      <div>
        <span class="eyebrow">PUBLIC HACKATHON</span>
        <h1><strong>Challenge Overview</strong></h1>
        <p class="font-medium">정적 JSON으로 운영되는 공개 해커톤 리더보드입니다.</p>
      </div>
    </section>
    <p v-if="store.loading" class="panel">데이터를 불러오는 중입니다…</p>
    <p v-else-if="store.error" class="panel form-error" role="alert">
      {{ store.error }} 
      <button class="table-action" @click="store.load">다시 시도</button>
    </p>
    <section class="metrics-grid grid gap-4 md:grid-cols-2 xl:grid-cols-4">
      <MetricCard label="참가 팀" :value="store.teams.length" hint="teams.json" tone="blue">
        <AppIcon name="users" />
      </MetricCard>
      <MetricCard label="완료 결과" :value="store.completedCount" hint="COMPLETED" tone="orange">
        <AppIcon name="trophy" />
      </MetricCard>
      <MetricCard label="시나리오" :value="store.scenarios.length" hint="tasks.json" tone="violet">
        <AppIcon name="grid" />
      </MetricCard>
      <MetricCard label="전체 태스크" :value="store.scenarios.flatMap((item) => item.tasks).length" hint="공개 과제" tone="green">
        <AppIcon name="database" />
      </MetricCard>
    </section>
    <section class="section-block">
      <div class="section-heading">
        <div>
          <span class="eyebrow">CHALLENGES</span>
          <h2><strong>시나리오별 태스크</strong></h2>
        </div>
        <RouterLink class="text-link" to="/leaderboard">
          전체 리더보드 
          <AppIcon name="arrow" />
        </RouterLink>
      </div>
      <div v-if="store.scenarios.length" class="scenario-grid grid gap-5 md:grid-cols-2 xl:grid-cols-3">
        <article v-for="(scenario, index) in store.scenarios" :key="scenario.id" class="scenario-card" :class="scenario.tone">
          <div class="scenario-number">
            {{ String(index + 1).padStart(2, '0') }}
          </div>
          <span class="scenario-icon">
            <AppIcon :name="scenarioIcon(index)" :size="24" />
          </span>
          <h3><strong>{{ scenario.name }}</strong></h3>
          <p class="font-medium">{{ scenario.description }}</p>
          <div class="task-list">
            <RouterLink v-for="task in scenario.tasks" :key="task.id" 
              :to="task.status === 'active' ? `/leaderboard/${scenario.id}/${task.id}` : ''" 
              :class="{ disabled: task.status !== 'active' }" :aria-disabled="task.status !== 'active'"
            >
              <span class="font-semibold">{{ task.name }}</span>
              <small v-if="task.status !== 'active'">준비 중</small>
              <AppIcon v-else name="chevron" />
            </RouterLink>
          </div>
        </article>
      </div>
      <div v-else class="panel">등록된 시나리오가 없습니다.</div>
    </section>
  </div>
</template>
