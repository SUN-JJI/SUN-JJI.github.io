<script setup lang="ts">
import { computed, ref } from 'vue'
import { useChallengeStore } from '../stores/challenge'
import AppIcon from '../components/ui/AppIcon.vue'

const store = useChallengeStore()
const query = ref('')
const checkingAsset = ref('')
const unavailableDownload = ref<{
  scenario: string
  task: string
  asset: string
  checkFailed: boolean
} | null>(null)
const releaseRepository = import.meta.env.VITE_GITHUB_RELEASE_REPOSITORY
const releaseTag = import.meta.env.VITE_GITHUB_RELEASE_TAG
const useLatestRelease = releaseTag === 'latest'
const releasePageUrl = releaseRepository && releaseTag
  ? useLatestRelease
    ? `https://github.com/${releaseRepository}/releases/latest`
    : `https://github.com/${releaseRepository}/releases/tag/${encodeURIComponent(releaseTag)}`
  : ''
const releaseDownloadUrl = (asset: string) => releasePageUrl && asset
  ? useLatestRelease
    ? `https://github.com/${releaseRepository}/releases/latest/download/${encodeURIComponent(asset)}`
    : `https://github.com/${releaseRepository}/releases/download/${encodeURIComponent(releaseTag)}/${encodeURIComponent(asset)}`
  : ''

let releaseAssetsRequest: Promise<string[]> | null = null
const releaseApiUrl = releaseRepository && releaseTag
  ? useLatestRelease
    ? `https://api.github.com/repos/${releaseRepository}/releases/latest`
    : `https://api.github.com/repos/${releaseRepository}/releases/tags/${encodeURIComponent(releaseTag)}`
  : ''

const downloadAsset = async (scenarioName: string, taskName: string, asset: string) => {
  if (!releaseApiUrl || checkingAsset.value) return
  checkingAsset.value = asset
  try {
    releaseAssetsRequest ??= fetch(releaseApiUrl, {
      headers: { Accept: 'application/vnd.github+json' },
    }).then(async (response) => {
      if (!response.ok) throw new Error(`GitHub API 요청 실패 (${response.status})`)
      const release = await response.json() as { assets?: { name?: string }[] }
      return release.assets?.flatMap((item) => typeof item.name === 'string' ? [item.name] : []) ?? []
    })
    const assets = await releaseAssetsRequest
    if (assets.includes(asset)) {
      window.location.assign(releaseDownloadUrl(asset))
      return
    }
    unavailableDownload.value = {
      scenario: scenarioName,
      task: taskName,
      asset,
      checkFailed: false,
    }
  } catch {
    releaseAssetsRequest = null
    unavailableDownload.value = {
      scenario: scenarioName,
      task: taskName,
      asset,
      checkFailed: true,
    }
  } finally {
    checkingAsset.value = ''
  }
}

const normalizedQuery = computed(() => query.value.trim().toLocaleLowerCase('ko'))
const scenarios = computed(() => store.scenarios.map((scenario) => ({
  ...scenario,
  tasks: scenario.tasks.filter((task) =>
    `${scenario.name} ${task.name}`.toLocaleLowerCase('ko').includes(normalizedQuery.value),
  ),
})).filter((scenario) => scenario.tasks.length > 0))
</script>

<template>
  <div class="page-stack flex flex-col gap-6">
    <div class="page-heading">
      <div>
        <span class="eyebrow">PUBLIC DATA</span>
        <h1><strong>Data</strong></h1>
        <p class="font-medium">현재 공개된 시나리오와 태스크 정보를 확인하세요.</p>
      </div>
      <div class="data-page-actions">
        <a
          v-if="releasePageUrl"
          :href="releasePageUrl"
          target="_blank"
          rel="noopener noreferrer"
          class="data-page-action release-shortcut"
        >
          <AppIcon name="file" :size="17" />
          릴리즈 바로가기
        </a>
        <button
          class="data-page-action refresh-action"
          type="button"
          :disabled="store.loading"
          @click="store.load"
        >
          새로고침
        </button>
      </div>
    </div>
    <div class="toolbar">
      <label class="search-box">
        <AppIcon name="search" />
        <input v-model="query" type="search" placeholder="시나리오 또는 태스크 검색" />
      </label>
      <span>시나리오 {{ store.scenarios.length }}개 · 태스크 {{ store.scenarios.flatMap((item) => item.tasks).length }}개</span>
    </div>
    <p v-if="store.loading" class="panel" role="status">
      데이터를 불러오는 중입니다…
    </p>
    <section v-else-if="store.error" class="panel" role="alert">
      <h2><strong>데이터를 표시할 수 없습니다.</strong></h2>
      <p class="form-error">{{ store.error }}</p>
      <button class="table-action" type="button" @click="store.load">다시 시도</button>
    </section>
    <section v-else-if="scenarios.length" class="resource-grid grid gap-4 md:grid-cols-2 xl:grid-cols-3">
      <article v-for="scenario in scenarios" :key="scenario.id" class="resource-card">
        <div class="resource-top">
          <span class="file-tile">
            <AppIcon name="database" />
          </span>
          <span class="category-badge">
            {{ scenario.id }}
          </span>
        </div>
        <h3>{{ scenario.name }}</h3>
        <p>{{ scenario.tasks.length }}개 태스크</p>
        <div class="task-list">
          <template v-for="task in scenario.tasks" :key="task.id">
            <div v-if="task.download?.asset && releasePageUrl" class="download-task-row">
              <a
                :href="releaseDownloadUrl(task.download.asset)"
                class="download-task-link"
                :aria-busy="checkingAsset === task.download.asset"
                @click.prevent="downloadAsset(scenario.name, task.name, task.download.asset)"
              >
                <span>
                  <b>{{ task.name }}</b>
                  <small>{{ checkingAsset === task.download.asset ? '파일 확인 중…' : 'ZIP 다운로드' }}</small>
                </span>
                <AppIcon name="chevron" />
              </a>
            </div>
            <RouterLink
              v-else
              :to="task.status === 'active' ? `/leaderboard/${scenario.id}/${task.id}` : ''"
              :class="{ disabled: task.status !== 'active' }"
            >
              <span>
                <b>{{ task.name }}</b>
                <small>{{ task.status === 'active' ? task.id : '준비 중' }}</small>
              </span>
              <AppIcon v-if="task.status === 'active'" name="chevron" />
            </RouterLink>
          </template>
        </div>
      </article>
    </section>
    <section v-else class="panel">
      <h2><strong>{{ query ? '검색 결과가 없습니다.' : '등록된 과제가 없습니다.' }}</strong></h2>
      <p v-if="!query">
        <code>public/data/tasks.json</code>에 시나리오와 태스크를 추가해 주세요.
      </p>
    </section>
    <Teleport to="body">
      <div
        v-if="unavailableDownload"
        class="modal-backdrop"
        role="presentation"
        @click.self="unavailableDownload = null"
      >
        <section class="download-modal" role="dialog" aria-modal="true" aria-labelledby="download-modal-title">
          <button class="modal-close" type="button" aria-label="닫기" @click="unavailableDownload = null">
            <span aria-hidden="true">×</span>
          </button>
          <div class="download-modal-icon" aria-hidden="true">
            <AppIcon name="file" />
          </div>
          <span class="download-modal-label">자료 다운로드</span>
          <h2 id="download-modal-title">
            <strong>{{ unavailableDownload.checkFailed
              ? '파일 상태를 확인할 수 없습니다.'
              : '아직 자료가 준비되지 않았습니다.' }}</strong>
          </h2>
          <p class="download-modal-context">
            {{ unavailableDownload.scenario }} · {{ unavailableDownload.task }}
          </p>
          <p class="download-modal-description">
            <template v-if="unavailableDownload.checkFailed">
              잠시 후 다시 시도하거나<br />GitHub 릴리즈 페이지에서 파일 상태를 확인해 주세요.
            </template>
            <template v-else>
              담당자가 ZIP 파일을 릴리즈에 업로드하면<br />바로 다운로드할 수 있습니다.
            </template>
          </p>
          <div class="download-file-chip">
            <AppIcon name="file" />
            <code>{{ unavailableDownload.asset }}</code>
          </div>
          <div class="download-modal-actions">
            <a
              :href="releasePageUrl"
              target="_blank"
              rel="noopener noreferrer"
              class="download-modal-button secondary"
            >
              릴리즈 확인
            </a>
            <button class="download-modal-button primary" type="button" @click="unavailableDownload = null">
              확인
            </button>
          </div>
        </section>
      </div>
    </Teleport>
  </div>
</template>
