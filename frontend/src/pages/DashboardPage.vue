<script setup lang="ts">
import { computed } from 'vue';
import { useRouter } from 'vue-router';
import {
  AlertCircleOutline,
  CalendarOutline,
  DocumentTextOutline,
  TimeOutline,
  ArrowForwardOutline
} from '@vicons/ionicons5';
import { useCompanyStore } from '@/stores/company';
import { useInterviewStore } from '@/stores/interview';
import { usePositionStore } from '@/stores/position';
import { useReflectionStore } from '@/stores/reflection';
import { InterviewResult, interviewStageLabels, interviewModeLabels } from '@/types/enums';
import { formatDateTime, formatDate } from '@/utils/format';
import dayjs from 'dayjs';

const router = useRouter();
const companyStore = useCompanyStore();
const positionStore = usePositionStore();
const interviewStore = useInterviewStore();
const reflectionStore = useReflectionStore();

const now = dayjs();
const threeDaysLater = now.add(3, 'day');
const oneWeekAgo = now.subtract(7, 'day');

type UrgencyLevel = 'critical' | 'high' | 'medium';

interface DashboardItem {
  id: string;
  type: 'upcoming-interview' | 'missing-reflection' | 'stuck-position';
  urgency: UrgencyLevel;
  urgencyScore: number;
  title: string;
  subtitle: string;
  description: string;
  actionText: string;
  positionId?: string;
  interviewId?: string;
  dateLabel: string;
}

const upcomingInterviews = computed(() => {
  return interviewStore.interviews
    .filter((interview) => {
      if (interview.result !== InterviewResult.Pending) return false;
      const scheduled = dayjs(interview.scheduledAt);
      return scheduled.isAfter(now) && scheduled.isBefore(threeDaysLater);
    })
    .sort((a, b) => new Date(a.scheduledAt).getTime() - new Date(b.scheduledAt).getTime());
});

const interviewsWithoutReflection = computed(() => {
  const completedInterviews = interviewStore.interviews.filter(
    (interview) => interview.result === InterviewResult.Passed || interview.result === InterviewResult.Rejected
  );
  return completedInterviews
    .filter((interview) => !reflectionStore.byInterview(interview.id))
    .sort((a, b) => new Date(a.scheduledAt).getTime() - new Date(b.scheduledAt).getTime());
});

const stuckPositions = computed(() => {
  return positionStore.positions
    .filter((position) => {
      const updated = dayjs(position.updatedAt);
      return updated.isBefore(oneWeekAgo);
    })
    .sort((a, b) => new Date(a.updatedAt).getTime() - new Date(b.updatedAt).getTime());
});

function getCompanyName(positionId: string): string {
  const position = positionStore.findPosition(positionId);
  if (!position) return '未知公司';
  const company = companyStore.findCompany(position.companyId);
  return company?.name ?? '未知公司';
}

function getPositionTitle(positionId: string): string {
  const position = positionStore.findPosition(positionId);
  return position?.title ?? '未知职位';
}

function calculateUrgencyScore(type: string, dateStr: string): number {
  const date = dayjs(dateStr);
  const daysDiff = date.diff(now, 'day');

  if (type === 'upcoming-interview') {
    return 1000 - daysDiff * 100;
  } else if (type === 'missing-reflection') {
    return 500 + Math.abs(daysDiff) * 10;
  } else {
    return 100 + Math.abs(daysDiff);
  }
}

function getUrgencyLevel(score: number): UrgencyLevel {
  if (score >= 800) return 'critical';
  if (score >= 400) return 'high';
  return 'medium';
}

const dashboardItems = computed<DashboardItem[]>(() => {
  const items: DashboardItem[] = [];

  upcomingInterviews.value.forEach((interview) => {
    const score = calculateUrgencyScore('upcoming-interview', interview.scheduledAt);
    const hoursLeft = dayjs(interview.scheduledAt).diff(now, 'hour');
    items.push({
      id: `interview-${interview.id}`,
      type: 'upcoming-interview',
      urgency: getUrgencyLevel(score),
      urgencyScore: score,
      title: `${getPositionTitle(interview.positionId)} · ${interviewStageLabels[interview.round]}`,
      subtitle: `${getCompanyName(interview.positionId)}`,
      description: `${interviewModeLabels[interview.mode]} · ${interview.interviewer}`,
      actionText: '查看详情',
      positionId: interview.positionId,
      interviewId: interview.id,
      dateLabel: hoursLeft < 24 ? `${hoursLeft} 小时后` : `${formatDateTime(interview.scheduledAt)}`
    });
  });

  interviewsWithoutReflection.value.forEach((interview) => {
    const score = calculateUrgencyScore('missing-reflection', interview.scheduledAt);
    const daysAgo = Math.abs(dayjs(interview.scheduledAt).diff(now, 'day'));
    items.push({
      id: `reflection-${interview.id}`,
      type: 'missing-reflection',
      urgency: getUrgencyLevel(score),
      urgencyScore: score,
      title: `${getPositionTitle(interview.positionId)} · ${interviewStageLabels[interview.round]}`,
      subtitle: `${getCompanyName(interview.positionId)} · ${interviewResultLabel(interview.result)}`,
      description: interview.feedback || '还没有记录反馈',
      actionText: '写复盘',
      positionId: interview.positionId,
      interviewId: interview.id,
      dateLabel: `${daysAgo} 天前结束`
    });
  });

  stuckPositions.value.forEach((position) => {
    const score = calculateUrgencyScore('stuck-position', position.updatedAt);
    const weeksStuck = Math.floor(Math.abs(dayjs(position.updatedAt).diff(now, 'day')) / 7);
    items.push({
      id: `position-${position.id}`,
      type: 'stuck-position',
      urgency: getUrgencyLevel(score),
      urgencyScore: score,
      title: `${position.title}`,
      subtitle: `${getCompanyName(position.id)} · ${interviewStageLabels[position.stage]}`,
      description: `最后更新: ${formatDate(position.updatedAt)}`,
      actionText: '推进状态',
      positionId: position.id,
      dateLabel: `已停滞 ${weeksStuck} 周`
    });
  });

  return items.sort((a, b) => b.urgencyScore - a.urgencyScore);
});

function interviewResultLabel(result: InterviewResult): string {
  return result === InterviewResult.Passed ? '已通过' : '已结束';
}

const urgencyConfig: Record<UrgencyLevel, { label: string; class: string; icon: typeof AlertCircleOutline }> = {
  critical: { label: '紧急', class: 'urgency-critical', icon: AlertCircleOutline },
  high: { label: '高优', class: 'urgency-high', icon: TimeOutline },
  medium: { label: '待办', class: 'urgency-medium', icon: DocumentTextOutline }
};

const typeConfig = {
  'upcoming-interview': { label: '即将面试', icon: CalendarOutline },
  'missing-reflection': { label: '待复盘', icon: DocumentTextOutline },
  'stuck-position': { label: '长期停滞', icon: TimeOutline }
};

function handleAction(item: DashboardItem) {
  if (item.positionId) {
    router.push(`/positions/${item.positionId}`);
  }
}

const stats = computed(() => ({
  upcoming: upcomingInterviews.value.length,
  noReflection: interviewsWithoutReflection.value.length,
  stuck: stuckPositions.value.length
}));
</script>

<template>
  <section class="page dashboard-page">
    <header class="page-header">
      <div>
        <p class="page-kicker">DASHBOARD</p>
        <h1 class="page-title">统一工作台</h1>
        <p class="page-subtitle">按紧急度聚合待办事项，集中处理最重要的事。</p>
      </div>
    </header>

    <div class="metric-strip">
      <div class="metric">
        <strong>{{ stats.upcoming }}</strong>
        <span>3 天内面试</span>
      </div>
      <div class="metric">
        <strong>{{ stats.noReflection }}</strong>
        <span>待复盘面试</span>
      </div>
      <div class="metric">
        <strong>{{ stats.stuck }}</strong>
        <span>停滞一周以上</span>
      </div>
    </div>

    <div class="dashboard-list">
      <div
        v-for="item in dashboardItems"
        :key="item.id"
        class="dashboard-card surface"
        :class="urgencyConfig[item.urgency].class"
        @click="handleAction(item)"
      >
        <div class="card-left">
          <div class="urgency-indicator">
            <n-icon :component="urgencyConfig[item.urgency].icon" :size="18" />
          </div>
          <div class="type-indicator">
            <n-icon :component="typeConfig[item.type].icon" :size="14" />
            <span>{{ typeConfig[item.type].label }}</span>
          </div>
        </div>

        <div class="card-main">
          <div class="card-header">
            <h3>{{ item.title }}</h3>
            <span class="urgency-badge" :class="urgencyConfig[item.urgency].class">
              {{ urgencyConfig[item.urgency].label }}
            </span>
          </div>
          <p class="card-subtitle">{{ item.subtitle }}</p>
          <p class="card-description">{{ item.description }}</p>
        </div>

        <div class="card-right">
          <div class="date-label">{{ item.dateLabel }}</div>
          <div class="action-link">
            <span>{{ item.actionText }}</span>
            <n-icon :component="ArrowForwardOutline" :size="14" />
          </div>
        </div>
      </div>

      <div v-if="dashboardItems.length === 0" class="empty-state">
        <n-icon :component="CalendarOutline" :size="48" />
        <p>暂时没有待办事项</p>
        <small>继续保持，所有事项都已处理完毕。</small>
      </div>
    </div>
  </section>
</template>

<style scoped>
.dashboard-list {
  display: grid;
  gap: 12px;
  margin-top: 24px;
}

.dashboard-card {
  display: grid;
  grid-template-columns: auto 1fr auto;
  gap: 16px;
  padding: 16px;
  border-radius: var(--radius-lg);
  cursor: pointer;
  transition: transform 0.15s ease, box-shadow 0.15s ease;
  border-left: 3px solid transparent;
}

.dashboard-card:hover {
  transform: translateX(2px);
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.08);
}

.dashboard-card.urgency-critical {
  border-left-color: var(--danger);
}

.dashboard-card.urgency-high {
  border-left-color: var(--warning);
}

.dashboard-card.urgency-medium {
  border-left-color: var(--info);
}

.card-left {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 8px;
  padding-top: 4px;
}

.urgency-indicator {
  display: grid;
  place-items: center;
  width: 36px;
  height: 36px;
  border-radius: var(--radius-md);
  background: var(--panel-muted);
}

.urgency-critical .urgency-indicator {
  background: color-mix(in srgb, var(--danger) 15%, transparent);
  color: var(--danger);
}

.urgency-high .urgency-indicator {
  background: color-mix(in srgb, var(--warning) 15%, transparent);
  color: var(--warning);
}

.urgency-medium .urgency-indicator {
  background: color-mix(in srgb, var(--info) 15%, transparent);
  color: var(--info);
}

.type-indicator {
  display: flex;
  align-items: center;
  gap: 4px;
  color: var(--text-muted);
  font-size: 11px;
}

.card-main {
  min-width: 0;
}

.card-header {
  display: flex;
  align-items: center;
  gap: 10px;
  margin-bottom: 4px;
}

.card-header h3 {
  margin: 0;
  font-size: 16px;
  font-weight: 600;
  color: var(--text-strong);
}

.urgency-badge {
  font-size: 11px;
  padding: 2px 8px;
  border-radius: var(--radius-sm);
  font-weight: 500;
}

.urgency-badge.urgency-critical {
  background: color-mix(in srgb, var(--danger) 12%, transparent);
  color: var(--danger);
}

.urgency-badge.urgency-high {
  background: color-mix(in srgb, var(--warning) 12%, transparent);
  color: var(--warning);
}

.urgency-badge.urgency-medium {
  background: color-mix(in srgb, var(--info) 12%, transparent);
  color: var(--info);
}

.card-subtitle {
  margin: 0 0 6px 0;
  color: var(--text-strong);
  font-size: 13px;
}

.card-description {
  margin: 0;
  color: var(--text-muted);
  font-size: 13px;
  line-height: 1.5;
}

.card-right {
  display: flex;
  flex-direction: column;
  align-items: flex-end;
  justify-content: space-between;
  text-align: right;
}

.date-label {
  font-size: 13px;
  font-weight: 500;
  color: var(--text-strong);
}

.action-link {
  display: flex;
  align-items: center;
  gap: 4px;
  font-size: 12px;
  color: var(--brand);
}

.empty-state {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 60px 20px;
  color: var(--text-muted);
  text-align: center;
}

.empty-state p {
  margin: 12px 0 4px 0;
  font-size: 16px;
  color: var(--text-strong);
}

.empty-state small {
  font-size: 13px;
}

@media (max-width: 600px) {
  .dashboard-card {
    grid-template-columns: 1fr;
    gap: 12px;
  }

  .card-left {
    flex-direction: row;
  }

  .card-right {
    flex-direction: row;
    align-items: center;
    justify-content: space-between;
  }
}
</style>
