<script lang="ts" setup>
import _ from "lodash"
import { NodeProp } from "../enums"
import type { Row } from "../interfaces"

interface Props {
  row: Row
  index: number
}
defineProps<Props>()
</script>
<template>
  <span class="tree-lines">
    <template v-for="i in _.range(row[0])">
      <template v-if="_.indexOf(row[3], i) != -1">&emsp;│</template
      ><template v-else-if="i !== 0">&emsp;&emsp;</template> </template
    ><template v-if="index !== 0">
      <template v-if="!row[1][NodeProp.SUBPLAN_NAME]">{{
        row[2] ? "&emsp;└" : "&emsp;├"
      }}</template
      ><template v-else>
        <template v-if="!row[2]">&emsp;│</template
        ><template v-else>&emsp;</template>
      </template>
    </template>
  </span>
</template>
