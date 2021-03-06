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
  <div
    class="g-droppable-listener"
    style="height:100%; width:100%; background-color: #333"
    v-shortkey.once="{bottom: ['ctrl', 'shift', 'h'], right: ['ctrl', 'shift', 'v']}"
    @dropped="droppedAt"
    @shortkey="addFromShortkey"
  >
    <g-splitpane
      v-if="splitpaneTree"
      :splitpaneTree="splitpaneTree"
      ref="splitpane"
    >
      <template v-slot="{item}">
        <g-terminal
          :uuid="item.uuid"
          :data="item.data"
          @terminated="onTermination(item)"
          @split="orientation => onSplit(item, orientation)"
        ></g-terminal>
      </template>
    </g-splitpane>
    <target-selection-dialog ref="targetDialog" :shootItem="shootItem"></target-selection-dialog>
  </div>
</template>

<script>

import { mapGetters } from 'vuex'
import get from 'lodash/get'
import map from 'lodash/map'
import pick from 'lodash/pick'
import cloneDeep from 'lodash/cloneDeep'
import difference from 'lodash/difference'
import GSplitpane from '@/components/GSplitpane'
import GTerminal from '@/components/GTerminal'
import TargetSelectionDialog from '@/dialogs/TargetSelectionDialog'
import { TargetEnum } from '@/utils'
import { shootItem } from '@/mixins/shootItem'
import { listTerminalSessions } from '@/utils/api'
import { GSymbolTree, Leaf, PositionEnum } from '@/lib/g-symbol-tree'

import 'splitpanes/dist/splitpanes.css'

function terminatedSessionIds (uuids, terminals) {
  const terminalSessionIds = map(terminals, 'metadata.identifier')
  return difference(uuids, terminalSessionIds)
}

export default {
  components: {
    GSplitpane,
    GTerminal,
    TargetSelectionDialog
  },
  data () {
    return {
      tree: new GSymbolTree(),
      splitpaneTree: undefined // splitpaneTree is a json object representation of the GSymbolTree`this.tree`
    }
  },
  mixins: [shootItem],
  computed: {
    ...mapGetters([
      'focusedElementId',
      'shootByNamespaceAndName',
      'hasControlPlaneTerminalAccess',
      'hasShootTerminalAccess'
    ]),
    shootItem () {
      return this.shootByNamespaceAndName(this.$route.params)
    },
    terminalCoordinates () {
      const coordinates = pick(this.$route.params, ['name', 'namespace', 'target'])
      return coordinates
    },
    storeKey () {
      const { name, namespace, target } = this.terminalCoordinates
      return `${target}--${namespace}--${name}`
    }
  },
  methods: {
    updateSplitpaneTree () {
      this.splitpaneTree = this.tree.toJSON(this.tree.root)

      this.store()
    },
    addFromShortkey ({ srcKey: position = PositionEnum.RIGHT } = {}) {
      return this.add({ position })
    },
    async add ({ position, targetId } = {}) {
      if (!targetId) {
        targetId = this.focusedElementId
      }
      if (!targetId) {
        const lastChild = this.tree.lastChild(this.tree.root, true)
        targetId = get(lastChild, 'uuid')
      }

      const data = cloneDeep(this.terminalCoordinates)
      if (!data.target) {
        let target
        if (this.hasControlPlaneTerminalAccess && this.hasShootTerminalAccess) {
          const defaultTaget = this.hasControlPlaneTerminalAccess ? TargetEnum.CONTROL_PLANE : TargetEnum.SHOOT
          target = await this.$refs.targetDialog.promptForTargetSelection({ target: defaultTaget })
        } else if (this.hasShootTerminalAccess) {
          target = TargetEnum.SHOOT
        }

        if (!target) {
          if (this.tree.isEmpty()) {
            this.leavePage()
          }
          return
        }
        data.target = target
      }

      const item = new Leaf({ data })
      if (targetId) {
        this.tree.appendChild(this.tree.root, item)
        const sourceId = item.uuid

        this.moveTo({ sourceId, targetId, position })
      } else {
        this.tree.appendChild(this.tree.root, item)
        this.updateSplitpaneTree()
      }
    },
    droppedAt ({ detail: { 'mouseOverId': position, 'sourceElementDropzoneId': sourceId, 'mouseOverDropzoneId': targetId } }) {
      this.moveTo({ sourceId, targetId, position })
    },
    moveTo ({ sourceId, targetId, position }) {
      this.tree.moveToWithId({ sourceId, targetId, position })

      this.updateSplitpaneTree()
    },
    removeWithId (id) {
      this.tree.removeWithId(id)

      this.updateSplitpaneTree()
    },
    store () {
      if (this.tree.isEmpty()) {
        this.$localStorage.removeItem(this.storeKey)
        return
      }
      this.$localStorage.setItem(this.storeKey, JSON.stringify({
        splitpaneTree: this.splitpaneTree
      }))
    },
    async load () {
      await this.restoreSessions()

      this.updateSplitpaneTree()
    },
    async restoreSessions () {
      const fromStore = this.$localStorage.getItem(this.storeKey)
      if (!fromStore) {
        this.add()
        return
      }

      let splitpaneTree
      try {
        const json = JSON.parse(fromStore)
        splitpaneTree = json.splitpaneTree
      } catch (err) {
        // could not restore session
      }
      if (!splitpaneTree) {
        this.add()
        return
      }

      const { namespace } = this.terminalCoordinates
      const { data: terminals } = await listTerminalSessions({ namespace })

      this.tree = GSymbolTree.fromJSON(splitpaneTree)

      const uuids = this.tree.ids()
      const terminatedIds = terminatedSessionIds(uuids, terminals)
      this.tree.removeWithIds(terminatedIds)

      if (this.tree.isEmpty()) { // nothing to restore
        this.add()
      }
    },
    leavePage () {
      const { name, namespace } = this.terminalCoordinates
      if (name) {
        return this.$router.push({ name: 'ShootItem', params: { namespace, name } })
      }
      return this.$router.push({ name: 'ShootList', params: { namespace } })
    },
    onTermination ({ uuid }) {
      this.removeWithId(uuid)
      if (this.tree.isEmpty()) {
        this.leavePage()
      }
    },
    onSplit ({ uuid: targetId }, orientation = 'horizontal') {
      switch (orientation) {
        case 'horizontal':
          this.add({ position: PositionEnum.RIGHT, targetId })
          break
        case 'vertical':
          this.add({ position: PositionEnum.BOTTOM, targetId })
          break
      } // ignore unknown orientations
    }
  },
  mounted () {
    this.load()
  }
}

</script>
