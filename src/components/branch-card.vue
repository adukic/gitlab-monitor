<template>
    <div v-if="showPipelinesOnly ? (pipelines !== null && pipelines.length > 0 && branchPipelines !== null && branchPipelines.length > 0) : true" :class="['branch-card', status]">
      <div class="content">
        <div class="title small">{{ project !== null ? project.namespace.name : '...' }} /</div>
        <div class="title">{{ project !== null ? project.name : 'Loading project...' }}</div>
        <div class="branch">
          <octicon name="git-branch" scale="0.9" />
            {{ branch.name }}
        </div>
        <div class="pipeline-container">
          <em v-if="pipelines !== null && pipelines.length === 0" class="no-pipelines">
            No recent pipelines
          </em>
          <div v-else-if="pipelines !== null && pipelines.length > 0">
            <div v-for="pipeline in branchPipelines" :key="pipeline.id">
              <pipeline-view :pipeline="pipeline" :branch="branch" :project="project" />
            </div>
          </div>
          <octicon v-else name="sync" scale="1.4" spin />
        </div>
      </div>
      <div class="spacer"></div>
      <div class="info">
        <div class="spacer"></div>
        <gitlab-icon class="calendar-icon" name="calendar" size="12" />
        <timeago v-if="project !== null" :since="project.last_activity_at" :auto-update="1"></timeago>
        <time v-else>...</time>
      </div>
    </div>
</template>

<script>
  import Octicon             from 'vue-octicon/components/Octicon';
  import {getQueryParameter} from '../util';
  import GitlabIcon          from './gitlab-icon';
  import PipelineView        from './pipeline-view';
  import 'vue-octicon/icons/git-branch';
  import 'vue-octicon/icons/clock';
  import 'vue-octicon/icons/sync';

  export default {
    components: {
      GitlabIcon,
      PipelineView,
      Octicon
    },
    name: 'branch-card',
    props: ['branch', 'pipelines', 'project'],
    data: () => ({
      branchPipelines: [],
      status: '',
      loading: false,
      refreshInterval: null
    }),
    computed: {
      showPipelinesOnly() {
        return getQueryParameter('pipelinesOnly') !== null ? getQueryParameter('pipelinesOnly') : false;
      }
    },
    mounted() {
      this.fetchBranchPipelines();
    },
    beforeDestroy() {
      if (this.refreshIntervalId) clearInterval(this.refreshIntervalId);
    },
    watch: {
      project() {
        this.fetchPipelines();
      },
      pipelines() {
        this.fetchBranchPipelines();
      },
      branchPipelines(branchPipelines) {
        if (this.$data.branchPipelines && this.$data.branchPipelines.length > 0) {
          this.$data.status = this.$data.branchPipelines[0].status;
          
          switch (this.$data.branchPipelines[0].status) {
            case 'pending':
            case 'running':
              this.$data.refreshInterval = 10000;
              break;
            default:
              this.$data.refreshInterval = 30000;
          }
        } else {
          this.$data.status = '';
          this.$data.refreshInterval = 120000;
        }
      },
      refreshInterval(newInterval, oldInterval) {
        if (newInterval !== oldInterval) {
          if (this.refreshIntervalId) clearInterval(this.refreshIntervalId);
          this.refreshIntervalId = setInterval(() => {
            if (!this.$data.loading) {
              this.fetchProject();
              this.fetchPipelines();
              this.fetchBranchPipelines();
            }
          }, newInterval);
        }
      }
    },
    methods: {
      async fetchProject() {
        this.$data.loading = true;

        this.$data.project = await this.$api(`/projects/${this.$props.project.id}`);
        this.$data.branches = await this.$api(`/projects/${this.$props.project.id}/repository/branches`);
        this.$emit('input', this.$props.project.last_activity_at);

        this.$data.loading = false;
      },
      async fetchPipelines() {
        this.$data.loading = true;

        const pipelines = await this.$api(`/projects/${this.$props.project.id}/pipelines`);

        if (pipelines.length === 0) {
          this.$data.pipelines = [];
        } else {
          this.$data.pipelines = pipelines;
        }

        this.$data.loading = false;
      },
      async fetchBranchPipelines() {
        this.$data.loading = true;

        const pipelineIds = [];
        if(this.$props.pipelines != null && this.$props.pipelines.length > 0) {
          for (const pipeline of this.$props.pipelines) {
            if (pipeline.ref === this.$props.branch.name) {
              pipelineIds.push(pipeline.id);
            }
          }
        }
        
        let pipelineId = null;
        if(pipelineIds.length > 0) {
          pipelineId = pipelineIds[0];
        }

        const resolvedPipelines = [];
        if(pipelineId != null) {
          const resolvedPipeline = await this.$api(`/projects/${this.$props.project.id}/pipelines/${pipelineId}`);
          resolvedPipelines.push(resolvedPipeline);
        }
        this.$data.branchPipelines = resolvedPipelines;
        
        this.$data.loading = false;
      }
    }
  };
</script>

<style lang="scss" scoped>
  .branch-card {
    margin: 4px;
    border-radius: 3px;
    background-color: #424242;
    flex-grow: 1;
    display: flex;
    flex-direction: column;
    transition: background-color 200ms;

    &.success {
      background-color: #2E7D32;
    }

    &.running {
      background-color: #1565C0;
    }

    &.pending {
      background-color: #A93F00;
    }

    &.failed {
      background-color: #C62828;
    }

    &.canceled {
      background-color: #010101;
    }

    &.skipped {
      background-color: #4b4b4b;
    }

    .content {
      padding: 12px;

      .title {
        white-space: nowrap;
        font-size: 16px;
        font-weight: bold;
        text-shadow: 1.5px 1.5px rgba(0, 0, 0, 0.4);

        &.small {
          font-size: 12px;
          line-height: 0.6;
        }
      }

      .branch {
        color: rgba(255, 255, 255, 0.5);
        display: flex;
        align-items: center;
        font-size: 14px;

        .octicon {
          margin-right: 4px;
        }
      }

      .pipeline-container {
        padding: 8px 0 0 0;
      }

      .no-pipelines {
        color: rgba(255, 255, 255, 0.5);
        font-size: 10px;
      }
    }

    .spacer {
      flex-grow: 1;
    }

    .info {
      padding: 12px;
      border-top: 1px solid rgba(255, 255, 255, 0.1);
      display: flex;
      align-items: center;
      font-size: 12px;
      color: rgba(255, 255, 255, 0.3);

      time {
        line-height: 1;
      }

      .calendar-icon {
        margin-right: 4px;
      }
    }
  }
</style>
