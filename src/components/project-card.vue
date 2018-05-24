<template>
  <div>
    <branch-card v-for="branch in branches" :key="branch.id" :branch="branch" :pipelines="pipelines" :project="project" />
  </div>
</template>

<script>
  import Octicon             from 'vue-octicon/components/Octicon';
  import {getQueryParameter} from '../util';
  import BranchCard         from './branch-card';
  
  export default {
    components: {
      Octicon,
      BranchCard
    },
    name: 'project-card',
    props: ['project-id'],
    data: () => ({
      project: null,
      branches: [],
      pipelines: null,
      loading: false
    }),
    mounted() {
      this.fetchProject();
    },
    watch: {
      project() {
        this.fetchPipelines();
      }
    },
    methods: {
      async fetchProject() {
        this.$data.loading = true;

        this.$data.project = await this.$api(`/projects/${this.$props.projectId}`);
        this.$data.branches = await this.$api(`/projects/${this.$props.projectId}/repository/branches`);
        this.$emit('input', this.$data.project.last_activity_at);

        this.$data.loading = false;
      },
      async fetchPipelines() {
        this.$data.loading = true;

        const pipelines = await this.$api(`/projects/${this.$props.projectId}/pipelines`);

        if (pipelines.length === 0) {
          this.$data.pipelines = [];
        } else {
          this.$data.pipelines = pipelines;
        }

        this.$data.loading = false;
      }
    }
  };
</script>

<style lang="scss" scoped>
.project-card {
    margin: 4px;
    border-radius: 3px;
    background-color: #424242;
    flex-grow: 1;
    display: flex;
    flex-direction: column;
    transition: background-color 200ms;
  }
</style>
