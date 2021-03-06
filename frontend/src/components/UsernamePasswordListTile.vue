<!--
Copyright (c) 2019 by SAP SE or an SAP affiliate company. All rights reserved. This file is licensed under the Apache Software License, v. 2 except as noted otherwise in the LICENSE file

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

     http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
-->

<template>
  <div v-show="(!!username || !!email) && !!password">
    <v-list-tile v-if="username">
      <v-list-tile-action>
        <v-icon class="cyan--text text--darken-2">perm_identity</v-icon>
      </v-list-tile-action>
      <v-list-tile-content>
        <v-list-tile-sub-title>User</v-list-tile-sub-title>
        <v-list-tile-title>{{username}}</v-list-tile-title>
      </v-list-tile-content>
      <v-list-tile-action>
        <copy-btn :clipboard-text="username"></copy-btn>
      </v-list-tile-action>
    </v-list-tile>
    <v-list-tile v-if="email">
      <v-list-tile-action>
        <v-icon v-if="!username" class="cyan--text text--darken-2">perm_identity</v-icon>
      </v-list-tile-action>
      <v-list-tile-content>
        <v-list-tile-sub-title>Email</v-list-tile-sub-title>
        <v-list-tile-title>{{email}}</v-list-tile-title>
      </v-list-tile-content>
      <v-list-tile-action>
        <copy-btn :clipboard-text="email"></copy-btn>
      </v-list-tile-action>
    </v-list-tile>
    <v-list-tile>
      <v-list-tile-action>
      </v-list-tile-action>
      <v-list-tile-content>
        <v-list-tile-sub-title>Password</v-list-tile-sub-title>
        <v-list-tile-title>{{passwordText}}</v-list-tile-title>
      </v-list-tile-content>
      <v-list-tile-action>
        <copy-btn :clipboard-text="password"></copy-btn>
      </v-list-tile-action>
      <v-list-tile-action>
        <v-tooltip top>
          <v-btn slot="activator" icon @click.native.stop="showPassword = !showPassword">
            <v-icon>{{visibilityIcon}}</v-icon>
          </v-btn>
          <span>{{passwordVisibilityTitle}}</span>
        </v-tooltip>
      </v-list-tile-action>
    </v-list-tile>
  </div>
</template>

<script>
import CopyBtn from '@/components/CopyBtn'

export default {
  components: {
    CopyBtn
  },
  props: {
    username: {
      type: String
    },
    email: {
      type: String
    },
    password: {
      type: String
    }
  },
  data () {
    return {
      showPassword: false
    }
  },
  methods: {
    reset () {
      this.showPassword = false
    }
  },
  computed: {
    passwordText () {
      if (this.showPassword) {
        return this.password
      } else {
        return '****************'
      }
    },
    passwordVisibilityTitle () {
      if (this.showPassword) {
        return 'Hide password'
      } else {
        return 'Show password'
      }
    },
    visibilityIcon () {
      if (this.showPassword) {
        return 'visibility_off'
      } else {
        return 'visibility'
      }
    }
  },
  watch: {
    password (value) {
      this.reset()
    }
  }
}
</script>
