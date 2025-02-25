<template>
  <div class="q-pa-md row items-start q-gutter-md justify-center" v-if="build">
    <span>
    </span>
    <!--<deploy-tool-link-window ref="deployToolWindow" :build_id="build_id"/>-->

    <q-card flat>
      <q-card-section class="text-h6 text-center">
        <router-link :to="{path: `/build/${build.id}`}">
        Build {{ build.id }}
        </router-link>
          created by
          <a :href="`mailto:${build.user.email}`">{{ build.user.username }}</a>
          at {{ buildCreatedTime }}
      </q-card-section>

      <q-card-section>

        <q-tabs v-model="tab">
         <q-tab name="summary" label="Summary"/>
          <q-tab
              v-for="target of Object.keys(buildTasks)"
              :name="target"
              :label="target"
              :key="target"
          />
        </q-tabs>

        <q-tab-panels v-model="tab">
          <q-tab-panel name="summary">
             <table class="text-left q-table horizontal-separator build-info-table">
              <thead>
                <tr>
                  <th><td/></th>
                  <th v-for="targetName of Object.keys(buildTasks)" :key="targetName" class="platform-name">
                    {{ targetName }}
                  </th>
                </tr>
              </thead>
              <tbody>
                <tr v-for="tasks in buildTasksByIndex" :key="tasks[0].index">
                  <td>
                    <buildRef :buildRef="tasks[0].ref"/>
                  </td>
                  <template v-for="targetName of Object.keys(buildTasks)" :key="targetName">
                    <td v-for="task in buildTasks[targetName][tasks[0].index]" :key=task.id>
                      <BuildStatusCircle :status="task.status" @click="openTaskLogs(task)"/>
                    </td>
                  </template>
                </tr>
              </tbody>
            </table>
          </q-tab-panel>

          <q-tab-panel
              v-for="target of Object.keys(buildTasks)"
              :name="target"
              :label="target"
              :key="target"
          >
             <table class="text-left q-table horizontal-separator build-info-table">
              <thead>
                <tr>
                  <th><td/></th>
                  <th>Status</th>
                  <th>Packages</th>
                </tr>
              </thead>
              <tbody>
                <tr v-for="tasks in buildTasksByIndex" :key="tasks[0].index">
                  <td>
                    <buildRef :buildRef="tasks[0].ref"/>
                  </td>
                  <template v-for="task in buildTasks[target][tasks[0].index]" :key=task.id>
                    <td :class="getTaskCSS(task)"
                        @click="openTaskLogs(task)"
                    >
                      {{ getTextStatus(task) }}
                    </td>
                    <td>
                      <div
                        v-for="pkg in getTaskPackages(task)"
                        :key="pkg.name"
                      >
                        <a class="text-tertiary" :href="pkg.downloadUrl">
                          {{ pkg.name }}
                        </a>
                      </div>
                    </td>
                  </template>
                </tr>
              </tbody>
            </table>
          </q-tab-panel>
        </q-tab-panels>

      </q-card-section>

      <q-card-section>
        <q-expansion-item label="Linked builds" expand-separator
                          icon="link" v-if="linked_builds">
          <q-card>
            <q-card-section v-for="linked_build in linked_builds" :key="linked_build">
              <q-item :to="`/build/${linked_build}`">
                <q-item-section>
                  {{ linked_build }}
                </q-item-section>
              </q-item>
            </q-card-section>
          </q-card>
        </q-expansion-item>
      </q-card-section>
      <q-card-actions align="right">
        <q-btn-dropdown label="Other Actions" color="primary" dropdown-icon="change_history"
                        style="width: 200px; height: 40px;">
          <q-list>
            <q-item clickable v-close-popup @click="add_to_distro = true" v-if="allowDistroModify">
              <q-item-section avatar>
                <q-avatar icon="playlist_add_check"/>
              </q-item-section>
              <q-item-section>
                <q-item-label>Add to a distribution</q-item-label>
              </q-item-section>
            </q-item>
            <q-item clickable v-close-popup @click="remove_from_distro = true" v-if="allowDistroModify">
              <q-item-section avatar>
                <q-avatar icon="delete_sweep"/>
              </q-item-section>
              <q-item-section>
                <q-item-label>Remove from a distribution</q-item-label>
              </q-item-section>
            </q-item>
          </q-list>
        </q-btn-dropdown>
      </q-card-actions>

    </q-card>
    <q-dialog v-model="add_to_distro">
      <q-card style="width: 400px;">
        <q-card-section>
          <div class="text-h6">Add to a distribution</div>
        </q-card-section>
        <q-card-section>
          <q-select v-model="current_distro" label="Choose distribution to add"
                    :options="existingDistros"/>
        </q-card-section>
        <q-card-actions align="right">
          <q-btn flat text-color="primary" label="Add" style="width: 150px"
                 :loading="loading"
                 @click="AddToDistribution">
          </q-btn>
          <q-btn :loading="loading" flat text-color="negative" label="Cancel"
                 v-close-popup @click="current_distro = []"/>
        </q-card-actions>
      </q-card>
    </q-dialog>
    <q-dialog v-model="remove_from_distro">
      <q-card style="width: 400px;">
        <q-card-section>
          <div class="text-h6">Remove from a distribution</div>
        </q-card-section>
        <q-card-section>
          <q-select v-model="current_distro" label="Choose distribution to add"
                    :options="existingDistros"/>
        </q-card-section>
        <q-card-actions align="right">
          <q-btn flat text-color="primary" label="Remove" style="width: 150px"
                 :loading="loading"
                 @click="RemoveFromDistribution"/>
          <q-btn flat text-color="negative" label="Cancel" :loading="loading"
                 v-close-popup @click="current_distro = []"/>
        </q-card-actions>
      </q-card>
    </q-dialog>
  </div>
</template>

<script>

import { defineComponent, ref } from 'vue'
import {exportFile, Loading, Notify} from 'quasar'
import BuildRef from 'components/BuildRef.vue'
import BuildStatusCircle from 'components/BuildStatusCircle.vue'
import { BuildStatus } from '../constants.js'

export default defineComponent({
  props: {
    buildId: String
  },
  data () {
    return {
      tab: 'summary',
      build: null,
      reload: true,
      refreshTimer: null,
      linked_builds: null,
      add_to_distro: false,
      remove_from_distro: false,
      loading: false,
      current_distro: []
    }
  },
  watch: {
    $route (to, from) {
      this.loadBuildInfo(to.params.buildId)
    }
  },
  computed: {
    allowDistroModify () {
      let allow_modify = true
      for (let i=0; i < this.build.tasks.length; i++) {
        if (this.build.tasks[i].status < 2) {
          allow_modify = false
        }
      }
      return allow_modify
    },
    existingDistros () {
      return this.$store.state.distributions.distributions.map(distribution => {
        return {label: distribution.name, value: distribution.name}
      })
    },
    buildTargets () {
      let targetsSet = new Set()
      let targets = []
      for (const task of this.build.tasks) {
        const targetKey = `${task.platform.name}.${task.arch}`
        if (targetsSet.has(targetKey)) {
          continue
        }
        targetsSet.add(targetKey)
        targets.push({name: task.platform.name, arch: task.arch, key: targetKey})
      }
      return targets
    },
    buildTasks () {
      let response = {}
      for (let task of this.build.tasks) {
        const x = `${task.platform.name}.${task.arch}`
        if (!response.hasOwnProperty(x)) {
          response[x] = {}
        }
        const y = task.index
        if (!response[x].hasOwnProperty(y)) {
          response[x][y] = []
        }
        response[x][y].push(task)
      }
      for (let sortX in response) {
        for (let sortY in response[sortX]) {
          response[sortX][sortY].sort((a, b) => (a.id > b.id) ? 1: -1)
        }
      }
      // TODO: sort here should use platform arch build order, instead
      //       of alphabetical.
      const ordered = Object.keys(response).sort().reduce(
        (obj, key) => {
          obj[key] = response[key];
          return obj;
        },
        {}
      )
      return ordered
    },
    buildTasksByIndex () {
      return this.buildTasks[Object.keys(this.buildTasks)[0]]
    },
    buildCreatedTime () {
      return new Date(this.build.created_at).toLocaleString()
    }
  },
  created () {
    this.loadBuildInfo(this.buildId)
    // update the build state every minute
    this.refreshTimer = setInterval(() => {
      if (this.reload) {
        this.loadBuildInfo(this.buildId)
      }
    }, 60000)
    // don't forget clear the timer while leaving the page
  },
  beforeUnmount () {
    if (this.refreshTimer) {
      clearInterval(this.refreshTimer)
      this.refreshTimer = null
    }
  },
  methods: {
    AddToDistribution () {
      this.loading = true
      this.$api.post(`/distro/add/${this.buildId}/${this.current_distro.label}/`)
        .then(() => {
          this.loading = false
          Notify.create({
            message: `Packages of build ${this.buildId} has been added to ${this.current_distro.label} distribution`,
            type: 'positive',
            actions: [
              { label: 'Dismiss', color: 'white', handler: () => {} }
            ]
          })
          this.current_distro = []
        })
        .catch(error => {
          this.loading = false
          Notify.create({
            message: error.response.data.detail,
            type: 'negative',
            actions: [
              { label: 'Dismiss', color: 'white', handler: () => {} }
            ]
          })
        })
    },
    RemoveFromDistribution () {
      this.loading = true
      this.$api.post(`/distro/remove/${this.buildId}/${this.current_distro.label}/`)
        .then(() => {
          this.loading = false
          Notify.create({
            message: `Packages of build ${this.buildId} has been removed to ${this.current_distro.label} distribution`,
            type: 'positive',
            actions: [
              { label: 'Dismiss', color: 'white', handler: () => {} }
            ]
          })
          this.current_distro = []
        })
        .catch(error => {
          this.loading = false
          Notify.create({
            message: error.response.data.detail, type: 'negative',
            actions: [
                { label: 'Dismiss', color: 'white', handler: () => {} }
              ]})
        })
    },
    loadBuildInfo (buildId) {
      this.reload = false
      this.linked_builds = null
      Loading.show()
      this.$api.get(`/builds/${buildId}/`)
        .then(response => {
          Loading.hide()
          this.build = response.data
          if (this.build.linked_builds.length) {
            this.linked_builds = this.build.linked_builds
          }
          this.reload = true
        })
        .catch(error => {
          Loading.hide()
          this.reload = true
        })
    },
    getTextStatus (task) {
      return BuildStatus.text[task.status]
    },
    getTaskCSS (task) {
        let css = ['cursor-pointer']
        if (task.status === BuildStatus.FAILED) {
          css.push('text-negative', 'bg-red-1')
        }
        else if (task.status === BuildStatus.IDLE) {
          css.push('text-grey-6')
        }
        else if (task.status === BuildStatus.STARTED) {
          css.push('text-black-6')
        }
        else if (task.status === BuildStatus.COMPLETED) {
          css.push('text-green-7')
        }
        return css
    },
    downloadArtifact (artifact) {
      this.$api.get(`/artifacts/${artifact.id}/`)
        .then((response) => {
          exportFile(artifact.name, response.data.content)
        })
    },
    openTaskLogs (task) {
      this.$router.push(`/build/${this.buildId}/logs/${task.id}`)
    },
    getTaskPackages (task) {
      return task.artifacts.filter(item => item.type === 'rpm').map(item => {
        let arch = task.arch
        if (item.name.includes('.src.')) {
          arch = 'src'
        }
        let debugSuffix = '-debug'
        if (!item.name.match(/-debug(info|source)/)) {
          debugSuffix = ''
        }
        item.downloadUrl = `${window.origin}/pulp/content/builds/${task.platform.name}-${arch}-${this.buildId}${debugSuffix}-br/Packages/${item.name[0]}/${item.name}`
        return item
      })
    }
  },
  components: {
    BuildRef,
    BuildStatusCircle
  }
})
</script>

<style scoped>
  .q-tab {
    text-transform: none !important;
  }

  table.build-info-table tr:last-child td {
    border: none !important;
  }

  .mock-options dd {
    font-family: monospace;
    font-size: medium;
    padding: 0.5em 0 1em 1em;
    width: 30vw;
  }

  .mock-options dd:last-child {
    padding-bottom: 0;
  }

  th.platform-name {
    text-overflow: ellipsis;
    overflow: hidden !important;
    width: 60px;
  }
</style>
