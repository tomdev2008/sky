<!--
  - Copyright 2014-2018 the original author or authors.
  -
  - Licensed under the Apache License, Version 2.0 (the "License");
  - you may not use this file except in compliance with the License.
  - You may obtain a copy of the License at
  -
  -     http://www.apache.org/licenses/LICENSE-2.0
  -
  - Unless required by applicable law or agreed to in writing, software
  - distributed under the License is distributed on an "AS IS" BASIS,
  - WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  - See the License for the specific language governing permissions and
  - limitations under the License.
  -->

<template>
  <table class="sessions table is-fullwidth">
    <thead>
      <tr>
        <th>Session Id</th>
        <th>Created at</th>
        <th>Last accessed at</th>
        <th>Expiry</th>
        <th>Max. inactive<br>interval</th>
        <th>Attributes</th>
        <th>
          <sba-confirm-button class="button"
                              :class="{ 'is-loading' : deletingAll === 'executing', 'is-info' : deletingAll === 'completed', 'is-danger' : deletingAll === 'failed' }"
                              :disabled="deletingAll !== null"
                              v-if="sessions.length > 1" @click="deleteAllSessions()"
          >
            <span v-if="deletingAll === 'completed'">Deleted</span>
            <span v-else-if="deletingAll === 'failed'">Failed</span>
            <span v-else><font-awesome-icon icon="trash" />&nbsp;Delete</span>
          </sba-confirm-button>
        </th>
      </tr>
    </thead>
    <tr v-for="session in sessions" :key="session.id">
      <td>
        <router-link v-text="session.id"
                     :to="{ name: 'instances/sessions', params: { 'instanceId' : instance.id}, query: { sessionId : session.id } }"
        />
      </td>
      <td v-text="session.creationTime.format('L HH:mm:ss.SSS')" />
      <td v-text="session.lastAccessedTime.format('L HH:mm:ss.SSS')" />
      <td>
        <span v-if="session.expired" class="tag is-info">Expired</span>
      </td>
      <td>
        <span v-if="session.maxInactiveInterval >= 0" v-text="`${session.maxInactiveInterval}s`" />
        <span v-else>unlimited</span>
      </td>
      <td>
        <span class="tag" v-for="name in session.attributeNames" :key="`${session.id}-${name}`"
              v-text="name"
        />
      </td>
      <td>
        <button class="button"
                :class="{ 'is-loading' : deleting[session.id] === 'executing', 'is-info' : deleting[session.id] === 'completed', 'is-danger' : deleting[session.id] === 'failed' }"
                :disabled="session.id in deleting" @click="deleteSession(session.id)"
        >
          <span v-if="deleting[session.id] === 'completed'">Deleted</span>
          <span v-else-if="deleting[session.id] === 'failed'">Failed</span>
          <span v-else><font-awesome-icon icon="trash" />&nbsp;Delete</span>
        </button>
      </td>
    </tr>
    <tr v-if="sessions.length === 0">
      <td class="is-muted" colspan="7 ">
        <p v-if="isLoading" class="is-loading">Loading Sessions...</p>
        <p v-else>No Sessions found.</p>
      </td>
    </tr>
  </table>
</template>

<script>
  import Instance from '@/services/instance';
  import {concatMap, from, listen, map, of, tap} from '@/utils/rxjs';
  import prettyBytes from 'pretty-bytes';

  export default {
    props: {
      sessions: {
        type: Array,
        default: () => []
      },
      instance: {
        type: Instance,
        required: true
      },
      isLoading: {
        type: Boolean,
        default: false
      }
    },
    data: () => ({
      deletingAll: null,
      deleting: {},
    }),
    methods: {
      prettyBytes,
      deleteAllSessions() {
        const vm = this;
        vm.subscription = from(vm.sessions)
          .pipe(
            map(session => session.id),
            concatMap(vm._deleteSession),
            listen(status => vm.deletingAll = status)
          )
          .subscribe({
            complete: () => {
              vm.$emit('deleted', '*');
            }
          });
      },
      deleteSession(sessionId) {
        const vm = this;
        vm._deleteSession(sessionId)
          .pipe(listen(status => vm.$set(vm.deleting, sessionId, status)))
          .subscribe({
            complete: () => vm.$emit('deleted', sessionId),
          });
      },
      _deleteSession(sessionId) {
        const vm = this;
        return of(sessionId)
          .pipe(
            concatMap(async sessionId => {
              await vm.instance.deleteSession(sessionId);
              return sessionId;
            }),
            tap({
              error: error => {
                console.warn(`Deleting session ${sessionId} failed:`, error);
              }
            })
          );
      }
    }
  }
</script>
<style lang="scss">
  .sessions {
    td, th {
      vertical-align: middle;
    }
  }
</style>
