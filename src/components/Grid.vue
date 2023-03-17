<script lang="ts" setup>
import _ from "lodash"
import { FontAwesomeIcon } from "@fortawesome/vue-fontawesome"
import { inject, onBeforeMount } from "vue"
import type { Ref } from "vue"
import type { IPlan, Node, Row } from "@/interfaces"
import { PlanKey } from "@/symbols"
import { EstimateDirection, NodeProp } from "../enums"
import LevelDivider from "@/components/LevelDivider.vue"
import { duration, factor } from "@/filters"
const plan = inject(PlanKey) as Ref<IPlan>

let plans: Row[][] = [[]]

onBeforeMount((): void => {
  flatten(plans[0], 0, plan.value.content.Plan, true, [])

  _.each(plan.value.ctes, (cte) => {
    const flat: Row[] = []
    flatten(flat, 0, cte, true, [])
    plans.push(flat)
  })
})

function flatten(
  output: Row[],
  level: number,
  node: Node,
  isLast: boolean,
  branches: number[]
) {
  // [level, node, isLastSibbling, branches]
  output.push([level, node, isLast, _.concat([], branches)])
  if (!isLast) {
    branches.push(level)
  }

  _.each(node.Plans, (subnode) => {
    flatten(
      output,
      level + 1,
      subnode,
      subnode === _.last(node.Plans),
      branches
    )
  })
  if (!isLast) {
    branches.pop()
  }
}

function isCTE(node: Node): boolean {
  return _.startsWith(node[NodeProp.SUBPLAN_NAME], "CTE")
}

function nodeType(row: Row): string {
  return row[1][NodeProp.NODE_TYPE]
}

/*
function rounded(value: number, total: number): number {
  const roundedTotal = parseFloat(total.toPrecision(1))
  return parseFloat(
    (parseFloat((value / roundedTotal).toFixed(2)) * roundedTotal).toPrecision(
      3
    )
  )
}
*/
</script>

<template>
  <div>
    <table class="m-1">
      <thead>
        <tr>
          <th class="text-center">id</th>
          <th class="text-center">time</th>
          <th class="text-center">rows</th>
          <th class="text-center">esti</th>
          <th class="text-center">loops</th>
          <th class="text-center">
            <font-awesome-icon
              fixed-width
              icon="filter"
              class="text-muted"
            ></font-awesome-icon>
          </th>
          <th style="width: 100%"></th>
        </tr>
      </thead>
      <tbody v-for="(flat, index) in plans" :key="index">
        <tr v-if="index === 0 && plans.length > 1">
          <th colspan="3" class="subplan">Main Query Plan</th>
        </tr>
        <template v-for="(row, index) in flat" :key="index">
          <tr v-if="row[1][NodeProp.SUBPLAN_NAME]">
            <td colspan="3"></td>
            <td
              class="subplan pr-2"
              :class="{ 'font-weight-bold': isCTE(row[1]) }"
              :colspan="isCTE(row[1]) ? 3 : 2"
            >
              <!-- FIXME (use a divider component -->
              <span class="tree-lines">
                <template v-for="i in _.range(row[0])">
                  <template v-if="_.indexOf(row[3], i) != -1">│</template
                  ><template v-else-if="i !== 0">&emsp;</template> </template
                ><template v-if="index !== 0">{{
                  row[2] ? "└" : "├"
                }}</template>
              </span>
              <a class="font-italic text-reset" href="">
                {{ row[1][NodeProp.SUBPLAN_NAME] }}
              </a>
            </td>
          </tr>
          <tr>
            <td class="node-index">
              <span class="font-weight-normal">#{{ row[1].nodeId }} </span>
            </td>
            <td class="text-right grid-progress-cell">
              {{ duration(row[1][NodeProp.EXCLUSIVE_DURATION]) }}
              <div class="grid-progress">
                <div
                  class="bg-secondary-light border-left border-secondary-light"
                  :class="{
                    'border-left': row[1][NodeProp.EXCLUSIVE_DURATION] > 0,
                  }"
                  :style="
                    'height: 100%; ' +
                    'width: ' +
                    (row[1][NodeProp.EXCLUSIVE_DURATION] /
                      (plan.planStats.executionTime ||
                        plan.content.Plan[NodeProp.ACTUAL_TOTAL_TIME])) *
                      100 +
                    '%;'
                  "
                ></div>
              </div>
            </td>
            <td class="text-right grid-progress-cell">
              {{ row[1][NodeProp.ACTUAL_ROWS_REVISED].toLocaleString() }}
              <div class="grid-progress">
                <div
                  class="bg-secondary-light border-left border-secondary-light"
                  :class="{
                    'border-left': row[1][NodeProp.EXCLUSIVE_DURATION] > 0,
                  }"
                  :style="
                    'height: 100%; ' +
                    'width: ' +
                    Math.round(
                      (row[1][NodeProp.ACTUAL_ROWS_REVISED] /
                        plan.planStats.maxRows) *
                        100
                    ) +
                    '%'
                  "
                ></div>
              </div>
            </td>
            <td class="text-right">
              <span v-if="row[1][NodeProp.PLANNER_ESTIMATE_FACTOR] != 1">
                <span v-html="row[1][NodeProp.PLANNER_ESTIMATE_FACTOR]"></span>
                <span
                  v-if="
                    row[1][NodeProp.PLANNER_ESTIMATE_DIRECTION] ===
                    EstimateDirection.under
                  "
                >
                  ↑
                </span>
                <span
                  v-if="
                    row[1][NodeProp.PLANNER_ESTIMATE_DIRECTION] ===
                    EstimateDirection.over
                  "
                >
                  ↓
                </span>
              </span>
              <span v-else>---</span>
            </td>
            <td class="text-right">
              <span v-if="row[1][NodeProp.ACTUAL_LOOPS] != 1">
                {{ row[1][NodeProp.ACTUAL_LOOPS] }}
              </span>
            </td>
            <td class="text-right">?</td>
            <td class="node-type">
              <level-divider :row="row" :index="index"></level-divider>
              <div class="d-inline-block">
                <b>
                  {{ nodeType(row) }}
                </b>

                <template v-if="row[1][NodeProp.RELATION_NAME]">
                  <span class="text-muted">on</span>
                  <span v-if="row[1][NodeProp.SCHEMA]"
                    >{{ row[1][NodeProp.SCHEMA] }}.</span
                  >{{ row[1][NodeProp.RELATION_NAME] }}
                  <span v-if="row[1][NodeProp.ALIAS]">
                    <span class="text-muted">as</span>
                    {{ row[1][NodeProp.ALIAS] }}
                  </span>
                </template>
              </div>
            </td>
          </tr>
        </template>
      </tbody>
    </table>
  </div>
</template>

<style lang="scss">
.grid-progress-cell {
  z-index: 0;
  position: relative;
}
.grid-progress {
  position: absolute;
  width: 100%;
  height: 100%;
  top: 0;
  left: 0;
  z-index: -1;
  padding: 1px;
}
</style>
